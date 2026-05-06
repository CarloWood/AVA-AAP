# ncursesw WINDOW* function coverage decisions

Candidates are grouped by their invisible-island ncurses man page URL. Record one decision per candidate before implementation proceeds.

Decision values:

- `support`: add or keep a terminal::Window API wrapper.
- `covered`: already supported by an existing terminal::Window API wrapper.
- `exclude`: intentionally do not expose through terminal::Window.
- `research`: needs investigation before classification.

## https://invisible-island.net/ncurses/man/curs_add_wchstr.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| mvwadd_wchstr| support  | Add convenience overload equivalent to move plus `wadd_wchstr`; public API uses `ComplexChar` rather than `cchar_t`. |
| mvwadd_wchnstr| support | Add convenience overload equivalent to move plus `wadd_wchnstr`; public API uses `ComplexChar` rather than `cchar_t`. |
| wadd_wchnstr | support  | Add bounded complex-character string output; public API uses `ComplexChar` rather than `cchar_t`. |
| wadd_wchstr  | support  | Add complex-character string output to the public `Window` API; implementation support exists only inside `Window::Impl` today. |

## https://invisible-island.net/ncurses/man/curs_addch.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| waddch       | exclude  | Uses the older `chtype` interface; prefer wide-character/`ComplexChar` APIs. |
| wechochar    | exclude  | Uses the older `chtype` interface; prefer wide-character/`ComplexChar` APIs. |

## https://invisible-island.net/ncurses/man/curs_addchstr.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| waddchnstr   | exclude  | Uses the older `chtype` string interface; prefer wide-character/`ComplexChar` APIs. |

## https://invisible-island.net/ncurses/man/curs_addwstr.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| mvwaddnwstr  | support  | Add convenience overload equivalent to move plus `waddnwstr`; public API can accept `wchar_t const*`. |
| mvwaddwstr   | support  | Add convenience overload equivalent to move plus `waddwstr`; public API can accept `wchar_t const*`. |
| waddnwstr    | support  | Add bounded wide-character string output; public API can accept `wchar_t const*`. |
| waddwstr     | support  | Add wide-character string output; public API can accept `wchar_t const*`. |

## https://invisible-island.net/ncurses/man/curs_attr.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| mvwchgat     | support  | Has a `WINDOW *win` argument and uses `attr_t`/color pair parameters rather than legacy `int attrs`. |
| wattr_get    | support  | Has a `WINDOW *win` argument and uses `attr_t`/color pair output parameters rather than legacy `int attrs`. |
| wattr_off    | support  | Has a `WINDOW *win` argument and uses `attr_t` rather than legacy `int attrs`. |
| wattr_on     | support  | Has a `WINDOW *win` argument and uses `attr_t` rather than legacy `int attrs`. |
| wattr_set    | support  | Has a `WINDOW *win` argument and uses `attr_t`/color pair parameters rather than legacy `int attrs`. |
| wchgat       | support  | Has a `WINDOW *win` argument and uses `attr_t`/color pair parameters rather than legacy `int attrs`. |
| wcolor_set   | support  | Has a `WINDOW *win` argument and uses a color pair parameter rather than legacy `int attrs`. |
| wstandend    | support  | Has a `WINDOW *win` argument and no legacy `int attrs` parameter. |
| wstandout    | support  | Has a `WINDOW *win` argument and no legacy `int attrs` parameter. |

## https://invisible-island.net/ncurses/man/curs_bkgd.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| wbkgd        | exclude  | Uses the older `chtype` background interface; use wide-character background APIs instead. |
| wbkgdset     | exclude  | Uses the older `chtype` background interface; use wide-character background APIs instead. |

## https://invisible-island.net/ncurses/man/curs_border.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| wborder      | exclude  | Uses the older `chtype` border interface; use wide-character border APIs instead. |
| whline       | exclude  | Uses the older `chtype` line-drawing interface; use wide-character line APIs instead. |
| wvline       | exclude  | Uses the older `chtype` line-drawing interface; use wide-character line APIs instead. |

