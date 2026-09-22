# 02 — Explicit SIGINT/SIGTERM policy for non-TUI execution

## Failure mechanism

`main()` constructs `app::Application` before mode selection (`src/main.cpp:14–37`). Its core signal manager reserves SIGINT/SIGTERM, installs AVA callbacks, and leaves both signals blocked (`src/ava/core/Signals.cpp:21–25`). Only `run_interactive_composer()` calls `activate_handlers()` after terminal initialization (`src/ava/tui/runtime.cpp:212–217`).

Print, RPC, ACP, explicit line shell, implicit non-terminal line-shell fallback, and early non-TUI subcommands bypass that activation. Their main thread and workers created from it therefore retain the blocked mask. An idle line shell or open RPC/ACP input stream need not respond to process-directed SIGTERM. The connect provider menu installs temporary handlers, but those cannot receive SIGINT/SIGTERM while the same signals remain blocked.

Simply calling `activate_handlers()` everywhere is not a fix: it enables the flag-only AVA callback, and non-TUI frontends do not consume those flags. Merely changing handlers is also insufficient: exec preserves the signal mask, and a default action cannot run while the signal remains blocked everywhere.

## Recommended policy and its limits

Restore ordinary default SIGINT/SIGTERM delivery for a definitively non-TUI invocation. Retain the early blocked custom callback for a TUI invocation, with activation after ncurses is ready. This is the smallest policy correction consistent with the pre-regression behavior; it is not a new cooperative-shutdown implementation for every frontend.

Confirm this policy with the user before implementation. SIG_DFL termination bypasses C++ destructors, the normal supervisor cleanup in `src/main.cpp:39–46`, and any guarantee that separate-process-group descendants are stopped. If clean descendant shutdown is required in headless modes, expand this goal to explicit frontend cancellation/wakeup protocols rather than claiming default delivery supplies it.

Terminal-owning connect prompts are an exception to blindly leaving defaults active: scoped prompt handlers must remain able to restore terminal settings. Apply the baseline before entering those scopes, not repeatedly inside them.

## Concrete core API change

Add an explicitly named one-way method on `core::Signals`, such as `use_default_non_tui_policy()`, called on the dispatch thread before that mode starts work. Make failure reportable with the existing core error/result convention.

Recommended internal sequence:

1. Require the startup-blocked state, or return success without changing anything if this same policy was already selected. Do not silently switch a live TUI to default disposition. A small per-owner policy state can make this contract explicit.
2. While both signals remain blocked, install SIG_DFL for SIGINT and SIGTERM with direct, checked `sigaction()` calls owned by this core wrapper. Initialize the action mask and flags. Do not pass through SIG_IGN, drain pending signals, or unregister and re-reserve them.
3. After both installations succeed, unblock exactly those two signals on the calling thread using a checked `pthread_sigmask(SIG_UNBLOCK, ...)`. Preserve unrelated mask bits. Account for `pthread_sigmask` returning its error number directly rather than relying on errno.
4. If action installation fails, keep delivery blocked, restore the first changed action if needed, and return a startup error without entering the mode. A queued startup signal is intentionally eligible for default delivery once the successful transition unblocks it.
5. Retain reservation/registration ownership until the destructor's `block_and_unregister()` calls. The utility has no separate callback dispatch table; its registration set is ownership/debug bookkeeping. Document why this owner changes the OS disposition directly without registering another callback.

Do not use `signals_.default_handler()` here: its `unblock(signum, SIG_DFL)` path re-registers an already registered signal and hits the duplicate-registration assertion (`utils/Signals.h:62–65`, `utils/Signals.cxx:186–207`). Do not weaken that assertion. Do not unregister/re-reserve after workers exist; reservation requires early single-threaded initialization.

Keep `activate_handlers()` for the TUI; guard against calling it after non-TUI policy selection. Its purpose remains unblocking the already installed callback, not installing it again. Correct the misleading header comments while adding this API.

## Put the calls at actual dispatch boundaries

Use `core::Application::instance().signals_manager()` from app orchestration, not process-global policy changes hidden inside reusable stream-based mode implementations.

| Dispatch path | Source anchor | Required placement |
| --- | --- | --- |
| ACP | `src/ava/app/app.cpp:280–296` | After validating its standalone arguments, before diagnostics and `run_acp_mode()`. This early return never reaches the common parser tail. |
| Print and RPC | `app.cpp:755–790,854–872` | After final argument/mode validation, before prompt-file expansion, diagnostics, catalog/session startup, or input reading. Use the resolved flags, not a transition on the first individual option token. |
| Explicit/implicit line shell | `app.cpp:875–883`; `src/ava/app/line_shell.cpp:691–698` | Resolve whether the invocation will use line shell before `Session::open()`, then select the policy before session workers are created. |
| Connect/login/auth login | `app.cpp:329–460` | At the common connect-command dispatch boundary, before credential reads, browser launch, HTTP/OAuth work, or provider-menu raw mode. |
| Doctor/support export | `app.cpp:241–268` | Before their substantive non-TUI work. |
| Help/version and similar early output branches | Early returns in `app::run()` | Apply the non-TUI policy before potentially blocking output; preserve existing invalid-argument diagnostics and statuses. |

