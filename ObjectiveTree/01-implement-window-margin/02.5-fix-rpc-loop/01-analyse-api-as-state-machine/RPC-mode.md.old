# RPC Mode State Machine

## 1. Protocol Overview

The RPC mode is a line-oriented JSONL protocol over a pair of
`std::istream&`/`std::ostream&` references (in production these are
`std::cin`/`std::cout`; in tests they are backed by custom
`streambuf` types such as `BlockingInputBuf` and `ThreadSafeStringBuf`).
The client sends one JSON object per line on the input stream; the
server writes one JSON object per line on the output stream.  Every
client request has a unique `id`; every response echoes that `id`.
Two kinds of output records exist:

| Record kind     | Shape                                                    |
|-----------------|----------------------------------------------------------|
| **response**    | `{"id":…,"type":"response","success":true/false,…}`     |
| **event**       | `{"event_id":…,"name":…,"request_id":…,"correlation_id":…,"payload_type":…,"payload":{…}}` |

A successful response carries an optional `result` field; a failed
response carries an `error` object.  Every response is terminal — it
signals completion of the request with that `id`.  Events may be
emitted before the response (during a prompt run or as a side effect of
a synchronous command).

## 2. Participants (Threads)

| Thread            | Role                                                                 |
|-------------------|----------------------------------------------------------------------|
| **Main loop**     | The RPC command dispatch loop. Reads input-stream lines, parses commands, dispatches them. When it receives a `prompt` command it spawns the prompt worker thread. |
| **Prompt worker** | A `std::jthread` spawned by the main loop for `prompt` and `invoke_command`-with-prompt. Calls `run_prompt()` (the AI agent loop — provider requests, tool calls, streaming deltas), handles follow-up chaining, and writes events plus the terminal response for the prompt. |

While the prompt worker is running, the main loop continues reading
and processing input lines.  It does **not** merely queue everything:

- `steer` and `follow_up` are **queued** (they require `active_run = true` and are consumed by the worker).
- All other commands are **processed immediately** by the main loop — `cancel`, `permission_reply`, `question_reply`, `get_state`, `permission_grants`, `permission_rule_add`, etc. each produce their own events and responses written right away.

This means the main loop and the worker both write to the output
stream concurrently.  Output writes are serialized by
`RpcOutput::mutex` to prevent interleaved or corrupted output records.

The main loop is single-threaded: while it is blocked inside a
synchronous command (e.g. `compact`, `context`, `export`), no further
input lines are read.

The prompt worker runs concurrently with the main loop.  Both threads
share three mutable state objects, each with its own mutex:

| State object            | Protects                                                |
|-------------------------|---------------------------------------------------------|
| `RpcRunState`           | `active_run`, `cancel_requested`, `input_closed`, `active_request_id`, steering/follow-up queues, async error |
| `PendingResolverState`  | Pending permission requests, pending question requests, session grants, condition variable |
| `RuntimeSession`        | Session store, model, reasoning, paths, workspace, etc. (protected by `session_mutex`) |

Output writes are serialized by `RpcOutput::mutex`.

## 3. Externally Visible States

The state machine is defined by three boolean flags that the client
can observe (directly or indirectly):

| Flag                | Where observable                                         |
|---------------------|----------------------------------------------------------|
| `active_run`        | Indirectly: steer/follow_up acceptance, mutation rejection |
| `cancel_requested`  | `get_state` response field `"cancel_requested"`          |
| `input_closed`      | Terminal: loop exits, no further output                  |

These combine into the following states:

```
                        ┌──────────┐
                   ┌────│  IDLE    │◄─────────────────────────┐
                   │    │active=F │                           │
                   │    │cancel=F │                           │
                   │    │closed=F │                           │
                   │    └────┬─────┘                           │
                   │         │                                 │
            prompt │         │ compact                         │ reset_cancel
            invoke  │         │                                 │ (session switch)
                   │    ┌────▼─────────┐    ┌──────────────┐    │
                   │    │PROMPT_ACTIVE │    │COMPACT_ACTIVE│    │
                   │    │active=T      │    │active=T      │    │
                   │    │cancel=F      │    │cancel=F      │    │
                   │    │closed=F      │    │closed=F      │    │
                   │    └────┬─────────┘    └──────┬───────┘    │
                   │         │ cancel              │ done       │
                   │    ┌────▼──────────┐    ┌─────▼──────┐     │
                   │    │PROMPT_CANCEL  │    │  IDLE      │─────┘
                   │    │active=T       │    │(or CANCEL  │
                   │    │cancel=T       │    │ _LATCHED)  │
                   │    │closed=F       │    └────────────┘
                   │    └────┬──────────┘
                   │         │ worker finishes
                   │    ┌────▼──────────┐
                   │    │CANCEL_LATCHED │
                   │    │active=F       │
                   │    │cancel=T       │
                   │    │closed=F       │
                   │    └────┬──────────┘
                   │         │ prompt / compact / session-switch
                   │         │ (clears cancel flag)
                   │    ┌────▼─────┐
                   └───►│  IDLE    │
                        └──────────┘

  Any state ──► SHUTTING_DOWN (input_closed=T) ──► exit
```

