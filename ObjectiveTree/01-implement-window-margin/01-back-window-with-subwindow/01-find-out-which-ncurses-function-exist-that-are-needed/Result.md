# Result

Found the relevant ncurses APIs:

- `derwin(orig, nlines, ncols, begin_y, begin_x)` — best fit for margins. Coordinates are relative to parent `orig`.
- `subwin(orig, nlines, ncols, begin_y, begin_x)` — same kind of subwindow, but coordinates are screen-relative.
- `delwin(win)` — required; subwindows must be deleted before the parent window.
- `mvderwin(win, par_y, par_x)` — moves a derived/subwindow within its parent; probably not needed unless changing margins without recreating the subwindow.
- `touchwin(orig)` / `touchline(orig, ...)` — may be needed before refreshing a subwindow because parent/subwindow share memory and touch state is not automatically mirrored.
- `syncok(win, true)` / `wsyncup(win)` — optional: automatically/manually mark changed locations in ancestors when subwindow changes.
- `wsyncdown(win)` — called by `wrefresh`; likely not needed directly.
- `wcursyncup(win)` — only relevant if parent cursor should track subwindow cursor.

Recommendation for `Window::Margin`: use `derwin` for the writable area, delete/recreate it when margin changes, and keep parent-before-child destruction explicit. Consider `syncok(writable, TRUE)` or explicit `touchwin(parent)` depending on refresh strategy.
