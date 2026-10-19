# Canonical and frontend snapshot types — proposal for review

## Status and boundaries

**Proposal, not approved implementation.** No source files, builds or configuration were changed by this planning work. Declarations below are sketches for review, not generated source. Source references include Session.h, Session.cpp mutation sites, frontend ModelSelection/SessionStats, session/Stats.h, runtime/SessionStats.h, runtime/ReasoningSelection.h, agent/agent_config.h, project_trust.h and command_palette.h. Statistics names and placement were reconciled with the user's commit c0b9b06e: session::Stats and session::EntryCounts are reusable values; runtime::SessionStats_ts is the runtime wrapper. Earlier audit Result.md files supply historical producer/consumer and verification inventories.

I recommend preserving the existing pattern: wrap actual backend-owned cohesive data, copy only the required value members into a frontend projection, and format there. Do not create a second writable agent/model/mode/directory selection solely for UI consumption. No generic property system or registry replacement.

User-approved grouping preference: expect infrequent updates (seconds apart or longer) and negligible contention. Keep naturally related backend members together if splitting them would make the code awkward, even when some members are not displayed and may never be. Do not mechanically partition structs by UI usage or introduce finer locks to avoid hypothetical tiny costs. Concrete expensive work or independently meaningful responsibilities can justify a split; otherwise backend readability and semantic cohesion take precedence. Ambiguous grouping choices should be reviewed with the user, not decided by a rigid rule.

The proposal has three categories:

1. Existing model/statistics shapes, retained and accurately reproduced below.
2. Concrete candidate types for agent selection, directories, prompt metadata, trust and session identity. Several have explicit tradeoffs requiring approval, especially wrapping existing large backend bundles versus maintaining derived observation records.
3. Later catalog/resource/todo domains with **explicit unresolved type decisions**. They are not approved by approval of the core types. Their implementation slices remain blocked until a supplementary declaration proposal is reviewed; no unspecified structs may be invented during coding.

Themes, keybindings, rendering preferences, previews, draft and selector interaction are frontend-local, without UnlockedData or locks. Existing history/streaming and interactive request/reply interfaces remain unchanged. This document does not design their future replacements.

## 1. Existing ModelSelection — keep its value shape

Currently in app/runtime/Session.h:111–122:

```cpp
namespace ava::app::runtime {
struct ModelSelection {
  ava::config::ModelInfo model;
  std::optional<ReasoningSelection> reasoning = std::nullopt;
  std::optional<std::vector<std::string>> scoped_model_cycle = std::nullopt;
  AVA_DEBUG_PRINT_MEMBERS_ON
};
using ModelSelection_ts = UnlockedData<ModelSelection>;
}
```

ReasoningSelection already has its own value header, app/runtime/ReasoningSelection.h:

```cpp
namespace ava::app::runtime {
struct ReasoningSelection {
  std::string level;
  std::optional<std::string> provider_level = std::nullopt;
  std::optional<long long> budget_tokens = std::nullopt;
  std::string display;
  AVA_DEBUG_PRINT_MEMBERS_ON
};
}
```

Backend storage stays `ModelSelection_ts model_selection_` in Session_aggregate_base. It is already wrapped; there is no new replacement of a bare ModelSelection required. Session::switch_model/set_reasoning and scoped-model mutations remain writers. Provider and context-window facts remain in ModelInfo; no separate ProviderSelection or ContextWindow canonical object.

Existing frontend snapshot members remain:

```cpp
namespace ava::app::frontend {
class ModelSelection {
  // Existing canonical copies:
  config::ModelInfo model_snapshot_;
  std::optional<runtime::ReasoningSelection> reasoning_snapshot_ = std::nullopt;
  // Existing output caches:
  DisplayString model_;
  DisplayString provider_;
  DisplayString reasoning_;
  bool context_window_tokens_changed_{true};
  // Existing frontend pointer, index and atomic dirty flag omitted here only
  // to emphasize the value mapping; see shared mechanics below.
};
}
```

| Snapshot | Canonical source | Copy semantics |
|---|---|---|
| model_snapshot_ | ModelSelection::model | Whole ModelInfo copy, including currently undisplayed metadata |
| reasoning_snapshot_ | ModelSelection::reasoning | Whole optional ReasoningSelection copy |
| none today | ModelSelection::scoped_model_cycle | Deliberately not copied by the current class; do not pretend it already is |

Existing accessors stay `refresh_model()`, `refresh_provider()`, `refresh_reasoning()` returning RefreshResult<string const&>, and `context_window_tokens_refresh()` returning RefreshResult<optional<long long>>. Existing naming is retained rather than unrelated renaming.

**Review decision: no scoped_model_cycle snapshot, changed flag or refresh accessor is added.** The /models and /scoped-models lists are interactive popups, deferred with the other interactions. The current scoped selector shows the complete available-model list even when canonical scope is nullopt; its heading does not make that list passive always-visible state. Preserve its existing on-demand view/actions, including scope toggle/reorder/save, without an eager frontend scope subscription.

Canonical scoped_model_cycle remains unchanged with its existing optional semantics. This refactor neither expands nullopt into a stored model list nor removes the optional; that is a separate backend/UI behavior decision. The existing ModelSelection projection continues copying only model and reasoning. When a popup action actually changes the selected model/reasoning, those current displayed values still refresh through the existing canonical notification path.

