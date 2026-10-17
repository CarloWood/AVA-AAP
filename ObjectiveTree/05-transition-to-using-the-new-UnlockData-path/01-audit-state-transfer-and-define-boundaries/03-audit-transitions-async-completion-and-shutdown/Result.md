# Audit 03: normal-runtime ordering and transition compatibility

## Design priority and scope

The user's direction is authoritative: design the ordinary runtime path for clarity, maintainability and no duplicated state/publication code. Shutdown and /exit are exceptional consumers of that design, not reasons to choose a generalized lifecycle framework or reshape the frontend API. This report inventories obligations to preserve, separating normal-operation ordering from local cleanup concerns. It does not prescribe epochs, an event bus, unregister machinery or transactions unless later implementation demonstrates a concrete need.

UnlockedData is for backend state that is or may become visible. TUI appearance, local interaction and draft state remain tui-thread-owned in tui-specific files, without locks or UnlockedData. Session history/streaming and associated historical-content delivery remain on the existing path for now. No collection snapshots or replacement journal are proposed.

Read-only investigation of the current working tree. No code modifications, builds or tests were performed. Paths are relative to REPOROOT and line numbers are navigation aids. Explore collected locations; the primary planner inspected the transition implementation, worker settlement, prompt dispatch/selection, detached state, shutdown epilogue and primary-agent tests before drawing the conclusions below.

## 1. The ordinary runtime path to preserve

### Editing, submission and return to editing

Draft text is local frontend state. A submitted string reaches RuntimeSubmitController and RuntimeActiveRunController; the worker receives the submitted value and a TuiSubmitContext containing request identity, interaction callbacks and attachment references. `runtime_active_run_internal.cpp:634–651` launches the worker with make_async("tui_submit", ...). The TUI continues servicing input, prompts, plugin UI, event projection and rendering while the future runs (`733–863`). This existing concurrency is not justification for making the editable draft a shared backend object.

The app on_submit binding calls handle_interactive_submission and may process queued follow-ups (`app/tui/interactive_tui.cpp:484–565`). Ordinary text proceeds through with_provider_runtime and run_prompt (`app/interactive.cpp:49–95` and submission handling); backend commands instead run their command handler. Provider selection uses the Session's model provider and backend credentials/catalog. The model invocation is not selected from displayed TUI text.

`run_prompt` inspects/adopts run admission, rejects a different active request or exclusive maintenance reservation, and delegates to run_admitted_prompt (`app/runtime_prompt.cpp:77–135`). Backend dispatch/admission remains authoritative regardless of how current values are displayed.

At worker completion, the TUI clears/unblocks plugin submission presentation, obtains the result, records authorized ordinary-turn request IDs and currently applies its runtime snapshot before accepting another prompt (`runtime_active_run_internal.cpp:864–879`). It finishes the active queues, discards old-session run state when a transition was detected, otherwise drains remaining events and merges local-command tools (`880–919`). Completion settlement and rendering follow.

**Normal-operation requirement:** after a backend action changes the target session/agent/model/directories, subsequent submission must use the actual committed backend values. Relevant frontend status must be refreshed at the action/completion boundary rather than relying solely on whichever widget happens to render first. This can use the existing serialized action flow plus one shared refresh/transition helper; there is no evidence here that every domain needs a new revision protocol. Backend validation still decides what Enter dispatches.

### Where duplicated responsibilities currently exist

Backend operations update real state; application callbacks then rebuild broad state snapshots; runtime snapshot application copies state into several presentation owners and requests statistics recomputation; generic completion requests statistics again. Centralizing authoritative backend publication and frontend consumption removes this duplication. It should not be replaced with independent refresh logic at every keybinding or exit branch.

## 2. Current active agent: actual authority, not a UI label

The relevant backend values are `InvocationInputs::requested_primary_agent` and `selected_primary_agent` (`app/runtime/Session.h:64–65`, accessors around 327–328). Requested name is selection intent; selected AgentDefinition is the resolved effective definition. They are not interchangeable, particularly when project trust changes or a requested definition becomes unavailable.

Verified paths:

