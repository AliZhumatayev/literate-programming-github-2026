# Top words

`top_words` returns the 10 most common words in an article and the count of each. Articles may be English, Russian, or Kazakh. One procedure applies to every string. The function does not detect a language.

Python 3.11 or newer, standard library only. Define `top_words(text: str) -> list[tuple[str, int]]` in `top_words.py`. The only parameter is `text`. `top_words(text=...)` is valid, and a `str` subclass is a `str`. Any other call raises `TypeError`. `bytes` is not decoded. The exception message is not part of the contract. The function returns a new list, does not print, and does no I/O. Importing the module needs no network and no third-party package.

A quoted cell is a Python literal: `\n` is one newline and `\uXXXX` is one character. The tables were checked on CPython 3.13.15 (Unicode 15.1.0).

## Procedure

1. If `text` is not an instance of `str`, raise `TypeError`.
2. NFC-normalize the whole string.
3. Replace U+2019 and U+02BC with U+0027. U+02BC has category `Lm`, so this replacement happens before letters are detected. No other character changes.
4. Scan left to right, without overlap. A letter is a character whose general category starts with `L` (for one character, `ch.isalpha()`). A token starts at a letter and then takes more letters. U+0027 is included only when the token already contains a letter and the next character is a letter. Every other character is a separator and is not part of a word.
5. Store `NFC(token.casefold())` and count those strings. Casefold each token after the scan. Do not casefold the whole string first.
6. Sort by count descending, then by Python's `<` on `str` (code-point order; a proper prefix comes first). Not appearance order and not dictionary order.
7. Return the first 10 `(word, count)` pairs. Fewer than 10 distinct words produces a shorter list. No words produces `[]`. Do not pad. Do not add an 11th word that ties the 10th.

The count is the number of tokens. It is not an overlapping substring count and not a dictionary segmentation.

## Examples

The third column is a plausible other reading of the original problem. It is not the required result.

