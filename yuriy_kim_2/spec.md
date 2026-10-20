# Specification: Top words

## Overview

`top_words` finds the 10 most common words in an article's text and how many
times each appears. Articles may be in English, Russian, or Kazakh. The
function has no language detection and no language-specific rules: one set of
Unicode rules applies to all text. In particular there is no locale-aware or
Turkic casing (Latin `I` lowercases to `i`, never to `ı`), and Latin-script
Kazakh text follows the same rules as any other text.

## Interface

- Python 3.9 or later. The function is named `top_words` and lives in
  `top_words.py`.
- Signature: `top_words(text: str) -> list[tuple[str, int]]`
- Standard library only.
- Unicode behavior (`str.isalpha()`, `str.lower()`, NFC) is whatever the
  running Python's `unicodedata` database provides. The examples in this
  document were checked on Python 3.13 (Unicode 15.1); statements about "the
  only letter in Unicode" and similar refer to that version.
- The return value is a list of `(word, count)` tuples with at most 10 items.
  Words in the result are plain `str` and lowercase; counts are plain `int`.
- Ordering: by count, highest first. Words with equal counts are ordered by the
  word itself, ascending, using Python's default `str` comparison (code point
  order). ASCII letters therefore sort before Cyrillic letters, and within
  Cyrillic the order is by code point, not by dictionary alphabet (see Rule 5).
- If `text` is not a `str` (for example `None`, `bytes`, `bytearray`, a list,
  or a number), raise `TypeError`. An instance of a `str` subclass is a `str`
  and is accepted (Rule 7). The check happens before any other processing.

## Rules

The function applies these steps, in this order:

1. Type check (Rule 7).
2. NFC normalization of the whole text (Rule 1).
3. `str.lower()` on the whole text (Rule 1). The result is not normalized
   again.
4. Replace `’` (U+2019) with `'` (Rule 3).
5. Split into words (Rules 2 and 3). Whether a hyphen or apostrophe is a
   joiner is decided on the text as it is after steps 2 to 4.
6. Count words exactly as they are (Rule 4).
7. Sort, then keep the first 10 (Rules 5 and 6).

Each rule has a table of examples. Each example would produce a different
result under some other plausible reading of the rule.

Examples that involve combining marks, look-alike characters, or other
invisible characters are written as Python string literals with `\u` escapes,
and their expected output as a Python list of tuples, so that an editor that
normalizes or re-encodes text cannot change what the example tests. The other
tables use the shorthand `word: count`, listed in output order.

### 1. Normalization and case

Apply Unicode NFC normalization, then convert to lowercase with `str.lower()`.
Words that differ only by case, or only by how a letter is composed in Unicode,
are the same word, with the exceptions described below.

Both steps apply to the whole text before it is split into words, and in this
order. `str.lower()` is used, not `str.casefold()`, so `ß` stays `ß` and is not
the same word as `ss`.

The result of `str.lower()` is not normalized again. Lowercasing can produce a
sequence that NFC would compose. `T` followed by U+0308 has no precomposed form,
so NFC leaves it alone; lowercasing then gives `t` followed by U+0308, which is
not composed into `ẗ` (U+1E97). U+0308 is a combining mark and separates words
(Rule 2). So `T` + U+0308 and the precomposed `ẗ` are different words, even
though they differ only by case and composition. Normalizing after lowercasing
would give a different result.

`str.lower()` can also depend on context. Greek capital sigma `Σ` lowercases to
`ς` at the end of a word and to `σ` elsewhere, and "end of a word" is judged on
the whole text, so it can be affected by following punctuation. This only
concerns Greek, which is out of scope, but it follows from applying
`str.lower()` to the whole text and is not special-cased.

