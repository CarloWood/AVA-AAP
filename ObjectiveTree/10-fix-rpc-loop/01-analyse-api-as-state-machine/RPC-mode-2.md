Here's the complete overview. I'll build it up in layers because the reason this API feels impossible to grasp is that **three independent things are tangled together**: (a) one "god" command struct with string-dispatch, (b) hand-rolled string responses with no structs, and (c) **three different response-delivery mechanisms**. Once you see the three mechanisms, the rest falls into place.

---

## The dispatch spine

```
run_rpc_loop (rpc_mode.cpp:105)            <-- single thread reads stdin lines
  └─ parse_rpc_command_line -> RpcCommand  <-- the god struct (protocol.h:23)
     └─ if (type == "get_protocol") ...                              #1 inline
     └─ handle_session_rpc_command(...)  (session_commands.cpp:161)  #1 inline (a big batch)
     └─ permission_grants / _grant_revoke / _grants_clear            #1 inline
     └─ list_commands                                                #1 inline
     └─ invoke_command ─┐                                            #2 worker (if it yields a prompt)
     └─ permission_reply / question_reply                            #1 inline (resolves a paused worker)
     └─ steer / follow_up                                            #1 inline (queue into active run)
     └─ cancel                                                       #1 inline
     └─ context/export/export_html/compact/plugin/mcp ── run_command #1 inline (#3 for compact)
     └─ prompt ─────────────────────────────────────────── worker    #2 worker
     └─ else -> error "unknown RPC command type"
```

The key structural facts:
- **There is no command table.** Dispatch is a ~600-line `if/else-if` chain on `command.type == "..."` spread across `rpc_mode.cpp` + `session_commands.cpp` + `handlers.cpp`.
- **Every command shares one input struct** (`RpcCommand`) — ~40 `std::optional` fields, where each command only uses a handful. There is no per-command parameter struct.
- **`type` is the only required discriminator**; `id` is the request id echoed in the response. Everything else is optional and validated ad hoc inside each branch.

---

## The three response mechanisms (this is the heart of your question)

### #1 Synchronous / inline (the common case)
The receiving thread itself computes the result and writes `{"id","type":"response","success":true,"result":{...}}\n` (or the error form) **before reading the next line**. No `Task`, no callback. This is ~35 of the ~41 commands.

### #2 Async worker thread ("the Task")
Used only by **`prompt`** and **`invoke_command`** (when the invoked slash-command resolves to a `prompt_message`). The receiving thread does:
1. `set_active_run(run_state, true, command->id)` — locks out other active-run-gated commands.
2. `prompt_worker.emplace(make_rpc_prompt_worker(...))` — spawns a `std::jthread`.
3. Immediately goes back to reading stdin (so it can process `permission_reply`, `question_reply`, `steer`, `follow_up`, `cancel` for the in-flight run).

The worker (`prompt_worker.cpp`) is where your "Task finishes → sends response" model lives:
- It calls `run_prompt(session, message, provider, transport, options)` (the agent loop, `runtime.h:178`), returning `Result<AgentLoopResult>`.
- **During** the run, the agent emits `RuntimeEvent`s into an `EventBus`; `subscribe_event_envelope_writer` turns each into an `EventEnvelope` JSONL line written to stdout (the event stream).
- **On completion**: `write_success(output, request_id, prompt_result_json(...))` → the final response.
- **On failure**: `write_error(output, request_id, result.error())`.
- **Write failures** (stdout broken) can't be returned to the loop synchronously, so they're stashed via `record_async_error(run_state, ...)` and the main loop picks them up with `take_async_error(run_state)` on the next iteration.

There are **no callbacks in the C++ sense**. The "response when finished" is just the worker thread calling `write_success`/`write_error` directly. The main loop never blocks on the worker; it just reaps it when idle (`reap_finished_prompt`).

