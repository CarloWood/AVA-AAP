These findings describe the **current working tree**, including the extracted `ModelSelection.cxx` and `SessionStats.cxx`, not just the committed revision.

# Reference-contract audit

## 1. Ownership and locking

The current ownership split matches the intended design:

- `Session` owns canonical `UnlockedData<runtime::ModelSelection>` and `UnlockedData<session::SessionStats>`.
- `TuiFrontEnd` owns separate frontend projection objects.
- Notifications carry a registration index—not a copy of the canonical state.
- Frontend projections acquire the canonical object’s read guard when refreshing.

Each `UnlockedData` has its own `std::mutex`. This is distinct from the enclosing Session mutex. Read access also locks that mutex; it is not a reader/writer lock.

**Sources:** `app/runtime/UnlockedData.h`, `UnlockedDataPolicy.h`, `Session.h`, `tui/TuiFrontEnd.h`.

## 2. What write notification guarantees

`change()` is a convenience around a notifying write-access guard. Direct `wat` usage also notifies.

The notifier’s inheritance order deliberately releases the corresponding canonical-data lock **before** calling:

```cpp
frontend_->data_changed(index_);
```

That callback runs synchronously on the thread destroying the write guard.

However, **other locks can still be held**:

- A surrounding Session write guard can remain held.
- During `UnlockedData` move assignment, the destination notification occurs while the source write guard still exists.

Consequently, notification handlers must remain bounded and avoid acquiring Session or unrelated canonical locks, blocking, or doing terminal work. The current `TuiFrontEnd::data_changed()` merely classifies the registered object and marks its projection dirty.

Additional properties:

- A write guard notifies even if nothing changed.
- There is no equality test or rollback in `change()`.
- If an update callback throws after partially modifying data, guard destruction still notifies.
- Notification is issued from a destructor; throwing from `data_changed()` is not a viable error-reporting mechanism.

**Sources:** `UnlockedDataPolicy.h:40–66`, `UnlockedData.h:107–114,139–174`, `frontend/Data.cxx:19–23`.

## 3. Registration and lifetime are contractual—not automatically enforced

Registration uses raw pointers:

- `Data` stores `FrontEnd*`.
- `FrontEnd` stores `Data const*` entries.

There is no unregister operation, weak ownership, in-flight notification barrier, or synchronization around registry mutation. The present design therefore requires:

1. Registration completed before concurrent frontend use.
2. Stable canonical addresses.
3. No subsequent registry mutation while it is being read.
4. Publishers quiescent before frontend destruction.

The actual application constructs Session first, with unregistered canonical members, then constructs `TuiFrontEnd`, registers those members, and runs the frontend. This is supported by the implementation, despite the older `Data.h` comment describing frontend construction before Data construction.

A Session surviving frontend termination still retains its frontend registration pointer. A later write would attempt to notify through that pointer. **That is a lifetime obligation to prove—not evidence that such a write currently occurs.**

Repeated attachment of the same canonical instance to another frontend is not supported by the one-time registration contract.

### Concrete documentation discrepancy

`UnlockedData`’s frontend-taking constructor comment says `nullptr` is allowed for headless modes. But `Data(FrontEnd*)` dereferences the pointer unconditionally.

Currently, headless use must take the **unregistered construction path**. The comment and implementation need reconciliation in the implementation phase.

**Sources:** `frontend/Data.h:23–60`, `Data.cxx:7–22`, `FrontEnd.cxx:17–25`, `app/tui/interactive_tui.cpp:385–387`.

## 4. Movement and session replacement

The implemented rules are:

| Operation | Effect |
|---|---|
| Move-construct unregistered data | Transfer value; new wrapper remains unregistered |
| Move-construct registered data | Forbidden by assertion |
| Move-assign into existing data | Preserve destination address, mutex and registration; transfer protected value |

`Session::replace_with()` assigns into the existing Session, so its registered canonical members keep their addresses and indices.

But replacement **does not atomically replace all frontend-visible state**. Individual canonical members are assigned and notified separately. Frontend readers acquire their individual canonical locks, not the Session lock, so they could observe different domains at different stages of a concurrent replacement.

Detached-state construction creates fresh unregistered wrappers from source values.

**Sources:** `UnlockedData.h:150–174`, `Session.cpp:713 onward,784–800`.

## 5. Dirty flags provide eventual latest-state observation

The public refresh methods use:

```cpp
if (dirty_.exchange(false))
  refresh();
```

This avoids losing a notification between separate testing and clearing:

- A notification before the exchange requests the current refresh.
- A notification after the exchange remains pending.
- A writer can unlock before notifying, so a refresh may briefly use older frontend state.
- A refresh may read newly written data before its notification arrives, causing a harmless additional refresh later.

The canonical mutex protects the values. The atomic dirty flag only signals invalidation.

**Important limitation:** this is not delivery of every mutation. Multiple changes can coalesce, including a change followed by a return to the original value before any refresh.

That is appropriate for ordinary displayed state, but insufficient by itself for ordered events or exactly-once transition effects.

## 6. Per-output `changed` flags are independent—but not per-consumer

The current implementation correctly separates:

- Canonical state potentially changed: `dirty_`.
- A particular projected output needs updating: its accumulated `changed` flag.

