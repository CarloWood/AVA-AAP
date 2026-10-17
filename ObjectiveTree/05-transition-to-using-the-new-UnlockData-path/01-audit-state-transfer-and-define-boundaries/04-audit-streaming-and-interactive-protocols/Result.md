# Audit 04: retained interaction and content interfaces

## Decision and scope

These interfaces are identified here so the ordinary backend-state refactor does not accidentally replace or break them. **They are deferred, not candidates for implementation with UnlockedData in this iteration.** The user will first design and implement one popup where the backend requires a user reply, then decide how to generalize that design. No replacement request/reply API, pending-request canonical domain, journal, cursor protocol or history collection implementation is proposed here.

Preserve current functionality and interfaces by default, including protocol-specific status, identity/cancellation fields and existing dependencies. Partial removal or an unimplemented endpoint requires agreement on the particular feature and safe unsupported behavior. Never silently auto-answer, bypass authorization or remove a feature to meet a no-Snapshot rule.

Independent ordinary backend facts remain in scope: active agent receiving the next submitted prompt, model/variant/provider, backend directories/workspace root, tokens and other approved passive observations. They should not remain on obsolete bulk-state transfers merely because the same callback also participates in a deferred exchange. Conversely, do not split an interaction's own state into UnlockedData solely because it can be displayed.

Pure appearance, local selection, modal layout and drafts remain tui-thread-owned in tui-specific code without locks/UnlockedData. The existing cross-thread request queues/services below have their own synchronization; preserving that synchronization does not authorize new locks around ordinary local UI preferences. Draft editing is not backend state; submitting a string, explicitly queueing a follow-up or passing text to a frontend external-editor utility are distinct operations.

## Evidence and limits

Read-only investigation of the current working tree. No repository changes, builds or tests were performed. Source locations below are relative to REPOROOT. Explore located interface/test definitions; the primary planner inspected prompt request/response queueing, plugin endpoint contracts, app subagent bindings, inspector frame shape, branch coordinator API and Mermaid bridge behavior. This is a bounded compatibility inventory, not a complete concurrency or security audit. Existing tests are anchors to preserve/run during implementation, not claimed fresh results.

## 1. Permission and question popups

**Retained interface:** permissions::PermissionResolver accepting PermissionPrompt and returning Result<PermissionResolutionDecision>; agent::QuestionResolver accepting QuestionPrompt and returning Result<QuestionAnswer>. They travel through TuiSubmitContext into backend submission/tool handling. TuiRuntimeOptions::remember_permission_rule provides the persistent-rule action and returns a rule identifier.

**Principal path:** runtime constructs RuntimePromptCoordinator; its resolvers are supplied to the active-run worker in runtime_active_run_internal.cpp. The application passes them into handle_interactive_submission/run options. A backend resolver invocation queues a PendingPermissionRequest or PendingQuestionRequest; the TUI services it, builds the local prompt view, accepts input and completes the waiting result. Sources: tui/runtime_prompts_internal.h; runtime_prompts_internal.cpp:105 onward,636–779; app/tui/interactive_tui.cpp:484–486,803–805; permissions/permission.h and agent/question.h.

**Concrete constraints:**

* Each pending request owns its prompt and its own mutex/condition-variable/result. Resolver callbacks wait for a result or closure; the result is not a last-value display slot.
* There are separate permission/question deques. claim_pending_request_locked takes the first permission if any, otherwise the first question (662–675): do not describe this as globally FIFO across both types.
* service_pending_request removes a request under the queue mutex, then resolves/renders outside that mutex (691–702). Search input has explicit prompt-precedence handling (705–724).
* Closing acceptance fails queued permissions as Deny and questions as an interrupted error (636–659). This is existing behavior to preserve, not a newly selected cancellation API.
* Permission UI starts with Deny selected, checks session-grant eligibility, and separately determines whether allow/deny can be remembered. Missing/failing persistence must not turn a request into an Allow (105–199). Persistent rule creation remains backend authority.
* Prompt views, highlighted choices and custom-answer editing are local. Clearing a view is not equivalent to completing the backend waiter. User guidance and secret-question content must not be exposed by broad debug printing.

**Boundary exception:** retain prompt types/resolver callbacks, queue/waiter objects, completion paths, rule-action callback and prompt-related audit delivery. Do not implement a canonical PendingPrompt, dirty-bit decision delivery or new permission service during /05. A future popup design may revisit these interfaces, not this refactor.

