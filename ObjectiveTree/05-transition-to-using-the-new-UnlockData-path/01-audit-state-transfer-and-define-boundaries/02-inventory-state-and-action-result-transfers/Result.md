# Audit 02: ordinary state and action-result transfers

## Ownership and scope

The inventory describes existing code; migration recommendations below have been revised to separate backend facts, frontend-local state and explicitly deferred historical-content transport. Existing coupling is evidence, not an endorsed ownership model.

* UnlockedData is exclusively for backend state that is or may become visible. Priority examples are the true active agent receiving the next Enter submission, model, execution variant/reasoning, provider, workspace root and other backend directories, and used tokens. Actual active-agent ownership/selection writers and directory meanings must be investigated in the remaining audits; this inventory of existing UI transfers does not establish that missing backend state is already exposed.
* Theme, palettes, keybindings, previews, cursor appearance, layout, local selector interaction and other state with no backend effect are TUI-owned, located in tui-specific paths/files, and accessed only on the tui-thread. They need no locks or UnlockedData. UI configuration persistence is not backend ownership. Replace direct backend theme management by frontend responsibility, NOT by UnlockedData<Theme>. Where keybinding hints appear in backend-built catalogs, remove that presentation dependency rather than making bindings backend authority.
* Draft text is entirely frontend-owned until Enter submits a std::string. It must not be mirrored into canonical Session data. Image rendering preferences likewise differ from backend attachment authority; reasoning presentation differs from execution reasoning selection.
* Session history is backend-owned but too large for this snapshot-copy pattern. Preserve the existing history, transcript/streaming and related historical-content transport for this iteration; do not design a replacement journal or remove working behavior. A future collection-access API will need a separate design covering history objects, attribution, reasoning and timing metadata without whole-history copies. Precisely inventory this exception; it does not exempt unrelated current-agent/model/directory/statistics facts.
* Only genuinely backend-affecting facts within mixed overview/trust/catalog/interaction domains use UnlockedData. Formatting and local views remain frontend-owned. Existing history/streaming/inspector-content transport is retained explicitly, not converted into canonical copied collections.

The /05 root and child descriptions use the same scope. Audit 05 must preserve these distinctions in its final matrix.

## Scope, evidence and terminology

This is a read-only source investigation of the current working tree, following audit 01. No production changes, configuration, build or tests were performed. References are relative to REPOROOT; line numbers describe the inspected tree and may shift. Explore collected helper/callback locations; the primary planner inspected runtime.h, the application builder and action implementations, runtime_state.cpp, display reload consumption, catalog delivery and the direct theme setter before forming the conclusions below.

This inventory accounts for every TuiRuntimeOptions field and callback and every field of the broad runtime/submit/reload contracts. Ordinary state paths are analyzed here. Ordered interactions and lifecycle obligations are explicitly assigned to audits 03/04 rather than silently exempted from migration. Proposed domains are design recommendations for reconciliation, not new types already implemented.

Implementation-goal references are /05/03 identity/active-agent/directories/transitions, /05/04 settings ownership/trust/overview, /05/05 catalogs/selectors/todos, /05/06 backend run facts/statistics (history transport preserved), /05/07 specialized interactions, /05/08 final boundary removal, /05/09 verification. The common registration/refresh boundary belongs to /05/02.

## Executive findings

1. The broad snapshot is constructed after many unrelated actions, including selecting a model, reasoning or settings and completing any submission. It is not a single coherent transaction: the builder reads different sources separately.
2. Startup and later updates duplicate most presentation domains through separate plumbing. ModelSelection and SessionStats use the new path, but model/provider/reasoning-related facts are still duplicated in overview and selectors.
3. Selector callbacks are another major state-transfer channel, even though their types do not contain Snapshot. Scoped-model actions even pass an entire prior TUI SelectListView back into the application and receive a replacement view.
4. There is a non-payload bypass: application display-setting functions directly call tui::set_tui_config_theme. Removing snapshot structures alone will not decouple the frontend.
5. Catalog generation semantics, post-mutation warning semantics, reload applied/unchanged signals, and path/authority filtering must survive the migration. A generic action-success bool plus unqualified dirtiness is not enough to preserve all current behavior.

## A. Broad production and consumption chain

