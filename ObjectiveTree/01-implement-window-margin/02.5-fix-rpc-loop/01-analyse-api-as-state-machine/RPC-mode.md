# RPC Mode State Machine

## 1. Protocol Overview

The RPC mode is a line-oriented JSONL protocol over a pair of
`std::istream&`/`std::ostream&` references (in production these are
`std::cin`/`std::cout`; in tests they are backed by custom
`streambuf` types). The client sends one JSON object per line on the
input stream; the server writes one JSON object per line on the output
stream.

Every client request carries a unique `id`. Two kinds of output
records exist:

| Record kind     | Shape                                                    |
|-----------------|----------------------------------------------------------|
| **response**    | `{"id":…,"type":"response","success":true/false,…}`     |
| **event**       | `{"event_id":…,"name":…,"request_id":…,"correlation_id":…,"payload_type":…,"payload":{…}}` |

A successful response carries an optional `result` field; a failed
response carries an `error` object. Every response is terminal — it
signals completion of the request with that `id`. Events may be
emitted before the response.

## 2. Client-Side State Machine

The client's state is determined by which commands it has sent and
which terminal responses it has received. The central question is
whether a prompt is **pending** — sent but not yet completed by a
terminal response.

### States

```
                          ┌──────────────────────────────┐
                          │           IDLE               │
                          │ No pending prompt.            │
                          │ All commands accepted.        │
                          │ steer/follow_up rejected.     │
                          └──────┬───────────────────────┘
                                 │ send prompt / invoke_command
                                 │ (with prompt_message)
                          ┌──────▼───────────────────────┐
                          │       PROMPT_PENDING          │
                          │ Prompt sent, terminal         │
                          │ response not yet received.    │
                          │                               │
                          │  steer / follow_up : accepted │
                          │  cancel              : accepted │
                          │  permission_reply    : accepted │
                          │  question_reply      : accepted │
                          │  read-only queries   : accepted │
                          │  prompt / mutations  : rejected │
                          │  compact             : rejected │
                          └──┬──────────┬──────────┬──────┘
                             │          │          │
              send cancel    │          │ recv     │ recv
              (→ CANCELING)  │          │ terminal │ terminal
                             │          │ response │ response
                             │          │ (success)│ (error)
                             │          │          │
                    ┌────────▼───┐  ┌───▼────┐  ┌─▼────────┐
                    │ CANCELING  │  │ IDLE   │  │ IDLE     │
                    │ cancel     │  │       │  │          │
                    │ sent;      │  └───────┘  └──────────┘
                    │ waiting for│
                    │ terminal   │
                    │ response   │
                    │            │
                    │ steer/     │
                    │ follow_up  │
                    │ rejected   │
                    └───────┬────┘
                            │ recv terminal response
                            │ (canceled error)
                    ┌───────▼────┐
                    │ IDLE        │
                    └─────────────┘

  IDLE ──send compact──► COMPACT_PENDING ──recv response──► IDLE
                           (no other commands
                            processed until response)
```

### State Definitions

| State             | Description                                                                 |
|-------------------|-----------------------------------------------------------------------------|
| **IDLE**          | No pending prompt. All commands accepted. `steer`/`follow_up` rejected (no active prompt). |
| **PROMPT_PENDING** | A `prompt` (or `invoke_command`-with-prompt) has been sent; its terminal response has not yet been received. `steer`, `follow_up`, `cancel`, `permission_reply`, `question_reply`, and read-only queries are accepted. New prompts, session mutations, and `compact` are rejected. |
| **CANCELING**     | A sub-state of PROMPT_PENDING: `cancel` was sent while a prompt was pending. `steer`/`follow_up` are now rejected. The client waits for the prompt's terminal response (which will be an error). |
| **COMPACT_PENDING** | A `compact` was sent; its terminal response has not yet been received. The server processes `compact` synchronously and will not read or process any other input until it completes. |

### Pending Sub-States within PROMPT_PENDING