**Regression anchors:** tests/tui_permission_tests.cpp (includes question handling), prompt-related runtime/composer tests; terminal main_permission_flow and main_question_flow. Preserve simultaneous queued requests, modal precedence, deny/cancel, persistent-rule failure and session-grant isolation. This inspection does not claim those files already cover every interleaving.

## 2. Plugin UI requests, replies and invocation surfaces

**Retained interface:** TuiPluginUiEndpoint, Binding, Request and Reply in tui/runtime_plugin_ui.h. Binding identifies plugin, command and invocation; requests add a hidden request_id and Status/Widget/Select/Confirm payload. Replies are Ack/Select/Confirm/Cancel with optional chosen option identity.

**Principal path:** RuntimePluginUiCoordinator::begin_submission creates the endpoint; TuiSubmitContext carries it to app/tui/interactive_tui.cpp's invocation-capability adapter. The app uses endpoint.present/close; runtime_plugin_ui_internal.* services the queued presentation and input and finishes it with the submission. Relevant policy is also captured by tests/plugin_ui_authority_inventory_test.py.

**Concrete constraints:** endpoint validity requires a live weak runtime token, a nonzero deadline and present/close callbacks (runtime_plugin_ui.h:91–101). The presenter receives request/deadline/cancel callback. Header limits include four widgets, 32 widget lines, 8 KiB text, 64 queued records and a 120-second invocation deadline (15–19). Hidden request/option IDs are protocol data, not rendered labels. plugin_ui_host_chrome_fits requires complete identifying host chrome/controls to fit before displaying the plugin-controlled surface (66–69).

**Boundary exception:** preserve the complete existing invocation-bound exchange, including status/widgets that appear passive but are tied to the invocation protocol. Do not independently canonicalize those fields or convert a reply into a dirty flag. Dock/modal layout remains frontend presentation; this does not make theme/draft/preferences backend-owned.

**Regression anchors:** tests/plugin_ui_tests.cpp; plugin_ui_authority_inventory_test.py; terminal plugin_ui. Preserve invocation identity, deadline/cancel, geometry rejection, hidden IDs and late-reply behavior.

## 3. Active-run control, queues and ordered event/content delivery

**Retained interfaces:** TuiActiveRunQueues, TuiQueuedFollowUp, TuiRestoredQueuedMessage, TuiSubmitContext event/cancellation/steering callbacks, RuntimeEvent/EventEnvelope and subagent-launch notifications. TuiActiveRunQueues includes queue_steering, queue_follow_up, take_steering_messages, skip_active_steering, take_next_follow_up, mark_follow_up_started, restore_latest, run_nonblocking_command and finish, plus active_request_id (tui/runtime.h:25–33,88–103,178–195).

**Principal path:** app/tui/interactive_tui.cpp:create_active_run_queues binds an InteractiveRunQueue to the active session/request; app/interactive_run_queue.* owns backend queue handling. runtime_active_run_internal.cpp binds event request/correlation context, services controls and drains RuntimeEventQueue. app/tui/interactive_tui_support.cpp:run_queued_follow_ups_until_session_transition stops old-session follow-ups after a transition. Detailed transition evidence is in audit 03.

**Content exception:** preserve transcript/tool streaming and event reducers, TuiSubmitResult output/tool_timeline, local_command_result, conversation_output/conversation_tool_timeline, ordinary_turn_committed and ordinary_turn_request_ids. Preserve existing initial transcript/onboarding delivery, command-output settlement and historical user-turn picker/exact-text retrieval. History remains too large for copied canonical snapshots; the future history-object collection API is not selected here.

**Minimal constraints:** local-command events do not automatically become conversation history. The active-run state keeps request-segmented projection and only authorizes command conversation events from returned ordinary-turn IDs; missing authority must remain fail-closed. Queue restore returns a queued message to the local draft without implicitly submitting it. Old-session queue/event/content handling remains as today when ordinary state snapshots are removed. Keep submission/control delivery separate from mutable draft editing.

**Not exempt:** TuiSubmitResult::state_snapshot and duplicate passive context/statistics/model/agent/directory transfers still need their in-scope replacement. Removing that field must not discard output/control fields or trigger a redesign of the event stream.

**Regression anchors:** tests/events_tests.cpp:619–653 (follow-up /new stops remaining old-session work); tui_command_output_tests.cpp (local output versus transcript); tui_composer_rendering_tests.cpp (active run and session transition); app_rpc_queue_tests.cpp for related backend queue invariants, not proof of all TUI behavior. Terminal active_run, restore_followup and streaming_scroll.

