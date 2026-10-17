# Canonical and frontend snapshot types — proposal for review

## Status and boundaries

**Proposal, not approved implementation.** No source files, builds or configuration were changed. Declarations below are sketches for review, not generated source. Source inspection used the current working tree: Session.h, Session.cpp mutation sites, frontend ModelSelection/SessionStats, session/stats.h, runtime/ReasoningSelection.h, agent/agent_config.h, project_trust.h and command_palette.h. Earlier audit Result.md files supply producer/consumer and verification inventories.

I recommend preserving the existing pattern: wrap actual backend-owned cohesive data, copy only the required value members into a frontend projection, and format there. Do not create a second writable agent/model/mode/directory selection solely for UI consumption. No generic property system or registry replacement.

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

For the later scope selector, candidate addition **requiring approval with that slice**:

```cpp
std::optional<std::vector<std::string>> scoped_model_cycle_snapshot_ = std::nullopt;
bool scoped_model_cycle_changed_{true};
RefreshResult<std::optional<std::vector<std::string>> const&> scoped_model_cycle_refresh();
```

This copies the existing optional vector (absence retains existing no-explicit-scope semantics). It is not a new writable scope. Caller is the local scope-selector controller, not SessionStats. Catalog availability still requires the later catalog decision; this addition alone does not implement a selector.

## 2. Existing SessionStats — keep canonical statistics and whole snapshot

The current source includes recently added equality operators. Preserve them. Nested counters in session/stats.h:

```cpp
namespace ava::session {
struct SessionEntryCounts {
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
  bool operator==(SessionEntryCounts const& other) const = default;
  AVA_DEBUG_PRINT_MEMBERS_ON
};
struct SessionStats {
  std::size_t entry_count = 0;
  std::string first_timestamp;
  std::string last_timestamp;
  SessionEntryCounts counts;
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
  bool operator==(SessionStats const& other) const = default;
  AVA_DEBUG_PRINT_MEMBERS_ON
};
}
namespace ava::app::runtime {
using SessionStats_ts = UnlockedData<ava::session::SessionStats>;
}
```

Default-constructed optionals are disengaged. Active history and system-prompt tokens remain separate members of this one object, published together. These counters are not the history collection itself.

Backend storage remains `SessionStats_ts session_stats_`. Frontend remains:

```cpp
ava::session::SessionStats stats_snapshot_;  // Copy of entire canonical value.
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
  std::optional<std::string> requested_primary_agent = std::nullopt;
  std::optional<ava::agent::AgentDefinition> selected_primary_agent = std::nullopt;
  AVA_DEBUG_PRINT_MEMBERS_OPT_OUT
};
using AgentSelection_ts = UnlockedData<AgentSelection>;
}
```

Proposed backend storage: `AgentSelection_ts agent_selection_` in Session_aggregate_base. **Remove** the two corresponding members from InvocationInputs. Backend resolution/dispatch access the wrapped selection, not retained originals. OpenContext and transient resolution candidates may still carry input copies; they are not live competing authorities.

AgentDefinition is the existing type in agent/agent_config.h with name, description, system_prompt, tool_preset, max_tool_iterations, hidden, provenance and path. It has no model/provider member. Keep that existing definition and its safety-oriented printer. Do not wrap it alone: requested intent and effective selection form one backend group, especially when trust causes an effective selection to disappear.

Proposed frontend class and value members:

