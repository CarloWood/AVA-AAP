# Incremental proposal review

Review target: ../01-propose-canonical-and-frontend-snapshot-types/Proposal.md, including the corrections made during the user's earlier review. The user requests one question at a time, with feedback resolved before moving to the next topic. Do not present the full checklist as a batch or treat completion of proposal writing as approval.

## Explicit decisions already recorded in the conversation

* Selected agent is a real nonoptional canonical definition, resolved before composer input; default selection is performed by the backend, not invented as a frontend fallback. SelectedAgentMetadata is nonoptional, uses selected/selection terminology, and omits requested CLI intent from frontend snapshots.
* Prompt state uses option 3: nested PromptModeAndContextSources_ts, one canonical mode/vector, frontend copies mode/vector size only.
* Git branch uses runtime GitBranch_ts = UnlockedData<std::string>, preserving existing observation moments, with frontend string snapshot/DisplayString.
* No speculative scope-vector, path, mode/count or identity typed refresh accessors; the established numeric context-window dependency remains.
* Model/scoped-model popups and other interaction/history interfaces remain deferred. UI-only state stays tui-thread-local, not UnlockedData.
* No per-domain _ts.h split. User's c0b9b06e establishes session::Stats/EntryCounts in session/Stats.h and runtime::SessionStats_ts in app/runtime/SessionStats.h.

These record specific user instructions, not blanket approval of the entire proposal. The concrete default-agent policy, if not already defined in backend configuration, still needs resolution before implementing it.

## Approved: directories

User explicitly answered "Yes" to moving workspace_dir, current_dir and additional_writable_dirs from InvocationInputs into one runtime::Directories canonical struct wrapped in Directories_ts, with frontend::Directories holding an exact whole-value directories_snapshot_. Only workspace/current directory display strings are initially proposed; additional writable paths are retained but not displayed. No duplicate backend path fields, path-refresh accessor or new permission authority.

Approval rationale is semantic grouping weighed against synchronization cost, not an assumption that every backend member should be in an observable wrapper. When only X contributes to frontend snapshots and Y does not, UnlockedData gives Y no frontend-observation benefit. Updating Y can nevertheless notify the frontend and cause unnecessary X copying/comparison; reads/writes of Y also take the same mutex and can increase contention/lock duration. Consider mutation frequency, size/work under lock, expected contention and whether X/Y genuinely belong together in backend design. Prefer splitting if it improves backend structure or materially avoids these costs; do not split solely because a semantically related member is not currently displayed. Other backend concurrency obligations are separate from whether frontend observation requires the wrapper.

In this particular approved whole-value Directories snapshot, additional_writable_dirs is actually copied too, though not displayed; its copying cost must also be considered, rather than calling it an uncopied Y field. The approval does not establish a general rule that every projection copies its whole canonical object, or authorize grouping unrelated frequently updated/large fields for cosmetic reasons.

User clarified the practical weighting: updates are expected to be at least seconds apart, with no noticeable contention. Default to keeping a naturally cohesive backend struct together when splitting it would be awkward, even if some members are never displayed. Do not turn the theoretical cost model into a requirement for fine-grained locks, measurements or additional struct splits. Backend readability and intuitive semantic grouping dominate unless a concrete cost or independent responsibility warrants separation. Present genuinely debatable grouping choices for user judgment rather than mechanically separating displayed from undisplayed fields.

## Decided: trust excluded

After verifying that the proposed summary is the subtitle of /settings -> Workspace, the user explicitly instructed: "lets not wrap TrustState in an UnlockedData." Withdraw both the earlier per-field projection and the whole-value snapshot suggestion. Preserve existing trust storage/policy and on-demand presentation; no new TrustState_ts, projection, display strings or registration index. Independently approved agent/model facts still publish when a trust operation changes them.

## Approved scope: live sidebar and drawer data

User explicitly classifies sidebar/drawer fields as live status data, not modal popup content. Backend-originating changes (including backend /commands) update canonical UnlockedData; frontend projections refresh snapshots and DisplayStrings without command handlers knowing which frontend is present. Session ID/path displayed there are in scope. This approves the live-data requirement, not the previously proposed duplicate SessionIdentity backend observation or any particular storage factoring; that ownership choice remains to be settled.

User's distinction: on-demand modal content is obtained on frontend initiative, then represents a local candidate selection/edit until Apply/OK; cancel must not mutate canonical state. Passive persistent status/sidebar data instead observes canonical updates. A backend command triggered by a user is still a backend mutation, not a reason to manually update the UI. Pure UI commands/preferences remain local. Existing concurrent/deferred popup protocol behavior is preserved; this classification does not claim that all backends stop while a modal is open.

## Remaining review, not yet presented for decision

Session ID/path canonical storage without duplicate authority (live display requirement now approved); necessity and shape of proposed registration API changes; scope of later supplementary type proposals; bounded foundation coding authorization. Address these separately, without reopening explicit decisions absent new evidence.

## Authorization

Review in progress. No new /05 implementation is authorized by this record. Previously requested standalone header extraction was a separate explicit coding task, not approval of this migration.
