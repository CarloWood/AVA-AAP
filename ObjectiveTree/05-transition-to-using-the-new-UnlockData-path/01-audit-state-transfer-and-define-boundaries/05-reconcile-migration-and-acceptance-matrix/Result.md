# Audit 05: consolidated migration and acceptance plan

## Authority, evidence and completion meaning

This report reconciles sibling audits 01–04 and the subsequent user corrections. Its classifications and the revised /05/02–09 descriptions govern implementation where earlier reports made broader recommendations. The reports remain source inventories, not authorization to implement every architecture they originally suggested.

Source baseline: HEAD f4cd7d8e86c0c5a6f7da541c6ee6cc6741fc19ca plus the working tree inspected by the audits. The reference implementation includes the extracted frontend translation units. Rechecked the selected-agent type and ownership definitions during reconciliation. No repository edits, builds or tests were performed. Source references below are relative to REPOROOT; exact assertion details and call chains are in sibling Result.md files.

Completion means migrating ordinary observable backend facts and assigning UI-only responsibilities correctly, NOT eliminating all existing frontend/backend interaction. History/content and interactive request/reply protocols are explicitly retained. No whole-history copy, journal, new popup protocol, generic lifecycle framework or mandatory unregister redesign belongs to this objective. Preserve working features; removal/stubbing requires a separate explicit agreement.

## 1. Settled ownership and API rules

### B — ordinary backend facts

Use backend-owned canonical UnlockedData and frontend-owned projections following ModelSelection/SessionStats. Backend mutations publish at authoritative commit points; the frontend refreshes on the tui-thread. Initial and later values use the same path. Domains are cohesive backend facts, not a wrapped TuiRuntimeStateSnapshot. A small backend-derived observation is permissible when it is maintained by the same authoritative mutation path; do not create competing writable selections or recompute it separately in every TUI callback.

### U — frontend-only state

Themes, palettes, binding/help presentation, layout, previews, cursor appearance, image rendering preferences, local selector query/highlight/scroll and drafts are tui-thread-owned, in src/ava/tui, src/ava/app/tui or another explicitly tui-named file. No locks or UnlockedData for these local mutable values. Approved shared config/file utilities may load/save them; persistence does not make them backend state. TUI-specific app code can legitimately own them. Remove backend-command dependence on these values, not legitimate frontend calls merely because they live under app/tui.

Draft editing remains unknown to the agent backend until submitted as std::string. External-editor invocation and explicitly queued submissions are distinct operations, not a canonical draft mirror. Reasoning selection that changes invocation is B; whether/how reasoning text is drawn is U.

### D — deferred content and exchanges

Preserve existing history/transcript/tool streaming, historical picker/inspector content, permission/question popups, plugin UI, subagent controls/inspection, branch-summary confirmation and image/editor/Mermaid service exchanges. Their protocol-specific status and existing synchronization/dependencies are retained too. Do not split pending prompt/operation status into UnlockedData in anticipation of the user's future popup design. Existing worker queues may need locks; this is not a reason to lock frontend preferences.

### A — narrow action/control outcomes

For in-scope ordinary-state mutations, actions report success/unchanged/error and necessary feedback/control, not replacement UI state. Preserve existing backend permission and validation layers. Reuse current action dispatch where possible; do not design a universal command bus. Commit resulting canonical facts before returning success. A successful backend switch followed by a catalog warning must still expose the committed switch.

### I — immutable frontend dependencies

Version constants and genuine immutable services can remain initialization inputs. They do not require dirty notifications. Backend state must not be disguised as an immutable initial option that becomes stale.

## 2. Reference contracts retained, with minimal corrections

