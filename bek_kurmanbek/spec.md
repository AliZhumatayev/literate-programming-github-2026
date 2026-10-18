# Top words: specification

Source: [Top words, Literate Programming Book](https://github.com/bjones1/literate-programming-book/blob/main/exercises/top-words.md).

## Purpose and interface

A news site needs the ten most common words in an article and their occurrence counts. Articles may contain English, Russian, Kazakh, or a mixture of these languages.

Implement `top_words(text)` in `top_words.py`, using Python 3. The function accepts one argument, `text`, which must be a Python `str` (a subclass of `str` is also accepted). It returns a new `list` of `(word, count)` tuples. Each word is a normalized string, and its count is a positive integer. The limit is always ten; there is no additional argument for choosing the limit.

The function returns its result without printing, reading files, accessing the network, or changing external state. Its result depends only on the supplied text. Non-string arguments raise `TypeError`; the exception message is not part of the contract.

## Normalization and definition of a word

1. Normalize the entire input to Unicode NFC, apply Unicode case folding, and normalize to NFC again, in that order. In Python notation, the result is `unicodedata.normalize("NFC", unicodedata.normalize("NFC", text).casefold())`. Thus uppercase and lowercase forms are counted together, and canonically equivalent spellings are counted together.
2. A word starts with a Unicode letter: a character whose general category is `Lu`, `Ll`, `Lt`, `Lm`, or `Lo`. After its first letter, it contains the longest uninterrupted sequence of letters and combining marks (categories `Mn`, `Mc`, and `Me`). Combining marks encountered when no word is currently being collected are ignored: they do not attach across a separator or become a prefix of a later word. Apply these rules only after completing the entire normalization pipeline in Rule 1.
3. Every character outside the letter and combining-mark categories in Rule 2 separates words. Examples include whitespace, digits, punctuation, control characters, the straight apostrophe U+0027, the curly apostrophe U+2019, the hyphen-minus U+002D, the underscore U+005F, and the smiling-face emoji U+1F642. Unicode general categories are authoritative: a character's name, visual appearance, or emoji presentation does not override Rule 2. Thus U+02BC (MODIFIER LETTER APOSTROPHE, category `Lm`) is a letter, and U+2139 followed by U+FE0F is a letter followed by an attached mark. Separators split words; they are not deleted to join the letters on either side. Consecutive separators do not create empty words.
4. Preserve accents and differences between letters that remain after the normalization and case folding required by Rule 1. For example, `е` and `ё`, `а` and `ә`, and visually similar Latin and Cyrillic letters remain different. Do not perform additional transliteration, accent stripping, or alphabet substitution. Unicode letters outside the three named languages are also accepted under the same rules.
5. Count every resulting word, including single-letter words and common words such as `the`, `и`, and `және`. Do not remove stop words, stem words, or merge grammatical forms. Process the input as plain text; do not parse or remove HTML, URLs, or other markup.

For a reproducible definition of normalization, case folding, and character categories, this specification uses Unicode 15.0. Python 3.12 is recommended because its standard library provides this Unicode database. A different implementation is acceptable if it gives the same behavior.

## Counting, ordering, and result size

Count every occurrence of each normalized word in the whole input. Include each distinct normalized word only once in the result.

Sort all distinct words by descending occurrence count. Break ties by ascending lexicographic order of the normalized words' Unicode code points, exactly as Python compares strings. This is not locale-specific dictionary order or order of first appearance.

Return the first `min(10, number_of_distinct_words)` entries after sorting. Do not include additional tied words beyond the tenth entry. With fewer than ten distinct words, return all of them. With no words, return `[]`.

## Examples and acceptance cases

The inputs and outputs below use Python literal notation. For example, `\n` is a newline, `\t` is a tab, `\u00a0` is a non-breaking space, and `\u0301` is a combining acute accent. For invalid input, the expected output column names the required exception. The outputs are complete lists, not abbreviated displays.

| ID | Decision checked | Input | Expected output |
| --- | --- | --- | --- |
| T01 | Count occurrences; return tuples; return fewer than ten when appropriate | `"cat dog cat bird dog cat"` | `[("cat", 3), ("dog", 2), ("bird", 1)]` |
| T02 | English case folding | `"Cat CAT cat Dog"` | `[("cat", 3), ("dog", 1)]` |
| T03 | Russian case folding | `"КОТ кот Кот"` | `[("кот", 3)]` |
| T04 | Kazakh text and case folding | `"ҚАЗАҚ қазақ ӘЛЕМ әлем"` | `[("қазақ", 2), ("әлем", 2)]` |
| T05 | Preserve all nine additional Kazakh letters; Unicode tie order | `"Ә Ғ Қ Ң Ө Ұ Ү Һ І ә ғ қ ң ө ұ ү һ і"` | `[("і", 2), ("ғ", 2), ("қ", 2), ("ң", 2), ("ү", 2), ("ұ", 2), ("һ", 2), ("ә", 2), ("ө", 2)]` |
| T06 | Canonically equivalent accents merge | `"café cafe\u0301 CAFÉ"` | `[("café", 3)]` |
| T07 | Accented and unaccented words remain distinct | `"cafe café"` | `[("cafe", 1), ("café", 1)]` |
| T08 | NFC also applies to Cyrillic letters | `"Ё е\u0308 ё"` | `[("ё", 3)]` |
| T09 | Do not merge Russian е and ё | `"елка ёлка"` | `[("елка", 1), ("ёлка", 1)]` |
| T10 | Do not merge Kazakh letters with similar Russian letters | `"а ә а"` | `[("а", 2), ("ә", 1)]` |
| T11 | Do not merge Latin a and Cyrillic а | `"a а a"` | `[("a", 2), ("а", 1)]` |
| T12 | Punctuation separates words without joining them | `"cat,dog;cat!"` | `[("cat", 2), ("dog", 1)]` |
| T13 | Apostrophes U+0027 and U+2019 split words | `"don't don’t"` | `[("don", 2), ("t", 2)]` |
| T14 | Hyphens and underscores split words | `"well-known well_known"` | `[("known", 2), ("well", 2)]` |
| T15 | Digits separate words and are not counted | `"abc123def 123 abc"` | `[("abc", 2), ("def", 1)]` |
| T16 | Spaces, tabs, line breaks, and non-breaking spaces separate words | `"cat\tdog\ncat\u00a0dog"` | `[("cat", 2), ("dog", 2)]` |
| T17 | U+0023, U+1F642, and U+002B are separators; the hashtag marker is not part of a word | `"#cat🙂dog+cat"` | `[("cat", 2), ("dog", 1)]` |
| T18 | A combining mark after a letter belongs to that word even without a precomposed form | `"q\u0301 q q\u0301"` | `[("q\u0301", 2), ("q", 1)]` |
| T19 | Ignore unattached combining marks | `"\u0301cat \u0301 cat"` | `[("cat", 2)]` |
| T20 | Keep stop words and one-letter words | `"the the и и және a"` | `[("the", 2), ("и", 2), ("a", 1), ("және", 1)]` |
| T21 | Do not stem or merge grammatical forms | `"cat cats cat"` | `[("cat", 2), ("cats", 1)]` |
| T22 | Input is plain text; no special HTML or URL extraction | `"<b>cat</b> https://x.kz"` | `[("b", 2), ("cat", 1), ("https", 1), ("kz", 1), ("x", 1)]` |
| T23 | Ties are alphabetical by code point, not first appearance | `"zebra apple zebra apple"` | `[("apple", 2), ("zebra", 2)]` |
| T24 | Mixed scripts use Unicode ordering, not separate language groups | `"қазақ кот apple"` | `[("apple", 1), ("кот", 1), ("қазақ", 1)]` |
| T25 | Keep the first ten after sorting, including a tie at the cutoff | `"k j i h g f e d c b a"` | `[("a", 1), ("b", 1), ("c", 1), ("d", 1), ("e", 1), ("f", 1), ("g", 1), ("h", 1), ("i", 1), ("j", 1)]` |
| T26 | Frequency takes priority over alphabetical order before the cutoff | `"a b c d e f g h i j k k"` | `[("k", 2), ("a", 1), ("b", 1), ("c", 1), ("d", 1), ("e", 1), ("f", 1), ("g", 1), ("h", 1), ("i", 1)]` |
| T27 | Empty input | `""` | `[]` |
| T28 | Input containing no words, including isolated marks | `"123 !!! 🙂 \u0301\n\t"` | `[]` |
| T29 | Non-string input is rejected, not converted to text | `None` | `TypeError` |
| T30 | Numbers are invalid as the entire input | `123` | `TypeError` |
| T31 | Bytes are not decoded automatically | `b"cat"` | `TypeError` |
| T32 | Unicode case folding, not just lowercasing | `"Straße STRASSE"` | `[("strasse", 2)]` |
| T33 | Control characters also separate words | `"cat\x00dog cat"` | `[("cat", 2), ("dog", 1)]` |
| T34 | Prefix words sort before longer words when counts tie | `"aa a"` | `[("a", 1), ("aa", 1)]` |
| T35 | Initial NFC must happen before case folding | `"\u03b1\u0345\u0301"` | `[("\u03ac\u03b9", 1)]` |
| T36 | Apply NFC again after case folding | `"\u01f0 J\u030c"` | `[("\u01f0", 2)]` |
| T37 | Use canonical NFC, not compatibility NFKC | `"\uff21 a"` | `[("a", 1), ("\uff41", 1)]` |
| T38 | Spacing combining marks (Mc) remain part of a word | `"\u0915\u093e \u0915"` | `[("\u0915", 1), ("\u0915\u093e", 1)]` |
| T39 | Enclosing combining marks (Me) remain part of a word | `"a\u20dd a a\u20dd"` | `[("a\u20dd", 2), ("a", 1)]` |
| T40 | An apostrophe-like character in category Lm is a letter | `"don\u02bct"` | `[("don\u02bct", 1)]` |
| T41 | Emoji presentation does not override letter and mark categories | `"\u2139\ufe0f"` | `[("\u2139\ufe0f", 1)]` |
| T42 | Letters can follow a combining mark within the same word | `"q\u0301b q\u0301b"` | `[("q\u0301b", 2)]` |
| T43 | A mark after a separator cannot attach backward or become a prefix | `"q,\u0301b"` | `[("b", 1), ("q", 1)]` |
| T44 | Case folding can turn an initially isolated mark into a letter before tokenization | `"\u0345"` | `[("\u03b9", 1)]` |
| T45 | A script change without a separator does not split a word | `"catкот catкот"` | `[("catкот", 2)]` |
| T46 | An invisible format character is a separator | `"a\u200db"` | `[("a", 1), ("b", 1)]` |
| T47 | A combining mark introduced by case folding remains attached | `"İ i\u0307 i"` | `[("i\u0307", 2), ("i", 1)]` |
| T48 | Letters outside the Basic Multilingual Plane follow the same rules | `"\U00010400 \U00010428"` | `[("\U00010428", 2)]` |

For interface behavior, the following checks supplement the table:

- Calling `top_words("cat")` returns `[("cat", 1)]` and writes nothing to standard output or standard error.
- Calling `top_words("cat")`, then `top_words("dog")`, then `top_words("cat")` returns `[("cat", 1)]`, `[("dog", 1)]`, and `[("cat", 1)]`, respectively. Counts do not accumulate across calls.
- Mutating a returned list does not change the result of a later call with the same input.
- Two calls to `top_words("")` return equal empty lists that are different objects: if their results are stored as `first` and `second`, then `first == second == []` is true and `first is second` is false.
- If `ArticleText` is a subclass of `str`, `top_words(ArticleText("Cat"))` returns `[("cat", 1)]`.

## Scope of the specification

The rules above define the observable result and input validation. They do not require a particular counting algorithm or data structure. Network retrieval, article extraction, a user interface, and storage are outside this function's responsibility.