### State Definitions

| State              | active_run | cancel_requested | input_closed | Description                                                       |
|--------------------|------------|------------------|---------------|-------------------------------------------------------------------|
| **IDLE**           | false      | false            | false         | No prompt running. All commands accepted.                         |
| **PROMPT_ACTIVE**  | true       | false            | false         | A prompt (or invoke_command-with-prompt) is running on the worker. Queuing, cancellation, and resolver replies are accepted. Session mutations and new prompts are rejected. |
| **PROMPT_CANCEL**  | true       | true             | false         | Cancel was requested during an active prompt. Worker is still running but will stop. Queuing is rejected. Resolver replies are still accepted (they may be needed to unblock the worker). |
| **CANCEL_LATCHED** | false      | true             | false         | Cancel was requested while idle (or the worker finished after cancel). The cancel flag persists until the next prompt, compact, or session-switch clears it. |
| **COMPACT_ACTIVE** | true       | false            | false         | A `compact` command is running synchronously on the main loop. No other commands can be processed (main loop is blocked). Cancellable only via the external `cancel_requested` callback. |
| **SHUTTING_DOWN**  | *          | *                | true          | Input closed or write failure occurred. Active prompt is canceled, queues are cleaned up, and the loop exits. |

### Notes on COMPACT_ACTIVE

`compact` is the only command that both:
- Runs synchronously on the main loop thread (blocking all input processing).
- Sets `active_run = true` during its execution.

This means no RPC command (including `cancel`) can be processed while
compact is running.  The only cancellation path is the
`base_cancel_requested` callback supplied in `RuntimeRunOptions`,
which the compact handler polls internally.

Other synchronous commands (`context`, `export`, `export_html`,
plugin, MCP) check `active_run` but do NOT set it, because they are
expected to be fast and non-interruptible.  They still reject if a
prompt is active.

## 4. State Transitions

### IDLE → PROMPT_ACTIVE

**Trigger:** Client sends `prompt` (or `invoke_command` that produces a `prompt_message`).

**Preconditions:** `active_run` is false.

**Side effects:**
1. If a previous worker thread is finished but not yet joined, `reap_finished_prompt()` joins it.
2. `set_active_run(true, command->id)` — sets `active_run = true`, `active_request_id = command->id`, clears `cancel_requested`.
3. For `prompt`: image attachments are imported under `session_mutex`.
4. For `invoke_command`: `run_command()` is called under `session_mutex`; if it returns `prompt_message`, the worker is launched with that message.
5. Worker thread is launched. The worker will:
   - Resolve provider credentials.
   - Set up permission and question resolvers that emit `permission_requested` / `question_requested` events and block until the client replies.
   - Call `run_prompt()` which runs the agent loop (tool calls, provider requests, streaming events).
   - After the prompt completes, check for queued follow-ups and chain them.
   - Write the terminal success/error response for the request id.
   - Call `deactivate_and_clear_queued_messages()` to set `active_run = false` and clear remaining queues.

### IDLE → COMPACT_ACTIVE

**Trigger:** Client sends `compact`.

**Preconditions:** `active_run` is false.

**Side effects:**
1. Lock `run_state.mutex`, set `active_run = true`, `active_request_id = command->id`, clear `cancel_requested`.
2. Resolve provider and runtime options under `session_mutex`.
3. Call `run_command(session, {"/compact", ...})` synchronously on the main loop thread. This calls `generate_compaction_summary()` which makes a provider request.
4. The `cancel_requested` callback polls both `run_state.cancel_requested` and the base callback.
5. On completion (success or error): `clear_compact_active_run()` sets `active_run = false`. → IDLE.