* Keep existing Data/FrontEnd registration and UnlockedData notification machinery. Final canonical addresses remain stable; Session replacement assigns values into registered destinations, and detached copies remain unregistered.
* Backend notifications only invalidate projections. They run synchronously after their own canonical mutex is released but possibly with other backend locks held. Do not read Session, reset views or call terminal APIs there.
* Projection snapshots and DisplayStrings are tui-thread-only. Keep dirty_.exchange(false) and independent output flags. Numeric/structured RefreshResult values are allowed where strings are unsuitable.
* A changed flag has one consumption owner. Preserve SessionStats as the consumer of the context-window numeric notification. For multiple widgets, one frontend refresh owner can distribute its result/current value; do not let widgets independently rely on consuming the same flag. Add per-consumer revisions only if a demonstrated consumer requirement cannot be met by this ownership rule.
* String references alias mutable projection caches; copy if a caller needs a historical value. Unknown, zero, unavailable, empty and unchanged must not be conflated.
* Use existing serialized action/completion boundaries for refresh/transition processing. Stable identity plus current backend values is the starting point, not a universal revision/transaction mechanism. Add a narrow generation check only where real asynchronous publication can otherwise update the wrong owner. Preserve existing statistics stale-publication checks.
* Backend read guards protect copies; expensive list/formatting work must not automatically remain under those guards. No per-render filesystem discovery or statistics recomputation.
* Correct demonstrably misleading comments: nullptr constructor registration is not currently supported by Data; absent reasoning prints "unknown"; late registration of stable pre-existing objects is supported; data_changed does not itself implement an event-loop wake. Verify idle observation against existing loop scheduling rather than assume an atomic flag schedules rendering.
* Keep normal runtime maintainability primary. Existing stop/join/enclosing lifetimes should satisfy notification lifetime where possible. A concrete teardown failure warrants a local fix/test, not speculative ownership infrastructure.

Neutral declarations/read bindings must remove Session implementation exposure for migrated facts. Session-specific registration stays in the backend/app adapter. Existing ModelSelection.h/SessionStats.h and FrontEnd.h expose Session/session_ts today; isolate that coupling incrementally without inventing a second registry. Deferred callback/type dependencies are explicit exceptions, not a reason to weaken general module boundaries.

## 3. Required backend domains and authoritative writers

Names below are descriptive, not mandated C++ names. Fields can share a domain when their backend mutation/coherence requirements justify it; do not split just to match existing UI labels.

| Domain / source evidence | Canonical authority and writers | Frontend consumer and required values | Goal / deletion / acceptance |
|---|---|---|---|
| Effective primary agent: Session.h InvocationInputs requested_primary_agent/selected_primary_agent | CLI request -> OpenContext -> Session construction; trust re-resolution/clearing; whole Session replacement; detached inputs copy. Session.cpp:458–464,603–611,695–717,914–915,1018–1033; command_trust.cpp; runtime_prompt_state.cpp:389–415,513–520 | A dedicated TuiFrontEnd projection exposes bounded requested/effective identity and availability/provenance as needed. No prompt text/source authority in display cache. Backend selected definition determines instructions/tool restrictions, not inspector focus. | /03: cover all real writers and next-Enter behavior. Absent selection is not a fabricated named agent. Requested-but-unavailable differs from no explicit selection; presentation can use truthful fallback labels. No new /agent command required. |
| Model/provider/execution variant/reasoning: Session::model_selection | Existing switch_model/set_reasoning paths, construction/replacement; prompt invocation reads this canonical data | Existing ModelSelection projection; selected model/provider labels, effective reasoning/variant information and numeric window. Scoped cycle belongs to backend model choices, not local selector highlight. | /02 preserves pattern, /03 covers consistent identity, /05 handles catalog/scope. Do not create parallel selected-model state. |
| Workspace root and current directory: InvocationInputs workspace_dir/current_dir | Directory resolution/open_at/construction; replacement_open_context and whole replacement; detached inputs copy. Session.cpp:105–106,198–202,606–607,910–911; app.cpp:188–196 | Distinct workspace-root and effective/current-directory values and display strings. Current TUI "workspace" prefers current_dir, so it is not sufficient evidence of root display. | /03: delete duplicated path fields in ordinary options/snapshots. Tests use different directories and verify tool/context behavior, not just labels. No new directory-navigation feature implied. |
| Session identity, mode and context-source count | Session store identity/mode/resolved prompt context; open/new/clear/resume/fork/branch and mode/trust/reload writers | Session presentation projection plus one shared local transition-effects helper; context count retains unknown vs zero | /03: delete corresponding ordinary initial/update copies and submit-result count. Preserve actual session transition effects; no whole-screen atomicity requirement without a consumer need. |
| Statistics including active history estimate, system prompt estimate and provider usage | SessionStats canonical plus backend computation from authorized history/usage/prompt; validated publication. interactive_tui_support.cpp:refresh_session_statistics currently triggered from TUI | Existing SessionStats projection including exact total and percentage; later strings can use retained metadata | /06: backend-owned trigger/publication shared across relevant mutations; delete refresh_session_statistics runtime callback and presentation-triggered recomputation. Exact versus compact changes, compaction, model-only percentage and no-op writes tested. |
| Trust/resource facts and safe overview inputs | Session trust decision and loaded context/freshness/resource availability, changed by trust/reload/open | Backend exposes safe facts; local overview composes these with existing model/agent projections and local theme/bindings | /04: remove ProjectTrustSnapshot/StartupOverviewSnapshot transport, retain filtering/bounds and truthful lower-bound counts. Permissions themselves remain backend authority; popup protocol remains D. |
| Model/command/session/workspace catalog facts, scope and committed todos | Existing registry/catalog coordinator/session read authority. Current adapters: command_palette_views.cpp, reasoning_selector.cpp, todos_for_session | Bounded domain projections for actual backend choices/availability, composed with local UI-only commands/hints. Query/highlight/sort/scroll and formatting are U. | /05: remove ordinary returned SelectListView bundles and previous-view arguments. Preserve generations/error/empty semantics and current history-backed picker exceptions. Do not use this slice to design general large-collection access. |