`app/tui/interactive_tui.cpp:240–291` defines SessionPresentation and state_snapshot. SessionPresentation reads mode, session identity/path, effective workspace, context source count and project trust under Session access. state_snapshot separately refreshes title catalog information, obtains application catalog delivery, reads session presentation, reads effective display settings, reads Mermaid configuration, builds startup overview, reads Git HEAD and loads todos. It constructs TuiRuntimeStateSnapshot with these results.

The startup path (`376–410`) separately constructs almost the same information in TuiRuntimeOptions. initial_transcript here contains optional first-run authentication onboarding, not a demonstrated full session-history replay. `tui/runtime_state.cpp:initial_snapshot` and RuntimePresentationState's constructor (`94–105`) distribute options into ComposerSnapshot and SidebarSnapshot. Those destination snapshots are frontend-local state, not automatically prohibited types; their backend-built input fields are the transfer being replaced.

Later, RuntimePresentationState::apply_runtime_state_snapshot (`117–173`) overwrites composer/sidebar fields, handles catalog generations, resets image/Mermaid presentation when required, and calls refresh_token_status. Higher transition wrappers perform additional lifecycle work assigned to audit 03. Submit completion applies TuiSubmitResult::state_snapshot and later refreshes statistics again (runtime_active_run_internal.cpp). There is no necessary relationship between every action returning a bulk snapshot and every field having changed.

## B. Complete ordinary field inventory

All non-callback fields of TuiRuntimeOptions (runtime.h:297–318) and all TuiRuntimeStateSnapshot fields (106–135) are accounted for in these rows.

| Domain and exact transported fields | Authoritative source and present publication | Consumers/formatting; proposed destination and removal |
|---|---|---|
| Identity: mode, session_id, session_path, workspace | Session mode, store identity/path, current_dir/workspace_dir; initial presentation and state_snapshot after commands, session selection and submit completion | RuntimePresentationState copies to composer/sidebar; app stringifies mode and paths. /03: backend canonical session-presentation facts with frontend identity/mode/path projections; remove initial and snapshot copies and on_toggle_mode's returned current-state string. Transition identity is not merely a display string. |
| git_branch | git_branch_for_sidebar reads workspace/.git/HEAD at startup and each bulk snapshot; returns empty on unreadable/missing input, strips ref prefix or truncates detached text to 12 chars | Sidebar rendering; app currently reads and formats. /03: backend workspace observation with frontend label formatting; remove repeated bulk-builder read. Preserve current fallback first; support for worktrees etc. is a separate improvement, not assumed today. External changes need an explicit refresh policy. |
| context_source_count | Session::context_sources().size(), initial options, snapshot, and TuiSubmitResult | Composer/sidebar; optional absence means unknown, zero is known empty. /03 (publication coordination with /06): one context-metadata canonical source, independent numeric/display projections. Remove duplicate submit-result field and assignments. |
| initial_status / snapshot.status | Startup keybinding/display-load diagnostics; action-specific success/warning strings; submit builder passes empty status | Local status/command settlement, not a durable session fact. /06 and /08: bounded correlated feedback/result handling; do not make a single last-value slot that can lose important messages. Remove status as an excuse to return an entire state snapshot. |
| app_version | version::kDisplayVersion, initial options only | Sidebar version; immutable construction metadata, not backend session state. /08 may retain as a neutral immutable dependency or frontend build constant. No invalidation mechanism required. |
| initial_transcript | App-created TranscriptItem for first-run onboarding, if present | Initial composer transcript/tool index. Preserve this content delivery with the deferred transcript path; do not introduce a canonical history snapshot or require removal of the vector in this iteration. Keep unrelated startup backend facts outside this exception. |
| slash_commands and slash_catalog_generation | ApplicationCatalogCoordinator built from command availability, session/config/resources and keybinding hotkeys; refreshed after commands/reloads/title changes | Completion/palette state. App project_slash_command_items constructs TUI records; runtime_state avoids replacement on identical present generation. /05: canonical neutral catalog with revision, frontend completion projection; remove initial/delivery vectors. |
| file_references and workspace_catalog_generation | ApplicationCatalogCoordinator workspace catalog; refreshed on workspace-changing submissions/reload | Completion candidates and ranking cache; app project_file_reference_items creates TUI records. /05: canonical workspace-reference catalog, revision-aware projection. Must preserve valid empty list versus omitted unchanged payload semantics; remove old delivery protocol. |
| custom_themes | available_tui_custom_themes(invocation_paths), called at startup and each state_snapshot; validates palette data and includes name, path detail, palette and revision | /04: tui-thread-owned theme discovery/catalog, labels and previews in tui-specific files, using approved configuration utilities. No UnlockedData or backend theme owner. Remove duplication through backend snapshots; preserve validation and persistence safety. |
| project_trust | Session ProjectTrustState; project_trust_snapshot copies decision/resources labels, workspace/matched/trust-file paths, protected count, diagnostic | Settings and overview/presentation consumers. /04: sanitized trust-presentation facts, not permission authority; remove ProjectTrustSnapshot transfer and initial duplicate. Full details are not path-free, unlike startup overview. |
| initial_todos / snapshot.todos | Session read authority -> load entries -> project latest committed todowrite snapshot -> map statuses to TUI TodoItem | Sidebar, also live event updates. /05 plus /06: coherent canonical todo state and frontend items; reconcile live updates with history hydration. Remove backend-built TodoItem vectors. Current read failure/no authority/no todo all return empty: do not invent a stronger existing error distinction. |
| show_images, image_width_cells, cursor | App effective display settings, currently protected by its local settings lock, loaded/applied at startup and settings actions/poll | /04: tui-thread-only preferences and application, without locks/UnlockedData. Preserve precedence, preview behavior and preview clearing on disable. Remove backend-snapshot copies and move reload ownership to tui-specific files. Attachment authority remains separate backend state. |
| mermaid_config_epoch / mermaid_enabled; initial mermaid_render.config_epoch/enabled | App Mermaid bridge's presentation configuration, updated with effective settings | /04: local rendering configuration and invalidation, no backend canonical appearance domain. /07 and audit 04 preserve correlated worker-service completion/cancellation; transferring immutable job parameters does not make frontend preferences shared mutable state. |
| startup_overview | build_startup_overview_snapshot reads loaded Session context/freshness, model/trust, theme and keybindings and constructs bounded TUI summary | /04: frontend composes projected safe backend resource/model/trust facts with its own local theme/bindings. Query/selection/layout remain local. Remove transported StartupOverviewSnapshot, not safety filtering; do not create backend copies of its UI-only fields. |
| key_bindings | App loads keybinds.json into TuiKeyBindings; initial options and TuiKeyBindingReloadResult; hotkeys also influence command catalog and overview | /04: tui-thread-local bindings, reload, dispatch and help; no UnlockedData or locks. Merge local hotkey hints with backend command facts on frontend side. Eliminate broad reload result, preserve errors and coherent local application. |