| Input                                     | Expected output  |
| ----------------------------------------- | ---------------- |
| `Cat CAT cat`                             | `cat: 3`         |
| `Привет ПРИВЕТ`                           | `привет: 2`      |
| `Қала қала`                               | `қала: 2`        |
| `Қала`, newline, `қала`, tab, `ҚАЛА`      | `қала: 3`        |
| `"\u0438\u0306 \u0439"` (`и` + combining breve, space, `й`) | `[('\u0439', 2)]` (`й: 2`) |
| `"T\u0308"` (`T` + U+0308) | `[('t', 1)]` (NFC does nothing, `lower()` gives `t` + U+0308, the mark separates; lowercasing first and normalizing after would give `ẗ`) |
| `"\u1e97"` (precomposed `ẗ`) | `[('\u1e97', 1)]` (a different word from the previous row) |
| `"Stra\u00dfe Strasse"` | `[('strasse', 1), ('stra\u00dfe', 1)]` (`lower()`, not `casefold()`) |
| `"I \u0131"` (`I`, space, dotless `ı`) | `[('i', 1), ('\u0131', 1)]` (no Turkic casing: `I` becomes `i`) |
| `"\u0391\u03a3 \u0391\u03a3.\u0392"` (`ΑΣ ΑΣ.Β`) | `[('\u03b1\u03c2', 1), ('\u03b1\u03c3', 1), ('\u03b2', 1)]` (the same capital word becomes `ας` or `ασ` depending on what follows) |

### 2. What a letter is

A character is a letter if `str.isalpha()` returns `True` for it. Digits,
punctuation, symbols, emoji, and whitespace are not letters and separate words.

Combining marks (such as the stress mark U+0301) are not letters either, so
they also separate words. The only exception is a mark that NFC normalization
(Rule 1) has already composed into a precomposed letter, as with `й` and `ё`.
A stress mark on a vowel has no precomposed form, so a stressed word is split
around the mark. This is intended: the function does not strip or ignore
combining marks.

Lowercasing (Rule 1) can itself produce a combining mark. `İ` (U+0130, Latin
capital I with dot above) lowercases to `i` followed by U+0307 (combining dot
above), and NFC has no precomposed form for that pair. U+0307 is a combining
mark, so it separates words like any other. The function has no special case
for `İ`. This is the only letter in Unicode (version 15.1) whose `str.lower()`
result contains a non-letter.

NFC can also turn a letter into a letter followed by a mark. For example,
U+0958 (Devanagari letter qa) is decomposed by NFC into U+0915 and the nukta
U+093C, and the nukta is a combining mark that separates words. This affects
no Latin or Cyrillic letter. As with `İ`, there is no special case.

Whitespace and invisible characters are not letters. This includes the
no-break space U+00A0, the zero-width space U+200B, and the zero-width joiner
and non-joiner U+200D and U+200C. They separate words and are not removed or
ignored.

| Input                        | Expected output                      |
| ---------------------------- | ------------------------------------ |
| `Hello, world! Hello — world…` | `hello: 2, world: 2`                 |
| `«Привет» привет`            | `привет: 2`                          |
| `http://x.com/a 😀`          | `a: 1, com: 1, http: 1, x: 1`        |
| `covid19 covid`              | `covid: 2`                           |
| `b2b`                        | `b: 2`                               |
| `2024 42`                    | `[]` (no words)                      |
| `"m\u00b2"` (`m²`)           | `[('m', 1)]` (`²` is not a letter; a regex like `[^\W\d_]+` would wrongly keep it) |
| `"\u2163 \u00bd"` (`Ⅳ ½`)    | `[]` (a Roman numeral and a fraction are not letters; same regex trap) |
| `snake_case`                 | `case: 1, snake: 1` (underscore is not a letter) |
| `"\u0434\u043e\u0301\u043c"` (`до` with a stress mark, then `м`) | `[('\u0434\u043e', 1), ('\u043c', 1)]` (`до: 1, м: 1`; the mark separates, and no precomposed `о́` exists) |
| `"\u0435\u0308\u043b\u043a\u0430"` (`е` + U+0308, `лка`) | `[('\u0451\u043b\u043a\u0430', 1)]` (`ёлка: 1`; NFC composes `е` + U+0308 into `ё`) |
| `"\u0130stanbul"` (`İstanbul`) | `[('i', 1), ('stanbul', 1)]` (`İ` lowercases to `i` + U+0307; the mark separates) |
| `"\u0958"` (U+0958) | `[('\u0915', 1)]` (NFC gives U+0915 + U+093C; the nukta separates) |
| `"a\u200bb"` (`a`, zero-width space, `b`) | `[('a', 1), ('b', 1)]` (an invisible character separates) |
| `"a\u00a0b"` (`a`, no-break space, `b`) | `[('a', 1), ('b', 1)]` (not only ASCII space separates) |

### 3. Words