## 4. Subagent selector, inspector and controls

**Retained interface:** TuiRuntimeOptions list_subagents, inspect_subagent, cancel_subagent and promote_subagent; app/tui/subagent_workspace.* adapters; agent::SubagentInspectorFrame and SubagentLiveMessage; runtime_subagent_workspace_internal.* consumer. Preserve the existing selector/control/status exchange as well as its history-bearing content, rather than extracting pending operation state into UnlockedData now.

**Verified binding:** interactive_tui.cpp:681–729 captures the current parent session ID and coordinator under Session access, then calls owner-bound list/inspect/cancel/promote adapters. inspect passes session_id, job_id and optional known_generation to coordinator->inspect; missing coordinator yields an explicit unavailable error. Selector values carry exact hidden job IDs; visible labels are not authority. Cancel/promote map to narrow outcomes rather than raw backend errors.

**Inspector content:** agent/subagent_inspector.h:44–59 exposes generation, not_modified, terminal, freeze_pending, unavailable, refresh_unavailable and messages. Messages are bounded path-free committed user/assistant text; the current MVP intentionally excludes tools/reasoning (32–38). Header caps include 1,000 entries, 1 MiB source-file bytes, 256 messages, 256 KiB total text and 64 KiB per item. The published content generation differs from an internal source/freeze epoch. Failed refresh can retain prior content with explicit refresh_unavailable; absence is not equivalent to a successful empty transcript.

**Boundary exception:** retain inspector frames and their operation/status fields, polling, hidden identities and control callbacks. No new canonical InspectorFrame or full-history copy. An inspected or selected child is not the effective primary agent that receives the user's next prompt; that ordinary parent-agent fact needs its own backend projection independently.

**Regression anchors:** tests/tui_selector_tests.cpp, tui_composer_rendering_tests.cpp, subagent_coordinator_tests.cpp, agent_loop_background_subagent_tests.cpp, session_subagent_job_history_tests.cpp; terminal subagent_workspace. Preserve parent ownership, stale generation behavior, eviction/unavailability, cancel/promote outcomes and existing message bounds. Do not claim the current inspector already implements the user's future richer history metadata model.

## 5. Branch-summary preparation and confirmation

**Retained interface:** app::BranchSummaryCoordinator prepare/confirm/cancel/snapshot, app::BranchSummarySnapshot -> tui::TuiBranchSummarySnapshot mapping, on_branch_summary_prepare, branch_summary_snapshot, on_branch_summary_confirm/cancel and the existing refresh-catalog integration. Sources: app/branch_summary_coordinator.h:192–240; app/tui/interactive_tui.cpp:353–374,798–802; interactive_tui_support.cpp:tui_branch_summary_snapshot; tui/runtime.cpp branch-summary handlers.

**Minimal constraints:** preparation is read-only and rejects a conflicting operation rather than queueing it. Confirmation is a separate action identified by generation and starts the operation deadline. Pre-confirmation cancellation is nonmutating; work that follows confirmation retains the existing lease/revalidation/append authority behavior. TUI state carries bounded labels, phase and fixed eligibility/failure/append-commit codes, not raw source payloads. Completion's refresh_required must not be dropped if ordinary session catalog plumbing changes.

**Boundary exception:** preserve the complete operation protocol and its snapshot/status fields. Do not migrate the status half merely because it is displayable while leaving its confirmation half awaiting design. The general session catalog can still migrate independently with a minimal adapter preserving the operation's existing refresh behavior.

**Regression anchors:** tests/branch_summary_coordinator_tests.cpp, tui_selector_tests.cpp and terminal branch_summary. Preserve explicit consent, generation matching, cancel before confirmation, stale-source/error codes and truthful append outcomes.

## 6. Image import/load, external editing and Mermaid rendering services

These are not all backend popups, but they are request/result interfaces distinct from passive state observation. Preserve their existing contracts rather than sweeping them into UnlockedData during the ordinary state refactor.

### Images

on_attach_image, on_paste_clipboard_image and on_load_image_attachment are bound in app/tui/interactive_tui.cpp:567–595 to session import/load and clipboard helpers. They return ImageAttachmentRef, optional imported reference, or LoadedImageAttachment through Result. runtime_submit_internal.cpp and runtime_actions_internal.cpp use them for attachment actions/previews; session/attachments.* retains storage validation, clipboard_image.* handles the external helper.

