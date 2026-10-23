# Top words: specification

Source: [Top words, Literate Programming Book](https://github.com/bjones1/literate-programming-book/blob/main/exercises/top-words.md).

## Purpose and interface

Implement `top_words(text)` in `top_words.py`. It returns a new list of `(word, count)` tuples for the ten most frequent words in plain article text, which may contain English, Russian, Kazakh, or other Unicode letters. `text` must be a Python `str` or subclass; other types raise `TypeError`. The function performs no I/O and does not change external state.

## Word definition

1. Normalize the whole input to NFC, apply Unicode default case folding, then normalize to NFC again: `unicodedata.normalize("NFC", unicodedata.normalize("NFC", text).casefold())`. Apply tokenization only after this full pipeline.
2. A word is a maximal uninterrupted sequence that starts with a Unicode letter (general category `Lu`, `Ll`, `Lt`, `Lm`, or `Lo`) and may continue with letters or combining marks (`Mn`, `Mc`, or `Me`). Combining marks not attached to a word that has started are discarded; they do not attach across a separator or become a word prefix.
3. Every other character separates words. Separators end words but are not deleted to join surrounding text, and consecutive separators do not create empty words. Unicode general categories, not a character's appearance or name, determine whether it is a letter, mark, or separator. For example, U+02BC is a letter (`Lm`), while U+0027 and U+2019 are separators; U+2139 followed by U+FE0F is a letter followed by an attached mark.
4. Apostrophes, hyphens, and underscores deliberately split tokens. Thus contractions such as `don't` and hyphenated forms such as `well-known` produce separate fragments. This is a language-independent rule; the function does not preserve contractions or compounds as single words.
5. Case folding is locale-independent: no Turkish/Azerbaijani-specific tailoring is applied. In particular, `I` folds to `i`, while `İ` folds to `i\u0307`; these forms can therefore remain distinct. This behavior is deliberate, including for Kazakh Latin text.
6. Preserve differences that remain after the required normalization and case folding. For example, `е` and `ё`, `а` and `ә`, and visually similar Latin and Cyrillic letters stay distinct. Do not transliterate, strip accents, stem words, merge grammatical forms, or remove stop words. Count single-letter words and process markup such as HTML and URLs as plain text.

Character categories come from the Unicode database of the Python runtime used. The examples were checked against Unicode 15.0; Python 3.12 is a convenient baseline for that database, but a particular Unicode version is not otherwise pinned.

## Counting and ordering

Count every occurrence of each normalized word, but return each distinct word once. Sort by descending count, then by ascending lexicographic order of Unicode code points, as Python compares strings. Return only the first ten entries after sorting; do not expand a tie at position ten. Return all entries when fewer than ten distinct words exist, or `[]` when there are no words.

## Examples and acceptance cases

Inputs in this table use Python string-literal notation. Multiple examples in a cell are independent calls. Outputs are complete lists.

| ID | Behavior | Input → expected output |
| --- | --- | --- |
| T01 | Count occurrences and fold English, Russian, and Kazakh case | `"Cat CAT cat Dog"` → `[("cat", 3), ("dog", 1)]`<br>`"КОТ кот Кот"` → `[("кот", 3)]`<br>`"ҚАЗАҚ қазақ"` → `[("қазақ", 2)]` |
| T02 | Merge canonically equivalent spellings under NFC | `"café cafe\u0301 CAFÉ"` → `[("café", 3)]`<br>`"Ё е\u0308 ё"` → `[("ё", 3)]` |
| T03 | Keep accents, alphabets, and different letters distinct | `"cafe café"` → `[("cafe", 1), ("café", 1)]`<br>`"елка ёлка"` → `[("елка", 1), ("ёлка", 1)]`<br>`"a а a ә"` → `[("a", 2), ("а", 1), ("ә", 1)]` |
| T04 | Use case folding, including expansions, without locale-specific tailoring | `"Straße STRASSE"` → `[("strasse", 2)]`<br>`"İ I i\u0307 i"` → `[("i", 2), ("i\u0307", 2)]` |
| T05 | Separators split tokens; punctuation, digits, whitespace, and controls are not words | `"don't don’t"` → `[("don", 2), ("t", 2)]`<br>`"well-known well_known"` → `[("known", 2), ("well", 2)]`<br>`"#cat🙂dog+cat"` → `[("cat", 2), ("dog", 1)]`<br>`"abc123def 123 abc"` → `[("abc", 2), ("def", 1)]`<br>`"cat\x00dog cat\tdog\ncat\u00a0dog"` → `[("cat", 3), ("dog", 3)]` |
| T06 | Unicode categories, not visual appearance, decide what belongs to a word | `"don\u02bct"` → `[("don\u02bct", 1)]`<br>`"\u2139\ufe0f"` → `[("\u2139\ufe0f", 1)]` |
| T07 | Keep attached marks and discard unattached marks | `"q\u0301b q\u0301b"` → `[("q\u0301b", 2)]`<br>`"\u0301cat \u0301 cat"` → `[("cat", 2)]`<br>`"q,\u0301b"` → `[("b", 1), ("q", 1)]` |
| T08 | Apply NFC before and after case folding, before tokenization | `"\u03b1\u0345\u0301"` → `[("\u03ac\u03b9", 1)]`<br>`"\u01f0 J\u030c"` → `[("\u01f0", 2)]` |
| T09 | A change of script without a separator does not split a word | `"catкот catкот"` → `[("catкот", 2)]` |
| T10 | Keep stop words and grammatical forms; process markup and URLs as plain text | `"the the и и және a"` → `[("the", 2), ("и", 2), ("a", 1), ("және", 1)]`<br>`"cat cats cat"` → `[("cat", 2), ("cats", 1)]`<br>`"<b>cat</b> https://x.kz"` → `[("b", 2), ("cat", 1), ("https", 1), ("kz", 1), ("x", 1)]` |
| T11 | Break ties by code-point order, not appearance or locale; shorter prefixes come first | `"zebra apple zebra apple"` → `[("apple", 2), ("zebra", 2)]`<br>`"aa a"` → `[("a", 1), ("aa", 1)]`<br>`"қазақ кот apple"` → `[("apple", 1), ("кот", 1), ("қазақ", 1)]` |
| T12 | Return only the first ten after sorting, including when the cutoff is tied | `"k j i h g f e d c b a"` → `[("a", 1), ("b", 1), ("c", 1), ("d", 1), ("e", 1), ("f", 1), ("g", 1), ("h", 1), ("i", 1), ("j", 1)]` |
| T13 | Frequency takes priority over code-point order before the cutoff | `"a b c d e f g h i j k k"` → `[("k", 2), ("a", 1), ("b", 1), ("c", 1), ("d", 1), ("e", 1), ("f", 1), ("g", 1), ("h", 1), ("i", 1)]` |
| T14 | Return an empty list when no words are present | `""` → `[]`<br>`"123 !!! 🙂 \u0301\n\t"` → `[]` |
| T15 | Reject non-string inputs | `None` → `TypeError`<br>`123` → `TypeError`<br>`b"cat"` → `TypeError` |

Every call returns a fresh list, including an empty result; mutating one result does not affect a later call. Counts do not persist between calls, the function writes nothing to standard output or standard error, and a subclass of `str` is accepted.