A word is a maximal run of one or more letters. In addition, a hyphen-minus
(`-`, U+002D) or an apostrophe (`'` U+0027, or `’` U+2019) is part of a word
when it has a letter immediately before it and a letter immediately after it.
The typographic apostrophe `’` is converted to `'` first, so both spellings are
the same word. Any other hyphen or dash (such as `—` or `–`) is a separator.
A hyphen or apostrophe at the start or end of a word, or next to another
non-letter, is a separator.

Only these three code points are joiners: U+002D, U+0027, and U+2019. Characters
that merely look like them are not:

- U+02BC (modifier letter apostrophe) is a letter under Rule 2, so it is
  simply part of the word. It is not converted to `'`, so `donʼt` and `don't`
  are different words.
- U+2018 (left single quotation mark), U+0060 (grave accent) and U+00B4 (acute
  accent) are separators, so `don‘t` is the two words `don` and `t`.
- U+2010 (hyphen) is a separator like any other dash. `—` and `–` are
  separators even when they have no spaces around them, as they often do in
  Russian text (`слово—слово` is two occurrences of `слово`).
- U+00AD (soft hyphen) and U+2011 (non-breaking hyphen) are separators. A word
  broken by a soft hyphen, as often happens in text copied from PDFs or web
  pages, is split into two words. The function does not remove soft hyphens.

"Immediately before" and "immediately after" refer to the text after NFC
normalization, lowercasing, and the `’` replacement (see the order of steps at
the start of this section). A combining mark next to a hyphen or apostrophe is a
non-letter, so the joiner is a separator. This also applies to a mark produced
by lowercasing (`İ`).

| Input                  | Expected output                  |
| ---------------------- | -------------------------------- |
| `из-за стола из-за`    | `из-за: 2, стола: 1`             |
| `state-of-the-art`     | `state-of-the-art: 1`            |
| `don't don’t dont`     | `don't: 2, dont: 1`              |
| `'hello' hello`        | `hello: 2`                       |
| `-cat- a--b`           | `a: 1, b: 1, cat: 1`             |
| `covid-19`             | `covid: 1` (a hyphen followed by a digit is a separator) |
| `1-a`                  | `a: 1` (a hyphen preceded by a digit is a separator) |
| `"don\u2019t don\u02bct"` (`don’t`, `donʼt` with U+02BC) | `[("don't", 1), ("don\u02bct", 1)]` (U+02BC is a letter, not converted) |
| `"don\u2018t"`         | `[('don', 1), ('t', 1)]` (U+2018 is a separator) |
| `"don\u0060t"`         | `[('don', 1), ('t', 1)]` (U+0060 is a separator) |
| `"don\u00b4t"`         | `[('don', 1), ('t', 1)]` (U+00B4 is a separator) |
| `слово—слово`          | `слово: 2` (an em dash with no spaces is a separator) |
| `a–b`                  | `a: 1, b: 1` (an en dash with no spaces is a separator) |
| `"a\u2010b"`           | `[('a', 1), ('b', 1)]` (U+2010 is a separator) |
| `"\u0130-i"` (`İ`, hyphen, `i`) | `[('i', 2)]` (`İ` becomes `i` + U+0307, so the hyphen follows a mark and separates; with a plain `I` it would be one word, `i-i`) |
| `"\u0434\u043e\u0301-\u043c"` (`до` + stress mark, hyphen, `м`) | `[('\u0434\u043e', 1), ('\u043c', 1)]` (the hyphen follows a mark and separates) |
| `"\u0438\u0306-\u0438\u0306"` (`и` + breve, hyphen, `и` + breve) | `[('\u0439-\u0439', 1)]` (NFC has composed `й` first, so the hyphen is between letters) |
| `"\u043f\u0435\u0440\u0435\u00ad\u043d\u043e\u0441"` (`пере`, soft hyphen, `нос`) | `[('\u043d\u043e\u0441', 1), ('\u043f\u0435\u0440\u0435', 1)]` (`нос: 1, пере: 1`; the soft hyphen separates) |
| `"\u0438\u0437\u2011\u0437\u0430"` (`из`, non-breaking hyphen, `за`) | `[('\u0437\u0430', 1), ('\u0438\u0437', 1)]` (`за: 1, из: 1`; U+2011 is a separator) |

### 4. No other processing

Words are counted exactly as they appear after rules 1 to 3.

- No stop words are removed, so `the`, `и`, and `я` are counted like any word.
- No stemming or lemmatization.
- `ё` and `е` are different letters, so `ёлка` and `елка` are different words.
- Letters that look alike in different scripts are different letters, so Latin
  `a` (U+0061) and Cyrillic `а` (U+0430) are different words.