### Nested shapes and semantics

ProjectTrustSnapshot (`composer.h:317–327`) contains decision, project_resources, workspace, matched_path, trust_file, protected_resource_count, diagnostic. `interactive_tui_support.cpp:71–79` performs this projection. The backend must continue validating authority and deciding what information is exposed; frontend formatting does not grant access to the underlying trust store.

StartupOverviewSnapshot (`composer.h:356–384`) includes mode/provider/model/trust/resource count, instruction-source count, bounded resource groups, skill/prompt-command/plugin names, optional plugin-resource failure count and lower-bound markers, theme name/badge, toggle keys, key hints and precomputed compact/detail lines. `startup_overview.cpp:245,478` builds it. Path/secret filtering, work caps, UTF-8 limits and truthful lower-bound counts must remain at the boundary; merely moving all raw context metadata into frontend memory would weaken the existing contract. Safe fact filtering belongs on the producer side; presentation layout/labels belong on the frontend side. Optional overview absence is distinct from an available overview with empty resource groups.

SelectListView (`composer.h:551–567`) contains title/subtitle, item vector, selected index, query, placeholder, empty text, footer and layout flags. Items contain labels/descriptions/groups/details/badges/current/enabled/disabled reason and hidden values. Application-created versions mix authoritative choices with local interaction and formatting. Do not store the entire view as the new canonical domain. Local query, highlight, scroll and layout flags should stay frontend-local; backend validates selected identifiers against current authority.

### Copying and generation costs