### IDLE → CANCEL_LATCHED

**Trigger:** Client sends `cancel` while `active_run` is false.

**Side effects:**
1. `cancel_requested.store(true)`.
2. Queues are cleared (both are empty since no prompt is active).
3. Pending resolvers are canceled (none should exist).
4. `cancel_requested` event is emitted.
5. Response: `{"cancel_requested":true,"active_run":false,"cleared_steer":0,"cleared_follow_up":0}`.

### PROMPT_ACTIVE → PROMPT_CANCEL

**Trigger:** Client sends `cancel` while `active_run` is true.

**Side effects:**
1. Lock `run_state.mutex`:
   - `cancel_requested.store(true)`.
   - Capture `active_run` and `active_request_id`.
   - Move all steering messages to `cleared.steering_messages`.
   - Move all follow-up messages to `cleared.follow_up_messages`.
2. `cancel_pending_resolvers()` — resolves all pending permission/question requests as denied/canceled, notifies condition variable.
3. Write `steer_skipped` / `follow_up_skipped` events for each cleared message.
4. Write `cancel_requested` event with `active_run`, cleared counts, and `active_request_id`.
5. Write `follow_up_skipped` error responses for each cleared follow-up.
6. Response: `{"cancel_requested":true,"active_run":true,"cleared_steer":N,"cleared_follow_up":M}`.
7. The worker notices `cancel_requested` (via its `cancel_requested` callback or the next resolver check) and stops the agent loop.
8. Worker writes its terminal response (canceled error for the prompt).
9. Worker calls `deactivate_and_clear_queued_messages()` → `active_run = false`. → CANCEL_LATCHED.

### PROMPT_ACTIVE → IDLE (normal completion)

**Trigger:** The worker finishes the prompt and all follow-ups without cancel.

**Side effects:**
1. Worker writes the terminal success response for the last prompt/follow-up.
2. Worker calls `deactivate_and_clear_queued_messages()`:
   - `active_run = false`.
   - `active_request_id` cleared.
   - Remaining queues cleared (should be empty).
3. Worker writes `steer_skipped` / `follow_up_skipped` events for any leftover messages.
4. Worker exits. → IDLE (cancel_requested is still false).

### PROMPT_ACTIVE → PROMPT_ACTIVE (follow-up chaining)

**Trigger:** The worker finishes a prompt and finds a queued follow-up.

**Side effects:**
1. `before_terminal_response()` calls `prepare_next_follow_up()`:
   - `take_next_follow_up_message()` dequeues the next follow-up (unless `cancel_requested` or `input_closed`).
   - If a follow-up exists: `set_active_request_id(follow_up->request_id)` — sets `active_run = true` with the new request_id.
   - If no follow-up: `set_active_run(false)` — sets `active_run = false`.
2. Worker writes the terminal response for the just-completed prompt.
3. If a follow-up was dequeued: `follow_up_started` event is emitted, then `run_one_prompt()` is called with the follow-up message.
4. The follow-up runs as a new prompt with its own request_id, permission/question resolvers, and steering message correlation.

### CANCEL_LATCHED → IDLE

**Trigger:** Client sends `prompt`, `compact`, or a session-switch command (`new_session`, `open_session`, `switch_session`, `fork_session`, `clone_session`).

**Side effects:**
- For `prompt`/`compact`: `set_active_run(true, …)` or the compact handler clears `cancel_requested`.
- For session-switch: `reset_cancel_after_session_switch()` clears `cancel_requested`.

### CANCEL_LATCHED → PROMPT_ACTIVE

**Trigger:** Client sends `prompt` while `cancel_requested` is latched.

**Side effects:** Same as IDLE → PROMPT_ACTIVE, but `set_active_run(true, …)` clears the latched `cancel_requested` flag as part of the transition.

### Any state → SHUTTING_DOWN

**Trigger:** EOF on input (stdin closed) or output write failure.

**Side effects:**
1. `close_input_and_cancel()`:
   - `input_closed = true`.
   - `cancel_requested = true`.
   - All queues cleared.
2. `cancel_pending_resolvers()` — unblocks any pending permission/question waits.
3. If a worker is running: `steer_skipped` / `follow_up_skipped` events for cleared messages.
4. `follow_up_skipped` error responses for cleared follow-ups.
5. Worker thread is joined (`prompt_worker.reset()`).
6. Check for async errors.
7. Loop exits. Return to caller.