```cpp
namespace ava::app::frontend {
class AgentSelection {
  std::optional<std::string> requested_primary_agent_snapshot_ = std::nullopt;
  std::optional<std::string> selected_name_snapshot_ = std::nullopt;
  std::optional<ava::agent::AgentDefinitionProvenance> provenance_snapshot_ = std::nullopt;
  std::optional<ava::agent::AgentToolPreset> tool_preset_snapshot_ = std::nullopt;
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

These are selected-field copies from one locked AgentSelection read, not a second canonical AgentIdentity. Mapping:

* requested_primary_agent_snapshot_ <- canonical requested_primary_agent.
* selected_name_snapshot_ <- selected_primary_agent->name if present, otherwise nullopt.
* provenance_snapshot_ <- selected_primary_agent->provenance with the same presence rule.
* tool_preset_snapshot_ <- selected_primary_agent->tool_preset with the same presence rule.

Do not copy agent system_prompt, source path, description or iteration policy into frontend just to show identity. Those fields remain in the canonical selected definition for backend behavior; a future visible field can add an explicit safe snapshot member. Do not expose the full AgentDefinition through a frontend accessor.

Proposed display semantics for review: no request/no resolved definition -> "default" (a display state, not a fabricated agent named default); requested but no effective definition -> "unavailable: <requested name>"; resolved -> its name. Unknown provenance remains the actual Unknown enum, not absence. Distinguish configured tool preset from effective permission/tool visibility: do not label the preset as the complete effective permission policy.

Writers: construction after primary resolution; command_trust's resolution/clearing paths; apply_trust_prompt_state_and_refresh; Session replacement. Detached state copies this canonical value into a fresh unregistered wrapper. Model switches and reload keep the already-resolved definition under current semantics. Session methods that currently return const references into InvocationInputs must become guard/value-based access; returning an unlocked reference from a temporary read guard is forbidden.

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
  bool workspace_root_path_changed_{true};
 public:
  RefreshResult<std::string const&> workspace_root_refresh();
  RefreshResult<std::string const&> current_directory_refresh();
  RefreshResult<std::filesystem::path const&> workspace_root_path_refresh();
};
}
// TuiFrontEnd:
app::frontend::Directories directories_;
```

The whole Directories value is copied. workspace_root_ formats snapshot.workspace_dir; current_directory_ formats snapshot.current_dir; additional_writable_dirs is retained but initially undisplayed. The typed path accessor exists only if the frontend's presentation-only Git observation actually needs it; one local controller owns its changed flag. If no typed consumer survives the implementation design, omit that accessor/flag rather than adding unused interface. It must not authorize backend tool access; backend continues validating against its roots/anchors.

Writers: Session construction/open_at/replacement and any verified future directory mutation. Existing backend workspace_dir/current_dir reference-returning APIs require guard/value migration as with AgentSelection. Do not create a new cd/navigation feature. Test distinct workspace and current paths, detached copies and replacement. /05/03 owns implementation.

**Review choice:** whole directories snapshot includes additional writable paths for future display. If that is not desired for frontend exposure, use two explicit path members instead and leave the additional vector uncopied; do not silently select that policy during coding.

## 5. Prompt metadata: recommended wrapper around the EXISTING backend bundle

Mode and context-source metadata currently belong to ResolvedPromptState, alongside assembled prompt strings. A separate writable Mode or ContextCount object would duplicate authority. My preferred candidate wraps the existing bundle but copies only small relevant values to frontend:

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

```cpp
namespace ava::app::frontend {
class PromptMetadata {
  ava::agent::Mode mode_snapshot_ = ava::agent::Mode::Build;
  std::optional<std::size_t> context_source_count_snapshot_ = std::nullopt;
  DisplayString mode_;
  DisplayString context_source_count_;
  bool mode_value_changed_{true};
  bool context_source_count_value_changed_{true};
 public:
  RefreshResult<std::string const&> mode_refresh();
  RefreshResult<std::string const&> context_source_count_refresh();
  RefreshResult<ava::agent::Mode> mode_value_refresh();
  RefreshResult<std::optional<std::size_t>> context_source_count_value_refresh();
};
}
// TuiFrontEnd:
app::frontend::PromptMetadata prompt_metadata_;
```

mode_snapshot_ copies canonical mode. context_source_count_snapshot_ is a **derived scalar observation**, computed as canonical context_sources.size() under the same guard, not a canonical count field. Nullopt means not yet observed; an initialized empty vector yields known zero. Full prompt strings, source paths, freshness arrays and base-prompt bodies are NOT copied. No history copy is involved. Structured values are consumed once by the local runtime presentation controller, strings by its rendering refresh pass.

All writers must go through one backend bundle publication: construction, apply_prompt_state and the direct replacement currently in switch_model (Session.cpp:993–1000,1059–1064). Avoid implementing mode publication separately in every command. Existing prompt getters currently relying on the Session lock need explicit guard/value access changes. This is a real cost of this proposal and belongs in /05/03, not the foundation slice.

**Alternative requiring a decision:** keep ResolvedPromptState protected only by Session and add a small read-only backend observation struct `{Mode mode; size_t context_source_count;}` maintained by one commit helper. This has lower accessor churn but duplicates derived metadata and requires reliable synchronization with all writers. I prefer wrapping the actual bundle to avoid that duplicate state. Approval must explicitly choose; do not implement both.