ApplicationCatalogCoordinator::delivery_snapshot (`app/command_palette.cpp:887–903`) copies slash/reference vectors only when its generation differs from its **last delivered generation**, and updates that delivery cursor immediately. Runtime state compares delivered generations to **last applied generations**; absent generation means apply, identical present generation means retain current data. Empty vectors can therefore mean omitted unchanged data, not necessarily empty catalog. Initialization uses a full coordinator snapshot instead.

Migration must preserve revision-controlled work avoidance without treating delivery as successful application. If packets can be discarded or multiple consumers appear, producer-side delivered cursors need scrutiny; the current code alone does not establish reliable multi-consumer delivery. New frontend projections should own their observation progress and copy only on necessary revision changes. Lists may be large; avoid per-render reconstruction and long canonical lock-held formatting.

Other bulk fields have no comparable change test in the builder: custom theme discovery, Git HEAD, todo history load and overview construction can recur after unrelated actions. state_snapshot calls refresh_title_catalog and ignores its Result at that particular point. Session selector snapshot does likewise. Failure behavior must be inventoried, not retrospectively described as always propagated.

## C. Complete TuiRuntimeOptions callback inventory

`I` below denotes app/tui/interactive_tui.cpp; `R` tui/runtime.cpp; `A` tui/runtime_actions_internal.cpp; `S` tui/runtime_submit_internal.cpp. All callbacks are declared at runtime.h:320–389. Groups list every field explicitly.