Do not classify all invocations with TTY streams as TUI: explicit print/RPC/connect remain non-TUI. Do not classify piped stdin alone as print: without a print/RPC selection it currently falls back to line shell. Agent build/plan mode is not frontend selection.

For the common frontend, use one resolved choice. The current interactive predicate is `!force_line_shell && terminal_is_tty()`, where `terminal_is_tty()` requires both stdin and stdout to be terminals. Move that decision ahead of common session startup and pass it to `run_interactive()` explicitly. Update its declaration in `src/ava/app/line_shell.h` and the one production call in `app.cpp`; do not recompute a conflicting policy later. Use the same result for the farewell card. Retain the composer's defensive TTY validation; a later failure should fail startup rather than silently switch frontends under a mismatched signal policy.

This early resolution matters: `Session::open()` constructs subagent-delivery/title coordinators (`src/ava/app/runtime/Session.cpp:436–441,589–594`). The supervisor monitor itself starts lazily, not in the top-level Supervisor constructor. A worker already created with a blocked mask is not retroactively unblocked by this transition, although an unblocked main thread suffices for process-directed default termination.

## Connect and child-process checks

The provider menu's existing `ScopedTerminalMode` installs SIGINT/SIGTERM/SIGHUP handlers and restores termios (`src/ava/app/connect_openai.cpp:36–79,118–150,335–344`). Selecting the non-TUI baseline before the wizard allows these handlers to work and preserves their controlled-cancellation behavior. Do not reapply defaults inside their lifetime.

Check the secret-entry path separately: `ScopedTerminalEcho` at `connect_openai.cpp:91–110`, used near `496–505`, disables echo but lacks the same cleanup handlers. Default termination during that read can bypass its destructor and leave echo disabled. This predates the new blocked-policy regression, but restoring defaults exposes the existing behavior again. Before claiming terminal-safe connect support, either reuse an appropriate scoped interruption/termios-cleanup mechanism for secret entry or document and obtain approval for deferring it. Verify real stream-interruption behavior with a PTY; do not assume setting a flag alone wakes a buffered read.

Keep existing child-side mask/disposition resets in the supervisor (`src/ava/process/supervisor_spawn_posix.cpp:234–250,644–657`), Bash (`src/ava/tools/bash_tool.cpp:262–276`), and Mermaid (`src/ava/app/mermaid_render_coordinator.cpp:168–176`). Direct browser, MCP, and LSP fork/exec paths may preserve the launching thread's mask; review the applicable launch path rather than relying on exec to unblock signals. Relevant sources are `src/ava/app/browser_open.cpp:68–94`, `src/ava/mcp/stdio_client.cpp:268–293`, and `src/ava/lsp/lsp_process.cpp:331–379`. The browser is reached from connect. Comprehensive child policy remains the separate child-process discussion, not an excuse to remove existing resets.

## Acceptance checks

Use fresh subprocesses, isolated HOME/configuration, no live credentials/network requirements, explicit readiness, timeouts, and unconditional reaping. Test SIGINT and SIGTERM independently.

- Low-level policy test: construction installs custom actions and blocks both signals; non-TUI transition installs defaults and unblocks exactly those bits; no duplicate-registration assertion occurs. A startup-pending signal takes its default action upon unblocking. Repeated selection must not overwrite a later temporary prompt handler.
- Print waiting for stdin EOF, RPC waiting for another request, ACP with its input open, explicit line shell, and implicit fallback all terminate promptly under the documented default policy. Include the redirected-stdout fallback, not just piped stdin. Test aliases at the parser level rather than duplicating every expensive end-to-end case.
- Check actual wait status (`WIFSIGNALED`/`WTERMSIG`, or Python's negative signal return code), not shell-style 130/143 for default termination. TUI cooperative signal exit continues to use its existing status 130 convention.
- Provider-menu and secret-prompt PTY tests verify cancellation and restored echo/canonical settings where termios was changed; credential input must not be committed after cancellation.
- Existing TUI lifecycle coverage still verifies callbacks survive ncurses setup, activation happens after terminal initialization, and SIGTERM restores terminal state. `tests/tui_terminal_lifecycle_smoke.py` is opt-in via `AVA_TUI_TERMINAL_LIFECYCLE_SMOKE=1`.
- Reuse fixtures in `tests/rpc_parser_subprocess_test.py`, `tests/acp_subprocess_test.py`, `tests/line_shell_cli_test.py`, and `tests/line_shell_pty_test.py`. Register focused new policy/process cases in `tests/CMakeLists.txt`; do not replace stream-level unit tests with a large UI suite.

Record the selected policy and tests in this artifact. Neither simply unblocking callback delivery nor adding a call only at the common parser tail satisfies this goal.
