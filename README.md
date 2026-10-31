# ArKitty

Readable Arabic in the [kitty](https://sw.kovidgoyal.net/kitty/) terminal.

Kitty has no bidirectional-text support and does not join Arabic letters, so Arabic printed with a plain `print()` comes out with separated letters, words in the wrong order, reversed numbers and misplaced signs. `arfix` prepares each line **before** you print it, so that kitty's left-to-right, cell-by-cell drawing ends up looking like correct Arabic.

It works on text **your own script prints** (status bars, tables, dates, prayer times, CLI tools). It cannot fix output from other programs.

> The module file in this project is named `arfix.py`. If you saved it under another name, use that name in the imports below.

## Requirements

- Python 3.10+ (current `arabic-reshaper` releases no longer support older versions)
- [`arabic-reshaper`](https://pypi.org/project/arabic-reshaper/)
- A font with the Arabic Presentation Forms glyphs (most Arabic-capable fonts have them)

```
pip install arabic-reshaper
```

If you use a virtual environment, install into it and run your script with that environment's Python. `python-bidi` is **not** needed.

## Quick start

```python
from arfix import fix, cells, show

print(fix("السلام عليكم ورحمة الله وبركاته"))      # one line
print(fix("الظهر -42 د"))                           # signs and numbers
print(cells(["الخميس", "1", "أكتوبر", "04:54"]))    # one table row
show("a long paragraph of Arabic text ...")          # wraps to the window width
```

## API

| Function | What it does |
|---|---|
| `fix(line)` | Turns one logical line into the text to print. Lines with no Arabic are returned unchanged, so it is safe to call on everything. |
| `cells(parts)` | Fixes each item of a list and joins them with two spaces. For table rows, where the column order must stay as it is. |
| `fix_wrapped(text, width)` | Wraps the logical text to `width`, fixes each line and right-aligns it. Use it instead of letting kitty wrap a long line. |
| `show(text)` | `fix_wrapped` at the current terminal width, printed. For paragraphs. |
| `unfix(text)` | Reverses `fix` on a full line (visual text back to logical text). |

Which one to use:

- Short single lines such as a bar, a date or a label: `print(fix(line))`
- Table rows: `print(cells([...]))`
- Long text that may be wider than the window: `show(text)`

## Settings

Two constants at the top of the module control the behavior. The defaults were chosen from side-by-side tests in kitty.

| Constant | Default | Meaning |
|---|---|---|
| `DIGITS_MODE` | `"split"` | How Arabic-Indic and Persian digits (`٠١٢٣`, `۰۱۲۳`) are handled. `split` keeps them as they are and places them correctly next to letters. `ascii` converts them to `0-9` (a simpler fallback that changes what the reader sees). `rev`, `flipall` and `asis` are experiment modes and are not recommended. |
| `PUNCT_MODE` | `"move"` | How Arabic punctuation (`،` `؛` `؟`) glued to a word is placed. `move` swaps it to the correct end of the word. `space` detaches it with a gap. `asis` leaves it, which draws it on the wrong side. |

Ordinary 0-9 digits glued to letters (`12د`, `3س`, `-42د`) are handled in every mode.

## What it handles

- Joined letter forms, including the lam-alef and Allah ligatures
- Correct word order (the first word at the right edge)
- Numbers glued to letters (`12د`), signs (`-42`, `+12`), percent and decimals
- Arabic-Indic and Persian digits, including times such as `١٢:٣٠`
- Arabic punctuation at the end of words
- Latin words and phrases inside Arabic lines (`kitty terminal` keeps its order), plus emails, URLs and file names
- Invisible direction marks (RLM, LRM), which would otherwise break the joined letters
- Diacritics and tatweel, which are removed because they render badly in a cell grid
- Tables, with each cell fixed and the column order kept
- Long text, wrapped before reordering so kitty never has to wrap it

## How it works

Kitty draws one cell at a time, left to right, with no joining. From testing, it also draws each word that contains Arabic from right to left, including any digits, signs or marks stuck to it. `fix` works around that in a fixed order:

1. Strip invisible direction marks.
2. Reshape the text into joined letter forms (one character per cell).
3. Split into words and move Arabic punctuation to the correct end.
4. Pre-flip digits, signs and other non-Arabic parts inside words that mix them with Arabic letters, so kitty's own reversal puts them right. Arabic-Indic digit runs are reversed in place and, next to letters, placed on the correct side.
5. Reverse the word order, keeping consecutive Latin words together.

`unfix` runs the same steps in reverse, which is why `unfix(fix(x))` gives back `x`.

## Known limitations

- **Copy and paste.** The printed text is in visual order and uses joined-form characters, so copied text is not the real text. Keep the original logical string for anything users may copy or search. `unfix` can restore a full line, but not a partial mouse selection.
- **Resizing the window.** Text printed earlier is re-wrapped by kitty when the window shrinks, and the halves of a wrapped line swap places. For a live bar or full-screen program, reprint on resize.
- **Whitespace.** Runs of spaces and tabs collapse to single spaces.
- **Other programs' output.** `ls`, logs and editors are not changed by this module.
- **Mixed text with several Latin words and numbers** can still be ordered differently from a full bidi implementation in unusual cases.
- **Terminals with real bidi support** (for example VTE-based terminals such as GNOME Terminal, or Konsole) already reorder text themselves, and `fix` would reverse it a second time. Use it only in terminals that draw strictly left to right.
- **Tested in kitty only.** The settings were verified visually in kitty. Other terminals that also lack bidi may behave differently.

## Testing

The fix was developed against a suite of 84 cases covering letters, digits, signs, punctuation, mixed Latin text, whitespace, symbols, long lines and tables. Each case has an automatic round-trip check (`unfix(fix(x)) == x`) and a written description of what a correct render looks like in kitty. If you change the fix, keep a suite like that and check the result in the terminal, because the automatic check cannot tell whether a line looks right on screen.

## Cleanup

If you only use the defaults, you can delete the unused modes (`asis`, `ascii`, `flipall`, `rev` and the `space` punctuation mode) and the helpers they use (`_TO_ASCII`, the `digits_are_letters=True` regexes).

## License

Released under the [MIT License](LICENSE). Use it, modify it and ship it in any project, including commercial and closed-source ones.

`arabic-reshaper`, the only dependency, is also MIT-licensed.