While in PROMPT_PENDING, the client may receive `permission_requested`
or `question_requested` events. These create an expectation that the
client must send a matching `permission_reply` or `question_reply`
before the prompt can make progress:

| Sub-state                  | Trigger                                              | Expected client action           |
|----------------------------|------------------------------------------------------|----------------------------------|
| Permission reply expected   | Received `permission_requested` event                | Send `permission_reply`          |
| Question reply expected    | Received `question_requested` event                 | Send `question_reply`            |

Multiple permission/question requests may be pending simultaneously
(though in practice they are sequential). The prompt remains blocked
until each is resolved.

### Transitions

| From              | To               | Trigger                                                              |
|-------------------|------------------|----------------------------------------------------------------------|
| IDLE              | PROMPT_PENDING   | Client sends `prompt` or `invoke_command` (with prompt_message).     |
| IDLE              | COMPACT_PENDING  | Client sends `compact`.                                              |
| PROMPT_PENDING    | IDLE             | Client receives terminal response for the pending prompt (success). |
| PROMPT_PENDING    | CANCELING        | Client sends `cancel`.                                               |
| PROMPT_PENDING    | PROMPT_PENDING   | Client receives terminal response for a prompt, but a follow-up was queued and starts running (see §6). |
| CANCELING         | IDLE             | Client receives terminal response for the canceled prompt (error).   |
| COMPACT_PENDING   | IDLE             | Client receives terminal response for `compact`.                     |

### Note on `cancel` while IDLE

If the client sends `cancel` while in IDLE, the server immediately
responds with `{"cancel_requested":true,"active_run":false,…}`. The
client stays in IDLE. However, the server latches a `cancel_requested`
flag that is observable via `get_state` and is automatically cleared
when the next `prompt`, `compact`, or session-switch command is sent.

### Note on `compact` blocking

While the server processes `compact`, it does not read any further
input. From the client's perspective, this means any command sent
during COMPACT_PENDING will not be processed until after the `compact`
response is received. This is a protocol-level constraint: the client
should not send commands during COMPACT_PENDING (or must accept that
they will be delayed).

## 3. Command Reference

Every command is a JSON object with at least `"id"` (unique string)
and `"type"` (command name) fields.

### 3.1 Commands accepted in all states

These commands produce an immediate terminal response and do not
change the client's state (unless explicitly noted).

| Command                  | Parameters                      | Response / Events                                                                 |
|--------------------------|---------------------------------|-----------------------------------------------------------------------------------|
| `get_protocol`           | —                               | Response with protocol version and supported session entry versions.              |
| `get_state`              | —                               | Response with session id, model, mode, reasoning, and `cancel_requested` flag.   |
| `list_sessions`          | —                               | Response with list of session ids.                                               |
| `session_tree`           | —                               | Response with session tree (path, children, metadata).                          |
| `list_models`            | —                               | Response with configured model catalog.                                          |
| `session_metadata`       | —                               | Response with session name, labels, actor, timestamps.                          |
| `permission_grants`     | —                               | Response with active session permission grants.                                  |
| `permission_grant_revoke`| `grant_id`                      | Event `permission_grant_revoked` + response with revoked grant.                 |
| `permission_grants_clear`| —                               | Event `permission_grants_cleared` + response with cleared count.                |
| `permission_reply`       | `request_id`, `correlation_id`, `decision`, `reason`? | Event `permission_replied` + response `{"success":true}`. Errors if no matching pending request or mismatched correlation_id. |
| `question_reply`        | `request_id`, `correlation_id`, `answer`? / `selected`? / `selected_options`? | Event `question_replied` + response `{"success":true}`. Errors if no matching pending request, mismatched correlation_id, or invalid selection. |
| `cancel`                 | —                               | Event `cancel_requested` + skip events for cleared queue items + response with `cancel_requested`, `active_run`, cleared counts. If prompt pending: transitions to CANCELING. |

