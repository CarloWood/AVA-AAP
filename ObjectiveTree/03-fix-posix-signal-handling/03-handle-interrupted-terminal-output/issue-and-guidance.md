# 03 — Interrupted output is not a failed frame

## Review finding

Removing the signal-dependent returns from `RuntimeRenderer::render_full()` and `paint()` was intentional and correct for the chosen design. A completed frame must not be reported as failed merely because SIGINT or SIGTERM is pending, and the frame scheduler must not latch such a notification as a permanent rendering failure.

Removing `SignalBlockGuard` also changed a separate property: SIGINT/SIGTERM can now interrupt system calls used by the drawing thread. `utils::Signal::register_callback()` installs actions with zero flags, not SA_RESTART (`utils/Signals.cxx:186–190`). AVA does direct output in addition to ncurses output:

- `src/ava/tui/composer.cpp:2624–2632`: OSC overlay cursor movement and text via `fwrite()`.
- `composer.cpp:2644–2662`: Kitty deletion and image/control sequences via `fwrite()`.
- `composer.cpp:2671–2680`: final cursor sequence and `fflush(stdout)`.
- `src/ava/tui/runtime_render_internal.cpp:389–392,493–496`: rendering is no longer masked and returns drawing success.
- `runtime_render_internal.cpp:127–131`: unsuccessful painting permanently sets `FrameScheduler::failed_`.

A short `fwrite()` or failed `fflush()` immediately becomes `fail_screen_draw()`. There is no recovery for EINTR. A signal overlapping an output write to a temporarily backpressured PTY can therefore make a draft-clearing SIGINT terminate an otherwise usable UI. A partially emitted terminal control/image sequence is also possible. The normal ncurses buffered flush retries EINTR in the inspected ncurses source; that does not repair AVA's direct stdio paths.

This is an output-interruption issue, not an argument that computing a frame should take a long time. The review established the source path; it did not reproduce the timing-dependent failure at runtime.

## Recommended implementation direction

Keep the renderer free of signal-bit checks. Resolve recoverable interruption in the output layer and leave the signal bit for the normal loop to consume after the frame.

1. First add a deterministic test seam around the direct terminal writer used by the cited draw path. Demonstrate interruption before any bytes, interruption after partial progress, and a genuine output failure. The test should distinguish pending signal notification from actual write failure and exercise the frame scheduler result.
2. Prefer a single output routine that owns its bytes and progress explicitly: advance by every positive write result, retry EINTR without dropping or duplicating bytes, and report genuine unrecoverable errors. A signal callback should set a bit; it should not cancel a partially emitted control sequence. Handle EAGAIN according to the descriptor's actual blocking policy rather than introducing an uncontrolled busy loop.
3. Route the OSC overlays, graphics/deletion sequences, and final cursor output through that routine. Preserve serialization under the existing renderer mutex and preserve ordering relative to ncurses' frame output. Do not mix raw descriptor writes with outstanding buffered stdout data. Establish the flush/ownership boundary while errors are still recoverable; document it in the writer's contract.
4. Do not assume that clearing a FILE error and blindly retrying `fflush()` will recover the exact unsent suffix: stdio buffering can obscure or discard progress after a failed flush. If retaining stdio, first establish an exact recovery strategy for the platform/library and test it. The preferred explicit-buffer writer must not inherit this ambiguity at its integration boundary.
5. If a correct output-layer retry integration would exceed this focused patch, discuss a narrowly scoped output mask as an interim fix. Use `utils::Signal::BlockGuard` with separate signal numbers around the actual terminal output/flush and restore the prior mask; this protects against interruption in the calling thread without reinstating signal-dependent render failure. Do not silently restore the old broad guard or claim masking is necessary because painting is slow. Record the user's selected approach before implementation.
6. A global SA_RESTART change is not a drop-in substitute. It changes blocking-read/wakeup behavior for input and other callers, may not restart every operation, and does not solve already partial writes. Consider it only with an explicit broader I/O policy and tests.

After a successful paint, preserve the ordinary caller checks: SIGINT with a nonempty draft can clear and repaint; SIGTERM can latch shutdown. Repeated signals during an already requested shutdown must not prevent output or terminal restoration needed to finish it. Genuine EIO/EPIPE/closed-terminal failures must still propagate as errors rather than being swallowed as interruption.

## Acceptance checks

- Inject EINTR before progress, partial progress followed by EINTR, multiple interruptions, and final success. Assert exact byte content/order and no duplication of graphics/control sequences.
- Inject a genuine unrecoverable write/flush error and verify drawing fails and the existing failure policy remains effective.
- A pending signal alone does not make `render_full()`/`paint()` return false or set the scheduler's permanent failure state.
- A controlled PTY case delivering SIGINT during direct image/overlay output with a nonempty draft remains usable, performs the draft-clear policy once, and can render another frame.
- A SIGTERM case, including a subsequent SIGINT/SIGTERM, completes the documented cleanup and restores the terminal. Validate complete protocol release bytes and termios where applicable, not just an exit status.
- Keep tests bounded and synchronized on the writer seam or PTY readiness, not assumptions about slow frame computation. Use the existing composer-rendering and terminal-lifecycle test fixtures where practical; do not claim production write coverage from a stand-in that never executes the writer.

## Completion record

Record whether the fix uses explicit retrying output or a user-approved narrow mask, why it is correct at the ncurses/stdio boundary, which failures were injected, and which real PTY scenarios were exercised. Removing signal checks alone does not complete this goal.