## 2. Existing session::Stats and runtime::SessionStats_ts — keep canonical statistics and whole snapshot

The current source includes equality operators; preserve them. Following commit c0b9b06e, the values are defined in session/Stats.h (capital S), and the wrapper alias is defined separately in app/runtime/SessionStats.h. This placement is settled, not a pending module-ownership question. The frontend class remains named frontend::SessionStats.

```cpp
namespace ava::session {
struct EntryCounts {
  std::size_t session_start = 0;
  std::size_t session_metadata = 0;
  std::size_t user_message = 0;
  std::size_t assistant_message = 0;
  std::size_t tool_call = 0;
  std::size_t tool_result = 0;
  std::size_t permission_decision = 0;
  std::size_t mode_change = 0;
  std::size_t model_change = 0;
  std::size_t reasoning_block = 0;
  std::size_t reasoning_change = 0;
  std::size_t compaction = 0;
  std::size_t branch_summary = 0;
  std::size_t error = 0;
  std::size_t cancel = 0;
  bool operator==(EntryCounts const& other) const = default;
  AVA_DEBUG_PRINT_MEMBERS_ON
};
struct Stats {
  std::size_t entry_count = 0;
  std::string first_timestamp;
  std::string last_timestamp;
  EntryCounts counts;
  std::optional<long long> input_tokens;
  std::optional<long long> output_tokens;
  std::optional<long long> reasoning_tokens;
  std::optional<long long> cache_read_tokens;
  std::optional<long long> cache_write_tokens;
  std::optional<long long> total_tokens;
  std::optional<long long> estimated_input_bytes;
  std::optional<long long> estimated_output_bytes;
  std::optional<long long> estimated_total_bytes;
  std::optional<std::size_t> active_context_tokens;
  std::optional<std::size_t> system_prompt_tokens;
  std::optional<long double> total_cost_usd;
  std::optional<long double> known_cost_usd;
  bool cost_complete = true;
  std::size_t unknown_cost_entries = 0;
  std::size_t exact_usage_entries = 0;
  std::size_t estimated_usage_entries = 0;
  bool operator==(Stats const& other) const = default;
  AVA_DEBUG_PRINT_MEMBERS_ON
};
}
namespace ava::app::runtime {
// Defined in app/runtime/SessionStats.h, not in session/Stats.h:
using SessionStats_ts = UnlockedData<ava::session::Stats>;
}
```

Default-constructed optionals are disengaged. Active history and system-prompt tokens remain separate members of this one object, published together. These counters are not the history collection itself.

Backend storage remains `SessionStats_ts session_stats_`. Frontend remains:

```cpp
ava::session::Stats stats_snapshot_;  // Copy of entire canonical value.
std::optional<long long> active_token_total_; // Derived saturated total, NOT another source.
DisplayString token_count_;
DisplayString percentage_;
bool percentage_recompute_pending_{true};
```

TuiFrontEnd member remains `frontend::SessionStats session_stats_`. `token_count_refresh()` and the existing spelling `contex_percentage_refresh()` return RefreshResult<string const&>. SessionStats remains the sole consumer of ModelSelection's numeric-window changed flag. Computing exact total/percentage from the snapshot is frontend derivation; computing estimates/provider usage is backend work. The latter publication-trigger changes belong to /05/06, not foundation coding.

## 3. Proposed AgentSelection — move actual selection, not a UI mirror

```cpp
namespace ava::app::runtime {
struct AgentSelection {
  ava::agent::AgentDefinition selected_primary_agent;
  AVA_DEBUG_PRINT_MEMBERS_OPT_OUT
};
using AgentSelection_ts = UnlockedData<AgentSelection>;
}
```

Proposed backend storage: `AgentSelection_ts agent_selection_` in Session_aggregate_base. Remove selected_primary_agent from InvocationInputs and make this nonoptional wrapped definition the actual selection used by backend execution. requested_primary_agent remains optional invocation/configuration intent in InvocationInputs/OpenContext; it is not the current selection and is not copied into the frontend. Its absence means resolve the configured/default agent, not publish an absent selected agent. No second writable selected definition remains elsewhere.

AgentDefinition is the existing type in agent/agent_config.h with name, description, system_prompt, tool_preset, max_tool_iterations, hidden, provenance and path. It has no model/provider member. Keep that existing definition and its safety-oriented printer. Resolve the selected definition during backend initialization after command-line/configuration processing, and construct/publish a valid AgentSelection before the composer permits submission. Without --agent, resolve the default there. Do not postpone selection until Enter, create only a display label, or use an empty default-constructed AgentDefinition as an initialized selection.

Proposed frontend value group: `SelectedAgentMetadata`. It contains copied metadata from the backend's selected_primary_agent, not the definition's prompt text or a popup's highlighted candidate. Preserve the backend's selected/selection terminology for these copies; any future terminology change must be project-wide rather than frontend-only. The group is nonoptional because an initialized backend has a selected agent. The member name identifies snapshot ownership, so individual fields do not need the `_snapshot_` suffix.