* CLI `--agent` sets requested_primary_agent (`app/command_line.cpp:249–253`); app passes it into OpenContext (`app.cpp:205`).
* Session construction resolves that request against loaded primary agents under current trust (`Session.cpp:458–464`) and stores requested and resolved values (`606–611`). A selected read-only preset also restricts tool visibility (`603` onward).
* `resolve_runtime_primary_agent` returns no selected definition when no name was requested; AllowUnavailable can also return no definition when the requested name is absent (`runtime_prompt_state.cpp:389–415`). Therefore null selected does not justify displaying a fabricated named agent. The desired display for default versus unavailable-requested needs an explicit choice based on these distinct facts.
* Prompt construction includes the permitted selected agent's name/instructions (`runtime_prompt_state.cpp:513–520`), alongside normal prompt/context composition. `select_runtime_prompt_state` reads the current selected definition and directories, not the draft or a frontend cache (`runtime_prompt.cpp:45–65`).
* Trust handling can re-resolve or clear a project agent and update tool visibility (`command_trust.cpp:127–173,217–227`; `Session::apply_trust_prompt_state_and_refresh`, Session.cpp:1018–1033). These are normal backend mutation paths that a future ActiveAgent projection must observe.
* Reload uses the resolved selected definition when rebuilding prompt state (`command_reload.cpp:138–146`). Changing an agent file does not imply automatic replacement of the selected definition. The inspected prompt tests explicitly require preserving the already-resolved definition across prompt reload and model switching.
* `replacement_open_context` carries requested_primary_agent and permits unavailability (`Session.cpp:914–915`), so opening a replacement may resolve differently under its current resource/trust context.
* `create_detached_state` copies InvocationInputs, including selection and directories (`Session.cpp:695–712`); detached work must not be mistaken for the next parent submission target.

A source search for assignments to requested_primary_agent/selected_primary_agent and literal `/agent` in app sources located CLI/construction/trust paths, not a verified runtime `/agent` selection handler. Do not invent that command or assume a picker exists. The mutation list is source-backed for explicit field assignments and replacement; it is not a formal proof against all indirect writes through mutable invocation_inputs().

Model/provider/reasoning remain distinct canonical state: invocation construction uses Session model_selection for model_id, reasoning, API family and capabilities (`runtime_prompt.cpp:447–468`), and with_provider_runtime uses its provider ID. The inspected primary-agent prompt path does not show an AgentDefinition model override being used for the ordinary parent request. Do not confuse subagent-definition model resolution with the current primary model or assert a universal override precedence without tracing it.

**Implementation implication:** expose bounded effective agent identity/status/provenance facts from the backend owner; do not copy an entire AgentDefinition including prompt text merely to show a name. Update that canonical fact at the actual selection/trust/replacement publication point, not by reading a selector label. Keep current backend execution semantics; a new runtime agent-switch feature is not implied by this audit.

## 3. Directories: workspace root differs from displayed effective directory

InvocationInputs owns workspace_dir and current_dir (`Session.h:58–59`, accessors 324–325). OpenContext carries both. `Session.cpp:105–106` defaults missing workspace to process cwd and missing current directory to workspace. `open_at` explicitly sets both (`198–202`); construction stores resolved values (`606–607`). CLI setup derives workspace/current directory from cwd and an optional requested current directory, enforcing containment (`app.cpp:188–196`). Replacement context preserves both (`Session.cpp:910–911`); detached state copies both with InvocationInputs.

These are backend facts: context loading, tool context, attachments, resource lookup and permissions use them. `runtime_prompt_state.cpp:418–432` passes both to context loading; invocation setup uses them in runtime_prompt.cpp:449–450. They must not be modeled as frontend path labels or confused with process-global cwd.

The current TUI SessionPresentation chooses current_dir when nonempty and calls that displayed value workspace (`interactive_tui.cpp:250–259`), while separately retaining workspace_dir for Git branch observation. Thus the existing `workspace` display does **not establish that workspace root is exposed distinctly**. The new Directory projection should distinguish backend workspace root from backend current/effective directory. Formatting/abbreviation belongs to the frontend; canonical paths retain backend meaning.

Inspected explicit assignments occur during context/open construction; no standalone TUI directory-switch writer was established. Future implementation should recheck mutations through whole InvocationInputs/Session assignment, not assume only member assignments matter. Do not add a filesystem navigation feature just to populate the projection.

