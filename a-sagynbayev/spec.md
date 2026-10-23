# Top Words — Specification

A news site shows a "Trending words" box beside each article. This spec defines
exactly which words appear in that box and with what counts. Articles may be in
English, Russian, or Kazakh (Cyrillic script), or any mix of them.

Notation: inputs are Python expressions, which are string literals except for
the D2 rows (so `́` is one character); outputs are Python values. A
`\uXXXX` escape in an expected output names the exact code point returned.

## D1. Interface

- Python 3, standard library only. File `top_words.py`, function:

  ```python
  def top_words(text: str) -> list[tuple[str, int]]: ...
  ```

- Returns a `list` of at most 10 `(word, count)` tuples, where `word` is a `str`
  and `count` is an `int` ≥ 1, in the order defined in D10.
- Each returned `word` is in its final form: normalized by D3 (NFC, `Mn` and
  `Cf` characters removed, apostrophe as U+0027) and lowercased by D7.
- The function is pure: it does not print, read files, read stdin, or modify
  global state.

| Input | Expected output | A different plausible reading would give |
|---|---|---|
| `"cat dog cat bird dog cat"` | `[("cat", 3), ("dog", 2), ("bird", 1)]` | `{"cat": 3, "dog": 2, "bird": 1}` or the string `"cat: 3, dog: 2, bird: 1"` |
| `""` | `[]` | `None` |
| `"   ,.!? 123 —"` | `[]` | `None`, or an error |

## D2. Invalid input type

If `text` is not an instance of `str` (for example `None`, `bytes`, a list),
raise `TypeError`. No other exceptions are raised for any `str` input.

| Input | Expected output | A different plausible reading would give |
|---|---|---|
| `None` | raises `TypeError` | `[]` |
| `b"cat"` | raises `TypeError` | `[("cat", 1)]` (decoding the bytes) |

## D3. Unicode normalization

Before finding words, the text is transformed in this order:

1. Normalize to Unicode NFC (`unicodedata.normalize("NFC", text)`), so a letter
   typed as base letter + combining mark becomes the single precomposed letter.
2. Delete every character whose Unicode category is `Mn` (non-spacing mark) or
   `Cf` (format character) that remains after step 1.
   - Removing `Mn` removes stress accents used in Russian texts (e.g.
     `о́`) so they do not split words.
   - Removing `Cf` removes invisible characters that occur inside words in web
     text: soft hyphen U+00AD, zero-width space U+200B, zero-width non-joiner
     and joiner U+200C/U+200D, word joiner U+2060, and BOM U+FEFF.
3. Replace `’` (U+2019) and `ʼ` (U+02BC) with `'` (U+0027). No other character
   is mapped to `'`; in particular `‘` (U+2018) and `` ` `` stay separators (D5).

Step 2 only removes marks that step 1 could not merge into a precomposed
letter. A mark that does have a precomposed form stays part of the letter:
`é` becomes `é` (U+00E9), and a grave stress mark on `и` or `е` becomes
`ѝ` (U+045D) or `ѐ` (U+0450). These are distinct letters (`ѝ` is a separate
word in Bulgarian), so `ѝ` and `и` are counted separately (D8).

| Input | Expected output | A different plausible reading would give |
|---|---|---|
| `"café café"` | `[("café", 2)]` | `[("cafe", 1), ("café", 1)]` or `[("cafe", 2)]` |
| `"моло́ко молоко"` | `[("молоко", 2)]` | `[("моло", 1), ("ко", 1), ("молоко", 1)]` |
| `"ѝ и"` | `[("ѝ", 1), ("и", 1)]` | `[("и", 2)]` |
| `"trend­ing trend​ing trending"` | `[("trending", 3)]` | `[("trend", 2), ("ing", 2), ("trending", 1)]` |
| `"don’t don't"` | `[("don't", 2)]` | `[("don’t", 1), ("don't", 1)]` |
| `"don‘t"` | `[("don", 1), ("t", 1)]` | `[("don't", 1)]` |

## D4. Letters

A **letter** is any character `c` for which `c.isalpha()` is `True` after D3.
This includes all Latin, Russian, and Kazakh letters (including
`ә ғ қ ң ө ұ ү һ і` and their capitals) and letters of other scripts.

Scripts that write vowels or other parts of a letter with combining marks
(Devanagari, Thai, Hebrew with points, and similar) are not supported: their
`Mn` marks are deleted by D3, and their `Mc` (spacing mark) characters are not
letters and therefore split words. The output for such text is still fully
determined by the rules, but the counted words are not the real words.

| Input | Expected output | A different plausible reading would give |
|---|---|---|
| `"қала қала"` | `[("қала", 2)]` | `[("ала", 2)]` (only `[a-zA-Zа-яА-Я]` counted as letters) |
| `"Өнер ұстаз"` | `[("өнер", 1), ("ұстаз", 1)]` | `[("нер", 1), ("стаз", 1)]` |
| `"हिन्दी"` | `[("ह", 1), ("नद", 1)]` | `[("हिन्दी", 1)]` |

## D5. Separators

Every character that is not a letter and not a joiner (D6) separates words.
This includes whitespace, digits, underscore, punctuation, quotation marks
(`" « » ‘ „` and `` ` ``), en and em dashes (`– —`), U+2010 hyphen, U+2011
non-breaking hyphen, symbols, and emoji. Only ASCII `-` joins words (D6).
Digits are never part of a word, and numbers are never counted.