- Mixed-script text is fine; each word is counted separately.

| Input                | Expected output                |
| -------------------- | ------------------------------ |
| `the cat the`        | `the: 2, cat: 1`               |
| `я и я`              | `я: 2, и: 1`                   |
| `ёлка елка ёлка`     | `ёлка: 2, елка: 1`             |
| `"a \u0430"` (Latin `a`, space, Cyrillic `а`) | `[('a', 1), ('\u0430', 1)]` (two words) |
| `"i \u0456"` (Latin `i`, space, Cyrillic Kazakh `і` U+0456) | `[('i', 1), ('\u0456', 1)]` (two words) |
| `кот cat`            | `cat: 1, кот: 1`               |

### 5. Ties and the cutoff

Sort first, then keep the first 10. Equal counts are ordered by the word itself
in code point order (see Interface), so the result is deterministic and the
cutoff among tied words is deterministic too.

"Code point order" is not the same as the dictionary alphabet of a language.
For Cyrillic text this matters: `ё` (U+0451) sorts after `я` (U+044F), not
between `е` and `ж`, and the Kazakh letters `ә`, `қ`, `ң`, `і`, `ү`, `ұ` and
others sort after `ё`. Among themselves they are in the order
`і`, `ғ`, `қ`, `ң`, `ү`, `ұ`, `һ`, `ә`, `ө`, which is not the Kazakh dictionary
order. Words are compared like any `str`, character by character, so a Kazakh
letter only decides the order at the first position where two words differ; a
word that merely contains one later in the word is not sorted last. ASCII
letters sort before all of them. The same holds for
Latin letters in the Latin-1, Latin Extended-A and Latin Extended-B blocks, but
not for all Latin letters: Latin Extended Additional (for example `ạ`, U+1EA1)
and fullwidth Latin letters sort after Cyrillic.

| Input                         | Expected output                                  |
| ----------------------------- | ------------------------------------------------ |
| `b a b a c`                   | `a: 2, b: 2, c: 1`                               |
| `a b c d e f g h i j k`       | `a` through `j`, each 1 (`k` is cut)             |
| `k j i h g f e d c b a`       | `a` through `j`, each 1 (`k` is cut by the sort, not by first appearance) |
| `k k a b c d e f g h i j`     | `k: 2`, then `a` through `i`, each 1             |
| `ёлка яблоко`                 | `яблоко: 1, ёлка: 1` (`я` before `ё`)            |
| `әке ёлка яблоко`             | `яблоко: 1, ёлка: 1, әке: 1` (words starting with a Kazakh letter come after `ё`) |
| `кот cat әке`                 | `cat: 1, кот: 1, әке: 1` (ASCII, then Cyrillic, then Kazakh-specific) |
| `"\u04d9\u043a\u0435 \u0456\u0441"` (`әке іс`) | `[('\u0456\u0441', 1), ('\u04d9\u043a\u0435', 1)]` (`і` U+0456 sorts before `ә` U+04D9; dictionary order would differ) |
| `ақ бала`                     | `ақ: 1, бала: 1` (words are compared character by character; `қ` comes second and does not decide) |

### 6. Short and empty results

Return every distinct word when there are fewer than 10, and an empty list when
there are none. This is not an error.

| Input                    | Expected output        |
| ------------------------ | ---------------------- |
| `cat dog cat bird dog cat` | `cat: 3, dog: 2, bird: 1` |
| `""` (empty string)      | `[]`                   |
| ` ... !!! `              | `[]`                   |

### 7. Invalid input

| Input        | Expected output |
| ------------ | --------------- |
| `None`       | `TypeError`     |
| `b"abc"`     | `TypeError`     |
| `bytearray(b"abc")` | `TypeError` |
| `["cat"]`    | `TypeError`     |
| `5`          | `TypeError`     |
| `True`       | `TypeError`     |

An instance of a subclass of `str` is accepted and handled like any `str`:
with `class S(str): pass`, `top_words(S("cat"))` returns `[('cat', 1)]`, and the
word in the result is a plain `str`.

## Out of scope

Language detection, stop-word lists, stemming, transliteration between Latin and
Cyrillic Kazakh, locale-aware or Turkic casing, scripts other than Latin and
Cyrillic (Greek and Devanagari behavior above is a consequence of the general
rules, not a requirement), streaming input, and performance on very large
inputs.