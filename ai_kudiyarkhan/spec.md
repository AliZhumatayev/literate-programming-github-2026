# Top Words — Specification

## Purpose

Implement a Python 3.13.5 function named `top_words` in a module named `top_words.py`. Given an article as text, return the ten most frequent words and their occurrence counts. The function must handle English, Russian, and Kazakh text according to the same rules.

## Interface

- **Signature:** `top_words(text: str) -> list[tuple[str, int]]`. Runtime type annotations are optional, but the function must accept exactly one argument and return the specified list.
- **Input:** One Python `str` (including subclasses of `str`, as determined by `isinstance(text, str)`) containing the article text. Every string is interpreted literally as text, even if it happens to resemble a filename or URL; the function never opens paths or fetches URLs.
- **Output:** A Python list of up to 10 `(word, count)` tuples. Each `word` is a Unicode-normalized, case-folded Python string; each `count` is a positive integer.
- The function returns the list directly. It does not print, read files, fetch URLs, or modify the input. It does not perform language detection.
- Passing a value for which `isinstance(text, str)` is false raises `TypeError`.

## Exact processing rules

1. **Normalize:** For an input string `text`, compute `unicodedata.normalize("NFC", unicodedata.normalize("NFD", text).casefold())`. **Order matters:** first NFD, then Unicode `casefold()`, then NFC. This implements canonical caseless matching under the Unicode data provided by Python 3.13.5. Do not transliterate, translate, remove accent marks, or use compatibility normalization (NFKC).
2. **Identify words:** Read the normalized string from left to right. A word begins with a character `c` satisfying `c.isalpha()`. Continue the same word while each following character is either alphabetic (`c.isalpha()`) or a Unicode combining mark (`unicodedata.category(c).startswith("M")`). A word ends immediately before any other character. A combining mark that appears without a preceding letter in the **current word** is ignored and cannot start a word; it also cannot join words across punctuation. Thus marks inside words are preserved, and unaccented and accented words can count separately. This rule supports Turkish/Kazakh-Latin `İ`, Russian stress marks, and letter-plus-mark scripts such as Devanagari. Spaces, punctuation, ASCII/curly apostrophes, hyphens, digits, underscores, and emoji are separators. The modifier letter apostrophe (`ʼ`, U+02BC) satisfies `isalpha()` and remains in a word. Adjacent alphabetic characters from different scripts belong to the same word. One-letter words are included.
3. **Count:** Count each occurrence of an identical normalized word. Include common words such as `the` and `и`: there is no stop-word filtering, stemming, or lemmatization.
4. **Sort:** Order word/count pairs by descending count. If counts are equal, order words by ascending Python string lexicographic order (Unicode code-point comparison), **not** by locale-dependent dictionary order.
5. **Limit:** Return the first 10 pairs in that order, or all pairs if there are fewer than 10 unique words. If there are no words, return `[]`.

## Python and Unicode reproducibility

The reference environment is **CPython 3.13.5**, using its bundled Unicode character database; this fixes the relevant case-folding, alphabetic-property, and mark-category definitions for exact cross-machine reproducibility. Other Python versions are not guaranteed to have identical Unicode tables. The return values below were tested in Python 3.13.5.

## Examples and required results

The following expressions are Python strings. Expected results are exact Python values, not printed display strings.

| Behavior tested | Input (`text`) | Expected result of `top_words(text)` |
| --- | --- | --- |
| Frequencies and ordering | `"cat dog cat bird dog cat"` | `[("cat", 3), ("dog", 2), ("bird", 1)]` |
| Case-insensitive matching | `"Cat CAT cat DOG dog"` | `[("cat", 3), ("dog", 2)]` |
| Punctuation, apostrophes, hyphens, digits, underscores | `"red-blue red's red_2blue 123"` | `[("red", 3), ("blue", 2), ("s", 1)]` |
| One-letter words; stop words included | `"I a a the THE"` | `[("a", 2), ("the", 2), ("i", 1)]` |
| Russian and Kazakh Unicode case folding | `"Қала қала ҚАЛА, город ГОРОД."` | `[("қала", 3), ("город", 2)]` |
| Case folding beyond lowercasing (`ß` to `ss`) | `"Straße STRASSE"` | `[("strasse", 2)]` |
| Apostrophe-like Unicode letter vs punctuation | `"don't donʼt"` | `[("don", 1), ("donʼt", 1), ("t", 1)]` |
| Combining mark stays inside word | `"x\u0301y"` | `[("x́y", 1)]` |
| No stemming or lemmatization | `"cat cats cat"` | `[("cat", 2), ("cats", 1)]` |
| No transliteration between writing systems | `"mir мир mir"` | `[("mir", 2), ("мир", 1)]` |
| No script-based word boundary | `"catкот"` | `[("catкот", 1)]` |
| A string resembling a filename is treated as text | `"README.md"` | `[("md", 1), ("readme", 1)]` |
| Unicode canonical equivalence | `"cafe\u0301 café"` | `[("café", 2)]` |
| Dotted I does not split a word | `"İstanbul"` | `[("i̇stanbul", 1)]` |
| Russian stress mark stays in the word | `"доро\u0301га дорога"` | `[("доро́га", 1), ("дорога", 1)]` |
| Devanagari vowel signs stay in the word | `"हिन्दी"` | `[("हिन्दी", 1)]` |
| Canonical equivalence under caseless matching | `"α\u0345\u0301 α\u0301\u0345"` | `[("άι", 2)]` |
| Decomposed Russian й and ё | `"и\u0306 й е\u0308 ё"` | `[("й", 2), ("ё", 2)]` |
| A leading mark cannot start a word | `"\u0301abc"` | `[("abc", 1)]` |
| Tie-break rule independent of first occurrence | `"zoo ant zoo ant"` | `[("ant", 2), ("zoo", 2)]` |
| Exactly ten distinct words | `"j i h g f e d c b a"` | `[("a", 1), ("b", 1), ("c", 1), ("d", 1), ("e", 1), ("f", 1), ("g", 1), ("h", 1), ("i", 1), ("j", 1)]` |
| Tie for 10th and 11th with count two | `"a a b b c c d d e e f f g g h h i i z z y y"` | `[("a", 2), ("b", 2), ("c", 2), ("d", 2), ("e", 2), ("f", 2), ("g", 2), ("h", 2), ("i", 2), ("y", 2)]` |
| Ten-word cutoff including ties | `"k j i h g f e d c b a"` | `[("a", 1), ("b", 1), ("c", 1), ("d", 1), ("e", 1), ("f", 1), ("g", 1), ("h", 1), ("i", 1), ("j", 1)]` |
| Whitespace, punctuation and emoji only | `" \n!?🙂42"` | `[]` |
| Empty article | `""` | `[]` |
| Non-string argument | `None` | Raises `TypeError` |

## Acceptance criteria

A conforming implementation must satisfy **all** the rules above and return exactly the expected values for every example. It must not depend on the machine's locale or on external stop-word dictionaries. For exact cross-machine matching, use CPython 3.13.5 and its bundled Unicode tables.