## https://invisible-island.net/ncurses/man/curs_delch.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| mvwdelch     | support  | Add convenience overload equivalent to move plus `wdelch`. |
| wdelch       | support  | Delete the character at this Window's cursor position. |

## https://invisible-island.net/ncurses/man/curs_deleteln.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| winsdelln    | support  | General insert/delete line primitive; `winsertln` and `wdeleteln` are trivial wrappers around it. |

## https://invisible-island.net/ncurses/man/curs_get_wch.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| unget_wch    | support  | Push a wide character back onto the input queue; complements `wget_wch`. |
| wget_wch     | support  | Read a wide character/key from this Window; omit `mvwget_wch` because callers can explicitly `move` first. |

## https://invisible-island.net/ncurses/man/curs_get_wstr.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| wget_wstr    | exclude  | Line-editing input API that reads until line feed/carriage return; AVA runs in `cbreak(); noecho(); nl();` mode and needs immediate per-key events. |
| wgetn_wstr   | exclude  | Bounded line-editing input API; AVA needs immediate per-key events via `wget_wch` instead. |

## https://invisible-island.net/ncurses/man/curs_getch.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| wgetch       | exclude  | Non-wide input API; use `wget_wch` for immediate wide-character/key events. |

## https://invisible-island.net/ncurses/man/curs_getstr.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| wgetnstr     | exclude  | Non-wide line-editing input API; AVA needs immediate wide-character/key events via `wget_wch`. |

## https://invisible-island.net/ncurses/man/curs_in_wch.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| win_wch      | research |                   |

## https://invisible-island.net/ncurses/man/curs_in_wchstr.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| win_wchnstr  | research |                   |

## https://invisible-island.net/ncurses/man/curs_inch.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| winch        | exclude  | Returns the older `chtype` representation; prefer wide-character/`ComplexChar` APIs. |

## https://invisible-island.net/ncurses/man/curs_inchstr.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| winchnstr    | exclude  | Uses the older `chtype` string interface; prefer wide-character/`ComplexChar` APIs. |

## https://invisible-island.net/ncurses/man/curs_inopts.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| nodelay      | research |                   |
| notimeout    | research |                   |
| wtimeout     | research |                   |

## https://invisible-island.net/ncurses/man/curs_ins_wch.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| wins_wch     | research |                   |

## https://invisible-island.net/ncurses/man/curs_ins_wstr.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| wins_nwstr   | research |                   |

## https://invisible-island.net/ncurses/man/curs_insch.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| winsch       | exclude  | Uses the older `chtype` insertion interface; prefer wide-character/`ComplexChar` APIs. |

## https://invisible-island.net/ncurses/man/curs_insstr.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| winsnstr     | research |                   |

## https://invisible-island.net/ncurses/man/curs_instr.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| winnstr      | research |                   |

## https://invisible-island.net/ncurses/man/curs_inwstr.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| winnwstr     | research |                   |

## https://invisible-island.net/ncurses/man/curs_kernel.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| ripoffline   | research |                   |

## https://invisible-island.net/ncurses/man/curs_overlay.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| overlay      | research |                   |
| overwrite    | research |                   |

## https://invisible-island.net/ncurses/man/curs_pad.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| pechochar    | exclude  | Uses the older `chtype` pad echo interface; prefer wide-character/`ComplexChar` APIs. |
| pecho_wchar  | research |                   |
| pnoutrefresh | research |                   |
| prefresh     | research |                   |
| subpad       | research |                   |

## https://invisible-island.net/ncurses/man/curs_printw.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| vw_printw    | research |                   |
| vwprintw     | research |                   |
| wprintw      | research |                   |

## https://invisible-island.net/ncurses/man/curs_scanw.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| vw_scanw     | research |                   |
| vwscanw      | research |                   |
| wscanw       | research |                   |

## https://invisible-island.net/ncurses/man/curs_scroll.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| wscrl        | research |                   |

## https://invisible-island.net/ncurses/man/curs_touch.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| wtouchln     | research |                   |

## https://invisible-island.net/ncurses/man/curs_util.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| putwin       | research |                   |

## https://invisible-island.net/ncurses/man/curs_window.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| wcursyncup   | research |                   |
| wsyncdown    | research |                   |