| Input | Expected output | A different plausible reading would give |
|---|---|---|
| `"covid19 COVID-19 2026"` | `[("covid", 2)]` | `[("covid19", 1), ("covid-19", 1), ("2026", 1)]` |
| `"snake_case"` | `[("snake", 1), ("case", 1)]` | `[("snake_case", 1)]` |
| `"pre—post pre–post"` | `[("pre", 2), ("post", 2)]` | `[("pre—post", 1), ("pre–post", 1)]` |
| `"pre‐post pre‑post pre-post"` | `[("pre", 2), ("post", 2), ("pre-post", 1)]` | `[("pre-post", 3)]` |
| `"«Алма» \"алма\""` | `[("алма", 2)]` | `[("«алма»", 1), ("\"алма\"", 1)]` |
| `"site.com/news"` | `[("site", 1), ("com", 1), ("news", 1)]` | `[("site.com/news", 1)]` |

## D6. Joiners: hyphen and apostrophe

A **joiner** is `-` (U+002D) or `'` (U+0027, after D3). A joiner is part of a
word only when it has a letter immediately before it and a letter immediately
after it. Otherwise it is a separator. Formally, a **word** is a maximal
substring matching `L+ (J L+)*`, where `L` is a letter and `J` is a joiner.

Consequences: a hyphenated compound is one word; leading and trailing hyphens
and apostrophes are dropped; two joiners in a row split the word; a word
hyphenated across a line break is two words.

| Input | Expected output | A different plausible reading would give |
|---|---|---|
| `"well-known well known"` | `[("well-known", 1), ("well", 1), ("known", 1)]` | `[("well", 2), ("known", 2)]` |
| `"кто-нибудь"` | `[("кто-нибудь", 1)]` | `[("кто", 1), ("нибудь", 1)]` |
| `"don't dont"` | `[("don't", 1), ("dont", 1)]` | `[("don", 1), ("t", 1), ("dont", 1)]` or `[("dont", 2)]` |
| `"'quoted' students' -dash-"` | `[("quoted", 1), ("students", 1), ("dash", 1)]` | `[("'quoted'", 1), ("students'", 1), ("-dash-", 1)]` |
| `"a--b a'-b"` | `[("a", 2), ("b", 2)]` | `[("a--b", 1), ("a'-b", 1)]` |
| `"trend-\ning trending"` | `[("trend", 1), ("ing", 1), ("trending", 1)]` | `[("trending", 2)]` |

## D7. Case

Matching is case-insensitive. After the words are found (D6), each word is
converted with `str.lower()`, and then every `Mn` character produced by the
conversion is deleted. The word is returned in that form.