### 3.2 Commands accepted only in IDLE (rejected in PROMPT_PENDING)

These are rejected with an error response when a prompt is pending.

| Command                  | Parameters                      | Response / Events                                                                 |
|--------------------------|---------------------------------|-----------------------------------------------------------------------------------|
| `prompt`                 | `message`, `attachments`?, `images`? | Many events (session_start, message_update, assistant_message, tool_call, etc.) + terminal response. Transitions to PROMPT_PENDING. |
| `invoke_command`         | `name`, `command_arguments`?    | Events + terminal response. May produce a prompt (transitions to PROMPT_PENDING) or return synchronously (stays IDLE). |
| `compact`                | `instructions`?                 | Events (compaction_start, compaction_end) + terminal response. Transitions to COMPACT_PENDING. |
| `context`                | —                               | Terminal response with context output.                                           |
| `export`                 | —                               | Terminal response with export text.                                              |
| `export_html`            | `outputPath`? / `output_path`?  | Terminal response with HTML export.                                               |
| `list_commands`          | —                               | Terminal response with command registry.                                          |
| `get_messages`           | —                               | Terminal response with session messages.                                         |
| `get_session_stats`      | —                               | Terminal response with session entry counters.                                    |
| `validate_session`       | —                               | Terminal response with validation result.                                        |
| `set_session_name`       | `session_name`                  | Terminal response with updated metadata.                                         |
| `set_session_labels`     | `labels`                       | Terminal response with updated metadata.                                         |
| `permission_rules`       | —                               | Terminal response with persistent permission rules.                               |
| `permission_rule_add`    | `action`, `operation`, `reason`, `scope`?, `mode`?, `target_path`?, `tool_name`?, `command`? | Event `permission_rule_added` + terminal response with added rule. |
| `permission_rule_remove` | `rule_id`                      | Event `permission_rule_removed` + terminal response with removed rule.          |
| `set_model`              | `model`, `provider`?            | Terminal response with updated state.                                            |
| `cycle_model`            | —                               | Terminal response with cycled model state.                                       |
| `set_reasoning`          | `reasoning_level`, `reasoning_budget_tokens`?, `reasoning_display`? | Terminal response with reasoning state.                              |
| `clear_reasoning`        | —                               | Terminal response with reasoning cleared.                                        |
| `fork_session`           | `session_id`?, `branch_from_entry_id`?, `session_name`?, `labels`? | Terminal response with new session state. Clears latched cancel.    |
| `clone_session`          | `session_id`?, `session_name`?, `labels`? | Terminal response with new session state. Clears latched cancel.          |
| `summarize_branch`       | `branch_root_entry_id`, `branch_tip_entry_id`, `summary`, `provider`, `model`, `reason`, `session_id`? | Terminal response with branch summary entry. |
| `new_session`            | —                               | Terminal response with new session state. Clears latched cancel.                  |
| `open_session`           | `session_id`                    | Terminal response with switched session state. Clears latched cancel.            |
| `switch_session`         | `session_id`                    | Same as `open_session`.                                                          |
| Plugin commands          | Various                         | Terminal response with plugin output.                                             |
| MCP commands             | Various                         | Terminal response with MCP output.                                               |

### 3.3 Commands accepted only in PROMPT_PENDING

| Command     | Parameters  | Response / Events                                                                    |
|-------------|-------------|--------------------------------------------------------------------------------------|
| `steer`     | `message`   | Event `steer_queued` + immediate terminal response `{"queued":true,"correlation_id":…}`. Rejected if cancel was already sent or input closed. |
| `follow_up` | `message`   | Event `follow_up_queued` only. No immediate terminal response — the response comes later when the follow-up is run or skipped. Rejected if cancel was already sent or input closed. |

**Gate check order for `steer`/`follow_up`:**
1. Input closed → error "RPC input is closed"
2. Cancel requested → error "agent loop canceled"
3. No active prompt → error "RPC command requires an active prompt"
4. Queue limit exceeded → error "RPC queued message limit exceeded"