### #3 Synchronous-but-blocking with active-run flag
Only **`compact`**. It runs on the receiving thread (so it blocks the next read), but it flips `active_run = true` first (rejecting concurrent commands), calls `run_command("/compact", ...)` with a `CompactionSummaryGenerator` lambda that itself may call a provider via `generate_compaction_summary`. Events still stream through the EventBus. Final response written inline. It clears active_run in its `write_command_success/error` wrapper.

---

## The two interaction protocols that happen *during* a worker (#2) run

These are what make the "Task" non-trivial. While a prompt worker is running, the main loop can receive commands that talk *to* it:

### Permission resolver (the pause/resume pattern)
When the agent needs permission, the resolver (`make_rpc_permission_resolver`, `resolvers.cpp:165`) — running on the **worker thread** — does:
1. Checks policy resolver + session grants. If none allow → writes a `permission_requested` **event** with a fresh `request_id`.
2. Blocks on `pending_state.cv.wait(...)` — the worker is parked.

The client then sends **`permission_reply`** (`request_id`, `correlation_id`, `decision`∈{allow,allow_session,deny}, `reason`). The main loop (#1 inline) calls `resolve_permission_reply`, which fills in `pending->resolution`, `notify_all()`s, and **unblocks the worker**. The worker returns the decision to the agent loop and continues. So `permission_reply` itself returns `{}` immediately; the *real* effect is waking the worker.

Questions (`question_requested` / `question_reply`) work identically (`make_rpc_question_resolver`, `resolvers.cpp:243`).

### Queueing (steer / follow_up)
`steer` and `follow_up` don't talk to a paused worker — they append to `run_state.steering_messages` / `follow_up_messages` and emit a `*_queued` event. The worker drains steering messages at its next safe point (`take_steering_messages` callback → `steer_applied` events) and runs follow-ups after the current prompt completes (`follow_up_started` events, then another `run_one_prompt`). `follow_up` is notable: it has **no immediate success response of its own** — it just gets a `follow_up_queued` event; its eventual result comes later as a separate success/error keyed by its own `request_id`.

---

## Every command, mapped

`active-run gated` = rejected with `active_run_reject_error` if a prompt/compact is in flight.
Params show only the fields each command actually reads from `RpcCommand`.

### A. Meta / protocol (inline #1)
| command | params | backend API | response |
|---|---|---|---|
| `get_protocol` | `protocol_version?` | `rpc_protocol_result_json()` | inline success |
| `list_commands` | — | `load_command_registry(...)` (active-run gated) | inline success |

### B. Session queries (inline #1, in `handle_session_rpc_command`)
| command | params | backend API | notes |
|---|---|---|---|
| `get_state` | — | `state_result_json(session, canceled)` | not active-run gated |
| `list_sessions` | — | `list_sessions_result_json` | |
| `session_tree` | — | `session_tree_result_json` | |
| `list_models` | — | `list_models_result_json` | |
| `get_messages` | — | `messages_result_json` | active-run gated |
| `get_session_stats` | — | `session_stats_result_json` (the one you dissected) | active-run gated |
| `validate_session` | — | `session_validation_result_json` | active-run gated |
| `session_metadata` | — | `load_session_metadata` | |

### C. Session mutation (inline #1)
| command | params | backend API | notes |
|---|---|---|---|
| `set_session_name` | `session_name` | `append_session_metadata` | active-run gated |
| `set_session_labels` | `labels[]` | `append_session_metadata` | active-run gated |
| `set_model` | `provider?`,`model?` | `resolve_requested_model`→`switch_runtime_model` | active-run gated |
| `cycle_model` | — | `next_runtime_model`→`switch_runtime_model` | active-run gated |
| `set_reasoning` | `reasoning_level`,`reasoning_budget_tokens?`,`reasoning_display?` | `set_runtime_reasoning` | active-run gated |
| `clear_reasoning` | — | `set_runtime_reasoning(nullopt)` | active-run gated |
| `new_session` | — | `create_new_session` → swaps `session` in place | active-run gated |
| `open_session`/`switch_session` | `session_id` | `open_requested_session` → swaps `session` | |
| `fork_session` | `session_id?`,`branch_from_entry_id?`,`session_name?`,`labels?` | `create_session_branch`→open | active-run gated |
| `clone_session` | `session_id?`,`session_name?`,`labels?` | `create_session_branch(Clone)`→open | active-run gated |
| `summarize_branch` | `branch_root_entry_id`,`branch_tip_entry_id`,`summary`,`provider`,`model`,`reason`,`session_id?` | `append_branch_summary` | active-run gated |

### D. Permission rule persistence (inline #1, active-run gated)
| command | params | backend API | extra |
|---|---|---|---|
| `permission_rules` | — | `load_persistent_permission_rules` | |
| `permission_rule_add` | `operation`,`mode?`,`scope?`,`tool_name?`,`path?`,`target_path?`,`command?`,`decision?` | `add_persistent_permission_rule` | also emits `permission_rule_added` **event** |
| `permission_rule_remove` | `rule_id` | `remove_persistent_permission_rule` | also emits `permission_rule_removed` event |

### E. In-flight permission grants (inline #1) — manage the resolver's session-grant list
| command | params | effect | extra |
|---|---|---|---|
| `permission_grants` | — | read grants list | inline success |
| `permission_grant_revoke` | `grant_id` | remove one grant | emits `permission_grant_revoked` event |
| `permission_grants_clear` | — | clear all | emits `permission_grants_cleared` event |

### F. Run control (inline #1, talk to a worker)
| command | params | mechanism | response |
|---|---|---|---|
| `prompt` | `message`,`attachments?`,`images?`,`provider?`,`model?`,`instructions?`,reasoning fields… | **#2 worker**; `run_prompt` | final success/error from worker, keyed by `id` |
| `invoke_command` | `name`,`command_arguments?` | `run_command` first; if it yields `prompt_message` → **#2 worker** | inline success if no prompt, else worker |
| `steer` | `message` | queue + `steer_queued` event | inline `{queued:true,correlation_id}` |
| `follow_up` | `message` | queue + `follow_up_queued` event | **no inline success**; result arrives later keyed by its `request_id` |
| `cancel` | — | set cancel flag, clear queues, cancel pending resolvers | emits `cancel_requested` event + inline `{cancel_requested,active_run,cleared_steer,cleared_follow_up}`; also emits follow-up errors |
| `permission_reply` | `request_id`,`correlation_id`,`decision`,`reason?` | `resolve_permission_reply` → wakes parked worker | inline `{}` |
| `question_reply` | `request_id`,`correlation_id`,`answer?`/`selected?`/`selected_options?` | `resolve_question_reply` → wakes parked worker | inline `{}` |

### G. Slash-command passthroughs (inline #1; `#3` for compact) — active-run gated
These translate the RPC command into a `/slash` string and run it via `run_command(session, CommandRequest{...})`. Events stream through the EventBus.
| command | params | slash translation |
|---|---|---|
| `context` | — | `/context` |
| `export` | — | `/export` |
| `export_html` | `output_path?` | `/export html [path]` |
| `compact` | `instructions?`,provider/model via session | `/compact [instructions]` (**#3**, holds active_run itself, supplies `CompactionSummaryGenerator`) |
| `list_plugins`,`plugin_failures`,`validate_plugin`(path),`inspect_plugin`,`enable_plugin`,`disable_plugin`,`list_plugin_prompts`,`list_plugin_skills`,`get_plugin_prompt`,`get_plugin_skill`,`run_plugin_command`(arguments) | `plugin_id`/`name`/`path`/`arguments` | `/plugins ...` or `/plugin run ...` |
| `list_mcp_servers`,`inspect_mcp_server`,`list_mcp_tools`,`restart_mcp_server` | `server_id` | `/mcp ...` |

That's **~41 command types** total.

---

## The event stream (records sent *before* the final response)

Two unrelated things both get serialized as envelope records via `serialize_event_envelope_jsonl`:

1. **`EventEnvelope`** (`events.h:215`) — the runtime event stream during a prompt/compact: `schema_version, event_id, timestamp, session_id, run_id?, turn_id?, message_id?, request_id?, correlation_id?, name, payload_type, payload_json`. Built from `RuntimeEvent` by `to_event_envelope`. Names come from `RuntimeEventType` (SessionStart, UserMessage, AssistantMessage, MessageUpdate/End, ReasoningStart/Delta/End, ProviderEvent, ToolStart/Progress/Result, CompactionStart/End, Retry/RetryTick, Canceled, Error, Done) + the payload families in `events.h:117-211` (Tool/Retry/Cancellation/Error/Completion).
2. **Resolver envelopes** (built directly via `resolver_event_envelope` in `output.cpp`) — the `permission_requested`/`permission_replied`/`permission_rule_added`/`permission_rule_removed`/`permission_grant_revoked`/`permission_grants_cleared`/`question_requested`/`question_replied`/`steer_queued`/`steer_applied`/`steer_skipped`/`follow_up_queued`/`follow_up_started`/`follow_up_skipped`/`cancel_requested` events. These set `payload_type` to Permission/Question/Cancellation/Queue via `payload_type_for_resolver_event`.

The envelope *is* a struct (`EventEnvelope`) — but its `payload_json` is still a hand-built string, and the success/error envelope has **no struct at all** (pure concatenation in `serialize_rpc_success_jsonl`).

---

## Why it's hard to grasp — and what a rewrite would consolidate

Your instinct is right: the data for a single record is scattered, and that's *because* there's no response model. Concretely, the rewrite would collapse three duplications:

1. **`RpcCommand` god-struct + string if/else** → one param struct per command + a `std::unordered_map<string, CommandHandler>` table. Each handler declares its own nlohmann-serializable params struct (your json03/json04 prototypes), and dispatch becomes a lookup. The 40-optional struct disappears.
2. **Hand-rolled envelope + payload strings** → a `Response { id; Success|Error body }` variant serialized by nlohmann (the json04 `SuccessResponse`/`FailureResponse` pattern, generalized), plus per-result payload structs. The "optional field omitted" behavior falls out for free from nlohmann's `*_WITH_DEFAULT` on `std::optional`. The scattered data problem you hit with `session_id`/`session_path` (from `SessionStore`) vs stats (from `SessionStats`) becomes explicit: a payload struct that the handler *assembles* from whatever sources, then serializes in one place.
3. **Three response mechanisms with ad-hoc wiring** → a single `Task` abstraction owning its lifecycle (running/done/errored), with the worker, the compact-in-place, and the resolver pause/resume all expressed as the same "Task emits events, then one terminal response" shape. The `record_async_error`/`take_async_error` side-channel and the `follow_up` "no immediate response" special case become first-class parts of the Task model rather than accidents.

The one subtlety to preserve in any rewrite: the **request thread must stay free** to handle `permission_reply`/`question_reply`/`steer`/`cancel` while a prompt runs. So "Task" can't mean "block the reader"; it means "spawn-and-remember, keep reading."

Topic List
1. json03 — empty-object `Empty` serialization — done
2. json04 — nested-object initializer-list gotcha + nested read-back — done
3. json04 — convert `rpc::json` structs to private-member classes with clean JSON keys — done
4. Map origin of every field in an emitted RPC stats record — done
5. Complete RPC command overview (all ~41 commands, params, backend API, response mechanism) — done
6. Event-record side: EventEnvelope stream + resolver envelopes — done
7. Rewrite strategy: per-command param structs + Response/payload structs + Task abstraction + dispatch table — needs discussion

Want me to go deeper on any one piece next — e.g. sketch the proposed `CommandHandler` table + per-command param structs, or model the `Task` lifecycle (states + the resolver pause/resume transitions) as the state machine your AAP objective was asking about?