The second step matters only for letters whose lowercase form contains a
combining mark, such as `İ` (U+0130), for which `str.lower()` returns `i`
followed by U+0307. With the step, `İ` lowercases to plain `i`.

| Input | Expected output | A different plausible reading would give |
|---|---|---|
| `"Алма алма АЛМА"` | `[("алма", 3)]` | `[("Алма", 1), ("алма", 1), ("АЛМА", 1)]` or `[("Алма", 3)]` |
| `"The the THE"` | `[("the", 3)]` | `[("The", 3)]` |
| `"İstanbul istanbul"` | `[("istanbul", 2)]` | `[("i̇stanbul", 1), ("istanbul", 1)]` |

## D8. Exact-form counting

Two words are the same only if their final forms (after D3 and D7) are equal.
There is no stemming, lemmatization, transliteration, or letter merging:

- `ё` and `е` are different letters.
- Latin and Cyrillic look-alike letters are different (`c` U+0063 vs `с` U+0441).
- A hyphenated compound does not count toward its parts.

| Input | Expected output | A different plausible reading would give |
|---|---|---|
| `"cat cats"` | `[("cat", 1), ("cats", 1)]` | `[("cat", 2)]` |
| `"Ёлка ёлка елка"` | `[("ёлка", 2), ("елка", 1)]` | `[("елка", 3)]` |
| `"cat сat"` | `[("cat", 1), ("сat", 1)]` | `[("cat", 2)]` |
| `"Қазақстан Казахстан"` | `[("қазақстан", 1), ("казахстан", 1)]` | `[("қазақстан", 2)]` |

## D9. No filtering

Every word counts. There is no stop-word list and no minimum word length.

| Input | Expected output | A different plausible reading would give |
|---|---|---|
| `"the cat and the dog"` | `[("the", 2), ("cat", 1), ("and", 1), ("dog", 1)]` | `[("cat", 1), ("dog", 1)]` |
| `"и я и a"` | `[("и", 2), ("я", 1), ("a", 1)]` | `[]` |

## D10. Order

Results are sorted by count, highest first. Words with equal counts are ordered
by the position of their first occurrence in the text (earlier first), where
"occurrence" means any occurrence of the final (lowercase) form.

| Input | Expected output | A different plausible reading would give |
|---|---|---|
| `"b a a b c"` | `[("b", 2), ("a", 2), ("c", 1)]` | `[("a", 2), ("b", 2), ("c", 1)]` (alphabetical) |
| `"B a b A"` | `[("b", 2), ("a", 2)]` | `[("a", 2), ("b", 2)]` |
| `"я а я а"` | `[("я", 2), ("а", 2)]` | `[("а", 2), ("я", 2)]` (alphabetical) |

## D11. Limit

Return the first 10 entries of the D10 order. If there are fewer than 10
distinct words, return all of them. Words tied with the 10th entry but
occurring later are dropped.

| Input | Expected output | A different plausible reading would give |
|---|---|---|
| `"a b c d e f g h i j k l"` | `[("a", 1), ("b", 1), ("c", 1), ("d", 1), ("e", 1), ("f", 1), ("g", 1), ("h", 1), ("i", 1), ("j", 1)]` | all 12 words (ties at the cutoff kept) |
| `"l l a b c d e f g h i j k"` | `[("l", 2), ("a", 1), ("b", 1), ("c", 1), ("d", 1), ("e", 1), ("f", 1), ("g", 1), ("h", 1), ("i", 1)]` | `[("l", 2), ("a", 1), …, ("j", 1), ("k", 1)]` (12 entries) |
| `"one two"` | `[("one", 1), ("two", 1)]` | padded with empty entries to 10 |

## D12. Input size and performance

There is no limit on the length of `text`. With `n = len(text)`, the function
runs in O(n log n) time (O(n) to normalize, scan, and count, plus sorting the
distinct words) and uses O(n) extra memory. It must not use an algorithm that
rescans the text per word (for example calling `text.count(word)` for each
word), which is O(n²) in the worst case.