**Debug prerequisite:** current UnlockedData::change logs `*write_access`. OPT_OUT alone may not provide a usable safe printer for this instantiation. Before wrapping a prompt/agent/trust aggregate, provide bounded non-content debug metadata or remove generic full-value logging as approved; never solve compilation by enabling full prompt printing. This is not permission to log sensitive strings.

Safe overview resource facts beyond mode/count remain a separate unresolved choice in section 9; copying raw source metadata to TUI is not approved by this wrapper.

## 6. Trust: wrap existing authority value; copy selected presentation fields

Existing ProjectTrustState:

```cpp
namespace ava::app {
enum class ProjectTrustDecision { Unknown, Trusted, Denied };
struct ProjectTrustResource {
  std::string kind;
  std::filesystem::path path;
  AVA_DEBUG_PRINT_MEMBERS_ON
};
struct ProjectTrustState {
  std::filesystem::path workspace_dir;
  std::filesystem::path trust_file;
  ProjectTrustDecision decision = ProjectTrustDecision::Unknown;
  std::filesystem::path matched_path;
  std::vector<ProjectTrustResource> protected_resources;
  std::string diagnostic;
  AVA_DEBUG_PRINT_MEMBERS_ON
};
}
namespace ava::app::runtime {
struct TrustState {
  app::ProjectTrustState project_trust;
  AVA_DEBUG_PRINT_MEMBERS_OPT_OUT
};
using TrustState_ts = UnlockedData<TrustState>;
}
```

Proposed storage replaces the bare `TrustState trust_state_` with `TrustState_ts trust_state_`. Keep existing validated trust mutation/persistence; write notification is not approval authority. Backend trust getters need protected access rather than leaking a reference. No change to the deferred permission popup protocol.

```cpp
namespace ava::app::frontend {
class ProjectTrust {
  app::ProjectTrustDecision decision_snapshot_ = app::ProjectTrustDecision::Unknown;
  std::filesystem::path workspace_snapshot_;
  std::filesystem::path matched_path_snapshot_;
  std::filesystem::path trust_file_snapshot_;
  std::size_t protected_resource_count_snapshot_ = 0;
  std::string diagnostic_snapshot_;
  DisplayString decision_;
  DisplayString project_resources_;
  DisplayString workspace_;
  DisplayString matched_path_;
  DisplayString trust_file_;
  DisplayString protected_resource_count_;
  DisplayString diagnostic_;
 public:
  RefreshResult<std::string const&> decision_refresh();
  RefreshResult<std::string const&> project_resources_refresh();
  RefreshResult<std::string const&> workspace_refresh();
  RefreshResult<std::string const&> matched_path_refresh();
  RefreshResult<std::string const&> trust_file_refresh();
  RefreshResult<std::string const&> protected_resource_count_refresh();
  RefreshResult<std::string const&> diagnostic_refresh();
};
}
// TuiFrontEnd:
app::frontend::ProjectTrust project_trust_;
```

Each named snapshot field copies the same-named/obvious source member of canonical project_trust; resource count derives from protected_resources.size() under that read guard. Resource path vector is deliberately uncopied. project_resources_ derives enabled/skipped from the actual trust decision rule, not a new boolean authority. These path/diagnostic fields are already exposed by the existing trust UI; they must not be fed into the deliberately path-free overview. A future overview composes only its approved safe subset.

Trust workspace is the workspace evaluated by the trust record, not a competing active workspace_dir. Preserve this meaning even if the paths normally match. /05/04 owns implementation and all trust writers, including the direct assignments in command_trust and apply_trust_prompt_state_and_refresh.