## 4. Session transition side effects are local consequences, not backend copies

Current detection is `snapshot.session_id != state.session_id` in `apply_runtime_state_snapshot_with_presentation_transition` (`tui/runtime_state.cpp:209–261`). On change it performs the following before applying the new ordinary state:

| Local owner | Current effects | Migration obligation |
|---|---|---|
| Transcript search | reset query, projection/cache, saved viewport/generation/dimensions and selection if open (`runtime_transcript_search_internal.cpp:957–973`) | Preserve existing history transition behavior; do not redesign storage. |
| Subagent workspace | reset calls close_to_parent (`runtime_subagent_workspace_internal.cpp:147–150`) | Leave old inspected-job view; preserve historical inspector transport. |
| Draft/editor | reset draft, selection, input-history/index, input buffer, jump/palette/completion/scroll/Escape state (`runtime_draft_internal.cpp:28–41`) | Entirely frontend-local. No canonical draft or backend reset payload. |
| Renderer | clear selection and pending shifts, transcript scroll, detached-output/sidebar/deferred viewport state, layout/row caches, wheel governor and position indicator (`runtime_render_internal.cpp:286–303`) | Local rendering consequence, executed by tui-thread. |
| Composer local presentation | clear transcript, increment generation; clear command output, tool indexes/eviction identities, queued-message view, permission/question views, sidebar/drawer and input/feedback state | Keep one shared transition helper; hiding a prompt is not proof its backend request was resolved. |
| Overview/selector | close expanded overview and any active select list; clear active selector | Local presentation. Same-session refresh instead rebuilds open overview while preserving query and clamping selection (`runtime_state.cpp:184–206`). |
| Sidebar | clear activity, modified files, todos and session-entry count | Hydrate new backend facts through projections; retain history-side existing behavior. |

`RuntimePresentationState::apply_runtime_state_snapshot` additionally calls command_session_grants.clear_for_session_transition and clears pending image-attachment references/previews if it reports a transition (`runtime_state.cpp:117–126`). Backend authorization stays authoritative; presentation code must neither retain old-session attachment/grant state nor manufacture permissions.

The current helper is already a useful centralization seam. Removing TuiRuntimeStateSnapshot should separate its trigger (observe a committed session transition) from its local reset body, not duplicate that body across /new, /clear, resume, fork, branch navigation and completion handlers.

The inspected draft reset is not a complete proof of prompt-stash policy: stash storage and restore rules need targeted verification before moving its integration. Preserve existing behavior and tests; do not infer that every draft-related object is reset simply because input was cleared.

## 5. Action success, failure and publication order

Session selector and parent/child branch navigation open a replacement then call Session::replace_with; already-open selection yields a distinct status, and unavailable targets return errors (`interactive_tui.cpp:310–352,995–1014`). Replacement performs value assignment into the stable current Session while retaining registered canonical addresses (`Session.cpp:784–800`). Detached construction creates unregistered ModelSelection/SessionStats wrappers (`713–717`). These facts support the existing pattern; no new registry is required.

Fork-from explicitly checks actual before/after identity/path. It refuses to report an opened session when authority did not change. After a committed switch, catalog refresh failure is a soft warning, not permission to retain old presentation (`interactive_tui.cpp:913–979`). Preserve this principle when replacing snapshot results with narrow action outcomes and canonical refresh: **failure of ancillary presentation/catalog work must not conceal a successful backend mutation**.

Trust changes can update prompt, effective agent and execution visibility together. Model changes update model/reasoning and rebuilt prompt state. Those ordinary operations determine the necessary grouping of backend facts. No observed code establishes atomic screen-wide refresh, and it should not be required without a concrete consumer need. At minimum avoid showing a new selected agent as if it is effective before backend publication is committed.

UI-only theme/settings reload is different: own and apply it locally on the tui-thread, preserving last-good settings, preview and applied/unchanged/error semantics. It does not belong in the backend session transition or locking protocol merely because the old bulk snapshot carried it.

## 6. Queued work and old-session results: preserve current boundaries