```cpp
namespace ava::app::frontend {
struct SelectedAgentMetadata {
  std::string name;
  ava::agent::AgentDefinitionProvenance provenance;
  ava::agent::AgentToolPreset tool_preset;
  AVA_DEBUG_PRINT_MEMBERS_OPT_OUT
};
class AgentSelection {
  SelectedAgentMetadata selected_agent_snapshot_;
  DisplayString agent_;
  DisplayString provenance_;
  DisplayString tool_preset_;
 public:
  RefreshResult<std::string const&> agent_refresh();
  RefreshResult<std::string const&> provenance_refresh();
  RefreshResult<std::string const&> tool_preset_refresh();
};
}
// TuiFrontEnd:
app::frontend::AgentSelection agent_selection_;
```

This is a grouped selected-field copy from one locked AgentSelection read: name, provenance and tool_preset are copied exactly from selected_primary_agent, including when that agent was selected by default rather than by --agent. requested_primary_agent is not exposed. Initialization/first refresh must populate the snapshot before any display/access; pre-initialization is not modeled as an optional ordinary state. No existing direct requested-agent TUI display was located.

Populate all three metadata fields together before their use. There is no frontend Unknown/Inherit fallback for a missing selected definition. The current file loader assigns Global or Project provenance; the chosen default must carry its actual provenance/preset as well. Preserve backend values, and do not invent them in the frontend. A default implemented by the backend must be a real execution definition, not solely a metadata record.

Do not copy agent system_prompt, source path, description or iteration policy into frontend just to show identity. Those fields remain in the canonical selected definition for backend behavior; a future visible field can add an explicit safe snapshot member. Do not expose the full AgentDefinition through a frontend accessor.

User decision: the selected agent is always known before the composer accepts a submission. This intentionally changes the old representation where no --agent meant an absent additional named definition. Investigate existing default configuration/prompt selection to resolve its real definition early; do not assume a default-name setting exists without checking. If a concrete default name/configuration policy is missing, present that narrow policy choice before coding it, not a renewed optional-selection design. Keep prompt composition correct: normal/default instructions must not be appended twice merely to introduce the explicit definition. Invalid explicit configuration remains a backend resolution error; do not publish an unusable selection and rely on UI guessing. Tool preset still describes definition configuration, not complete permission authority.

Writers: construction after explicit/default primary resolution; trust re-resolution; apply_trust_prompt_state_and_refresh; Session replacement. Paths that previously cleared a forbidden/unavailable definition must now resolve an allowed default or fail the relevant operation explicitly, never leave a submit-ready backend with no selected agent. Do not bypass trust restrictions to maintain the invariant. Detached state copies the selected value into a fresh unregistered wrapper. Model switches/reload retain the selected definition unless backend policy actually changes it. Session accessors must use guard/value-based access; no references escaping temporary guards. Enter reads the already-selected canonical definition, rather than selecting a default as part of submission.

**Grouping decision for approval:** AgentSelection remains separate from execution tool_visibility and resolved prompt state. Existing Session-level mutation coordinates them. Do not promise atomic frontend observation across those separate domains; the current agent display needs identity, not a new transaction framework. If user prefers a broader backend execution-selection aggregate, review it before adding fields.

Implementation allocation: /05/03, after the shared model/stats boundary is approved and implemented.

## 4. Proposed Directories — move actual backend directory inputs

```cpp
namespace ava::app::runtime {
struct Directories {
  std::filesystem::path workspace_dir;
  std::filesystem::path current_dir;
  std::vector<std::filesystem::path> additional_writable_dirs = {};
  AVA_DEBUG_PRINT_MEMBERS_OPT_OUT
};
using Directories_ts = UnlockedData<Directories>;
}
```

Proposed storage: `Directories_ts directories_` in Session_aggregate_base. Remove those three members from InvocationInputs. This preserves additional writable-directory configuration with the roots it complements; descriptor anchors and permission authority do NOT move into this object or frontend memory.

Construct with resolved backend paths: workspace defaults to process cwd; current defaults to workspace. An empty default path in an uninitialized wrapper is not permission to resolve paths in the UI. Registration/first rendering follows successful construction. OpenContext retains input paths; they are construction inputs, not the active canonical copy.

```cpp
namespace ava::app::frontend {
class Directories {
  runtime::Directories directories_snapshot_;
  DisplayString workspace_root_;
  DisplayString current_directory_;
 public:
  RefreshResult<std::string const&> workspace_root_refresh();
  RefreshResult<std::string const&> current_directory_refresh();
};
}
// TuiFrontEnd:
app::frontend::Directories directories_;
```

The whole Directories value is copied. workspace_root_ formats snapshot.workspace_dir; current_directory_ formats snapshot.current_dir; additional_writable_dirs is retained but initially undisplayed. No public workspace_root_path_refresh or separate path-change flag is proposed. Render-facing refresh results are display strings; a typed accessor needs a concrete cross-projection formatting dependency, like the existing numeric window used by SessionStats percentage, not a speculative consumer.

Git branch is a separate canonical backend observation as specified in section 4a below. It needs no public path-refresh accessor on Directories. Preserve the current observation moments rather than introducing a watcher or making rendering read the filesystem.

