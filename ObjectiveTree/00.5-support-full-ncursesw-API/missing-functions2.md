# ncursesw functions to support

Functions are grouped by their invisible-island ncurses man page URL. Every function listed here needs `terminal::Window` support.

## https://invisible-island.net/ncurses/man/curs_getyx.3x.html

| Function  | Rationale / Notes |
|-----------|-------------------|
| getbegyx  | Return this Window's beginning screen position through the public API as a `Position`. |
| getmaxyx  | Return this Window's dimensions through the public API as a `Dimension`. |
| getparyx  | Return this subwindow's beginning position relative to its parent through the public API as a `Position`; handle the ncurses `(-1, -1)` no-parent case explicitly. |
| getyx     | Return this Window's current cursor position through the public API as a `Position`. |