| Callback(s) | Current producer -> consumer / result | Migration classification and target |
|---|---|---|
| refresh_session_statistics | I:410 -> RuntimePresentationState::refresh_token_status; void callback loads/recomputes backend canonical stats | /06 remove UI-triggered publication callback; backend lifecycle publishes. Existing frontend SessionStats remains consumer. |
| create_active_run_queues | I:411 onward captures session/jobs binding and creates InteractiveRunQueue -> active-run controller; returns TuiActiveRunQueues | /06,/07; bound command/queue services with nested data/control operations, not ordinary static state. Audit 04 must resolve every nested operation, not retain as a broad exception. |
| on_submit | I:484–565 backend submission/queued followups -> active-run completion; TuiSubmitResult | /06 separate authorized output and control from canonical state; /08 delete state_snapshot. See full field inventory below. |
| on_attach_image | I:567–580 resolves/imports path under session authority -> S:392–402 | /07 action returns ImageAttachmentRef capability/metadata; neutralize backend type, retain validated import and correlation. Not a whole-UI snapshot. |
| on_paste_clipboard_image | I:582–587 backend clipboard import -> A attachment action | /07; Result<optional<ImageAttachmentRef>>, absence is distinct from import failure. Audit 04 owns payload/lifetime treatment. |
| on_external_editor | I:589–590 external editor -> A:167–193 | /07; Result<optional<string>> edited text/cancel outcome. Draft is frontend-owned; this is a correlated service result, not a persistent backend state mirror. Must not be silently discarded or forced into uncorrelated last-value state. |
| on_load_image_attachment | I:592–595 session load -> A:263 attachment preview and renderer capability plumbing | /07; loaded bytes/resource result under authority, not ordinary label state. Remove session implementation types from TUI API; retain bounded validated loading semantics. |
| on_toggle_mode | I:597–601 run /mode then return stringified current mode -> R:2059 onward assigns mode | /03; remove returned authoritative mode string and consume canonical mode projection; action can return success/error. |
| on_cycle_reasoning | I:603 delegates cycle_runtime_reasoning -> A:322–327 | /03,/05,/08; current string result is feedback while model-selection projection owns state. Preserve failure/feedback, avoid redundant bulk refresh. |
| on_cycle_model | I:604–612 switch_model_and_refresh -> A:502–508 | /03,/08; Result<runtime snapshot> replaced by success/unchanged/error/control plus canonical model and dependent domain publication. |
| on_reload_key_bindings | I:614–622 load bindings, update hotkeys/catalog -> R:1083–1090,1577–1584 | /04; frontend-local reload/application replaces backend callback plumbing and bulk result. Preserve local dispatch/help consistency and reload outcome; no canonical bindings. |
| on_reload_display_settings | I:624–634 apply/remember/watch/catalog -> S:286–291 | /04; frontend-local preference reload removes backend snapshot return, preserving explicit outcome/error. |
| on_maybe_reload_display_settings | I:636–660 compares watch state, retains last-good settings on invalid edit -> A:105–145 | /04; move watch/reload ownership to tui-specific frontend code. Preserve failure/unchanged/applied independently from displayed text under a preview overlay; do not replace this with backend UnlockedData. |
| model_selector_view | I:662–663 -> A:347–355; app model_selector_view_1 | /05 canonical available models/provider validity + frontend selector. Remove returned SelectListView. |
| reasoning_selector_view | I:664–667 -> A:366; optional view depends on configurable reasoning; chained bool changes footer | /05 model capabilities/selection facts + frontend view/footer. Optional absence must remain distinct from an empty selectable view. |
| scoped_model_selector_view | I:669–670 -> A:388–396 | /05 canonical scope/available models; frontend owns ordering presentation/selection. |
| session_selector_view | I:671–679 resets app-local sort/filter flags, builds catalog view -> A:406–414 | /05 session catalog projection; move purely local selector controls out of app closure. |
| list_subagents | I:681–691 -> runtime_subagent_workspace_internal.cpp:111–113,164–172 | /07 canonical sanitized job list + frontend selector. Result<SelectListView> is a state bypass. |
| inspect_subagent | I:692–707 -> runtime_subagent_workspace_internal.cpp:242–244 | /07/audit 04: retain existing historical inspector-content delivery under the history exception, including owner/generation/eviction safety. Separately classify current job/control facts for migration; do not redesign or copy the message collection here. |
| cancel_subagent, promote_subagent | I:708–729 -> subagent workspace controller | /07 explicit fixed outcome enums, not broad state payloads. Retain permission/owner validation; resulting status through canonical data. |
| on_open_fork_user_turn_selector, on_open_copy_user_turn_selector | I:730–740 -> A selector opening paths | Retain existing historical picker-content transport for now, with persistent-session eligibility and authorized IDs. Local interaction is frontend-owned; do not force historical choices into a copied canonical collection. /05 records this bounded deferral. |
| on_session_selector_sort_cycle, on_session_selector_named_filter_toggle, on_session_selector_path_display_toggle, on_session_selector_archived_filter_toggle, on_session_selector_label_timestamp_toggle | I:741–765 -> R:1352–1374 | /05 app mutates closure-local view controls and returns a new view; these interaction controls should be local, reading canonical catalog facts. |
| on_session_selector_archive, on_session_selector_unarchive | I:766–791 -> R:1486–1501 | /05 backend archive mutation remains action; refreshed catalog canonical, not returned SelectListView. Preserve errors and current-selection reconciliation. |
| on_session_selector_branch_parent, on_session_selector_branch_child | I:792–797 -> R:1430–1445; helpers I:310–352 open/replace Session | /03,/05 actions validate opaque selection and navigate; resulting identity/catalog publication replaces snapshot result. |
| on_branch_summary_prepare | I:353–369,798 -> R:463–473 | /07 operation generation/phase facts canonical; remove TuiBranchSummarySnapshot return. Preparation error remains explicit. |
| branch_summary_snapshot | I:799 -> R:437–449 | /07 replace polling snapshot callback with frontend projection; audit 04 preserves required sequencing. |
| on_branch_summary_confirm, on_branch_summary_cancel | I:800–801 -> R:499–520 | /07 Result<bool> correlated generation-based action outcome; not bulk state. Preserve accepted/rejected/error distinction. |
| on_branch_summary_refresh_catalog | I:371–374,802 -> R:389–397 | /05,/07 canonical session-catalog publication; remove Result<SelectListView>. |
| remember_permission_rule | I:803–805 -> runtime_prompts_internal.cpp:123,167–184 | /07 Result<TuiRememberedPermissionRule> with rule_id is authority/control correlation, not arbitrary replacement UI state. Audit 04 must preserve decision lifetime and neutralize backend prompt/action types. |
| on_settings_selected | I:807–878 -> R:1141–1147,1688–1690 | /04 split mixed dispatch: backend trust/authority actions publish backend facts; theme/display/binding choices and closing a view are frontend-local. Remove broad snapshot return; retain action feedback without backend UI preference state. |
| on_model_selected | I:879–894 -> R:1599–1601 | /03,/05 resolve selected provider/model and mutate Session; eliminate bulk result. |
| on_reasoning_selected | I:895–912 -> R:1617–1621 | /05 validates level or resets default; model-selection canonical already owns current value; remove bulk result while preserving default/unchanged feedback. |
| on_fork_user_turn_selected | I:913–980 -> R:1645–1666 | /03,/05 confirmed authority switch, identity/catalog publication; preserve post-mutation soft-warning semantics described below. |
| on_read_user_turn_text | I:981–983 -> R:1681–1683 | Preserve authorized exact historical-text retrieval under the deferred history exception. It fills a frontend-only draft; subsequent editing is invisible to backend until submission. No canonical draft or new historical-content protocol is required now. |
| on_scoped_model_toggled, on_scoped_model_enable_all, on_scoped_model_clear_all, on_scoped_model_toggle_provider, on_scoped_model_reorder | I:984–993 -> R:1190–1296 | /05 these accept previous SelectListView and return replacement SelectListView. Remove both directions of view coupling; backend receives validated identifiers/action, frontend retains query/highlight and observes canonical scope. |
| on_scoped_model_save | I:994 -> R:1310–1317 | /05 persistence action plus string feedback, not a replacement model list. Preserve error/ack, publish any changed scope/catalog. |
| on_session_selected | I:995–1014 -> R:1632–1634 | /03 open/replace or already-open result; canonical identity/catalog + explicit outcome replaces snapshot. |
| on_before_tui_shutdown | I:1016–1020 -> RuntimeBeforeShutdownGuard in R | /03,/07 lifecycle barrier/control, no state payload. Audit 03 proves complete quiescence; not removable simply because callbacks are being reduced. |