Additional source clarification: current agent::AgentDefinition (agent/agent_config.h:33–51) contains name, description, prompt, tool preset, max-tool iterations, hidden/provenance/path; it has no model/provider/variant member. The ordinary invocation explicitly obtains model/reasoning from Session::model_selection. Do not invent a primary-agent model override because subagent configuration has other concepts. If "variant" denotes a new execution setting beyond existing reasoning selection, identify its actual backend representation before adding fields; exposing existing semantics needs no new feature.

No verified runtime /agent picker/selection handler was found in audit 03. Publish actual current selection first; adding interactive agent selection is separate product work. Prompt tests explicitly preserve the resolved selected definition across file changes/model switching. Do not silently reinterpret selected-agent display as live reloading of its source file.

Git branch ownership was subsequently settled by the user: migrate the existing observed string to Session_aggregate_base::git_branch_ of type GitBranch_ts = UnlockedData<std::string>. Move its reader into app/runtime and change() the canonical value at the same initial setup/action/completion moments that currently construct runtime state snapshots. A frontend GitBranch projection copies the string and returns its DisplayString through git_branch_refresh(). Remove the old snapshot/options/sidebar transfer, without adding a watcher, timer or render-time filesystem work. This decision supersedes the earlier suggestion to keep branch observation frontend-owned.

## 4. Complete ordinary input disposition

This table closes all non-callback TuiRuntimeOptions fields and TuiRuntimeStateSnapshot fields listed in audit 02. "Remove" means remove the old transfer once replacement is live, not delete behavior.