## 5. Command-to-State Gate Matrix

### Commands accepted in ALL states (IDLE, PROMPT_ACTIVE, PROMPT_CANCEL, CANCEL_LATCHED)

These commands never check `active_run`. They are safe to run
concurrently with a prompt because they only read session state under
`session_mutex` or operate on independent state.

| Command                   | Notes                                                              |
|--------------------------|--------------------------------------------------------------------|
| `get_protocol`           | Returns protocol version. No state access.                        |
| `get_state`              | Reads `cancel_requested` and session snapshot under `session_mutex`. |
| `list_sessions`          | Reads session list under `session_mutex`.                          |
| `session_tree`           | Reads session tree under `session_mutex`.                          |
| `list_models`            | Reads model registry.                                              |
| `session_metadata`       | Reads session metadata.                                             |
| `permission_grants`      | Lists session grants from `pending_state`.                         |
| `permission_grant_revoke`| Revokes a grant from `pending_state`. Emits `permission_grant_revoked` event. |
| `permission_grants_clear`| Clears all grants. Emits `permission_grants_cleared` event.        |
| `permission_reply`       | Resolves a pending permission request. Requires matching `request_id` and `correlation_id`. Emits `permission_replied` event. |
| `question_reply`         | Resolves a pending question request. Requires matching `request_id` and `correlation_id`. Emits `question_replied` event. |
| `cancel`                 | Sets `cancel_requested`, clears queues, cancels pending resolvers. Works in all states (idle or active). |

**Note:** During COMPACT_ACTIVE, the main loop is blocked, so none of
these can actually be processed.  The gate matrix applies to states
where the main loop is running.

### Commands accepted only when NOT active (IDLE, CANCEL_LATCHED)

These commands are rejected with `active_run_reject_error` when
`active_run` is true.  They either mutate the session or run
synchronous commands that access the session store.

| Command                     | Reason for rejection during active run                          |
|-----------------------------|-----------------------------------------------------------------|
| `prompt`                    | Starts a new prompt; would conflict with the running one.       |
| `invoke_command`            | May produce a prompt or access the session store.               |
| `compact`                   | Runs a synchronous provider call; sets `active_run` itself.     |
| `context`                   | Runs `run_command` synchronously; accesses session store.       |
| `export`                    | Same as `context`.                                              |
| `export_html`               | Same as `context`.                                               |
| `list_commands`             | Loads command registry from session.                            |
| `get_messages`              | Reads session messages; could race with prompt writes.          |
| `get_session_stats`         | Reads session stats.                                              |
| `validate_session`          | Validates session replay.                                         |
| `set_session_name`          | Mutates session metadata.                                         |
| `set_session_labels`        | Mutates session metadata.                                         |
| `permission_rules`          | Reads permission rule files.                                      |
| `permission_rule_add`       | Writes permission rule files. Emits `permission_rule_added` event. |
| `permission_rule_remove`    | Writes permission rule files. Emits `permission_rule_removed` event. |
| `set_model`                 | Switches the active model.                                        |
| `cycle_model`               | Cycles to the next model.                                        |
| `set_reasoning`             | Sets reasoning configuration.                                    |
| `clear_reasoning`           | Clears reasoning configuration.                                   |
| `fork_session`              | Creates a session fork and switches to it.                       |
| `clone_session`             | Creates a session clone and switches to it.                      |
| `summarize_branch`          | Appends a branch summary to a session.                           |
| `new_session`               | Creates and switches to a new session. Clears `cancel_requested`. |
| `open_session`              | Opens an existing session. Clears `cancel_requested`.            |
| `switch_session`            | Same as `open_session`.                                          |
| Plugin commands (all)       | Run plugin slash commands synchronously.                         |
| MCP commands (all)          | Run MCP slash commands synchronously.                            |

### Commands accepted only when active (PROMPT_ACTIVE)

| Command     | Gate                                                            |
|-------------|-----------------------------------------------------------------|
| `steer`     | Requires `active_run = true`, `active_request_id` non-empty, `cancel_requested = false`, `input_closed = false`. Queues a steering message tagged with the current `active_request_id` as `correlation_id`. Emits `steer_queued` event. |
| `follow_up` | Same gates as `steer`. Queues a follow-up message. Emits `follow_up_queued` event. Does NOT write a success response (the response comes when the follow-up is run or skipped). |