Writers: Session construction/open_at/replacement and any verified future directory mutation. Existing backend workspace_dir/current_dir reference-returning APIs require guard/value migration as with AgentSelection. Do not create a new cd/navigation feature. Test distinct workspace and current paths, detached copies and replacement. /05/03 owns implementation.

**Approved by user:** keep workspace_dir, current_dir and additional_writable_dirs together in Directories and copy the whole value into directories_snapshot_, displaying only root/current paths initially. This is a semantic grouping decision, not a blanket preference for wrapping undisplayed data. Additional writable paths belong to backend directory configuration; weigh their actual mutation/copy cost and contention, rather than grouping solely for appearance. No additional writable-path display accessor is implied.

## 4a. Git branch — user-directed migration of the existing observed string

```cpp
namespace ava::app::runtime {
using GitBranch_ts = UnlockedData<std::string>;
}
// Session_aggregate_base:
GitBranch_ts git_branch_;

namespace ava::app::frontend {
class GitBranch {
  FrontEnd const* frontend_;
  DataListIndex index_;
  std::atomic_bool dirty_{false};
  std::string git_branch_snapshot_;
  DisplayString git_branch_;
  void refresh();
 public:
  void initialize(DataListIndex index);
  void dirty();
  RefreshResult<std::string const&> git_branch_refresh();
};
}
// TuiFrontEnd:
app::frontend::GitBranch git_branch_;
```

Canonical value is the existing observed branch string, not a new Git metadata struct. The projection copies the entire std::string into git_branch_snapshot_; its DisplayString owns the displayed representation and independent changed flag. Empty string retains the current unavailable/no-branch fallback; do not add an optional or change Git detection semantics in this migration.

Move the existing branch-reading helper (currently git_branch_for_sidebar in app/tui/interactive_tui_support.cpp) to app/runtime, removing its sidebar-specific naming. A backend refresh operation reads the Session workspace root, observes the branch outside backend locks, and publishes through git_branch().change(...). Preserve the same observation moments: initial TUI setup and every path currently constructing TuiRuntimeStateSnapshot. During incremental migration those callsites can invoke the backend refresh; after removing the bulk snapshot retain a shared app/backend refresh hook at those same action/completion points, not duplicated branch-reading code. If observation can race replacement, validate the captured session/workspace before publishing into the current object. No new watcher, timer or broader polling policy is required.

Remove TuiRuntimeStateSnapshot::git_branch and the duplicate initial TuiRuntimeOptions::git_branch transfer. Sidebar and drawer consume the frontend GitBranch::git_branch_refresh() result instead of storing another authoritative sidebar copy. Reading/observing Git is backend runtime work; the frontend accessor only snapshots/formats the already-observed value. A change in the canonical string marks this projection dirty, independently of directory changes. Preserve normal registered-address replacement and unregistered detached-copy rules. Implementation belongs to /05/03, not the shared model/stats foundation patch.

## 5. Prompt metadata: recommended wrapper around the EXISTING backend bundle

**User decision: option 3 is selected.** Nest PromptModeAndContextSources_ts inside ResolvedPromptState. There is exactly one backend mode and context_sources vector; the frontend copies mode and vector size only. Options 1 and 2 below are rejected alternatives retained for reference, not implementation instructions. This decision approves this type arrangement, not the entire proposal or the start of coding.

**Rejected option 1 (reference only): wrap the complete existing bundle.**

```cpp
namespace ava::app::runtime {
struct ResolvedPromptState {
  ava::agent::Mode mode = ava::agent::Mode::Build;
  BasePromptMetadata base_prompt;
  std::vector<ContextSourceMetadata> context_sources;
  std::vector<FreshnessSourceMetadata> freshness_sources;
  std::string system_prompt;
  std::string ambient_extension_free_system_prompt;
  AVA_DEBUG_PRINT_MEMBERS_OPT_OUT
};
using ResolvedPromptState_ts = UnlockedData<ResolvedPromptState>;
}
```

Fields above match the existing structure in Session.h:91–100; the intentional proposed change is its wrapper and debug opt-out, not extra prompt copies. Replace `ResolvedPromptState resolved_prompt_state_` with `ResolvedPromptState_ts resolved_prompt_state_`. Keep transient PromptState returned by load_runtime_prompt_state unchanged.

**Rejected option 2 (reference only): move the observed fields to a separately wrapped aggregate member.**

```cpp
namespace ava::app::runtime {
struct PromptModeAndContextSources {
  ava::agent::Mode mode = ava::agent::Mode::Build;
  std::vector<ContextSourceMetadata> context_sources;
  AVA_DEBUG_PRINT_MEMBERS_OPT_OUT
};
using PromptModeAndContextSources_ts = UnlockedData<PromptModeAndContextSources>;

struct ResolvedPromptState {
  BasePromptMetadata base_prompt;
  std::vector<FreshnessSourceMetadata> freshness_sources;
  std::string system_prompt;
  std::string ambient_extension_free_system_prompt;
  AVA_DEBUG_PRINT_MEMBERS_OPT_OUT
};
// Session_aggregate_base members:
// ResolvedPromptState resolved_prompt_state_;
// PromptModeAndContextSources_ts mode_and_context_sources_;
}
```