| Fields / old transport | Classification / destination | Removal owner and test concern |
|---|---|---|
| mode, session_id, session_path, workspace | B session/directory projections | /03; startup and action completion use same source; test distinct root/current path and switches |
| git_branch | B observed string in Session GitBranch_ts; frontend string snapshot/display | /03; move reader into runtime, preserve observation moments/fallback, remove bulk/options branch transfer |
| context_source_count (options, snapshot and submit result) | B context metadata | /03 with /06; one source, known zero not unknown |
| app_version | I build/version value | /08 keep narrow immutable initialization or local build constant |
| initial_status / status | U startup diagnostics and A correlated action feedback; not a generic canonical status bag | /04,/06,/08; do not lose warnings or hide successful mutations |
| initial_transcript | D existing onboarding/transcript content | retain; no new history copy |
| slash_commands, slash_catalog_generation | B backend command facts + U frontend-only commands/help/hotkeys | /05; preserve generation/no-op semantics, local hints not backend binding ownership |
| file_references, workspace_catalog_generation | B current backend workspace catalog facts where bounded existing contract permits | /05; unknown/unavailable/empty/unchanged distinct, not per-render discovery |
| custom_themes | U theme discovery/catalog/palette | /04; tui-thread-local config helpers; no UnlockedData or locks |
| project_trust | B safe trust/resource facts | /04; remove transported ProjectTrustSnapshot, preserve authority/diagnostics |
| initial_todos / todos | B committed todo facts, local rendering | /05; keep existing hydration semantics and compatibility with retained live event input |
| show_images, image_width_cells, cursor | U appearance/application | /04; no canonical backend settings; preserve local invalidation/preview behavior |
| startup_overview | U composed view of B resource facts and U preferences | /04; remove transported overview/compact lines, preserve path-free bounds |
| mermaid_render.config_epoch/enabled and snapshot mermaid_config_epoch/enabled | U preferences plus existing D service configuration/correlation | /04 owns local preference changes; retain service-compatible parameters and cancellation, no new shared preferences object |
| mermaid_render.enqueue/cancel/drain | D asynchronous service | retain exact interface/necessary synchronization, audit04 |
| key_bindings | U reload/dispatch/help | /04; no backend canonical bindings; test preview/reload/dispatch coherence |

Current catalog delivered-generation is not applied-generation (command_palette.cpp:887–903 vs runtime_state.cpp:131–145). Migration must give the actual frontend observation owner consistent progress; don't discard a canonical update merely because an old snapshot builder constructed an undelivered packet. Local filtering/selection stays independent of canonical revisions. Preserve existing unavailable-row behavior and todo errors collapsing to empty until separately approved changes.

## 5. Complete callback and result disposition

The following accounts for every TuiRuntimeOptions callback (runtime.h:320–389). It distinguishes narrowing an in-scope action from redesigning a deferred interaction.

| Callback names | Decision and owner |
|---|---|
| refresh_session_statistics | Remove in /06; backend publisher owns computation, frontend projection owns observation. |
| create_active_run_queues, on_submit | D controls/content preserved. Strip only ordinary state_snapshot/count transfer after /03–06; do not redesign submission protocol. |
| on_attach_image, on_paste_clipboard_image, on_load_image_attachment, on_external_editor | D correlated services preserved; U draft/preview remains local. |
| on_toggle_mode, on_cycle_reasoning, on_cycle_model | A validated backend mutations plus B projection refresh. /03,/05,/08 remove returned current-state/bulk bundles; preserve success/unchanged/error/feedback. |
| on_reload_key_bindings, on_reload_display_settings, on_maybe_reload_display_settings | U local configuration/reload ownership in /04, not backend state requests. Preserve explicit applied/unchanged/error and last-good behavior; no generation protocol needed for single-thread-local state. |
| model_selector_view, scoped_model_selector_view | D interactive /models and /scoped-models lists, explicitly deferred by subsequent user review. Preserve existing list delivery/actions; no scope snapshot/change flag/refresh accessor or eager model catalog. Current model/provider/reasoning display still uses existing B projection. |
| reasoning_selector_view, session_selector_view | Preserve no-choice/default/current/disabled distinctions. Do not infer approval of a popup redesign from passive-state migration; any popup-specific restructuring requires its own reviewed scope. Ordinary committed selection/identity facts can refresh independently. |
| list_subagents, inspect_subagent, cancel_subagent, promote_subagent | D complete existing exchange/status/content preserved; inspector focus never defines primary selection. |
| on_open_fork_user_turn_selector, on_open_copy_user_turn_selector, on_read_user_turn_text | D historical-content/picker retrieval preserved; copied draft remains U. |
| on_session_selector_sort_cycle, on_session_selector_named_filter_toggle, on_session_selector_path_display_toggle, on_session_selector_archived_filter_toggle, on_session_selector_label_timestamp_toggle | U selector controls in /05; remove callback round-trips that simply rebuild backend-owned views. Backend catalog facts remain validated. |
| on_session_selector_archive, on_session_selector_unarchive | A persistence mutation + B catalog update in /05, no replacement SelectListView result. |
| on_session_selector_branch_parent, on_session_selector_branch_child, on_session_selected | A navigation + B committed session/directory facts in /03,/05; shared local transition handling. |
| on_branch_summary_prepare, branch_summary_snapshot, on_branch_summary_confirm, on_branch_summary_cancel, on_branch_summary_refresh_catalog | D confirmation/status exchange preserved. Minimal /05 catalog adapter may be needed; do not migrate operation state or remove needed completion refresh. |
| remember_permission_rule | D permission/rule-authority request/result preserved. |
| on_settings_selected | Split in /04: B trust/backend changes through existing validated actions; U theme/render/binding/close-view actions stay local; no bulk snapshot return. |
| on_model_selected, on_reasoning_selected | A backend mutations + existing B projection; /03,/05,/08 remove bulk snapshot result and duplicated refresh work. |
| on_fork_user_turn_selected | A existing backend fork action and B committed transition refresh; preserve D historical picker. Ancillary catalog failure after mutation remains warning, not stale old identity. |
| on_scoped_model_toggled, on_scoped_model_enable_all, on_scoped_model_clear_all, on_scoped_model_toggle_provider, on_scoped_model_reorder | D existing scoped-model popup actions and returned views retained. No new scope projection/publication path in this refactor; backend canonical optional remains unchanged. |
| on_scoped_model_save | D existing popup persistence/feedback retained; no frontend scope subscription added. |
| on_before_tui_shutdown | D existing cleanup callback preserved; no automatic lifecycle framework. |