Consumer locations identify the relevant dispatch sites, not exhaustive secondary render uses. Definitions and signatures, not a Snapshot-name search, determine classification. Actions returning strings must be judged by actual meaning: on_toggle_mode returns current state, whereas save/status feedback and external-editor text have different semantics.

## D. Complete submit/reload and nested-service inventory

TuiSubmitResult (`runtime.h:146–165`, producer I:549–565) has:

* quit: control outcome, must survive until orderly shutdown.
* ordinary_turn_committed and ordinary_turn_request_ids: conversation authorization/settlement information, not UI decoration. /06 preserves fail-closed handling of missing IDs.
* output and tool_timeline: aggregated completion/fallback content, with backend-built TUI tool records; preserve this existing content path under the explicit transcript/streaming deferral, not a new UnlockedData history collection.
* local_command_result.output/tool_timeline: local-command-only content; must never implicitly authorize conversation events.
* conversation_output/conversation_tool_timeline: separately attributed conversation fallback; /06 must not merge indiscriminately with local command output.
* context_source_count: duplicate ordinary state, moves to the context metadata domain.
* state_snapshot: duplicate bulk state, deleted after its domains and transition effects migrate.

The active-run controller applies the state result before settlement and uses output/tool fallbacks when appropriate. Preserve those content paths; removing state_snapshot is not permission to discard command output or skip Session transition cleanup. TuiKeyBindingReloadResult is replaced by frontend-local binding reload/application and its local outcome; its broad state member disappears, with no backend binding publication.

TuiActiveRunQueues (`runtime.h:88–103`) includes active_request_id, queue_steering, queue_follow_up, take_steering_messages, skip_active_steering, take_next_follow_up, mark_follow_up_started, restore_latest, run_nonblocking_command, finish. These are bound queue/control operations with nested message/output delivery. TuiQueuedFollowUp has request_id/message; TuiRestoredQueuedMessage has message/steering. Their lossless ordering and session binding are audit 04 inputs, not ordinary catalog state exemptions.

TuiSubmitContext (`178–195`) carries request_id, permission_resolver, question_resolver, event_sink, cancel_requested, steering/follow-up operations, on_subagent_launch, plugin_ui, image_attachments. These expose event/agent/permission/session types directly. /07 and /08 classify and narrow in-scope implementation coupling without moving backend validation into frontend projections or breaking the explicitly retained historical-content/event contracts.

TuiMermaidRenderBridge carries configuration plus enqueue/cancel/drain. Requests/completions carry identity, config_epoch and source/text/accepted. /04 keeps rendering preferences and their local invalidation on the tui-thread without locks/UnlockedData; audit 04 distinguishes worker request/completion synchronization from ownership of those local preferences. Retain epoch rejection and bounded service execution, rather than introducing a canonical last-text or shared preferences object.