This alternative would remove mode and context_sources from ResolvedPromptState and keep the wrapper beside it. It is not selected; use the nested layout below instead.

**Selected option 3: keep the grouping, nest the wrapped observed fields.**

```cpp
namespace ava::app::runtime {
struct PromptModeAndContextSources {
  ava::agent::Mode mode = ava::agent::Mode::Build;
  std::vector<ContextSourceMetadata> context_sources;
  AVA_DEBUG_PRINT_MEMBERS_OPT_OUT
};
using PromptModeAndContextSources_ts = UnlockedData<PromptModeAndContextSources>;
struct ResolvedPromptState {
  PromptModeAndContextSources_ts mode_and_context_sources;
  BasePromptMetadata base_prompt;
  std::vector<FreshnessSourceMetadata> freshness_sources;
  std::string system_prompt;
  std::string ambient_extension_free_system_prompt;
  AVA_DEBUG_PRINT_MEMBERS_OPT_OUT
};
}
```

Session_aggregate_base retains its unwrapped resolved_prompt_state_ member, with the nested canonical object's final address registered. Assignment-based updates must preserve that registered address; do not reconstruct/move-construct a registered nested object. Option 3 keeps conceptual prompt-state grouping while avoiding wrapping prompt strings, and needs the same guarded access for mode/context_sources as option 2.

**Identical frontend value shape for all three options:**

```cpp
namespace ava::app::frontend {
class PromptMetadata {
  ava::agent::Mode mode_snapshot_ = ava::agent::Mode::Build;
  std::size_t context_source_count_snapshot_ = 0;
  DisplayString mode_;
  DisplayString context_source_count_;
 public:
  RefreshResult<std::string const&> mode_refresh();
  RefreshResult<std::string const&> context_source_count_refresh();
};
}
// TuiFrontEnd:
app::frontend::PromptMetadata prompt_metadata_;
```

mode_snapshot_ copies the single canonical mode. context_source_count_snapshot_ copies the result of canonical context_sources.size() under the same guard; it is frontend derived state, not a second backend count. Initialization/first refresh precedes display; an empty vector yields known zero, not unknown. Full context vectors, prompt strings, source paths, freshness arrays and base-prompt bodies are NOT copied. Consumers receive only formatted mode and decimal count strings. No typed value accessors or extra consumption flags are proposed.

Update construction, apply_prompt_state and the direct assignment currently in switch_model (Session.cpp:993–1000,1059–1064) for the nested arrangement, sharing the actual update path instead of duplicating mutation logic across commands. Only mode/context_sources access moves to the new lock. Prompt strings and other existing metadata retain their Session-level protection. This work belongs to /05/03, not the shared model/stats foundation slice.

When vector content changes but its size does not, notification may cause a refresh whose count comparison finds no display change. Accept that small redundant check; it does not justify a separately maintained count, content hash, second authority or more complex notification scheme. Implement only the selected nested wrapper, after the overall coding approval gate.

**Debug prerequisite:** current UnlockedData::change logs `*write_access`. OPT_OUT alone may not provide a usable safe printer for this instantiation. Before wrapping a prompt/agent/trust aggregate, provide bounded non-content debug metadata or remove generic full-value logging as approved; never solve compilation by enabling full prompt printing. This is not permission to log sensitive strings.

Safe overview resource facts beyond mode/count remain a separate unresolved choice in section 9; copying raw source metadata to TUI is not approved by this wrapper.

## 6. Trust — explicitly excluded from wrapping

User decision: do not wrap TrustState in UnlockedData. Keep existing TrustState/ProjectTrustState storage, synchronization, persistence and backend policy unchanged. The identified compact summary is shown in /settings -> Workspace, not an always-visible composer/sidebar field. Preserve its existing on-demand delivery and other existing trust presentation paths with the deferred interactions.

No TrustState_ts, frontend::ProjectTrust projection, trust snapshot/DisplayString members, new trust refresh accessors or registration index is proposed. This excludes both the earlier per-field display proposal and the later whole-value snapshot suggestion. When backend trust changes cause the selected agent or other separately approved canonical values to change, publish those values through their own existing/approved paths; that does not require wrapping trust itself. Do not remove trust enforcement or UI functionality to satisfy a blanket snapshot-removal rule.

## 7. Session identity — explicit derived observation candidate, not store replacement

**User-approved scope:** session ID/path in the sidebar/drawer are live display data, equivalent to status-bar fields, and require canonical UnlockedData-backed observation plus frontend DisplayStrings. Do not defer them as modal content. Backend /commands publish canonical changes without manually updating a particular UI. This settles the display requirement only; the duplicate-observation candidate below remains unapproved pending a storage decision that preserves one authoritative identity.

The store owns persistent identity/path; wrapping SessionStore would expose authority/handles and history machinery. For UI transition detection I propose a small backend-published observation instead, with this duplication explicitly acknowledged:

```cpp
namespace ava::app::runtime {
struct SessionIdentity {
  std::string session_id;
  std::filesystem::path session_path;
  bool sessionless = false;
  AVA_DEBUG_PRINT_MEMBERS_OPT_OUT
};
using SessionIdentity_ts = UnlockedData<SessionIdentity>;
}
```