### 3.4 `invoke_command` dual behavior

`invoke_command` dispatches a slash command. If the command produces
a `prompt_message`, it behaves like `prompt` (transitions to
PROMPT_PENDING, emits events, terminal response comes later). If not,
it returns synchronously with a terminal response and stays in IDLE.
The client cannot predict which path will be taken — it must wait for
the terminal response to know.

## 4. Event Reference

### Permission Events
| Event name               | When emitted                                          | Payload type |
|--------------------------|-------------------------------------------------------|-------------|
| `permission_requested`   | The running prompt needs permission for a tool call. | `permission`|
| `permission_replied`     | Client sent `permission_reply`.                       | `permission`|
| `permission_rule_added`  | Client sent `permission_rule_add`.                    | `permission`|
| `permission_rule_removed`| Client sent `permission_rule_remove`.                 | `permission`|
| `permission_grant_revoked`| Client sent `permission_grant_revoke`.              | `permission`|
| `permission_grants_cleared`| Client sent `permission_grants_clear`.            | `permission`|

### Question Events
| Event name           | When emitted                                   | Payload type |
|----------------------|------------------------------------------------|-------------|
| `question_requested` | The running prompt needs a question answered.  | `question`  |
| `question_replied`   | Client sent `question_reply`.                  | `question`  |

### Queue Events
| Event name           | When emitted                                              | Payload type |
|----------------------|-----------------------------------------------------------|-------------|
| `steer_queued`       | Client sent `steer`; message accepted into queue.        | `queue`     |
| `steer_applied`      | The prompt consumed a steering message before the next provider request. | `queue` |
| `steer_skipped`      | A steering message was cleared without being applied.     | `queue`     |
| `follow_up_queued`   | Client sent `follow_up`; message accepted into queue.    | `queue`     |
| `follow_up_started`  | The server begins running a queued follow-up.            | `queue`     |
| `follow_up_skipped`  | A follow-up was cleared without being run.                | `queue`     |

### Cancellation Events
| Event name          | When emitted               | Payload type   |
|---------------------|-----------------------------------------------------|---------------|
| `cancel_requested`  | Client sent `cancel`. | `cancellation`|

### Runtime Events (emitted during prompt execution)
| Event name            | When emitted                                           |
|-----------------------|--------------------------------------------------------|
| `session_start`       | A provider turn begins.                                |
| `message_update`      | Streaming text delta from the provider.                |
| `assistant_message`   | Final assistant text for a turn.                       |
| `tool_call`           | A tool was called.                                     |
| `retry`               | Transport retry.                                       |
| `canceled`            | The agent loop was canceled.                           |
| `error`               | The agent loop encountered an error.                   |
| `compaction_start`    | Compact operation begins.                              |
| `compaction_end`      | Compact operation completes.                            |

## 5. Correlation IDs

The `request_id` and `correlation_id` fields in events identify which
prompt a message belongs to:

- **`request_id`**: The `id` of the RPC command that caused this
  event. For `steer_queued`/`follow_up_queued`, this is the
  steer/follow_up command's `id`. For `permission_requested`/
  `question_requested`, this is the prompt command's `id`.

- **`correlation_id`**: The `id` of the currently running prompt.
  For `steer_queued`/`follow_up_queued`, this is the prompt `id` that
  was active when the message was queued. For `permission_requested`/
  `question_requested`, this is the prompt `id` that triggered the
  resolver. For `follow_up_started`, this is the follow-up's own `id`
  (it becomes the active prompt).

- For `permission_reply`/`question_reply` commands: the client must
  supply `request_id` (the resolver request id from the
  `permission_requested`/`question_requested` event) and
  `correlation_id` (the prompt `id` that the event's `correlation_id`
  matched). The server verifies that these match a pending request.

## 6. Steering Message Lifecycle

