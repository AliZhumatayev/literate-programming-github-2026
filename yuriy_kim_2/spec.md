# Specification: Top words

## Overview

`top_words` finds the 10 most common words in an article's text and how many
times each appears. Articles may be in English, Russian, or Kazakh. There is no
language detection and there are no language-specific rules: the same Unicode
rules apply to all text. In particular there is no Turkic casing (`I` always
lowercases to `i`).

## Interface

- Python 3.13 or later, standard library only. The function is `top_words` in
  `top_words.py`.
- Signature: `top_words(text: str) -> list[tuple[str, int]]`
- Returns at most 10 `(word, count)` tuples. Words are plain lowercase `str`;
  counts are plain `int`.
- Unicode behavior is that of the running Python's `unicodedata`. The examples
  below were checked on Python 3.13 (Unicode 15.1) and 3.14 (Unicode 16.0).

## Rules

The function applies these steps in order:

1. If `text` is not a `str`, raise `TypeError` (Rule 7).
2. Apply NFC normalization to the whole text (Rule 1).
3. Apply `str.lower()` to the whole text. The result is not normalized again.
4. Replace `’` (U+2019) with `'` (U+0027).
5. Split into words (Rules 2 and 3), using the text as it is after steps 2–4.
6. Count the words as they are (Rule 4).
7. Sort, then keep the first 10 (Rules 5 and 6).

Examples that contain invisible or combining characters are written as Python
literals with `\u` escapes. The others use the shorthand `word: count`, in
output order.

### 1. Normalization and case

Words that differ only in case, or only in how a letter is composed, are the
same word. Use `str.lower()`, not `str.casefold()`, so `ß` and `ss` differ.

| Input | Expected output |
| --- | --- |
| `Cat CAT cat` | `cat: 3` |
| `Привет ПРИВЕТ` | `привет: 2` |
| `"Қала\nқала\tҚАЛА"` (`Қала`, newline, `қала`, tab, `ҚАЛА`) | `[('қала', 3)]` |
| `"й й"` (`и` + breve, `й`) | `[('й', 2)]` |
| `"Straße Strasse"` | `[('strasse', 1), ('straße', 1)]` |
| `"I ı"` (`I`, dotless `ı`) | `[('i', 1), ('ı', 1)]` |

### 2. Letters

A character is a letter if `str.isalpha()` is `True` for it. Everything else
(digits, punctuation, symbols, emoji, whitespace, invisible characters and
combining marks) separates words. A combining mark survives only if NFC has
composed it into a letter (`й`, `ё`); otherwise it splits the word. `İ`
lowercases to `i` + U+0307, so it splits too.

| Input | Expected output |
| --- | --- |
| `Hello, world! Hello — world…` | `hello: 2, world: 2` |
| `«Привет» привет` | `привет: 2` |
| `http://x.com/a 😀` | `a: 1, com: 1, http: 1, x: 1` |
| `covid19 covid` | `covid: 2` |
| `2024 42` | `[]` |
| `"m² Ⅳ ½"` (`m² Ⅳ ½`) | `[('m', 1)]` |
| `snake_case` | `case: 1, snake: 1` |
| `"до́м"` (`до` + stress mark, `м`) | `[('до', 1), ('м', 1)]` |
| `"ёлка"` (`е` + U+0308, `лка`) | `[('ёлка', 1)]` |
| `"İstanbul"` (`İstanbul`) | `[('i', 1), ('stanbul', 1)]` |
| `"a b​c"` (no-break space, zero-width space) | `[('a', 1), ('b', 1), ('c', 1)]` |

### 3. Words

A word is a maximal run of letters, where a `-` (U+002D) or `'` (U+0027,
which includes every `’` after step 4) also belongs to the word if it has a
letter immediately on both sides. Every other character separates, including other
dashes and hyphens (`—`, `–`, U+2010, U+2011, soft hyphen U+00AD) and other
apostrophe-like characters (`‘`, `` ` ``, `´`). U+02BC `ʼ` is a letter, so it
is part of the word and is not converted.

| Input | Expected output |
| --- | --- |
| `из-за стола из-за` | `из-за: 2, стола: 1` |
| `state-of-the-art` | `state-of-the-art: 1` |
| `don't don’t dont` | `don't: 2, dont: 1` |
| `'hello' hello` | `hello: 2` |
| `-cat- a--b` | `a: 1, b: 1, cat: 1` |
| `covid-19 1-a` | `a: 1, covid: 1` |
| `"don’t donʼt"` | `[("don't", 1), ("donʼt", 1)]` |
| `"don‘t"` | `[('don', 1), ('t', 1)]` |
| `слово—слово` | `слово: 2` |
| `"İ-i"` (`İ`, hyphen, `i`) | `[('i', 2)]` |
| `"й-й"` (`и` + breve, hyphen, twice) | `[('й-й', 1)]` |
| `"пере­нос из‑за"` (`пере`, soft hyphen, `нос`; `из`, U+2011, `за`) | `[('за', 1), ('из', 1), ('нос', 1), ('пере', 1)]` |

### 4. No other processing

No stop words are removed and there is no stemming. `ё` and `е` are different
letters, and look-alike letters from different scripts are different letters.

| Input | Expected output |
| --- | --- |
| `the cat the` | `the: 2, cat: 1` |
| `ёлка елка ёлка` | `ёлка: 2, елка: 1` |
| `"a а"` (Latin `a`, Cyrillic `а`) | `[('a', 1), ('а', 1)]` |
| `"i і"` (Latin `i`, Kazakh `і`) | `[('i', 1), ('і', 1)]` |

### 5. Ties and the cutoff

Sort by count, highest first. Break ties by the word in ascending code point
order (Python's default `str` comparison), then keep the first 10. Code point
order is not dictionary order: `ё` sorts after `я`, and Kazakh letters such as
`ә`, `қ`, `і` sort after `ё`.

| Input | Expected output |
| --- | --- |
| `b a b a c` | `a: 2, b: 2, c: 1` |
| `k j i h g f e d c b a` | `a` through `j`, each 1 |
| `k k a b c d e f g h i j` | `k: 2`, then `a` through `i`, each 1 |
| `ёлка яблоко` | `яблоко: 1, ёлка: 1` |
| `әке ёлка яблоко` | `яблоко: 1, ёлка: 1, әке: 1` |
| `"әке іс"` (`әке іс`) | `[('іс', 1), ('әке', 1)]` |
| `ақ бала` | `ақ: 1, бала: 1` |

### 6. Short and empty results

With fewer than 10 distinct words, return all of them; with none, return `[]`.

| Input | Expected output |
| --- | --- |
| `cat dog cat bird dog cat` | `cat: 3, dog: 2, bird: 1` |
| `""` | `[]` |
| ` ... !!! ` | `[]` |

### 7. Invalid input

A non-`str` argument raises `TypeError`, for example `top_words(None)` or
`top_words(b"abc")`. A `str` subclass is accepted and processed as
`str(text)`, so methods it overrides are not used: with `class S(str): pass`,
`top_words(S("cat"))` returns `[('cat', 1)]`, where `'cat'` is a plain `str`.

## Out of scope

Language detection, stop words, stemming, Latin–Cyrillic transliteration,
locale-aware casing, scripts other than Latin and Cyrillic, streaming input,
and performance on very large inputs.