Add `SessionIdentity_ts session_identity_` to Session_aggregate_base. Do not remove identity/path from SessionStore, and do not give independent setters for these fields. One backend construction/replacement publication helper derives them from the installed store and effective sessionless state. Detached construction derives a fresh unregistered observation for its copied store. Before successful action return it must match actual authority; no UI callback publishes it. Empty strings in default-constructed storage are not an authorized session; initialize before frontend registration. An ephemeral session may have an ID but no persistent path, so sessionless is explicit.

```cpp
namespace ava::app::frontend {
class SessionIdentity {
  runtime::SessionIdentity identity_snapshot_;
  DisplayString session_id_;
  DisplayString session_path_;
 public:
  RefreshResult<std::string const&> session_id_refresh();
  RefreshResult<std::string const&> session_path_refresh();
};
}
// TuiFrontEnd:
app::frontend::SessionIdentity session_identity_;
```

The whole value is copied internally; the projection exposes only session_id_refresh() and session_path_refresh() display strings. No other projection has an established cross-projection formatting dependency on the complete SessionIdentity, so no identity_refresh() or aggregate change flag is proposed. The previously suggested typed accessor was for controller transition detection, not display-string composition, and is withdrawn. Preserve existing transition behavior while removing old ordinary-state transport; decide its minimal control integration separately rather than using a speculative typed projection accessor or treating display text/its consumed changed flag as backend transition authority. No canonical draft, grants, transcript or selector state belongs in this object, and no new transition generation is proposed.

**Decision required:** approve this narrowly derived identity observation, or prefer a different authoritative store-identity factoring. Unlike AgentSelection/Directories this is not simply moving existing fields. Keeping a general runtime snapshot solely for identity is not the desired final solution, but wrapping the store/history is not proposed. /05/03 implements only the chosen approach.

## 8. Shared declarations, bindings and frontend ownership

### Header layout proposal

Keep existing namespaces; neutral means value declarations without Session/process/store implementation authority, not renaming all types into frontend.

| Proposed header | Contents / consumers |
|---|---|
| app/runtime/ModelSelection.h | Extract canonical definition from Session.h; place ModelSelection_ts alias immediately below it in the same header |
| session/Stats.h | Existing session::Stats and session::EntryCounts values plus compute_session_stats API; no app-level wrapper dependency |
| app/runtime/SessionStats.h | Existing runtime::SessionStats_ts = UnlockedData<session::Stats>; includes UnlockedData.h and session/Stats.h. Preserve this user-established placement |
| app/runtime/AgentSelection.h | Actual canonical selection followed immediately by AgentSelection_ts alias; full AgentDefinition not copied frontend |
| agent/agent_definition_types.h | Extract existing AgentToolPreset and AgentDefinitionProvenance enums without changing values; frontend metadata snapshots and agent_config.h include it |
| app/runtime/Directories.h | Canonical value followed immediately by Directories_ts alias |
| app/runtime/git_branch.h/.cxx | GitBranch_ts string-wrapper alias in the ordinary branch header and existing observation/update implementation moved into runtime; no separate _ts header |
| app/frontend/GitBranch.h/.cxx | Exact string snapshot, DisplayString and frontend refresh accessor |
| app/runtime/PromptModeAndContextSources.h | Canonical mode/vector followed immediately by its wrapper alias; registered object is nested inside ResolvedPromptState |
| app/runtime/ResolvedPromptState.h | Backend bundle containing mode_and_context_sources wrapper plus existing other metadata/prompt strings; no ResolvedPromptState_ts alias. Frontend PromptMetadata header needs only mode and size_t value declarations |
| app/runtime/SessionIdentity.h | Candidate observation followed immediately by SessionIdentity_ts alias, subject to section 7 approval |
| app/frontend/AgentSelection.h/.cxx, Directories.h/.cxx, PromptMetadata.h/.cxx, SessionIdentity.h/.cxx | Frontend snapshots/formatting and protected read implementations |

**User decision: no per-domain `_ts.h` split.** For runtime-owned canonical structs, put `using Foo_ts = UnlockedData<Foo>;` immediately below the definition. Statistics now have an explicitly chosen cross-layer arrangement: reusable session::Stats stays in session/Stats.h, while runtime::SessionStats_ts lives in app/runtime/SessionStats.h. Do not move that alias back into the session layer or reopen the solved dependency issue. UnlockedData.h no longer includes FrontEnd.h; the earlier hypothetical circular-include rationale is withdrawn. The goal remains avoiding all of Session.h just to name a domain type, not mechanically splitting every value and alias.

All sketches omit implementation details of constructors/debug markers for frontend classes, not an intent to bypass repository C++ rules. Every actual class will retain the applicable debug marker, and .cxx code includes sys.h first. Sensitive canonical types need safe custom printing or a non-content generic notifier log; no generated prompt logging.

### Registration proposal: pass indices, not Session guards

Do not create another registry. Use existing DataListIndex and backend registration. Proposed shared handle-only binding value (shown at final core size; foundation implements only existing model/stats fields):