1. Client sends `steer` with `message` and a new `id`.
2. Server emits `steer_queued` event (with `request_id` = steer `id`,
   `correlation_id` = active prompt `id`).
3. Server sends immediate terminal response `{"queued":true,"correlation_id":…}`.
4. At some later point, the prompt consumes the steering message
   before the next provider request.
5. Server emits `steer_applied` event.
6. The steering message text is incorporated into the next provider
   request.
7. If the prompt finishes or is canceled before consuming the steering
   message: server emits `steer_skipped` event.

## 7. Follow-up Message Lifecycle

1. Client sends `follow_up` with `message` and a new `id`.
2. Server emits `follow_up_queued` event (with `request_id` =
   follow_up `id`, `correlation_id` = active prompt `id`).
3. **No immediate terminal response.** The client must wait.
4. After the current prompt's terminal response is sent, the server
   checks for queued follow-ups.
5. If a follow-up exists and cancel was not requested:
   a. Server emits `follow_up_started` event (with `request_id` =
      `correlation_id` = follow_up `id`).
   b. The follow-up runs as a new prompt with its own message.
   c. Runtime events are emitted as normal.
   d. Server sends terminal response for the follow_up `id`.
6. If cancel was requested, or input was closed:
   a. Server emits `follow_up_skipped` event.
   b. Server sends terminal error response for the follow_up `id`
      (error: "agent loop canceled" or "queued follow_up skipped").
7. After the follow-up completes, the server checks for more queued
   follow-ups. Steps 5–7 repeat until the queue is empty, at which
   point the client returns to IDLE.

**Key protocol property:** While a follow-up is running, the client is
still in PROMPT_PENDING. New `steer`/`follow_up` commands are
accepted and associated with the follow-up's `id` (the new active
prompt).

## 8. Permission Resolution Lifecycle

1. During a prompt, the agent needs permission for a tool call.
2. If a permission policy auto-allows or a session grant matches, no
   event is emitted and the prompt continues. The client never sees
   this.
3. Otherwise, the server emits `permission_requested` event with:
   - `request_id` = prompt `id`
   - `correlation_id` = prompt `id`
   - `resolver_request_id` (in payload) = a unique id for this
     resolver request (e.g. `"permission_…"`).
4. The prompt is now blocked. The client must send `permission_reply`
   with:
   - `request_id` = the `resolver_request_id` from the event payload.
   - `correlation_id` = the prompt `id`.
   - `decision` = `"allow"`, `"allow_session"`, or `"deny"`.
   - `reason` (optional).
5. Server validates the reply, emits `permission_replied` event, and
   sends terminal response `{"success":true}` for the
   `permission_reply` command's `id`.
6. If `decision` is `"allow_session"`: a session grant is created
   (deduplicated) that will auto-allow matching future permission
   requests without emitting an event.
7. The prompt unblocks and continues with the decision.

**If cancel is sent while a permission is pending:** The server
resolves the pending request as denied (canceled) and emits the
appropriate skip/cancel events. The client does not need to send a
`permission_reply`.

## 9. Question Resolution Lifecycle

1. During a prompt, the agent asks a question.
2. The server emits `question_requested` event with:
   - `request_id` = prompt `id`
   - `correlation_id` = prompt `id`
   - `resolver_request_id` (in payload) = a unique id for this
     resolver request (e.g. `"question_…"`).
   - Options, multiple-select flag, custom-answer flag (in payload).
3. The prompt is now blocked. The client must send `question_reply`
   with:
   - `request_id` = the `resolver_request_id` from the event payload.
   - `correlation_id` = the prompt `id`.
   - One of: `answer` (custom text, only if `allow_custom` is true),
     `selected` (single option value), or `selected_options` (array
     of option values, only if `multiple` is true).
4. Server validates the reply, emits `question_replied` event, and
   sends terminal response `{"success":true}` for the
   `question_reply` command's `id`.
5. The prompt unblocks and continues with the answer.