Nested result contracts:

* TuiSubmitResult.quit, ordinary_turn_committed, ordinary_turn_request_ids, output, tool_timeline, local_command_result, conversation_output and conversation_tool_timeline stay with the existing D/A submission/content path. context_source_count and state_snapshot are ordinary-state removal targets. Keep local-command/conversation authorization distinctions.
* TuiKeyBindingReloadResult.key_bindings/state ceases to be backend delivery: reload and apply bindings locally; no bulk state member.
* TuiActiveRunQueues retains active_request_id and all queue/take/skip/mark/restore/nonblocking-command/finish functions. TuiSubmitContext retains existing request identity, prompt resolvers, event sink, cancel callback, followup/steering operations, launch sink, plugin endpoint and attachments. No wholesale neutralization of those deferred types.
* TuiBranchSummarySnapshot and SubagentInspectorFrame remain explicitly allowed within their current operation protocols, including statuses and historical messages. Do not claim they are already migrated because the broad runtime snapshot disappears.

Current implementations and consumers are indexed in audit02 sections C/D and audit04. Changes must follow those actual callsites, not only grep for Snapshot names.

## 6. Exact retained boundaries and compatibility verification

| Retained boundary | Principal files/types | Preserve / regression anchors |
|---|---|---|
| Permission/question | runtime_prompts_internal.*, PermissionResolver/Prompt/Decision, QuestionResolver/Prompt/Answer, rule action | Correlation, waiting, Deny/interruption, separate queue precedence, rule persistence/authority. tui_permission_tests.cpp; main_permission_flow/main_question_flow. |
| Plugin invocation UI | runtime_plugin_ui.*, runtime_plugin_ui_internal.*, app invocation capability adapter | Binding/deadline/weak token/request identity, bounds and geometry-safe host chrome, cancel/reply. plugin_ui_tests.cpp; plugin_ui_authority_inventory_test.py; plugin_ui smoke. |
| Active-run/history/tool content | runtime_active_run_internal.*, runtime_state event queue, event reducers, interactive_run_queue.*, TuiSubmitContext/Result content fields | Request-authorized conversation vs local command, ordering, followup restore and old-session discard. events_tests.cpp; tui_command_output_tests.cpp; composer transition tests; active_run/restore_followup/streaming_scroll. |
| Historical pickers/readers | command_palette_views.cpp user-turn functions; interactive_tui callbacks | Source-authorized IDs, exact text reads, fork eligibility; no canonical history vector. Existing selector/fork/copy tests. |
| Subagent workspace | app/tui/subagent_workspace.*, runtime_subagent_workspace_internal.*, SubagentInspectorFrame | Parent ownership, hidden job identity, generation/not_modified/unavailable/eviction and content caps; preserve control/status as well as messages. subagent_coordinator_tests.cpp, tui_selector_tests.cpp; subagent_workspace. |
| Branch summary | branch_summary_coordinator.*, TuiBranchSummarySnapshot and callbacks | Separate generation-bound confirmation, nonmutating pre-confirm cancel, truthful append outcomes/catalog refresh. branch_summary_coordinator_tests.cpp; branch_summary smoke. |
| Images/editor/Mermaid | session attachment/clipboard helpers, external_editor.cpp, mermaid_tui_bridge.*, runtime_mermaid_internal.* | Authorized imports, no-image vs error, draft preservation, identity/config-epoch rejection, service bounds. clipboard/image, composer/editor, mermaid tests and terminal scenarios. |

