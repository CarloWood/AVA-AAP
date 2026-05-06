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
| wadd_wchnstr | research |                   |

## https://invisible-island.net/ncurses/man/curs_addch.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| waddch       | research |                   |
| wechochar    | research |                   |

## https://invisible-island.net/ncurses/man/curs_addchstr.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| waddchnstr   | research |                   |

## https://invisible-island.net/ncurses/man/curs_addwstr.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| waddnwstr    | research |                   |

## https://invisible-island.net/ncurses/man/curs_attr.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| wattr_off    | research |                   |
| wattr_on     | research |                   |
| wchgat       | research |                   |
| wcolor_set   | research |                   |

## https://invisible-island.net/ncurses/man/curs_bkgd.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| wbkgd        | research |                   |
| wbkgdset     | research |                   |

## https://invisible-island.net/ncurses/man/curs_border.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| wborder      | research |                   |
| whline       | research |                   |
| wvline       | research |                   |

## https://invisible-island.net/ncurses/man/curs_delch.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| wdelch       | research |                   |

## https://invisible-island.net/ncurses/man/curs_deleteln.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| winsdelln    | research |                   |

## https://invisible-island.net/ncurses/man/curs_get_wch.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| wget_wch     | research |                   |

## https://invisible-island.net/ncurses/man/curs_get_wstr.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| wgetn_wstr   | research |                   |

## https://invisible-island.net/ncurses/man/curs_getch.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| wgetch       | research |                   |

## https://invisible-island.net/ncurses/man/curs_getstr.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| wgetnstr     | research |                   |

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
| winch        | research |                   |

## https://invisible-island.net/ncurses/man/curs_inchstr.3x.html

| Function     | Decision | Rationale / Notes |
|--------------|----------|-------------------|
| winchnstr    | research |                   |

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
| winsch       | research |                   |

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
| pechochar    | research |                   |
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