TuiBranchSummarySnapshot (`281–292`) contains generation, phase, source/model labels, optional eligibility/failure/append-commit codes and refresh_required. These are canonical operation facts for /07; bool action outcomes remain control. No backend-internal operation snapshot should cross directly after migration.

## E. Authoritative selector data and current formatting

Source definitions inspected/located:

* app/tui/command_palette_views.cpp:17,53 maps neutral slash/reference catalog records into TUI types. No filtering/sorting occurs in these two simple mapping functions.
* model_selector_view_1 at :255 reads model/paths/provider catalog under Session access, then loads the model registry. Successful views include current model if missing, order/group providers and mark unavailable providers disabled. Load failure creates a disabled unavailable row rather than Result failure.
* scoped_model_selector_view_1 at :321 also reads scoped cycle; its view orders scope first and marks membership. Scope remains backend model-selection authority; presentation filter/highlight is not.
* app/tui/reasoning_selector.cpp:83,123 derives choices from supported levels, omits off/disabled choices, adds Default, uses readable labels, selects current and disambiguates labels. Optional absence signals no choices. Backend validates selection; frontend can derive labels from exposed safe capability facts.
* command_palette_views.cpp:527,558 builds user-turn views from authorized SessionUserTurn records; action-time exact text retrieval must revalidate selected ID, not trust a stale displayed preview.
* application_catalog_session_view at :573 reads the coordinator cache and produces sort/filter/path/time-sensitive session rows and parent-summary hints. Missing tree becomes a disabled unavailable row carrying cached diagnostic. It is not equivalent to an empty successful catalog.
* interactive_tui_support.cpp:252 todos_for_session loads authorized entries and maps committed todos. Errors currently collapse to empty.

The new catalog domains should expose stable identities, validity/disabled facts and necessary safe descriptions. Frontend projections own labels, footer instructions, selected index reconciliation and filtering. A rendered row or backend-stored previous SelectListView is not canonical backend state. Session-title or catalog background updates require observation independent of opening a selector; publication timing is for audits 03/04 to reconcile with the app coordinator.

## F. Non-snapshot bypass: theme globals

`app/tui/tui_display_settings.cpp:1383–1389` calls `ava::tui::set_tui_config_theme(settings->theme, settings->custom_theme)` directly. `app/tui/configuration_commands.cpp:573,584,604` also calls that setter. The app reads active_tui_theme for status/overview (`interactive_tui_support.cpp:65–68`, I:271,384).

Therefore selected effective theme currently also travels by direct TUI mutation. /04 must make theme discovery, persistence, reload and application frontend responsibilities in tui-specific files, on the tui-thread without locks or UnlockedData. A setter called by frontend code in app/tui is legitimate; a backend command mutating frontend preferences is the coupling to remove. Preserve environment/config precedence and preview overlays. Code location alone does not establish backend ownership, and this finding does not assert a demonstrated race.

## G. Correctness constraints not to lose

1. Display auto-reload: A:105–145 explicitly distinguishes unchanged/applied and suppresses repeated identical error feedback. Failed external edits retain last-good settings and do not advance watcher state. Preserve both behavior and errors; a changed display string is not an applied-reload acknowledgment.
2. Fork-from: I:939–979 checks that authority truly changed and rejects false "opened" success. After real mutation, a catalog-refresh failure is a soft visible warning and the new session still gets presented. A replacement action API must not return failure in a way that leaves old presentation over new backend authority.
3. Same-session navigation returns a distinct already-open outcome; invalid/missing targets and unavailable branch parents/children remain errors.
4. Unknown, empty, absent/no-choice and unavailable are not interchangeable. Preserve current semantics unless the user explicitly approves improvement (notably todo errors currently appear empty).
5. Safe output limits/path filtering remain backend boundary responsibilities even when layout/formatting moves frontend-side. Path-free overview/subagent contracts must not expand into raw prompt, session-source or coordinator access.
6. Initial and incremental publication of migrated backend facts must share one canonical source. Frontend-only preferences instead have one tui-thread-local owner. Temporary adapters need removal goals; explicitly deferred history delivery remains intact rather than acquiring a parallel writable authority.

## H. Verification and deletion matrix