`run_queued_follow_ups_until_session_transition` (`app/tui/interactive_tui_support.cpp:657–708`) first checks whether the session already changed. It stops on quit/cancel/no pending follow-up, marks a follow-up started before running it, and checks session identity afterward. When it changes, it substitutes the transition result while preserving accumulated quit/session-tree/ordinary-turn flags, and returns without running further old-session follow-ups.

The TUI's worker-completion path applies the returned state, finishes the old bound queue, then discards prior-session event/run state when session_changed (`runtime_active_run_internal.cpp:864–897`). Otherwise it drains/project events normally. Event callbacks bind request/correlation context (`569–605`); command event projection is gated by authorized ordinary-turn IDs. These are ordinary action-completion constraints, not shutdown special cases. Preserve them while changing only in-scope state delivery.

Evidence does not establish a blanket session-ID comparison on every event application: drain_events processes the existing queue/projection contracts, and the explicit transition discard occurs at completion. Do not invent stronger guarantees or replace this retained streaming machinery in the state migration. Audit 04 can document its exact retained interface.

Latest-value dirtiness suffices for display values, but not automatically for a required reset or command acknowledgment. In the current serialized action path, one shared transition check after completion may be sufficient. If implementation permits multiple unseen transitions (for example A -> B -> A before UI observation), session-ID equality alone would miss intermediate effects; demonstrate that scenario is possible before adding general transition generations or queues. Rapid sequential switches that each return to the action boundary should be tested with the ordinary helper first.

## 7. Locking and notification constraints for migrated facts

Keep backend mutation/access locking separate from tui-thread-local effects. Session guards often enclose canonical changes; as audit 01 established, data_changed runs after the individual data lock is released but can still run while Session/other locks remain held. The callback should mark dirty, not reacquire backend state or reset UI objects. Frontend projection refresh occurs later on the tui-thread.

Session replacement parks old resources and releases locks before potentially blocking detached-parent cleanup (`Session.cpp:784 onward,865–873`). Preserve this backend discipline without extending canonical critical sections across formatting, joins or filesystem work. One-object coherence is supplied by UnlockedData; shared identity coherence between active agent, directories and other facts should be specified only where normal consumers need it.

Canonical statistics computation already captures session identity, append target and prompt, computes outside Session locking, and checks these before publishing (`interactive_tui_support.cpp:206–233`). Moving its trigger into backend mutation/lifecycle ownership should preserve those stale-result checks, not hold a giant lock across history loading. History loading itself remains existing implementation, not a new history-access design.

## 8. /exit duplication: a local cleanup issue

The two observed calls are explained by the generic completion path:

1. on_submit returns state_snapshot even for quit (`interactive_tui.cpp:556–565`). After future.get, applying it reaches RuntimePresentationState::apply_runtime_state_snapshot, which calls refresh_token_status.
2. Generic completion later calls presentation_state_.refresh_token_status again, renders, then checks close_after_submit and result.quit (`runtime_active_run_internal.cpp:1037 onward; precise lines may shift`).

Both callbacks invoke refresh_session_statistics and publish via change(), even when statistics are identical. Neither call is a required /exit persistence operation. This is evidence of duplicated computation ownership, not a need for a shutdown-oriented frontend API.

First establish a single backend statistics publisher for normal operation and a single frontend consumption path. Then let quit skip unnecessary presentation refresh/render where safe, without bypassing actual result settlement, queue completion, pending decisions or resource cleanup. Do not apply a broad early-return patch during this audit.

## 9. Shutdown compatibility notes — secondary, not architecture drivers

RuntimeBeforeShutdownGuard is created near the beginning of run_interactive_composer, before the non-TTY early return (`runtime.cpp:95–126,215–219`). invoke is idempotent, marks itself invoked before calling the callback, catches callback exceptions and reports failure; its destructor invokes it as a fallback.

Normal loop exit shuts down local Mermaid presentation and plugin UI, invokes the callback, then returns an exit status (`runtime.cpp:2597–2601`). The app callback shuts down Mermaid bridge and branch-summary coordinator, and the app also calls their shutdown after frontend.run returns (`interactive_tui.cpp:1016–1023`). The callback comment requires use while curses is active, but exceptional unwinding order of every terminal owner was not fully proved in this audit.