Existing locks and backend includes needed by these paths are retained exceptions, not new desired dependencies. Guards should name interfaces/paths and prohibit expansion into ordinary facts, not exempt all of runtime.h or all app/tui. A broad no-backend-header check would contradict the agreed incremental scope.

## 7. Execution order and removal ownership

Each implementation goal leaves the tree buildable and verifies its own changed boundary. Do not leave two writable authorities. A temporary adapter may read the new canonical source to serve unmigrated consumers, but cannot maintain its own competing state. Record every temporary adapter with a removal owner below.

1. **/05/02 — minimal reusable boundary.** Preserve model/stats behavior; isolate Session-specific bindings; establish single-consumer output ownership and ordinary action-completion refresh seam. Correct reference-contract discrepancies. Keep existing lifetime model unless a concrete failure requires a local guard. No popup/cancellation API project.
2. **/05/03 — first missing backend facts.** Effective agent, distinct directories, session identity/mode/context and local transition trigger. Trace all real writers, preserve selection resolution semantics. Use the existing transition reset body once; keep history and prompt authority logic intact. Remove migrated initial/snapshot fields when callsites move.
3. **/05/04 — frontend preferences versus backend facts.** Move UI-only ownership/load/save/reload into tui-specific owners; no backend preference copies. Project only safe backend trust/resource facts and locally compose overview. Replace broad reload/snapshot coupling and direct backend theme management. Keep asynchronous service APIs unchanged.
4. **/05/05 — ordinary choices/catalogs/todos.** Replace ordinary backend-created views with projections/local controls, preserve bounded generation semantics and validation. Retain historical pickers/subagent/branch interfaces explicitly. Prefer existing bounded data contracts; do not introduce generalized list/history infrastructure.
5. **/05/06 — normal run facts/statistics.** Centralize backend computation/publication at actual relevant mutations, remove UI computation callbacks and duplicate completion work. Preserve submitted/streaming content. /exit optimization follows normal path simplification, never drives the API.
6. **/05/07 — compatibility, not interaction migration.** Verify retained protocols after surrounding changes and make only necessary adapters. Its historical directory name is not authorization to implement pending-prompt/operation projections.
7. **/05/08 — final ordinary-path deletion.** Remove TuiRuntimeStateSnapshot, state_snapshot, reload.state and obsolete builders/apply-data helpers once domains and local transition effects have replacements. Retain shared local transition helper and all enumerated D paths. Runtime options no longer duplicate ordinary backend current state.
8. **/05/09 — scoped verification.** Reconcile every matrix row with final implementation; enforce B/U/D/I distinctions, run full and terminal gates, document exact exceptions and extension pattern.

Adapter deadlines: old ordinary snapshot fields removed in the domain's slice; remaining aggregate removed by /08. Local preferences must not use backend snapshots after /04. Ordinary selector-return adapters removed by /05 or final shared-callsite cleanup /08. Branch/subagent/history/popup adapters are preserved interfaces, NOT temporary bridges due for deletion /08. Shared model/stats projection header isolation may finish /08 but must not introduce new backend coupling in intermediate slices.