These are existing relevant test locations and required migration assertions, not claims of execution in this audit.

| Goal/domain | Existing anchors | Required evidence before deleting old path |
|---|---|---|
| /03 identity/mode/context | tui_runtime_state_tests.cpp, tui_prompt_stash_tests.cpp, tui_permission_tests.cpp, app_runtime_startup_tests.cpp | initial state; switch/clear/fork; same/failed switch; late old-session result; one-time grants/attachment/search reset; context count unknown/zero; mode action no longer assigns returned state |
| /04 display/trust/overview/bindings | app_runtime_command_ui_tests.cpp; tui_composer_rendering_tests.cpp display_reload_poll/display_reload_overview cases; configuration/keybinding/theme tests | unchanged/applied/error local reload; last-good retention; precedence/preview; dispatch/help coherence; bounded backend facts in local overview; no backend-owned appearance state, locks or UnlockedData; frontend-local theme application is allowed |
| /05 catalog/selectors/todos | app_runtime_catalog_tests.cpp, session_title_coordinator_tests.cpp, tui_selector_tests.cpp, tui_composer_rendering_tests.cpp | generations/no-op copy avoidance; omitted versus empty list; unavailable disabled rows; stale selected IDs; sort/filter/highlight preservation; reasoning Default/no-choice; scoped order/bulk/save; todo hydration/live consistency |
| /06 submission/stats | app_runtime_startup_tests.cpp token-compaction checks; tui_selector_tests.cpp projection checks; runtime/composer event settlement tests | exact versus compact token changes; no duplicate /exit publication; preserved local/conversation attribution; completion content not lost; no broad submit snapshot |
| /07 interaction state | existing subagent workspace, plugin UI, branch summary, permission/question tests to be enumerated by audit 04 | generation/correlation/owner checks; remove in-scope state Snapshot transfers while preserving historical inspector content; safe cancellation and stale response rejection |
| /08,/09 boundary | module_dependency_rules_test.py and frontend contract checks to add | no in-scope backend-built replacement views or TuiRuntimeStateSnapshot/state_snapshot/reload.state; explicitly allow retained transcript/tool/history-picker/inspector content paths; no backend theme/draft authority or new locks/UnlockedData for UI-only state |

Real terminal scenarios to retain include startup_overview, main_startup_trust_keybinds, main_models_selectors, main_slash_completions, main_session_mgmt, theme_env, theme_persisted, nested_settings_preview, active_run, streaming_scroll, main_permission_flow, main_question_flow, subagent_workspace, branch_summary and plugin_ui under tests/tui_tmux_scenarios. Run the applicable scenarios for each slice and the full opt-in wave at /09. Tests for expensive work counts and delivery-versus-applied generations must be explicit; screen equality alone does not establish the new ownership contract.

## I. Reconciliation handoff and limits

Audit 03 owns lifecycle/transition effects hidden in apply_runtime_state_snapshot, publication sequencing, stale results and shutdown. Audit 04 classifies nested queues/services and decisions and records the precise existing historical-content paths that remain intact. No replacement journal or collection-access architecture is selected. The no-state-snapshot objective applies to migrated backend facts, with the explicit history exception above.

Open decisions for audit 05:

* Split backend catalog/resource/execution facts from frontend-local settings and views; do not create UnlockedData<TuiRuntimeStateSnapshot>, Theme, KeyBindings or Draft domains. Verify active-agent dispatch authority and workspace-root/directory meanings, which the existing transfer inventory does not establish.
* Decide how an action acknowledgment identifies the canonical revision/transition the frontend must observe before accepting another prompt.
* Preserve reload application semantics independently from string changed flags.
* Preserve existing historical text/content retrieval; classify image/editor services without making drafts backend-owned or redesigning history. Shared service utilities do not imply backend ownership of UI state.
* Define generation/consumer ownership for backend catalogs/model/execution facts; delivery cursors are not acknowledgments. Local settings application remains tui-thread-only.
* Move formatting without moving permission/resource-validation authority or exposing forbidden raw data.

No ordinary TuiRuntimeOptions field or callback is intentionally left unclassified. The table is the entry-contract inventory; the final full-tree producer/include audit in /08 remains necessary to catch future changes and additional paths outside this seam. Specialized protocol semantics are explicitly deferred to audit 04, not asserted safe without inspection.
