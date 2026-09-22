# 04 — Pass two signal numbers to the suspend guard

## Review finding

`src/ava/tui/runtime_actions_internal.cpp:234` constructs `utils::Signal::BlockGuard` with an initializer list containing the expression `SIGINT | SIGTERM`. Unlike the core pending-bit flags, POSIX signal constants here are numbers, not disjoint mask bits. On Linux, SIGINT is 2 and SIGTERM is 15; OR-ing them produces one value, 15. The guard therefore blocks only SIGTERM.

The utility constructor iterates over the initializer list and passes each entry to `sigaddset()` (`utils/Signals.cxx:230–236`). Its destructor restores the saved calling-thread mask (`:239–242`). There is no interpretation of an integer as a combined signal set.

This replacement lost the former guard's protection for SIGINT while `endwin()`, protocol release, suspension, and terminal restoration are in progress (`runtime_actions_internal.cpp:233–259`). In particular, direct protocol output can be interrupted by SIGINT without SA_RESTART. The low-level sequence writer in `src/ava/tui/terminal.cpp:79–89` ignores its stdio failures, so incomplete release/rearm sequences can disagree with AVA's protocol-ownership state. This is distinct from the ordinary rendering checks removed in goal 03.

## Concrete correction

Pass SIGINT and SIGTERM as two separate initializer-list entries: `{SIGINT, SIGTERM}`. Do not use `core::Signals::bit_SIGINT` or `bit_SIGTERM` here either; those are application pending-state bits, not OS signal numbers.

Retain the intended scope around suspend handoff and restoration, including the failure return from `kill(0, SIGTSTP)`. RAII must restore the exact prior mask on that return as well as on successful resume. Do not replace restoration with unconditional unblocking, because either signal may already have been blocked on entry.

This is a calling-thread guard only. Another unblocked worker may still execute the atomic bit-setting handler. That does not interrupt the calling thread's write or mutate ncurses state. The goal is not to turn this into a process-wide mask or decide the broader question of which thread should receive every signal.

Do not change `kill(0, SIGTSTP)` or ncurses job-control policy as part of this minimal correction. The process-group scope of suspension and the separate SIGTSTP/raw-mode discussion remain explicit follow-ups unless the regression tests demonstrate a directly related defect.

## Acceptance checks

1. Query the calling-thread mask before, during, and after the guard. Both SIGINT and SIGTERM must be blocked during its lifetime; unrelated signal bits and preexisting blocked bits must be preserved afterward.
2. Include cases where neither, one, and both signals were already blocked on entry. Run in an isolated process or restore test masks rigorously.
3. Cover the actual production guard argument, not just the utility constructor in isolation. Existing sequence tests in `tests/tui_terminal_input_tests.cpp:2671–2705` use a stand-in and did not detect this call-site error.
4. Exercise suspend/resume and the suspend-failure path under controlled PTY/process-group setup. Verify release/rearm order and terminal settings. Do not accidentally suspend the test runner's own process group; use an isolated child/session and bounded SIGCONT/cleanup handling.
5. Deliver SIGINT during the guarded interval to the controlled calling thread and verify it is handled after restoration rather than interrupting that thread's terminal handoff. Do not interpret a process-directed signal delivered to another eligible worker as failure of a per-thread guard.

A correct two-entry initializer fixes the identified defect; masks and terminal-lifecycle tests establish that the intended protection survived the replacement.
