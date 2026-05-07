# ncursesw functions to support

Functions are grouped by their invisible-island ncurses man page URL. Every function listed here needs `terminal::Window` support.

## https://invisible-island.net/ncurses/man/curs_getyx.3x.html

| Function  | Rationale / Notes |
|-----------|-------------------|
| getbegyx  | Return this Window's beginning screen position through the public API as a `Position`. |
| getmaxyx  | Return this Window's dimensions through the public API as a `Dimension`. |
| getparyx  | Return this subwindow's beginning position relative to its parent through the public API as a `Position`; handle the ncurses `(-1, -1)` no-parent case explicitly. |
| getyx     | Return this Window's current cursor position through the public API as a `Position`. |

## https://invisible-island.net/ncurses/man/curs_mouse.3x.html

| Function  | Rationale / Notes |
|-----------|-------------------|
| wenclose  | Return whether a screen-relative `Position` lies within this Window; boolean result is the queried property, not success/failure. |
| wmouse_trafo | Transform a `Position` between this Window's local coordinates and screen coordinates; boolean result reports whether the conversion is possible. |

## https://invisible-island.net/ncurses/man/curs_opaque.3x.html

| Function  | Rationale / Notes |
|-----------|-------------------|
| is_cleared | Const query returning the value set by `clearok`; boolean result is the queried property. |
| is_idcok  | Const query returning the value set by `idcok`; boolean result is the queried property. |
| is_idlok  | Const query returning the value set by `idlok`; boolean result is the queried property. |
| is_immedok | Const query returning the value set by `immedok`; boolean result is the queried property. |
| is_keypad | Const query returning the value set by `keypad`; boolean result is the queried property. |
| is_leaveok | Const query returning the value set by `leaveok`; boolean result is the queried property. |
| is_nodelay | Const query returning the value set by `nodelay`/`wtimeout`; boolean result is the queried property. |
| is_notimeout | Const query returning the value set by `notimeout`; boolean result is the queried property. |
| is_pad    | Const query returning whether this Window is a pad; boolean result is the queried property. |
| is_scrollok | Const query returning the value set by `scrollok`; boolean result is the queried property. |
| is_subwin | Const query returning whether this Window is a subwindow; boolean result is the queried property. |
| is_syncok | Const query returning the value set by `syncok`; boolean result is the queried property. |
| wgetdelay | Const query returning this Window's input delay timeout as set by `wtimeout`. |
| wgetparent | Const query returning this Window's parent as a `Window` wrapper or an explicit no-parent result without exposing `WINDOW*`. |
| wgetscrreg | Const query returning this Window's scrolling region top and bottom rows. |