Refreshing one output preserves pending changes for the others. Metadata-only changes can update snapshots without changing displayed text. Initial flags start true to ensure first delivery.

However, each output has **one shared consumption flag**, not one flag per caller or widget.

The numeric context-window accessor explicitly designates `SessionStats` as its consumer. Consider an additional consumer:

1. Initial percentage is consumed.
2. Only the model’s context window changes.
3. Another caller consumes `context_window_tokens_refresh().changed`.
4. Percentage refresh sees the new value but `changed == false`.
5. With no token change pending, its cached percentage can remain stale.

This is a concrete limitation of unrestricted multi-consumer use, **not a claim that the current designated-consumer wiring violates its contract**.

Before adding consumers, the plan must choose explicit consumption ownership/fan-out or independent revision tracking. It must not silently treat `changed` as a broadcast notification.

**Sources:** `ModelSelection.h:50–102`, `SessionStats.h` refresh accessors.

## 7. Snapshot and result semantics

Two details matter for future migration:

### Snapshot coverage is not universally complete

- `frontend::SessionStats` copies the complete canonical `session::SessionStats`.
- `frontend::ModelSelection` copies `ModelInfo` and reasoning, but **does not copy `scoped_model_cycle`**.

The pattern supports retaining currently undisplayed information, but we must not assume every existing projection already copies its entire canonical aggregate.

### String results are borrowed current values

`RefreshResult<std::string const&>` refers to the projection’s cached string. It is not an immutable historical result.

- A later refresh can modify that same string.
- Destruction of the projection ends its lifetime.
- Saved character pointers or `string_view`s may be invalidated by reassignment.
- A consumer needing a historical value must copy it.

Numeric optional results are returned by value.

**Sources:** `ModelSelection.cxx:71–73`, `SessionStats.cxx:65–87`, `FrontEnd.h:22–45`.

## 8. Coherence and lock duration

The two token estimates now live in one canonical `SessionStats`, so they are copied coherently under one lock.

Model metadata and statistics still use separate locks and have no shared generation. The percentage can therefore combine values from different publication moments. Whether this is acceptable depends on the domain; it is particularly relevant during session replacement.

Also, the private refresh implementations currently perform formatting while the canonical read guard remains alive. This is safe for mutual exclusion, but it is **not a short-lock-duration design guarantee**. Expensive catalog formatting should not inherit that scope without consideration.

## 9. Notification does not itself wake the UI

`TuiFrontEnd::data_changed()` marks projections dirty but contains no explicit queueing or wake operation.

Thus this mechanism establishes what happens **when refresh is called**, not when an idle frontend will call it. The later audit must trace event-loop scheduling before asynchronous publishers are migrated.

Similarly, the inspected shutdown callback handles Mermaid and branch-summary machinery, but that alone does not establish quiescence of every publisher on every exit path.

These are specific follow-up investigations for subgoals 03 and 04.

## 10. Existing test evidence

I inspected these assertions:

| Test location | Evidence |
|---|---|
| `tests/app_runtime_model_tests.cpp:72–121` | Construction/read accesses do not notify; unchanged writes do; moved write guard notifies once; callback can reacquire the canonical lock; assignment preserves destination notification |
| `tests/tui_selector_tests.cpp:41–82` | Independent string/numeric change flags and known-to-unknown transitions |
| `tests/tui_selector_tests.cpp:84–177` | Compact versus exact token changes, percentage behavior, refresh ordering and session replacement |
| `tests/app_runtime_startup_tests.cpp:486–544` | Detached estimates copied without registration; replacement lifecycle coverage |

These are primarily sequential tests. They do **not establish** concurrent teardown safety, idle wake behavior, multi-domain replacement coherence, or independent multi-consumer invalidation.

Those require targeted verification in the later implementation goals.

## 11. Backend coupling still present in the reference API

The existing projections are a behavioral reference, but not yet a fully decoupled interface:

- `ModelSelection.h` and `SessionStats.h` include `runtime/Session.h`.
- `FrontEnd.h` exposes `session_ts` and `SessionConstReadAccess`.
- `TuiFrontEnd` initialization directly accesses Session members.
- Notification routing classifies runtime canonical aliases.

The smallest direction is to separate neutral value declarations and typed projection/read bindings from Session-specific registration. There is no evidence here requiring a new registry or a replacement architecture.

# Handoff to reconciliation

The remaining audits need to resolve:

1. **Lifecycle:** Are all publishers quiescent before frontend destruction? Is detach/rebind required?
2. **Consumption:** Which outputs have multiple consumers, and who owns each change notification?
3. **Coherence:** Which domains require a shared session identity/revision rather than eventual independent refresh?
4. **Scheduling:** How do asynchronous canonical updates cause an idle frontend to refresh?
5. **Protocol semantics:** Which information cannot safely coalesce?
6. **Boundary cleanup:** Which neutral declarations and bindings eliminate Session dependencies without moving canonical ownership into the UI?

**Conclusion:** the reference mechanism is sound as a narrowly scoped, stable-lifetime, latest-state projection pattern. The migration must explicitly preserve—or strengthen—its lifetime, consumer and coherence contracts rather than assuming atomic dirtiness solves those concerns.
