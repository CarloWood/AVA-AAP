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
| mvwin_wch    | support  | Add convenience overload equivalent to move plus `win_wch`; useful for tests that inspect rendered TUI output. |
| win_wch      | support  | Read the complex character at this Window's cursor; useful for tests that inspect rendered TUI output. |

## https://invisible-island.net/ncurses/man/curs_in_wchstr.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| mvwin_wchnstr| support  | Add convenience overload equivalent to move plus `win_wchnstr`; useful for tests that inspect rendered TUI output. |
| mvwin_wchstr | support  | Add convenience overload equivalent to move plus `win_wchstr`; useful for tests that inspect rendered TUI output. |
| win_wchnstr  | support  | Read a bounded complex-character string from this Window; useful for tests that inspect rendered TUI output. |
| win_wchstr   | support  | Read a complex-character string from this Window; useful for tests that inspect rendered TUI output. |

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
| nodelay      | exclude  | Input timing defaults are set centrally in `Window::Impl::default_initialization`; `wtimeout` is the more general equivalent. |
| notimeout    | exclude  | Escape-sequence timeout policy is set centrally in `Window::Impl::default_initialization`. |
| wtimeout     | exclude  | Input timeout policy is set centrally in `Window::Impl::default_initialization`. |

## https://invisible-island.net/ncurses/man/curs_ins_wch.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| mvwins_wch   | support  | Add convenience overload equivalent to move plus `wins_wch`; public API uses `ComplexChar` rather than `cchar_t`. |
| wins_wch     | support  | Insert a complex character before this Window's cursor; public API uses `ComplexChar` rather than `cchar_t`. |

## https://invisible-island.net/ncurses/man/curs_ins_wstr.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| mvwins_nwstr | support  | Add convenience overload equivalent to move plus `wins_nwstr`; public API uses `wchar_t const*`. |
| mvwins_wstr  | support  | Add convenience overload equivalent to move plus `wins_wstr`; public API uses `wchar_t const*`. |
| wins_nwstr   | support  | Insert at most `n` wide characters before this Window's cursor; public API uses `wchar_t const*`. |
| wins_wstr    | support  | Insert a null-terminated wide-character string before this Window's cursor; public API uses `wchar_t const*`. |

## https://invisible-island.net/ncurses/man/curs_insch.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| winsch       | exclude  | Uses the older `chtype` insertion interface; prefer wide-character/`ComplexChar` APIs. |

## https://invisible-island.net/ncurses/man/curs_insstr.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| mvwinsnstr   | support  | Add convenience overload equivalent to move plus `winsnstr`, matching the existing `mvwaddnstr` coverage. |
| mvwinsstr    | support  | Add convenience overload equivalent to move plus `winsstr`, matching the existing `mvwaddstr` coverage. |
| winsnstr     | support  | Insert at most `n` narrow/UTF-8 characters before this Window's cursor, matching the existing `waddnstr` coverage. |
| winsstr      | support  | Insert a null-terminated narrow/UTF-8 string before this Window's cursor, matching the existing `waddstr` coverage. |

## https://invisible-island.net/ncurses/man/curs_instr.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| winnstr      | exclude  | Narrow string extraction is less useful for screen selection/copy; prefer wide-character extraction via `winwstr`/`winnwstr`. |

## https://invisible-island.net/ncurses/man/curs_inwstr.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| mvwinnwstr   | support  | Add convenience overload equivalent to move plus `winnwstr`; useful for select-and-copy from the screen. |
| mvwinwstr    | support  | Add convenience overload equivalent to move plus `winwstr`; useful for select-and-copy from the screen. |
| winnwstr     | support  | Extract at most `n` wide characters from this Window; useful for select-and-copy from the screen. |
| winwstr      | support  | Extract a wide-character string from this Window; useful for select-and-copy from the screen. |

## https://invisible-island.net/ncurses/man/curs_kernel.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| curs_set     | support  | Terminal cursor visibility is useful UI state to expose without requiring raw ncurses access. |
| def_prog_mode| exclude  | Low-level terminal mode snapshot API; ncurses/session lifecycle should manage this internally. |
| def_shell_mode| exclude | Low-level terminal mode snapshot API; ncurses/session lifecycle should manage this internally. |
| getsyx       | exclude  | Low-level virtual-screen cursor API; not a `Window` operation. |
| mvcur        | exclude  | Low-level physical cursor movement API; `Window` drawing/refresh should manage cursor placement. |
| napms        | exclude  | General sleep helper, not a `Window` operation. |
| reset_prog_mode| exclude| Low-level terminal mode restore API; ncurses/session lifecycle should manage this internally. |
| reset_shell_mode| exclude| Low-level terminal mode restore API; ncurses/session lifecycle should manage this internally. |
| resetty      | exclude  | Low-level terminal mode restore API; ncurses/session lifecycle should manage this internally. |
| ripoffline   | exclude  | Startup-time screen-line reservation hook with raw `WINDOW*` callback; not appropriate for `terminal::Window`. |
| savetty      | exclude  | Low-level terminal mode snapshot API; ncurses/session lifecycle should manage this internally. |
| setsyx       | exclude  | Low-level virtual-screen cursor API; not a `Window` operation. |

## https://invisible-island.net/ncurses/man/curs_overlay.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| overlay      | exclude  | Window-to-window compositing that mutates the destination contents; temporary overlays should be modeled without destroying underlying window state. |
| overwrite    | exclude  | Window-to-window compositing that mutates the destination contents, including blanks; not needed for temporary UI such as toasts. |

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