**Gate check order in `queue_rpc_message`:**
1. `input_closed` → error "RPC input is closed"
2. `cancel_requested` → error "agent loop canceled"
3. `!active_run || active_request_id.empty()` → error "RPC command requires an active prompt"
4. Queue size / message size limits → error "RPC queued message limit exceeded"

### Special: `invoke_command` dual behavior

`invoke_command` has two paths:
1. **No prompt_message:** `run_command()` returns a result synchronously. The result is written as a success response. No worker is launched.
2. **With prompt_message:** `run_command()` returns a `prompt_message`. The main loop then launches a prompt worker with that message, just like `prompt`. The transition to PROMPT_ACTIVE happens.

Both paths are gated by `active_run = false`.

### Special: `compact` as a synchronous active run

`compact` shares the code block with `context`/`export`/etc., but unlike them it sets `active_run = true` during its execution.  This allows:
- The `cancel_requested` callback to poll `run_state.cancel_requested`.
- Future `steer`/`follow_up` to be queued (though in practice the main loop is blocked, so this can't happen via RPC).

When `compact` finishes (success or error), `clear_compact_active_run()`
calls `set_active_run(false)`, returning to IDLE.

## 6. Event Taxonomy

### Permission Events
| Event name               | When emitted                                          | Payload type |
|--------------------------|-------------------------------------------------------|-------------|
| `permission_requested`   | Worker needs permission for a tool call.              | `permission`|
| `permission_replied`     | Client replied to a permission request.               | `permission`|
| `permission_rule_added`  | Client added a persistent permission rule.             | `permission`|
| `permission_rule_removed`| Client removed a persistent permission rule.          | `permission`|
| `permission_grant_revoked`| Client revoked a session grant.                     | `permission`|
| `permission_grants_cleared`| Client cleared all session grants.                  | `permission`|

### Question Events
| Event name           | When emitted                                   | Payload type |
|----------------------|------------------------------------------------|-------------|
| `question_requested` | Worker needs a question answered.              | `question`  |
| `question_replied`   | Client replied to a question request.         | `question`  |

### Queue Events
| Event name           | When emitted                                              | Payload type |
|----------------------|-----------------------------------------------------------|-------------|
| `steer_queued`       | Client sent `steer`; message accepted into queue.        | `queue`     |
| `steer_applied`      | Worker consumed a steering message before next provider request. | `queue` |
| `steer_skipped`      | Steering message was cleared without being applied.       | `queue`     |
| `follow_up_queued`   | Client sent `follow_up`; message accepted into queue.    | `queue`     |
| `follow_up_started`  | Worker begins running a queued follow-up.                | `queue`     |
| `follow_up_skipped`  | Follow-up was cleared without being run.                  | `queue`     |

### Cancellation Events
| Event name          | When emitted                                        | Payload type   |
|---------------------|-----------------------------------------------------|---------------|
| `cancel_requested`  | Client sent `cancel`.                               | `cancellation`|

### Runtime Events (from agent loop)
| Event name            | When emitted                                           |
|-----------------------|--------------------------------------------------------|
| `session_start`       | Prompt worker starts a provider turn.                  |
| `message_update`      | Streaming delta from the provider.                     |
| `assistant_message`   | Final assistant text for a turn.                       |
| `tool_call`           | A tool was called.                                     |
| `retry`               | Transport retry.                                       |
| `canceled`            | Agent loop was canceled.                               |
| `error`               | Agent loop encountered an error.                      |
| `compaction_start`    | Compact operation begins.                              |
| `compaction_end`      | Compact operation completes.                           |

## 7. Correlation IDs

The `correlation_id` field in events and queue messages identifies
which prompt request a message belongs to:

- For `steer`/`follow_up`: `correlation_id` = `active_request_id` at
  the time of queuing. This is the request id of the currently running
  prompt.
- For `permission_requested`/`question_requested`: `correlation_id` =
  `prompt_request_id` (the request id of the prompt that triggered the
  resolver).
- For `steer_applied`: `correlation_id` = the request id of the prompt
  that consumed the steering message (may differ from the original if
  the steer was queued during one prompt and applied during a
  follow-up, though in practice steers are always consumed by the
  prompt they were queued for).
- For `follow_up_started`: `correlation_id` = `request_id` = the
  follow-up's own request id.

The `request_id` field in events identifies the RPC command that
caused the event. For steer/follow_up queued events, `request_id` is
the steer/follow_up command's id. For permission/question events,
`request_id` is the prompt command's id.

## 8. Steering Message Lifecycle

1. Client sends `steer` with `message`.
2. Main loop calls `queue_rpc_message(steering_messages, …)`. The message is tagged with `correlation_id = active_request_id`.
3. `steer_queued` event is emitted.
4. Success response `{"queued":true,"correlation_id":…}` is written.
5. The worker calls `take_steering_messages(run_state, request_id)` which dequeues all steering messages with matching `correlation_id`.
6. For each dequeued message: `steer_applied` event is emitted, and the message text is appended to the steering messages list passed to the agent loop.
7. The agent loop incorporates the steering messages into the next provider request.
8. If the prompt finishes before consuming a steering message: `clear_queued_steering_messages()` dequeues all remaining, and `steer_skipped` events are emitted.

## 9. Follow-up Message Lifecycle

1. Client sends `follow_up` with `message`.
2. Main loop calls `queue_rpc_message(follow_up_messages, …)`. The message is tagged with `correlation_id = active_request_id`.
3. `follow_up_queued` event is emitted.
4. No success response is written yet (the response comes later).
5. After the current prompt completes, the worker calls `take_next_follow_up_message()`.
6. If a follow-up exists and no cancel is requested:
   a. `set_active_request_id(follow_up->request_id)` — updates `active_run` and `active_request_id`.
   b. `follow_up_started` event is emitted.
   c. `run_one_prompt()` is called with the follow-up message.
   d. Terminal response is written for the follow-up's request id.
7. If no follow-up exists: `set_active_run(false)`.
8. If cancel is requested: follow-up is not run. `follow_up_skipped` event and error response are emitted.

## 10. Permission Resolution Lifecycle

1. The agent loop calls a tool that requires permission.
2. The permission resolver (created by `make_rpc_permission_resolver`) is invoked.
3. If `cancel_requested` or `input_closed`: returns `canceled_error` immediately.
4. If the policy resolver allows: returns `Allow` (no event emitted).
5. If the policy resolver authoritatively denies: returns `Deny` (no event emitted).
6. If a matching session grant exists: returns `AllowSessionGrant` (no event emitted).
7. Otherwise: creates a `PendingPermissionRequest`, emits `permission_requested` event, and blocks on the condition variable.
8. Client sends `permission_reply` with `request_id`, `correlation_id`, and `decision` (allow/allow_session/deny).
9. `resolve_permission_reply()` finds the pending request, verifies `correlation_id`, resolves it, and notifies the condition variable.
10. If `allow_session`: a `PermissionSessionGrant` is created (deduplicated).
11. The resolver returns the decision to the agent loop.
12. The `permission_replied` event is emitted after the reply is processed.

## 11. Question Resolution Lifecycle

1. The agent loop calls a tool that asks a question.
2. The question resolver (created by `make_rpc_question_resolver`) is invoked.
3. If `cancel_requested` or `input_closed`: returns `canceled_error` immediately.
4. Otherwise: creates a `PendingQuestionRequest`, emits `question_requested` event, and blocks on the condition variable.
5. Client sends `question_reply` with `request_id`, `correlation_id`, and either `answer`, `selected`, or `selected_options`.
6. `resolve_question_reply()` validates the reply, resolves the request, and notifies the condition variable.
7. The `question_replied` event is emitted.
8. The resolver returns the answer to the agent loop.

## 12. Known Races and Design Weaknesses

These are the races that motivate the redesign. They all stem from
`active_run` being a single boolean that is read and written from
multiple threads without atomicity guarantees with respect to the
state transitions it guards.

### Race 1: Follow-up-vs-active-run window

In the prompt worker, `prepare_next_follow_up()` is called as the
`before_terminal_response` callback — before the terminal response is
written.  If there is no follow-up, it calls
`set_active_run(false)`, setting `active_run = false` while the worker
is still running (it still needs to write the terminal response and
do cleanup).

During this window:
- The main loop can accept a new `prompt` (it sees `active_run = false`).
- `reap_finished_prompt()` will join the old worker, but the old
  worker may still be writing its terminal response or cleanup events.
- A `steer`/`follow_up` sent during this window is rejected
  (`requires_active_prompt`), which is correct but may surprise the
  client if it was sent in response to seeing a partial output.

**Root cause:** `active_run` is set to false before the worker has
fully completed its cleanup.

### Race 2: Mutating-command-after-success window

After `prepare_next_follow_up()` sets `active_run = false` (no
follow-up), but before the worker finishes, the main loop can accept
session-mutating commands like `set_model`, `fork_session`, etc.
These commands mutate the session under `session_mutex`, which is also
used by the worker.  If the worker is still accessing the session
(e.g., writing the terminal response which includes `session_id`), the
mutation could produce inconsistent results.

**Root cause:** Same as Race 1 — `active_run` is cleared prematurely.

### Race 3: Cancel-during-follow-up-preparation

`prepare_next_follow_up()` calls `take_next_follow_up_message()`
which checks `cancel_requested` under `run_state.mutex`.  But after
it returns, the worker checks `cancel_requested()` again in the while
loop condition.  If cancel is requested between these two checks, the
follow-up was dequeued (and `active_request_id` was set) but will not
be run.  The follow-up is then handled by the cleanup path at the end
of the worker.

This is not strictly a data race (everything is under mutex), but it
means a follow-up can be transiently in an inconsistent state where
`active_request_id` has been updated but the prompt hasn't started.

### Race 4: `reap_finished_prompt` timing

`reap_finished_prompt()` checks `active_run()` (which locks the
mutex) and if false, calls `prompt_worker.reset()` (which joins the
thread).  But between the `active_run()` check and `reset()`, the
worker could theoretically restart (set `active_run = true` for a
follow-up).  In practice this can't happen because
`prepare_next_follow_up` is called before `active_run` is checked,
but the code is fragile because the check and the action are not
atomic.

### Race 5: Output write failure during cleanup

`output.on_write_failure` calls `close_input_and_cancel()` and
`cancel_pending_resolvers()`.  This can be triggered from either the
main loop or the worker thread.  If both threads encounter write
failures simultaneously, the cleanup is called twice, but the
mutex-protected operations inside are safe.  However, the
`on_write_failure` callback itself is invoked after unlocking
`output.mutex`, so it could be called concurrently from two threads.

### Race 6: `cancel` and `permission_reply` / `question_reply` interleaving

`cancel` calls `cancel_pending_resolvers()` which resolves all pending
requests as denied and clears the maps.  If a `permission_reply`
arrives concurrently (on the main loop, but `cancel` is also on the
main loop, so this can't happen — both are processed sequentially by
the main loop).  However, the resolver running on the worker thread
could be between checking `cancel_requested` and creating the pending
request when cancel is processed.  The code handles this with a
post-creation re-check, but the window exists.

### Race 7: `set_active_run` clearing `cancel_requested`

`set_active_run(true, …)` clears `cancel_requested` as a side effect.
This means that if `cancel` is sent while the main loop is between
checking `active_run` (which returns false) and calling
`set_active_run(true, …)` (which clears the cancel), the cancel is
silently dropped.  However, since both `cancel` and `prompt` are
processed by the single-threaded main loop, this can't happen — the
cancel would be processed before or after the prompt, never
concurrently.

This is not a race in the current single-main-loop design, but it
would become one if the design were ever changed to process input
asynchronously.

## 13. Summary of Required Behavior

The state machine must enforce the following invariants:

1. **At most one prompt worker exists at any time.** A new `prompt` or
   `invoke_command`-with-prompt can only start after the previous
   worker has fully finished — including writing its terminal response
   and all cleanup events.

2. **Session mutations are excluded during active prompts.** Any
   command that reads or writes the session store in a way that could
   conflict with the running prompt must be rejected while
   `active_run` is true.

3. **Steering and follow-up messages are only accepted during an
   active prompt.** They must be associated with the correct
   `active_request_id` (correlation).

4. **Cancel is always accepted.** When idle, it latches until the
   next prompt clears it.  When active, it cancels the worker, clears
   queues, and denies pending resolvers.

5. **Permission and question replies are always accepted.** They must
   match a pending request by `request_id` and `correlation_id`.
   Replies with no matching pending request are rejected with an error.

6. **Input closure triggers clean shutdown.** The active prompt is
   canceled, queued messages are cleared with skip events, and the
   loop exits.

7. **Follow-up chaining is atomic with respect to the active_run
   state.** The transition from one prompt to its follow-up must not
   create a window where `active_run` is false (and a new prompt could
   start).

8. **The `active_request_id` always reflects the currently running
   prompt.** Steering messages are tagged with this id so they are
   consumed by the correct prompt, even across follow-up transitions.
