# 05 — Preserve signal identity across input polling and active-run exit

## Review finding and causal trace

The newly added `terminal_signal_received` field is correctly forwarded for the explicit successful-claim branch: `RuntimeActiveRunOutcome` to `RuntimeSubmitOutcome` to the main composer. It means a signal requested application exit, not that any interrupt occurred. Retain that name and meaning.

Another exit route bypasses the assignment:

1. `RuntimeActiveRunController::run()` checks the signal hint at `src/ava/tui/runtime_active_run_internal.cpp:738` and sees no bit.
2. SIGINT or SIGTERM arrives before/during `poll_curses_input()` near `:788`.
3. `src/ava/tui/runtime_input.cpp:255–260` sees a pending terminal signal after `wget_wch()` and returns an ordinary `Key::CtrlC`. It does not claim the bit or retain signal provenance in the input type.
4. For an empty draft and normal bindings, `runtime_active_run_input_preemptive_internal.cpp:109–118` treats that event as a keyboard interrupt and calls `request_close_after_submit()`.
5. The active loop breaks on `close_after_submit` near `runtime_active_run_internal.cpp:799–800` or `854–855` without revisiting the explicit signal branch or setting `active_run_outcome.terminal_signal_received`.
6. The new completion check near `:931` consults only the remembered flag; the later close return still reports false for the signal cause. `runtime_submit_internal.cpp:599–602` and `runtime.cpp:639–645` forward that false value. With otherwise successful cleanup, the composer returns 0 instead of its signal-exit status 130 near `runtime.cpp:2593`.

This is a signal-caused exit, not merely a coincidental late signal during some unrelated exit. No competing consumer is required to reproduce it. Previously the pending bit was re-read later; the new outcome model requires explicitly preserving the cause on every signal-driven route.

## Desired behavior

| Successfully claimed request | Active-run draft state | Action and outcome |
| --- | --- | --- |
| SIGTERM, with or without SIGINT | Any | Request run cancellation and close after submit; latch `terminal_signal_received`; leave active loop. |
| SIGINT only | Empty | Request cancellation and close; latch the same exit-cause flag; leave active loop. |
| SIGINT only | Nonempty | Clear draft and render; keep active run running; do not latch a signal-exit cause. |
| Neither | Any | No signal action; preserve normal input handling. |

The initial signal read remains a hint. Only successful atomic claiming authorizes processing a request if competing consumers are allowed. Do not infer a claim from a remembered wakeup, a later pending-bit read, or every `close_after_submit` state. Genuine Ctrl-C keyboard events in raw mode must not be relabeled as POSIX delivery.

## Recommended implementation sequence

1. Extract the active loop's existing signal decision block into a small local operation with an explicit result such as no request handled, draft handled/continue, or close requested. Keep state changes and `terminal_signal_received` assignment together. Use atomic claim results, not independent load/store clearing. This avoids maintaining two copies of the policy table.
2. Preserve signal provenance in input polling instead of synthesizing an indistinguishable keyboard Ctrl-C. `RuntimeInput` is in `src/ava/tui/runtime_input_internal.h:13–19`; give the input/result representation an explicit terminal-signal wakeup indication. This notification carries no ownership claim. A marker with no real key must never reach ordinary key binding dispatch, even if another consumer has already cleared the signal.
3. Service signal requests at the existing top-of-loop boundary and again after input polling, before `dispatch_retained_input()` and its `close_after_submit` checks. On a signal-only wakeup whose bits can no longer be claimed, resume normal polling without issuing a synthetic interrupt. If a real key was also read, preserve it under the existing input-retention rules rather than inventing or duplicating an interrupt event.
4. Update the main composer and permission/question readers to understand the explicit notification. They already inspect pending bits, but must not treat a stale marker as a genuine key or lose a real decoded event. Inspect the startup input queue and bracketed-paste handling when propagating the result type. This is an input-origin correction, not permission for every nested reader to consume the same request.
5. Audit completion boundaries: the worker may become ready before another active-loop iteration. Route pending requests either through the same claimant before final input-driven closure or deliberately back to the outer composer; do not overwrite an already latched exit cause and do not declare a non-claimed hint a successful claim. Keep already requested shutdown idempotent when more signals arrive.
6. Preserve the outcome field on all returns after a claim, including render failure, queue finishing, event draining, and worker settlement. `request_close_after_submit()` persists the cancellation/closure request in run state (`runtime_active_run_internal.cpp:383–389`); the outcome separately persists why closure was requested. A local boolean that is never forwarded is not enough.
7. Keep `runtime_submit_internal.cpp` forwarding the field, and keep the main composer recording it before honoring BreakLoop. The main composer's invariant remains that a returned signal-exit flag implies BreakLoop. Do not merely set the field for every keyboard-driven exit or restore the old renderer-side signal checks.

If a smaller patch initially rechecks after polling, it must still distinguish signal-only synthetic input when the claim fails. A recheck alone leaves the competing-consumer case able to dispatch a stale Ctrl-C. The explicit wakeup representation is the recommended complete fix.

## Acceptance checks

Use deterministic input-poll hooks or a small input/decision adapter rather than relying exclusively on timing a real signal between two statements.

- No signal at loop entry, then SIGTERM appears during polling: with an empty or nonempty draft, cancellation/closure is requested, the claim succeeds, the outcome flag reaches the main composer, and the cooperative exit status is 130 after successful cleanup.
- The equivalent SIGINT with an empty draft exits with the flag; with a nonempty draft it clears once and leaves the flag false while the run continues.
- Both bits pending take the termination path. Subsequent notifications do not reset exit intent or abort required cleanup.
- Another consumer clears the bit after a wakeup is observed: no signal action and no fake Ctrl-C keyboard action occurs.
- A genuine Ctrl-C key with no pending signal follows existing bindings and does not set the POSIX signal-exit flag merely because it causes closure.
- A worker completes during the poll/dispatch boundary; no signal-driven close loses its cause and no handled draft interrupt is spuriously upgraded to an exit.
- Input retained around signal processing is neither dropped nor dispatched twice. Permission/question and startup-probe input handling continue to work with the new notification representation.
- Exercise outcome propagation through early failure returns as well as success. Do not claim exit-code coverage by checking only `request_close_after_submit()` in isolation.

Use focused TUI input/active-run tests for the interleavings, then one process/PTY lifecycle case for the end-to-end return code and terminal restoration. Record the exact tested boundaries before marking the goal achieved.