Maintain distinction between no clipboard image, failed import/load and a valid authorized reference. Preview visibility/width and local pending-attachment presentation are frontend preferences/state; attachment storage/authority and submitted references are backend concerns. Preserve service synchronization if required; do not add shared canonical appearance settings. Tests: clipboard_image_process_tests.cpp, clipboard_image_static_contract.cmake, agent_loop_core_tests.cpp attachment-load-before-provider checks, tui_composer_rendering_tests.cpp, tui_kitty_image_smoke.py and terminal main_paste_scrollback_attach.

### External editor

on_external_editor delegates to app/external_editor.cpp from the frontend action controller (interactive_tui.cpp:589–590; runtime_actions_internal.cpp:167–193). Result<optional<string>> represents edited content, cancellation/no replacement or error. This utility may receive a copy of the draft without making draft text part of backend Session state. Preserve original-draft-on-failure behavior and terminal restoration. Tests: tui_composer_tests.cpp and terminal main_editor_input. The file's current app location does not turn this frontend utility into agent backend authority.

### Mermaid

TuiMermaidRenderBridge exposes enqueue/cancel/drain and initial configuration fields (runtime.h:48–85). app/tui/mermaid_tui_bridge.cpp maps requests into MermaidRenderCoordinator; runtime_mermaid_internal.cpp drains completions into local rendering. Requests/completions carry identity and config_epoch. Bridge enqueue rejects shutdown/unavailable/disabled/stale-epoch requests, reports QueueFull when its try-lock fails, and maps coordinator outcomes (78–107). cancel checks identity/epoch; drain returns accepted text with correlation or empty failure text (110–149).

Retain the asynchronous service exchange and stale-completion checks. Its existing mutexes protect a deferred worker bridge, not a precedent for locking local display preferences. UI-only configuration/appearance remains frontend-owned in the target; any narrowly necessary parameters passed to the existing service are compatibility data, not a new backend settings domain. Tests: mermaid_render_coordinator_tests.cpp, tui_mermaid_tests.cpp and terminal mermaid. No replacement job/completion API is required now.

## 7. Precise exceptions for reconciliation and boundary checks

| Retain for this iteration | What must not be hidden by the exception |
|---|---|
| Permission/question resolver types, waiters, prompt views, rule actions and audit/cancel paths | New canonical pending-popups, automatic approvals or backend-owned local draft/appearance state |
| Plugin endpoint/capability, invocation/request IDs, replies and status/widget/modal delivery | Unrelated Session facts delivered as plugin status instead of their approved projections |
| Active-run queues, event/content reducers, attributed completion content and historical pickers/readers | Ordinary bulk runtime snapshot, duplicate current model/agent/directories/token/context facts |
| Subagent list/control/inspector interface and frame/status/content delivery | Treating focused child inspector as authoritative next-prompt primary-agent selection |
| Branch-summary prepare/confirm/cancel/status protocol and catalog-refresh adapter | Returning an unrelated full ordinary runtime state bundle solely to refresh passive metadata |
| Existing image/editor/Mermaid request-result services and necessary dependencies | UnlockedData-backed theme/cursor/layout/draft or other purely local preferences |

Do not impose a universal ban on backend headers, Snapshot names or TUI view payloads across these preserved interfaces. Scope guards to migrated ordinary-state boundaries and explicitly record existing exceptions by interface/path, not a blanket directory exemption. Allow minimal compatibility adapters where removing bulk state would otherwise break retained behavior; do not preserve two writable authorities for in-scope facts.

Some earlier audit tables recommended migrating pending/operation state or neutralizing all callback types. Those recommendations are superseded by this narrower scope. Audit 05 must use this retained-interface list, the corrected ordinary-state ownership rules and the user's normal-runtime design priority.

## 8. Verification and completion of this audit

No design question here requires selecting a new protocol now. The first backend-requested popup and future history collection API remain explicit future design work. Existing interactions should keep working through surrounding refactors; run focused tests only when implementation touches their integration, and relevant real terminal smokes before final handoff. A test filename is an anchor, not proof of exhaustive late-response/cancel/race coverage.

Before any proposed partial removal, obtain agreement on the specific feature and safe user-visible unsupported behavior; absence of an implementation must not masquerade as permission granted, a successful reply or an empty successful history result.

This audit identifies principal contracts, constraints and regression anchors without changing them. It intentionally does not prove all concurrency interleavings, invent missing metadata, or expand the current refactor into streaming/interaction redesign. No statuses have been advanced automatically.