## 7. Session identity — explicit derived observation candidate, not store replacement

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
  bool identity_changed_{true};
 public:
  RefreshResult<std::string const&> session_id_refresh();
  RefreshResult<std::string const&> session_path_refresh();
  RefreshResult<runtime::SessionIdentity const&> identity_refresh();
};
}
// TuiFrontEnd:
app::frontend::SessionIdentity session_identity_;
```

The whole value is copied; the local runtime transition controller owns identity_refresh's change consumption. It checks identity using existing semantics and calls one local transition-reset helper. Renderers use the separate string flags. No canonical draft, grants, transcript or selector state in this object. No new transition generation is proposed absent a demonstrated concurrency requirement.

**Decision required:** approve this narrowly derived identity observation, or prefer a different authoritative store-identity factoring. Unlike AgentSelection/Directories this is not simply moving existing fields. Keeping a general runtime snapshot solely for identity is not the desired final solution, but wrapping the store/history is not proposed. /05/03 implements only the chosen approach.

## 8. Shared declarations, bindings and frontend ownership

### Header layout proposal

Keep existing namespaces; neutral means value declarations without Session/process/store implementation authority, not renaming all types into frontend.

| Proposed header | Contents / consumers |
|---|---|
| app/runtime/ModelSelection.h | Extract existing canonical value declaration from Session.h; uses ModelInfo/ReasoningSelection, no Session |
| app/runtime/ModelSelection_ts.h | Alias + UnlockedData include; backend registration and frontend .cxx only |
| session/stats_data.h | Move SessionEntryCounts/SessionStats value declarations unchanged; session/stats.h retains compute API and includes stats_data.h. Avoid SessionStore transitively in projection header |
| app/runtime/SessionStats_ts.h | Existing wrapper alias; no need to include full Session |
| app/runtime/AgentSelection.h and AgentSelection_ts.h | Actual canonical selection and wrapper; full AgentDefinition used backend/.cxx, not copied frontend |
| agent/agent_definition_types.h | Extract existing AgentToolPreset and AgentDefinitionProvenance enums without changing values; frontend metadata snapshots and agent_config.h include it |
| app/runtime/Directories.h and Directories_ts.h | Path-only canonical value plus separately included wrapper |
| app/runtime/ResolvedPromptState.h and ResolvedPromptState_ts.h | Existing backend bundle and wrapper; frontend PromptMetadata header needs only core mode and std optional/size_t |
| app/project_trust_types.h | Existing trust enums/value structs separated from persistence/staging API; app/project_trust.h includes it |
| app/runtime/TrustState.h and TrustState_ts.h | Existing trust bundle and new wrapper |
| app/runtime/SessionIdentity.h and SessionIdentity_ts.h | Candidate observation and wrapper, subject to section 7 approval |
| app/frontend/AgentSelection.h/.cxx, Directories.h/.cxx, PromptMetadata.h/.cxx, ProjectTrust.h/.cxx, SessionIdentity.h/.cxx | Frontend snapshots/formatting and protected read implementations |

The per-domain `_ts.h` split is a **proposal to approve**, not a requirement of C++: it keeps UnlockedData/FrontEnd mechanics out of frontend public projection headers and avoids a circular include when they carry value snapshots. An acceptable reviewed alternative is a small backend wrapper-alias header grouping only these aliases. Do not include Session.h just to find an alias.

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
  DataListIndex resolved_prompt_state;
  DataListIndex project_trust;
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
app::frontend::PromptMetadata prompt_metadata_;// proposed
app::frontend::ProjectTrust project_trust_;     // proposed
app::frontend::SessionIdentity session_identity_; // proposed, explicit decision
```

Construct projections with this; SessionStats also receives &model_selection_. Do not instantiate all proposed members in /05/02 merely because they are approved; /03 and /04 add their respective domains. Runtime private UI state is not moved into these projections.

## 9. Later ordinary domains — unresolved declarations, explicit blockers

The earlier plan included catalogs, safe resource overview and todos. I do not recommend pretending their exact types are settled by the seven core sketches. The current structures mix backend facts with presentation or contain large/authority-bearing aggregates. The following decisions require a supplementary concrete declaration proposal before their implementation; approval of sections 1–8 is not blanket approval for unspecified types.

| Domain / current source | Concrete choice to resolve | Proposed ownership direction / blocked slice |
|---|---|---|
| Safe overview resources: ResolvedPromptState context_sources/freshness_sources; startup_overview.cpp safe filtering/aggregation | Copy which already-safe facts versus store a backend-prepared safe resource summary? Exact group/name/count/lower-bound fields and limits must be specified without copying source paths or prompt-derived text. Wrapping ResolvedPromptState does not authorize copying its vectors | Backend owns resource facts/filtering; frontend owns overview theme/key hints/formatting. /04 resource-overview portion blocked pending supplement; mode/count proposal above can proceed independently |
| Available model/provider catalog | Session current selection is authoritative, but registry discovery/provider availability currently occurs in selector builder. Choose the canonical owner/cache and exact model/availability record fields; no second mutable selected model | /05 model-list portion blocked. Scope-vector snapshot addition in section 1 is a concrete candidate, not a complete catalog |
| Slash/workspace completion catalogs: ApplicationCatalogCache | Existing SlashCommandCatalogItem/FileReferenceCatalogItem may carry UI hotkeys/formatting; exact records after separating local commands/hints are not yet approved. Preserve absent/empty/error and generation semantics without a generic catalog template invented on the fly | Backend factual discovery, frontend-only commands/hints/query; /05 completion portion blocked pending record declarations and bounded-copy review |
| Session selector: ApplicationCatalogCache::session_tree | SessionTreeIndex/SessionSummary can contain backend authority-related data. Select exact safe metadata and identity fields; determine acceptable bounded copy cost without designing history access | /05 session-list portion blocked. Do not wrap/copy entire ApplicationCatalogCache or SessionTreeIndex merely to eliminate SelectListView |
| Todos: latest committed todowrite state | Choose reusing agent::TodoItem value records versus a session-owned neutral todo value declaration, and whether absence/load error remains the current empty-list behavior or has explicit observation state. Specify publication owner replacing hydration and any overlap with retained live events | /05 todo portion blocked; no second writable todo history. Existing history/event delivery stays intact until that reviewed ordinary projection can be fed coherently |

