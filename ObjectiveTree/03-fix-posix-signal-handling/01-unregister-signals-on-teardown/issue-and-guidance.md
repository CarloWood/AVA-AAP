# 01 — Signal-manager destruction and supported reconstruction

## Review finding and current correction

The original uncommitted `core::Signals` destructor called `signals_.default_handler(SIGINT)` and the equivalent for SIGTERM. Those utility calls do not simply replace the OS action: `utils::Signal::default_handler()` calls `unblock(signum, SIG_DFL)`, which calls `register_callback()` again. Construction had already registered both callbacks. With the registration assertion enabled, destruction therefore asserted even on an ordinary exit without any received signal.

Evidence:

- `src/ava/core/Signals.cpp:21–25`: early reservation and callback registration.
- `utils/Signals.h:62–65`: default-handler/unblock chain.
- `utils/Signals.cxx:186–207`: second registration and the assertion against an existing `m_callback_set` entry. The OS action is changed before the assertion fires.
- `tests/core_tests.cpp:340–367`: the runner destroys its Application for `core_mode`, then creates another for later suites.
- `tests/core_mode_json_permissions_tests.cpp:196–229`: explicit Application lifecycle/death tests.

The user has already replaced the destructor at `src/ava/core/Signals.cpp:28–32` with `utils::Signal::block_and_unregister()` for SIGINT and SIGTERM. Retain this fix. The old duplicate-registration report must not be presented as an unfixed defect in this updated source.

## Correct meaning of teardown

The utility does not save the process's previous dispositions and masks for restoration when a `utils::Signals` object is destroyed. Normally the owner lives until program termination, when restoring an earlier application handler is unnecessary.

`utils::Signal::block_and_unregister()` (`utils/Signals.cxx:210–227`) is the supported operation that makes reservation/registration of the signal possible again: it blocks that signal in the calling thread, installs process-wide SIG_IGN, removes its reservation, and clears its debug callback-registration entry. It does not install SIG_DFL and does not restore the state that existed before initial construction. Ignored/pending signals may be discarded at this boundary. Do not promise deferred delivery across destruction and reconstruction.

This postcondition is appropriate for the existing single-threaded lifecycle tests and avoids the duplicate-registration assertion. It does not retroactively block all other threads. Its use assumes workers and other consumers of the old Application have already finished.

## Remaining implementation and verification guidance

1. Keep unregistering each signal in the destructor; do not call `default_handler()` afterward. That would require registration that has just been removed and would contradict the intended ignored teardown state.
2. Document that reconstruction is supported only under the utility's reservation preconditions. `utils/Signals.cxx:94–100` asserts, in the libcwd debug configuration, that threads have not been created. Unregistering does not reset that global condition. Test sequential lifetimes before worker creation, or use fresh test processes. Do not weaken the utility's thread-safety assertion just to permit arbitrary runtime reconstruction.
3. Inspect `core::Signals::s_received_`, which is static and currently initialized only once. A callback bit left by a previous lifetime must not contaminate a newly constructed test Application. Reset this recorded state once the new constructor has reserved/blocked the signals and before it installs its callbacks. This clears old application state, not a kernel signal that arrives after reservation. Do not clear pending state during TUI activation, where it could erase a legitimate startup request.
4. Update stale comments in `src/ava/core/Signals.h:12–15` and `Signals.cpp:23`: only the reserved SIGINT/SIGTERM signals are blocked, not every POSIX signal; callbacks are installed during construction; activation unblocks delivery. Goal 02 defines the non-TUI policy separately.
5. Keep the member-order guarantee: `terminal_context_` is destroyed before `signals_manager_` (`src/ava/core/Application.h:27–30`). The flag-only static callback remains valid while terminal cleanup runs. Do not move unregistering ahead of terminal restoration merely to make tests quiet.

## Acceptance checks

- In a fresh assertion-enabled test process, construct and destroy one Application without a signal. No callback-registration assertion occurs.
- Query SIGINT/SIGTERM after destruction: both are ignored; both are blocked in the destroying thread. Verify unrelated mask bits are unchanged. Restore the test harness state externally or let the isolated process end.
- Construct a second Application before any worker creation. Both callbacks register successfully and both signals remain blocked until activation.
- Record a callback bit in the first lifetime, destroy it, construct the next lifetime, and verify the recorded bit does not leak. Use an isolated single-threaded process and controlled delivery.
- Retain the existing Application lifecycle death tests and `ava_tests.core_mode`; distinguish the intentional duplicate-live-Application assertion from an accidental signal-registration assertion.
- Exercise ordinary executable teardown (for example help/version) in the debug build. Do not mark the goal achieved merely because the destructor source has been edited.

## Remaining limits

This goal does not promise restoration of inherited launcher signal policy, safe reconstruction while workers exist, or preservation of signals received during teardown. Those are different contracts. The accepted fix is unregistering for process-lifetime cleanup and the supported test reconstruction sequence.