**If cancel is sent while a question is pending:** Same as
permission — the server resolves the pending request as canceled.

## 10. Cancellation Lifecycle

1. Client sends `cancel` with a new `id`.
2. Server clears all queued steering and follow-up messages.
3. Server resolves all pending permission/question requests as denied
   (canceled).
4. Server emits `steer_skipped` / `follow_up_skipped` events for each
   cleared queue item.
5. Server emits `cancel_requested` event with `active_run` flag,
   cleared counts, and the active request id.
6. Server sends terminal error responses for each cleared follow-up
   (`"agent loop canceled"`).
7. Server sends terminal response for the `cancel` command's `id`
   with `cancel_requested`, `active_run`, and cleared counts.
8. If a prompt was active: the prompt receives the cancel signal and
   eventually emits its terminal response (error: canceled). The
   client is then in CANCELING until that response arrives.

**Cancel while IDLE:** The server still sets a `cancel_requested`
flag (observable via `get_state`). The next `prompt`, `compact`, or
session-switch command clears it automatically. No queue cleanup is
needed (queues are empty).

## 11. Input Closure

When the input stream reaches EOF, the server:

1. Cancels any active prompt (as if `cancel` was sent).
2. Clears all queued messages.
3. Emits `steer_skipped` / `follow_up_skipped` events for cleared
   items.
4. Sends terminal error responses for cleared follow-ups.
5. Stops processing and exits.

The client should close the input stream only after receiving all
expected terminal responses, or when it wants to force shutdown.

## 12. Protocol-Level Ambiguities

These are protocol-level issues that the redesign should address:

### 12.1 Follow-up rejection race

The client sends `follow_up` while in PROMPT_PENDING. If the prompt
finishes and transitions to IDLE before the server processes the
`follow_up`, the follow-up is rejected with "RPC command requires an
active prompt." The client has no way to know whether the follow-up
will be accepted until it receives either the `follow_up_queued`
event or the error response.

### 12.2 Follow-up terminal response ordering

After a prompt completes, the server may start a queued follow-up.
The terminal response for the original prompt and the
`follow_up_started` event for the follow-up are emitted in the
correct order (prompt response first, then follow_up_started). But
the client must track which request ids are still pending.

### 12.3 `invoke_command` unpredictability

The client cannot predict whether `invoke_command` will produce a
prompt (transitioning to PROMPT_PENDING with a delayed terminal
response) or return synchronously (staying in IDLE with an immediate
terminal response). The client must handle both cases.

### 12.4 Cancel flag latching

An idle `cancel` latches a flag that is observable via `get_state`.
This flag is cleared by the next `prompt`/`compact`/session-switch.
The client may see `cancel_requested:true` in `get_state` responses
even though no prompt was ever active. This is a minor confusion but
not a correctness issue.

## 13. Required Invariants

The protocol must enforce:

1. **At most one prompt is active at a time.** A `prompt` sent while
   another prompt is pending is rejected.

2. **Session mutations are excluded during an active prompt.**
   Commands that read or write the session in ways that could
   conflict with the running prompt are rejected.

3. **Steering and follow-up messages are only accepted during an
   active prompt.** They are associated with the active prompt's `id`
   as `correlation_id`.

4. **Cancel is always accepted.** When sent during a prompt, it
   cancels the prompt and clears queues. When sent while idle, it
   latches until the next prompt/compact/session-switch.

5. **Permission and question replies are always accepted** if they
   match a pending request. Replies with no matching pending request
   are rejected with an error.

6. **Every command except `follow_up` receives an immediate terminal
   response.** `follow_up` receives a `follow_up_queued` event and its
   terminal response comes later (when the follow-up is run or
   skipped).

7. **Input closure triggers clean shutdown.** The active prompt is
   canceled, queued messages are cleared with skip events, and
   follow-ups receive error responses.

8. **Follow-up chaining does not create a gap.** The transition from
   one prompt to its follow-up must not allow a new `prompt` to be
   accepted in between.