```cpp
namespace ava::app::frontend {
struct FrontendDataIndices {
  DataListIndex model_selection;
  DataListIndex session_stats;
  DataListIndex agent_selection;
  DataListIndex directories;
  DataListIndex git_branch;
  DataListIndex mode_and_context_sources;
  DataListIndex session_identity;
  AVA_DEBUG_PRINT_MEMBERS_OPT_OUT
};
}
```

Replace the frontend virtual initializer's Session read-guard parameter with `FrontendDataIndices const&`. Backend registration remains responsible for selecting/registering the stable canonical instances; it fills this indices struct after registration and calls the frontend initializer. FrontEnd.h should not expose session_ts/SessionConstReadAccess. Move its session-registration orchestration to an app/backend adapter with narrowly scoped access to the existing register method; preserving the current Badge restriction requires an explicit approved friend/binding arrangement, not making arbitrary registry mutation public to every runtime caller.

Each projection uses `void initialize(DataListIndex index)` plus its existing FrontEnd pointer. Its .cxx looks up Data, checks the expected wrapper type and copies under that wrapper's read guard. The header's refresh accessors touch only local dirty/cache state and call private out-of-line refresh. TuiFrontEnd::data_changed compares registered indices and marks its corresponding projection dirty, rather than naming runtime wrapper types or dereferencing Session. This allows TUI headers to stop depending on Session without another subscription system.

Common per-projection members are `FrontEnd const* frontend_`, `DataListIndex index_`, `std::atomic_bool dirty_{false}` (or the existing domain-specific index name) and the exact snapshots/displays listed above. initialize requests initial refresh; calls before initialization remain programmer errors. SessionStats additionally retains ModelSelection* for its numeric dependency. No data copy is performed by notification and no terminal operation occurs there.

Concrete binding declaration proposal (replacing the current frontend Session-taking registration method):

```cpp
// FrontEnd.h: only a forward declaration of the backend binder, no session_ts.
namespace ava::app::runtime { class SessionFrontendBinding; }
namespace ava::app::frontend {
class FrontEnd {
  friend class ava::app::runtime::SessionFrontendBinding;
  DataListIndex connect_canonical_data(Data& data);
  // Implementation invokes Data::register_with_front_end with FrontEnd's badge.
 public:
  virtual void initialize_frontend_state(FrontendDataIndices const& indices) = 0;
  // Existing registration storage, data_changed and get_data remain.
};
}
// Separate backend-only runtime/SessionFrontendBinding.h:
namespace ava::app::runtime {
class SessionFrontendBinding {
 public:
  static void initialize(session_ts& session, frontend::FrontEnd& frontend);
  AVA_DEBUG_PRINT_MEMBERS_OPT_OUT
};
}
```

SessionFrontendBinding's .cxx takes the Session write guard, calls the private connect_canonical_data for each final canonical member, fills FrontendDataIndices and invokes the neutral initializer. connect_canonical_data returns the registered object's index. The existing direct Data-constructor registration and registry append mechanics remain available under their existing contract; this adapter does not create another registry. The backend-only binder header includes session_ts and forward-declares FrontEnd, while FrontEnd.h only forward-declares the binder class. This preserves the badge restriction without exposing Session in TUI projection headers.

This exact friend/binder choice requires approval along with the value shapes. It does not remove the Session lock around setup or permit runtime re-registration. A handle-only struct is not a bulk state snapshot: it carries no values and is used only for setup. Do not add handles for unimplemented domains in the foundation patch.

### Final core TuiFrontEnd member map

```cpp
app::frontend::ModelSelection model_selection_; // existing
app::frontend::SessionStats session_stats_;     // existing
app::frontend::AgentSelection agent_selection_; // proposed
app::frontend::Directories directories_;       // proposed
app::frontend::GitBranch git_branch_;           // user-directed observed string migration
app::frontend::PromptMetadata prompt_metadata_;// proposed
app::frontend::SessionIdentity session_identity_; // proposed, explicit decision
```

Construct projections with this; SessionStats also receives &model_selection_. Do not instantiate all proposed members in /05/02 merely because they are approved; /03 and /04 add their respective domains. Runtime private UI state is not moved into these projections.

## 9. Later ordinary domains — unresolved declarations, explicit blockers

The earlier plan included catalogs, safe resource overview and todos. I do not recommend pretending their exact types are settled by the seven core sketches. The current structures mix backend facts with presentation or contain large/authority-bearing aggregates. The following decisions require a supplementary concrete declaration proposal before their implementation; approval of sections 1–8 is not blanket approval for unspecified types.

