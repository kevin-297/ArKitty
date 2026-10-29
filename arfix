#!/usr/bin/env python3

import re
import textwrap
import unicodedata
import shutil

import arabic_reshaper

# --------------------------------------------------------------------------- #
# THE FIX (copy this block into your real script)
# --------------------------------------------------------------------------- #
DIGITS_MODE = "split"   # chosen from the experiments: keeps Arabic-Indic digits as they are
PUNCT_MODE = "move"     # chosen from the experiments (asis | move | space)

DIGIT_CHARS = "0-9\u0660-\u0669\u06F0-\u06F9"
ARB = "\u0600-\u06FF\u0750-\u077F\uFB50-\uFDFF\uFE70-\uFEFF"
ARAB_PUNCT = "\u060C\u061B\u061F"          # ،  ؛  ؟

reshaper = arabic_reshaper.ArabicReshaper(configuration={
    "delete_harakat": True,       # diacritics attach to the wrong cell after reordering
    "delete_tatweel": True,       # the stretch bar renders as '_' in a cell grid
    "support_ligatures": True,
})
_MARKS = re.compile("[\u200e\u200f\u061c\u202a-\u202e\u2066-\u2069]")   # invisible bidi marks
_TO_ASCII = str.maketrans("٠١٢٣٤٥٦٧٨٩۰۱۲۳۴۵۶۷۸۹", "0123456789" * 2)
_latin = re.compile(r"[A-Za-z]")
_ARDIG = re.compile("[\u0660-\u0669\u06F0-\u06F9]")      # Arabic-Indic / Persian digits


def _regexes(digits_are_letters):
    letters = f"[{ARB}]" if digits_are_letters else f"(?:(?![{DIGIT_CHARS}])[{ARB}])"
    other = f"(?:(?!{letters}).)"
    return re.compile(f"{letters}+|{other}+", re.S), re.compile(letters)


_R = {True: _regexes(True), False: _regexes(False)}


def _flip_runs(word, runs, has):
    """In a token that mixes Arabic letters with other stuff (12د, -42د, 50%د),
    reverse every non-Arabic run in place: kitty draws the whole token right to left."""
    parts = runs.findall(word)
    if len(parts) > 1 and any(has.search(p) for p in parts):
        return "".join(p if has.search(p) else p[::-1] for p in parts)
    return word


_AD = "\u0660-\u0669\u06F0-\u06F9"
_NUM = re.compile(rf"([{_AD}]+(?:[:.,/\u066B\u066C][{_AD}]+)*)")      # a number: ١٢:٣٠ , ٣٫٥
_ADRUN = re.compile(rf"[{_AD}]+")


def _one(w, runs, has, digits):
    """Per-token fix. In 'flipall' mode a token made only of Arabic-Indic digits and
    symbols (١٢:٣٠) is reversed whole, because kitty draws it right to left too."""
    if digits == "flipall" and not has.search(w) and _ARDIG.search(w):
        return w[::-1]
    if digits in ("rev", "split") and _ARDIG.search(w):
        # kitty reverses every run of Arabic-script digits in place and lays it out as
        # its own piece, so: reverse each digit run in place ...
        pieces = [p for p in _NUM.split(w) if p]
        out = [_ADRUN.sub(lambda m: m.group()[::-1], p) if _NUM.fullmatch(p)
               else _flip_runs(p, runs, has) for p in pieces]
        if digits == "split" and has.search(w):      # ... and, next to letters, swap the pieces
            out.reverse()
        return "".join(out)
    return _flip_runs(w, runs, has)


def _punct(word, mode):
    """Arabic punctuation glued to a word (بخير؟) -> list of tokens."""
    if mode == "asis":
        return [word]
    m = re.match(rf"^([{ARAB_PUNCT}]*)(.*?)([{ARAB_PUNCT}]*)$", word, re.S)
    lead, core, trail = m.groups()
    if not (lead or trail) or not core:
        return [word]
    if mode == "move":                      # swap the punctuation to the other end
        return [trail + core + lead]
    return [t for t in (lead, core, trail) if t]      # "space": separate tokens


def _reorder(words, has):
    """Reverse the word order, but keep consecutive Latin words in their own order."""
    is_latin = lambda w: bool(_latin.search(w)) and not has.search(w)
    groups = []
    for w in words:
        if groups and is_latin(w) and is_latin(groups[-1][-1]):
            groups[-1].append(w)
        else:
            groups.append([w])
    return " ".join(" ".join(g) for g in reversed(groups))


def fix(line, digits=None, punct=None):
    digits, punct = digits or DIGITS_MODE, punct or PUNCT_MODE
    line = _MARKS.sub("", line)
    if digits == "ascii":
        line = line.translate(_TO_ASCII)
    runs, has = _R[digits == "asis"]
    if not has.search(line) and not (digits in ("flipall", "rev", "split") and _ARDIG.search(line)):
        return line                          # no Arabic: leave the line alone
    words = []
    for w in reshaper.reshape(line).split():
        words.extend(_punct(w, punct))
    words = [_one(w, runs, has, digits) for w in words]
    return _reorder(words, has)


def unfix(text):
    """Undo fix(): visual text back to logical text, full lines.
    Digits stay ASCII in ascii mode (the original Arabic-Indic style cannot be recovered)."""
    runs, has = _R[DIGITS_MODE == "asis"]
    text = unicodedata.normalize("NFKC", text)
    if not has.search(text) and not (DIGITS_MODE in ("flipall", "rev", "split") and _ARDIG.search(text)):
        return text
    words = []
    for w in text.split():
        words.extend(_punct(_one(w, runs, has, DIGITS_MODE), "move" if PUNCT_MODE == "move" else "asis"))
    return _reorder(words, has)


def cells(parts):
    return "  ".join(fix(p) for p in parts)


def fix_wrapped(text, width):
    """Wrap the LOGICAL text first, then fix each line and right-align it.
    Use this instead of letting kitty wrap a long line (kitty wraps the visual text)."""
    lines = textwrap.wrap(text, width=width, break_long_words=False) or [""]
    return "\n".join(fix(l).rjust(width) for l in lines)

def show(text):                      
    width = shutil.get_terminal_size((80, 24)).columns
    print(fix_wrapped(text, width))