## 8. Acceptance matrix

| Area | Minimum proof before handoff |
|---|---|
| Reference projection contract | initial/unknown values, identical writes, independent string/numeric changes, changed-flag consumption order, update during refresh with deterministic synchronization, stable registered destination replacement; existing app_runtime_model and tui_selector checks retained |
| Effective agent | requested/effective/default/unavailable distinction; construction/trust/replacement writers publish; next normal request uses selected instructions/tool visibility. app_runtime_prompt_tests.cpp:210–254 and trust-selection cases around 404–550; add frontend projection tests, no invented model override/name |
| Directories | different root/current values, correct defaults/open_at/replacement/detach, actual context/tool path usage. app_runtime_startup_tests.cpp:388–426 plus new projection/render assertions |
| Local state | theme/binding/render/draft changes do not mutate backend canonical preferences; real reload/persistence/preview failures remain correct; file location and ownership checks, not just UI output |
| Session transitions | one shared reset; normal failed/successful/same-session/fork-with-warning paths; queue-bound old-session results discarded; existing changed_session/session_transition tests and events_tests.cpp followup case retained |
| Catalogs | known empty vs omitted/unchanged/unavailable, generation/no-op work avoidance, selection preservation/invalidation, validated IDs, local hotkeys/filtering; app_runtime_catalog_tests.cpp, session_title_coordinator_tests.cpp, tui_selector_tests.cpp |
| Statistics | exact vs compact text, coherent active+prompt totals, provider usage retained, compaction/replacement/stale compute rejection, model-window-only update; publication/work-count test proves rendering doesn't recompute stats and /exit doesn't duplicate it |
| Deferred interfaces | focused existing tests above pass when integration changes; no stubs or silent authorization/content changes; explicit exception inventory matches actual source |
| Boundary | no in-scope backend current-state copies in options/action bundles; no backend-owned appearance/draft domains; no new UnlockedData protocol state; no generic Snapshot word ban or blanket deferred-directory exemption |

Run builds using existing canonical debug-enabled configuration; do not configure a new tree as part of this planning objective. During implementation use aap-build -- --quiet, regenerate print-members first if its generated definitions fail, and run focused CTest filters discovered from the actual test registry for each slice. After all slices, run the full credential-free suite, applicable module dependency/debug generation/coverage checks and the complete opt-in terminal wave. Existing documented command is `AVA_TUI_TMUX_SMOKE=1 scripts/run-tests.sh --build-dir "$BUILDDIR" --jobs 23 -R '^ava_tui\.tmux_smoke_'`; confirm build directory and harness configuration then. Relevant scenarios include startup_overview, main_startup_trust_keybinds, main_models_selectors, main_slash_completions, main_session_mgmt, theme_env/theme_persisted/nested_settings_preview and all retained interaction/content scenarios above. Record actual passed/skipped/blocked results and exact rerun instructions; no skipped test counted as passing. Format changed C++ and run git diff --check.

If AVA_DEBUG_OUTPUT_DIR points to an unwritable/stale location, report the environment issue and use an explicitly permitted local output directory; do not confuse it with a product regression. No test run is claimed by this audit.

## 9. Decisions not to invent during implementation

Remaining local verification questions are assigned, not guessed: any non-reasoning variant field actually present (/03), exact bounded ordinary catalog payloads (/05), and idle refresh scheduling under existing runtime (/02). Git branch backend ownership is settled as above. The later declaration proposal/review gate still blocks implementation of unapproved types, including unresolved catalog choices. If investigation reveals a large collection or inseparable interaction requiring a new design, return that specific choice to the user instead of extending this refactor.

Do not invent agent switching commands, new agent/model precedence, default agent names, backend draft storage, general revision buses or new history/popup APIs. Select honest local formatting for existing facts, preserving absence distinctions. Require user agreement before removing features or substituting unimplemented endpoints.

The outcome is a maintainable normal-runtime state path with clear ownership, central backend publication, reusable frontend refresh and precise retained compatibility boundaries. Shutdown exceptions are tested as needed but do not dictate the architecture.