Active-run code gets the submit future after cancellation/loop completion before ordinary settlement, so a loop break is not evidence that the worker was abandoned. This does not prove all plugin/subagent/coordinator producers have stopped on every exceptional exit.

For registered raw frontend pointers, the minimal lifetime requirement is still real: no publisher may notify after the frontend is destroyed. Prefer using the existing stop/join and enclosing lifetime structure to satisfy that requirement. Do not introduce unregister/refcount/epoch machinery solely for hypothetical shutdown races. If an actual path violates the rule, fix that path or add the smallest common cleanup guard and test it. Suppressed shutdown callback exceptions are not proof of quiescence; document failures accurately.

## 10. Existing tests and targeted verification plan

Existing source evidence (not tests executed here):

* tests/tui_composer_rendering_tests.cpp dispatches changed_session and session_transition cases around 1144–1157, covering presentation reset and prior-session run/event/queue discard. Existing history assertions must stay intact.
* tests/events_tests.cpp:619–653 exercises /new as a queued follow-up: later old-session follow-up is not executed, new result replaces old content, and control flags survive.
* tests/app_runtime_startup_tests.cpp:388–426 checks distinct current/workspace directories through replacement; create_at and detached/process-scope tests cover related construction/lifetime behavior.
* tests/app_runtime_prompt_tests.cpp:210–254 verifies requested selected-primary instructions, retaining the resolved definition despite file edits and model switches. Around 404–550, trust downgrade/re-resolution and unavailable replacement retain selection intent while removing unsafe effective project instructions. tests/agent_loop_subagent_tests.cpp:1180–1218 covers primary-agent eligibility/trust resolution.

Migration tests should favor normal runtime behavior:

| Scenario | Deterministic assertion |
|---|---|
| Startup, selected agent and directories | Projection reports actual effective agent/request status and distinct workspace root/current directory; absent selected agent is not fabricated. |
| Trust/model changes then Enter | Backend dispatch uses committed effective selection/model; updated labels derive from canonical facts, without duplicate agent instructions or UI-owned authority. |
| Session switch/new/clear/resume/fork/branch | One shared local transition helper resets appropriate presentation, grants/attachments and existing history views; subsequent submission targets new authority. |
| Failed switch vs committed switch plus ancillary failure | Failed mutation preserves old target; committed mutation remains observable with warning even if catalog refresh fails. |
| Queued follow-up changes session | Remaining old-session queue work does not execute; retained event/content path cannot repopulate new session with old state. |
| Rapid sequential switches and delayed completion | Serialize with explicit latches/barriers, not sleeps; refresh committed facts at action boundaries and preserve existing stale-result rejection. Add generations only if demonstrated concurrency requires them. |
| Local draft/settings changes | No backend canonical write or permission change until explicit submission/action; local settings stay single-thread-owned. |
| Normal statistics mutations and repeated reads | One authoritative publication trigger; rendering/refreshing does not reload history or recompute backend stats. |
| Exit/cancel and errors (secondary) | Required settlement/joins still occur; no duplicate /exit statistics recomputation; no notification after frontend lifetime; terminal restoration remains intact. |

No dedicated all-exit-path publisher-quiescence test was established by this inspection. No dynamic stress run or proof of every indirect InvocationInputs writer is claimed. These gaps should become narrowly scoped tests where implementation actually changes the relevant path, not blockers demanding a speculative lifecycle redesign.

## 11. Handoff to audits 04/05

Normal-runtime design should centralize backend publication and frontend action-boundary refresh, retaining the existing projection model. Prioritize effective active agent and distinct directories alongside model/stats. Keep TUI-only controls local and history transport unchanged. Reuse a shared frontend transition-effects helper rather than distributing reset code across commands.

Audit 04 should record exact existing request/decision/streaming correlation contracts and retained content paths, not design a new history stream. Audit 05 should choose minimal coherence and acknowledgment semantics from demonstrated normal-runtime needs, preserve the concrete compatibility rules above, and treat shutdown remedies and /exit optimization as local follow-up work. Neither this audit nor the earlier inventory authorizes general-purpose lifecycle infrastructure or redundant per-action state copies.