| Input | Expected output | Other reading |
| --- | --- | --- |
| `cat dog cat bird dog cat` | `[("cat", 3), ("dog", 2), ("bird", 1)]` | a dict, prose, or first-seen order |
| `The THE the` | `[("the", 3)]` | three pairs, or `[("The", 3)]` |
| `Straße strasse` | `[("strasse", 2)]` | `str.lower` leaves `straße` and `strasse` apart |
| `Σ σ ς` | `[("σ", 3)]` | `str.lower` leaves final sigma `ς` apart |
| `Ёж ёж еж` | `[("ёж", 2), ("еж", 1)]` | folding `ё` into `е` yields `[("еж", 3)]` |
| `іс ис` | `[("ис", 1), ("іс", 1)]` | folding Kazakh `і` (U+0456) into Russian `и` |
| `I І` | `[("i", 1), ("і", 1)]` | folding Kazakh `І` (U+0406) into Latin `i` |
| `ІС іс` | `[("іс", 2)]` | case-sensitive pairs |
| `"\u0130 i"` | `[("i", 1), ("i\u0307", 1)]` | casefold-before-scan yields `[("i", 2)]`; the second word is U+0069 then U+0307 |
| `Қазақ қазақ` | `[("қазақ", 2)]` | a Latin transliteration |
| `Ән Ғалым` | `[("ғалым", 1), ("ән", 1)]` | `ғ` (U+0493) precedes `ә` (U+04D9); dictionary order can disagree |
| `"cafe\u0301 café"` | `[("café", 2)]` | the decomposed spelling stays a second word |
| `"cafe\u0301s cafés"` | `[("cafés", 2)]` | skipping NFC before the scan yields `cafe`, `s`, and `cafés` |
| `"E\u0301 e\u0301 É é"` | `[("é", 4)]` | skipping NFC before the scan yields `e` twice and `é` twice |
| `"don't don\u2019t"` | `[("don't", 2)]` | U+2019 stays a different character; the result uses U+0027 |
| `"o\u02bcclock o'clock"` | `[("o'clock", 2)]` | U+02BC stays a letter, so the two spellings differ |
| `don't dont` | `[("don't", 1), ("dont", 1)]` | deleting every apostrophe merges them |
| `students' student` | `[("student", 1), ("students", 1)]` | the trailing apostrophe stays on `students` |
| `a''b` | `[("a", 1), ("b", 1)]` | one token `a''b` |
| `'cat' cat` | `[("cat", 2)]` | the quotes stay on the first word |
| `rock 'n' roll` | `[("n", 1), ("rock", 1), ("roll", 1)]` | one joined word |
| `rock'n'roll` | `[("rock'n'roll", 1)]` | a split on every apostrophe |
| `"'''"` | `[]` | a word made only of apostrophes |
| `well-known well known` | `[("known", 2), ("well", 2)]` | `well-known` is one token |
| `"well-\nknown"` | `[("known", 1), ("well", 1)]` | rejoining the line break yields `wellknown` |
| `covid19 covid` | `[("covid", 2)]` | digits are letters, so `covid19` is its own word |
| `foo_bar foo` | `[("foo", 2), ("bar", 1)]` | `\w` keeps `foo_bar` |
| `cat, cat! cat.` | `[("cat", 3)]` | punctuation stays attached |
| `"cat\x00dog"` | `[("cat", 1), ("dog", 1)]` | a C string stops at the null and drops `dog` |
| `"cat\u00addog"` | `[("cat", 1), ("dog", 1)]` | deleting the soft hyphen joins the halves |
| `"\ufeffcat"` | `[("cat", 1)]` | the BOM stays glued to `cat` |
| `<b>cat</b> cat` | `[("b", 2), ("cat", 2)]` | stripping tags yields `[("cat", 2)]`; `<b>` and `</b>` each contain `b` |
| `"\u0026amp; cat"` | `[("amp", 1), ("cat", 1)]` | decoding `&amp;` does not produce `amp`; the input is those characters |
| `the cat the` | `[("the", 2), ("cat", 1)]` | a stopword list leaves only `cat` |
| `a cat a` | `[("a", 2), ("cat", 1)]` | a minimum length of 2 leaves only `cat` |
| `cats cat` | `[("cat", 1), ("cats", 1)]` | stemming merges them |
| `catcat cat` | `[("cat", 1), ("catcat", 1)]` | a dictionary split yields `[("cat", 3)]` |
| `Cat кот cat` | `[("cat", 2), ("кот", 1)]` | a separate list per language |
| `dog cat` | `[("cat", 1), ("dog", 1)]` | first-seen order puts `dog` first |
| `я я а` | `[("я", 2), ("а", 1)]` | sorting only alphabetically puts `а` first |
| `ё я` | `[("я", 1), ("ё", 1)]` | dictionary order puts `ё` (U+0451) before `я` (U+044F) |
| `aa a` | `[("a", 1), ("aa", 1)]` | the longer word comes first |
| `k j i h g f e d c b a` | `[("a", 1), ("b", 1), ("c", 1), ("d", 1), ("e", 1), ("f", 1), ("g", 1), ("h", 1), ("i", 1), ("j", 1)]` | keep first-seen `k`, or return all 11 ties |
| `aa aa bb bb cc cc dd dd ee ee ff ff gg gg hh hh ii ii zz yy xx ww vv` | `[("aa", 2), ("bb", 2), ("cc", 2), ("dd", 2), ("ee", 2), ("ff", 2), ("gg", 2), ("hh", 2), ("ii", 2), ("vv", 1)]` | also keep `ww`, `xx`, `yy`, and `zz`, which tie `vv` |
| `zz zz zz yy yy a b c d e f g h i j k` | `[("zz", 3), ("yy", 2), ("a", 1), ("b", 1), ("c", 1), ("d", 1), ("e", 1), ("f", 1), ("g", 1), ("h", 1)]` | keep `i`, `j`, and `k` |
| `""` | `[]` | `None` |
| `"   "`, `"\n"`, `"\t"` | `[]` for each | a placeholder word |

Digits, `_`, emoji, `<`, `>`, NBSP (U+00A0), and any other non-letter are separators under step 4, the same way punctuation is. Stemming, stopword lists, HTML parsing, transliteration, and a caller-chosen length are not part of the function.