| Domain / current source | Concrete choice to resolve | Proposed ownership direction / blocked slice |
|---|---|---|
| Safe overview resources: ResolvedPromptState context_sources/freshness_sources; startup_overview.cpp safe filtering/aggregation | Copy which already-safe facts versus store a backend-prepared safe resource summary? Exact group/name/count/lower-bound fields and limits must be specified without copying source paths or prompt-derived text. Wrapping ResolvedPromptState does not authorize copying its vectors | Backend owns resource facts/filtering; frontend owns overview theme/key hints/formatting. /04 resource-overview portion blocked pending supplement; mode/count proposal above can proceed independently |
| Available model/provider catalog and scoped cycling list | Explicitly deferred interactive /models and /scoped-models popups. Keep existing discovery, list delivery and scope actions; no new canonical catalog/scope snapshot or refresh accessor for these popups | Not a type-proposal blocker for this refactor. Current selected model/provider/reasoning remains in scope via existing ModelSelection, independently of popup list delivery |
| Slash/workspace completion catalogs: ApplicationCatalogCache | Existing SlashCommandCatalogItem/FileReferenceCatalogItem may carry UI hotkeys/formatting; exact records after separating local commands/hints are not yet approved. Preserve absent/empty/error and generation semantics without a generic catalog template invented on the fly | Backend factual discovery, frontend-only commands/hints/query; /05 completion portion blocked pending record declarations and bounded-copy review |
| Session selector: ApplicationCatalogCache::session_tree | SessionTreeIndex/SessionSummary can contain backend authority-related data. Select exact safe metadata and identity fields; determine acceptable bounded copy cost without designing history access | /05 session-list portion blocked. Do not wrap/copy entire ApplicationCatalogCache or SessionTreeIndex merely to eliminate SelectListView |
| Todos: latest committed todowrite state | Choose reusing agent::TodoItem value records versus a session-owned neutral todo value declaration, and whether absence/load error remains the current empty-list behavior or has explicit observation state. Specify publication owner replacing hydration and any overlap with retained live events | /05 todo portion blocked; no second writable todo history. Existing history/event delivery stays intact until that reviewed ordinary projection can be fed coherently |

These are material type/ownership decisions, not implementation details to delegate unseen. Options for user review: (A) approve the core model/stats/agent/directories/prompt/trust/identity design first and require per-domain supplements before /04–05 affected slices; or (B) require all supplements before authorizing even foundation coding. Either is compatible with the no-coding-before-approval rule. The current review gate must record which scope was approved and which remains blocked.

Git branch ownership is settled by user direction: use GitBranch_ts = UnlockedData<std::string> in Session and preserve existing observation moments, as in section 4a. No generic live-run/status object is proposed: protocol-specific run/interaction state is deferred. Independent future counters can be added only after their actual backend owner/type is reviewed.

## 10. Explicit exclusions

No canonical Theme, KeyBindings, CursorStyle, Layout, Draft, Preview or SelectListView. No snapshot copy of SessionStore, complete session history, transcript/tool event stream, permission/question queue, plugin endpoint/invocation state, SubagentInspectorFrame or branch-confirmation operation. No copying agent instructions or assembled system prompts to frontend snapshots. Existing request/reply, history-picker, image/editor/Mermaid service APIs stay intact, including synchronization they already require.

Frontend-local theme/binding/settings ownership can be corrected without new canonical type approval; its implementation must still respect the reviewed /04 scope and not silently redesign deferred services.

## 11. Review checklist and implementation allocation

Approval questions:

1. Settled by user: AgentSelection contains a nonoptional actual selected definition, resolved explicitly or by default before composer input; frontend SelectedAgentMetadata is also nonoptional. Requested CLI intent remains backend configuration, not a snapshot field. Review any missing concrete default-name/configuration policy separately.
2. Settled by user: move workspace/current/additional writable paths into Directories, wrap it and copy the whole value; display root/current paths only for now. Approval is based on semantic cohesion versus mutation/copy/locking cost, not a rule to wrap all undisplayed fields.
3. Settled by user: option 3, PromptModeAndContextSources_ts nested in ResolvedPromptState. Register the nested stable object and copy only mode/vector size to frontend; other prompt fields retain existing protection.
4. Settled by user: no UnlockedData wrapper or frontend projection for TrustState; preserve existing trust behavior/presentation paths.
5. Add narrowly derived SessionIdentity observation maintained at actual store-install points, or request a different identity factoring?
6. Settled by user: no per-domain _ts.h split; runtime-owned structs have adjacent aliases. Statistics specifically retain session::Stats/EntryCounts in session/Stats.h and runtime::SessionStats_ts in app/runtime/SessionStats.h, per c0b9b06e. Neutral index-only initialization remains a separate proposal for review; these placement decisions do not approve that binding API.
7. Review core now with later catalog/resource/todo supplements, or complete every supplementary type proposal before any coding?

Existing ModelSelection/SessionStats value shapes and outputs are not otherwise redesigned. Foundation /05/02/03 implements only approved shared model/stats header/binding work and relevant safe logging/documentation corrections; /05/03 implements approved agent/directory/identity/prompt facts; /05/04 handles approved trust/local settings/resource portion; /05/05 requires the additional catalog/todo decisions; /05/06 changes statistics publication ownership, not its shape; /05/07 preserves interfaces; /05/08 deletes superseded ordinary transports; /05/09 verifies scoped behavior.

For every approved wrapper change, tests must prove canonical ownership has moved rather than been duplicated, all real writers notify, detached construction is unregistered, assignment preserves registered addresses, and backend getters cannot escape guards. Projection tests cover exact source mapping, absence/zero semantics, unchanged text with changed metadata, numeric/string consumption order and replacement. Add tests for actual next-prompt selection and distinct directories. No special lifecycle framework is needed to review these types; verify existing normal-runtime lifetimes and fix concrete problems locally.

This proposal is ready for discussion but intentionally reports unresolved later-domain choices. It is not authorization to code, and it does not mark the review gate approved.
