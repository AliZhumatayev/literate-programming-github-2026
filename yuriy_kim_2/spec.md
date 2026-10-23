# Specification: Top words

## Overview

`top_words` returns the 10 most common words in an English, Russian, or Kazakh
article and how many times each appears. There is no language detection and
there are no language-specific rules, e.g. no Turkic casing (`I` becomes `i`).

## Interface

- `top_words(text: str) -> list[tuple[str, int]]` in `top_words.py`; Python
  3.13 or later, standard library only.
- Returns at most 10 `(word, count)` tuples of plain lowercase `str` and `int`.
- Unicode behavior follows the running Python's `unicodedata`; the examples
  were checked on Python 3.13 (Unicode 15.1) and 3.14 (Unicode 16.0).

## Rules

The function applies these steps in order:

1. If `text` is not a `str`, raise `TypeError` (Rule 7).
2. Apply NFC normalization to the whole text (Rule 1).
3. Apply `str.lower()` to the whole text. The result is not normalized again.
4. Replace `’` (U+2019) with `'` (U+0027).
5. Split into words (Rules 2 and 3), using the text as it is after steps 2–4.
6. Count the words as they are (Rule 4).
7. Sort, then keep the first 10 (Rules 5 and 6).

Examples with invisible or combining characters are Python literals with `\u`
escapes; the others use the shorthand `word: count`, in output order.

### 1. Normalization and case

Words that differ only in case, or only in how a letter is composed, are the
same word. Use `str.lower()`, not `str.casefold()`, so `ß` and `ss` differ.

| Input | Expected output |
| --- | --- |
| `Cat CAT cat` | `cat: 3` |
| `"\u049a\u0430\u043b\u0430\n\u049b\u0430\u043b\u0430\t\u049a\u0410\u041b\u0410"` (`Қала`, newline, `қала`, tab, `ҚАЛА`) | `[('\u049b\u0430\u043b\u0430', 3)]` |
| `"\u0438\u0306 \u0439"` (`и` + breve, `й`) | `[('\u0439', 2)]` |
| `"Stra\u00dfe Strasse"` | `[('strasse', 1), ('stra\u00dfe', 1)]` |
| `"I \u0131"` (`I`, dotless `ı`) | `[('i', 1), ('\u0131', 1)]` |

### 2. Letters

A character is a letter if `str.isalpha()` is `True` for it. Everything else
(digits, punctuation, symbols, emoji, whitespace, invisible characters, and
combining marks that NFC has not composed into a letter like `й` or `ё`)
separates words. `İ` lowercases to `i` + U+0307, so it splits the word.

| Input | Expected output |
| --- | --- |
| `http://x.com/a 😀` | `a: 1, com: 1, http: 1, x: 1` |
| `covid19 covid` | `covid: 2` |
| `"m\u00b2 \u2163 \u00bd"` (`m² Ⅳ ½`) | `[('m', 1)]` |
| `"\u0434\u043e\u0301\u043c"` (`до` + stress mark, `м`) | `[('\u0434\u043e', 1), ('\u043c', 1)]` |
| `"\u0435\u0308\u043b\u043a\u0430"` (`е` + U+0308, `лка`) | `[('\u0451\u043b\u043a\u0430', 1)]` |
| `"\u0130stanbul"` (`İstanbul`) | `[('i', 1), ('stanbul', 1)]` |
| `"a\u00a0b\u200bc"` (no-break space, zero-width space) | `[('a', 1), ('b', 1), ('c', 1)]` |

### 3. Words

A word is a maximal run of letters, plus any `-` or `'` (U+002D, U+0027, and
`’` after step 4) with a letter immediately on both sides. Every other
non-letter separates, including other dashes and hyphens (`—`, `–`, U+2010,
U+2011, soft hyphen U+00AD). U+02BC `ʼ` is a letter and is not converted.

| Input | Expected output |
| --- | --- |
| `из-за стола из-за` | `из-за: 2, стола: 1` |
| `"don't don\u2019t dont don\u02bct"` | `[("don't", 2), ('dont', 1), ('don\u02bct', 1)]` |
| `'cat' -cat- a--b` | `cat: 2, a: 1, b: 1` |
| `covid-19 1-a` | `a: 1, covid: 1` |
| `"don\u2018t"` | `[('don', 1), ('t', 1)]` |
| `слово—слово` | `слово: 2` |
| `"\u0130-i"` (`İ`, hyphen, `i`) | `[('i', 2)]` |
| `"\u0438\u0306-\u0438\u0306"` (`и` + breve, hyphen, twice) | `[('\u0439-\u0439', 1)]` |
| `"\u043f\u0435\u0440\u0435\u00ad\u043d\u043e\u0441 \u0438\u0437\u2011\u0437\u0430"` (soft hyphen; U+2011) | `[('\u0437\u0430', 1), ('\u0438\u0437', 1), ('\u043d\u043e\u0441', 1), ('\u043f\u0435\u0440\u0435', 1)]` |

### 4. No other processing

No stop words, no stemming; `ё`/`е` and look-alikes across scripts differ.

| Input | Expected output |
| --- | --- |
| `the cat the` | `the: 2, cat: 1` |
| `ёлка елка ёлка` | `ёлка: 2, елка: 1` |
| `"a \u0430 i \u0456"` (Latin `a`, Cyrillic `а`, Latin `i`, Kazakh `і`) | `[('a', 1), ('i', 1), ('\u0430', 1), ('\u0456', 1)]` |

### 5. Ties and the cutoff

Sort by count, highest first, and break ties by the word in ascending code
point order (Python's default `str` comparison); then keep the first 10. This
is not dictionary order: `ё` sorts after `я`, and `ә`, `қ`, `і` after `ё`.

| Input | Expected output |
| --- | --- |
| `k j i h g f e d c b a` | `a` through `j`, each 1 |
| `k k a b c d e f g h i j` | `k: 2`, then `a` through `i`, each 1 |
| `әке ёлка яблоко` | `яблоко: 1, ёлка: 1, әке: 1` |
| `"\u04d9\u043a\u0435 \u0456\u0441"` (`әке іс`) | `[('\u0456\u0441', 1), ('\u04d9\u043a\u0435', 1)]` |
| `ақ бала` | `ақ: 1, бала: 1` |

### 6. Short and empty results

With fewer than 10 distinct words, return all of them: input
`cat dog cat bird dog cat` gives `cat: 3, dog: 2, bird: 1`. With none, such as
`""` or ` ... !!! `, return `[]`; this is not an error.

### 7. Invalid input

A non-`str` argument, such as `None` or `b"abc"`, raises `TypeError`. A `str`
subclass is processed as `str(text)`, so its overridden methods are not used:
with `class S(str): pass`, `top_words(S("cat"))` returns `[('cat', 1)]`.

## Out of scope

Language detection, stop words, stemming, Latin–Cyrillic transliteration,
locale-aware casing, other scripts, streaming, and performance on huge inputs.