These are material type/ownership decisions, not implementation details to delegate unseen. Options for user review: (A) approve the core model/stats/agent/directories/prompt/trust/identity design first and require per-domain supplements before /04–05 affected slices; or (B) require all supplements before authorizing even foundation coding. Either is compatible with the no-coding-before-approval rule. The current review gate must record which scope was approved and which remains blocked.

Git branch has only a located presentation helper, not established backend authority. No UnlockedData<GitBranch> is proposed; retain a tui-specific cached observation from the exposed workspace root if needed. No generic live-run/status object is proposed: protocol-specific run/interaction state is deferred. Independent future counters can be added only after their actual backend owner/type is reviewed.

## 10. Explicit exclusions

No canonical Theme, KeyBindings, CursorStyle, Layout, Draft, Preview or SelectListView. No snapshot copy of SessionStore, complete session history, transcript/tool event stream, permission/question queue, plugin endpoint/invocation state, SubagentInspectorFrame or branch-confirmation operation. No copying agent instructions or assembled system prompts to frontend snapshots. Existing request/reply, history-picker, image/editor/Mermaid service APIs stay intact, including synchronization they already require.

Frontend-local theme/binding/settings ownership can be corrected without new canonical type approval; its implementation must still respect the reviewed /04 scope and not silently redesign deferred services.

## 11. Review checklist and implementation allocation

Approval questions:

1. AgentSelection as the actual moved requested/resolved selection, with field-only frontend copies rather than AgentDefinition snapshot?
2. Directories as workspace/current/additional writable inputs moved out of InvocationInputs; whole frontend copy versus only root/current paths?
3. Wrap existing ResolvedPromptState, copying only mode/count for now, versus a small derived observation maintained by a commit helper? The former avoids duplicated mode/count but requires protected-access migration for existing backend getters.
4. Wrap existing TrustState and copy selected existing presentation fields, keeping protected resource paths uncopied?
5. Add narrowly derived SessionIdentity observation maintained at actual store-install points, or request a different identity factoring?
6. Approve value/wrapper header split and neutral index-only initialization boundary? Backend registration stays narrow; frontend never gets Session access.
7. Review core now with later catalog/resource/todo supplements, or complete every supplementary type proposal before any coding?

Existing ModelSelection/SessionStats value shapes and outputs are not otherwise redesigned. Foundation /05/02/03 implements only approved shared model/stats header/binding work and relevant safe logging/documentation corrections; /05/03 implements approved agent/directory/identity/prompt facts; /05/04 handles approved trust/local settings/resource portion; /05/05 requires the additional catalog/todo decisions; /05/06 changes statistics publication ownership, not its shape; /05/07 preserves interfaces; /05/08 deletes superseded ordinary transports; /05/09 verifies scoped behavior.

For every approved wrapper change, tests must prove canonical ownership has moved rather than been duplicated, all real writers notify, detached construction is unregistered, assignment preserves registered addresses, and backend getters cannot escape guards. Projection tests cover exact source mapping, absence/zero semantics, unchanged text with changed metadata, numeric/string consumption order and replacement. Add tests for actual next-prompt selection and distinct directories. No special lifecycle framework is needed to review these types; verify existing normal-runtime lifetimes and fix concrete problems locally.

This proposal is ready for discussion but intentionally reports unresolved later-domain choices. It is not authorization to code, and it does not mark the review gate approved.
