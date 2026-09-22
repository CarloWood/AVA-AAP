# POSIX signal fixes: scope and implementation contract

## Purpose and workflow

This objective records the five findings from the review of AVA's uncommitted signal-handling changes, together with sufficiently detailed English guidance for the user to implement each fix personally. It is not authorization for an agent to edit application code. Discuss and implement one subgoal at a time. The repository was inspected read-only; no build, signal-injection experiment, or runtime test has established completion of these goals.

The child goals correspond, in order, to the five numbered review findings. Each child has an `issue-and-guidance.md` artifact containing the failure mechanism, source locations, recommended changes, and acceptance checks. Read that artifact as well as the concise description. Goal 01 records the destructor fix the user has already made; its remaining work is lifecycle verification and any demonstrated reconstruction-state issue, not rewriting that destructor back to defaults.

All source paths in these artifacts are relative to `$REPOROOT`. Line numbers reflect the inspected working tree and will drift as the user edits. Locate the named functions again before making a patch. Do not replace unrelated uncommitted work, delete `#if 0` blocks, or change unrelated dirty submodule files.

## Agreed baseline

- Commit `2cb3d22fb15476e3b` replaced `CursesSession::enter()` with `terminal::Context::initialize()`. The old path called `newterm()`, then installed AVA's flag-only `mark_terminal_signal` handler for SIGINT/SIGTERM and restored a temporary calling-thread mask after initialization.
- The new design deliberately replaces that installation with a process-lifetime `core::Signals`, owned by `core::Application`, wrapping `utils::Signals` reserved for SIGINT/SIGTERM. Its constructor installs an atomic pending-bit callback while these signals remain blocked. TUI activation unblocks the calling thread only after terminal initialization.
- `initscr()` calls `newterm()` in the inspected ncurses implementation. Ncurses does not unblock SIGINT/SIGTERM during initialization and leaves an existing custom handler for them alone.
- Signal dispositions are process-wide. Masks are per-thread, and new threads inherit their creator's mask. Early construction protects workers created before activation; it does not by itself establish main-thread-only delivery for workers created afterward.
- The callback records bits only. A pending bit is not a queued callback count; same-signal occurrences may coalesce. A successful atomic clear claims a request. An initial read is only a hint when competing consumers are possible.
- `terminal_signal_received` in an outcome means that a successfully handled signal requested application exit. It does not mean that every signal receipt, draft clear, or local cancellation should set it. Keep the user's chosen name and document this contract.
- Rendering reports drawing success, not whether a terminal signal is pending. Do not restore the removed pre/post-render signal tests or poison the frame scheduler merely because a signal arrived.
- Once a termination request has been accepted, further SIGINT/SIGTERM requests must not downgrade or derail orderly shutdown. They do not implicitly mean force-abort. Genuine output failures must remain distinguishable from notification of a pending signal.

## Policy choice requiring confirmation in goal 02

The recommended bounded regression fix restores ordinary default SIGINT/SIGTERM termination for non-TUI frontends, while retaining cooperative TUI shutdown. Default termination bypasses C++ destructors and does not guarantee cleanup of separate-process-group descendants. If graceful supervisor/child cleanup is required for every non-TUI termination, say so before implementing goal 02: that requires cooperative cancellation in each frontend, not merely a different disposition or mask. The guide states exactly where to apply the recommended default policy and where terminal-owning connect prompts require scoped cleanup handling.

## Verification strategy

1. Use small isolated process tests for dispositions, masks, callback registration, teardown, reconstruction, and claiming/outcome decisions. Process-global signal tests must not contaminate unrelated tests.
2. Use deterministic injected I/O results to test EINTR, short writes, and genuine failures. Add a PTY-backed integration case only where actual terminal protocol restoration or signal delivery is the behavior under test.
3. Use subprocess tests for non-TUI SIGINT/SIGTERM delivery and a small number of TUI lifecycle tests for return status and terminal restoration. Synchronize on readiness rather than guessing a startup delay; every test needs a timeout and guaranteed child reaping.
4. Keep the debug/libcwd configuration enabled for lifecycle tests so callback-registration assertions are exercised. Assert that the pending-bit atomic is lock-free on supported targets, or resolve signal-safe storage before broadening platform support.
5. The reviewer ran `git diff --check`, not a build. After changes, the user can build the already-configured tree with `cmake --build "$BUILDDIR"` and run selected registered tests with `ctest --test-dir "$BUILDDIR" --output-on-failure -R '<selected tests>'`. Confirm test names with `ctest --test-dir "$BUILDDIR" -N`; do not claim a stale binary verifies new source. Do not reconfigure merely because a build was requested. If configuration is explicitly needed, follow repository instructions and stop on configuration failure.

## Objective-wide completion

All five child acceptance checks must pass. Verify in combination that SIGTERM causes the documented result before TUI activation, during an idle TUI, during a submitted run, during output, during summary cancellation, and during suspend/resume. Verify the intended SIGINT distinction between an empty draft (exit), a nonempty draft (clear), and an active branch summary (cancel without exit). Cover both pending bits and repeated requests after shutdown is latched.

The forwarding chain is `RuntimeActiveRunOutcome` to `RuntimeSubmitOutcome` to the main composer's remembered exit flag. Preserve it across all early returns, not just the explicit top-of-loop signal branch.

Record actual commands, test results, and residual limitations in the relevant goal artifact before marking it achieved with an `aap-*` helper. Until then, a proposed fix or an already edited source file is not a verified completion.

## Related discussion retained outside these five fixes

The broader Topic List still includes SIGTSTP/raw-mode policy, application construction in each frontend/child path, supervisor-monitor masks, and worker inheritance. These are not silently declared solved by this objective. Goal 02 records the mode/child consequences needed for its fix; goal 04 fixes the signal-number error without deciding all job-control policy. Review callbacks that used a successful render as an implicit cancellation gate (editor, selector, permission/question results) when deciding operation-boundary policy; do not insert checks after every render indiscriminately.
