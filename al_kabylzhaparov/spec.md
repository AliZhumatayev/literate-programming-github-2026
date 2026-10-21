# Top words

A news site shows a "Trending words" box beside an article. `top_words` finds the 10 most common words in that article's text, and how many times each word appears. Articles may be English, Russian, or Kazakh. The function does not detect the language and does not keep a separate rule set per language. One Unicode procedure handles every string.

The original one-line problem still allows many programs. This document picks one output for every input below. Each table is an input that would come out differently under another plausible reading of the original problem. Those other readings are named so the choice is visible. They are not the required behavior.

The tables were produced by running this procedure on CPython 3.13.15 (Unicode 15.1.0). Where a cell is a quoted Python literal, read it as Python: `\n` is one newline, and `\uXXXX` or `\UXXXXXXXX` is one character.

## Signature

- Python 3.11 or newer. Standard library only.
- The function is named `top_words` and is defined in `top_words.py`.
- Signature: `top_words(text: str) -> list[tuple[str, int]]`.
- The parameter name is `text`. `top_words(text="cat")` is a valid call.
- The module may define helpers. `from top_words import top_words` must work offline, with no third-party packages, and must not read files, the network, or standard input.
- The function returns a new list. It does not print. It does not read or write files.

A call with no argument, or with any argument besides `text`, raises `TypeError`.

| Call | Expected result |
| --- | --- |
| `top_words("cat")` | `[("cat", 1)]` |
| `top_words(text="Cat cat")` | `[("cat", 2)]` |
| `top_words()` | raises `TypeError` |
| `top_words("cat", 10)` | raises `TypeError` |
| `top_words(b"cat")` | raises `TypeError` |
| `top_words(None)` | raises `TypeError` |

`bytes` is not decoded. `None` is not treated as an empty article. A second parameter is not a way to ask for a different list length. The length is always 10.

A subclass of `str` is a `str`. `top_words(S("Cat cat"))` returns `[("cat", 2)]` when `S` inherits from `str`.

The message inside `TypeError` is not part of the contract.

## Procedure

Do these steps in this order. Later steps never see the original string.

1. If `text` is not an instance of `str`, raise `TypeError` immediately.
2. Let `s` be `unicodedata.normalize("NFC", text)`.
3. In `s`, replace each U+2019 RIGHT SINGLE QUOTATION MARK and each U+02BC MODIFIER LETTER APOSTROPHE with U+0027 APOSTROPHE. No other character is rewritten here. U+02BC has Unicode category `Lm` (it is a letter). This step runs before letters are detected, so U+02BC is a joiner, not a letter of its own.
4. Walk `s` from left to right and take tokens. A character is a letter when `unicodedata.category(ch)` starts with `L` (`Lu`, `Ll`, `Lt`, `Lm`, `Lo`). For one character this is the same test as `ch.isalpha()`.
   - Every character that is not a letter is a separator, with one exception: U+0027 is copied into the current token when it has a letter immediately after it and the token already contains a letter.
   - A token starts at a letter. It then consumes letters. When the current character is U+0027 and the next character is a letter, consume both and continue. Otherwise a non-letter ends the token and is not part of it. A following letter starts a new token.
   - Tokens do not overlap. The scan does not go back.
   - A run of apostrophes, or an apostrophe that is not between two letters, never becomes a word and is not glued onto the neighboring word.
5. For each token, let the word be `unicodedata.normalize("NFC", token.casefold())`. Count identical words. Casing, apostrophe lookalikes, and decomposed accents are already gone, so each distinct word has one spelling in the count.
6. Sort the pairs by count descending. Break remaining ties with Python 3's default `str` order (`word_a < word_b`), which walks Unicode code points, not a dictionary alphabet and not the order the words appeared. If one word is a proper prefix of another, the shorter word comes first.
7. Return the first 10 pairs as a list of `(word, count)` tuples. If there are fewer than 10 distinct words, return all of them. If there are none, return `[]`. Do not pad the list. Do not keep an 11th word because it ties the 10th.

The count is the number of tokens, not the number of overlapping substrings and not the number of times the word could be segmented out of a longer token.

## Decisions

### The book example

The box lists words and counts, most frequent first.

| Input | Expected output |
| --- | --- |
| `cat dog cat bird dog cat` | `[("cat", 3), ("dog", 2), ("bird", 1)]` |

Another reading would print a sentence, or return a dict, or order the pairs by the first time each word appeared.

### Case uses `str.casefold`, and the stored spelling is the casefolded one

`The`, `THE`, and `the` are one word. The pair stores `the`, not the first spelling that showed up in the article.

| Input | Expected output |
| --- | --- |
| `The THE the` | `[("the", 3)]` |

A case-sensitive reading would return three pairs with count 1. A reading that keeps the first spelling would return `[("The", 3)]`.

`casefold` is not `lower`. On these two inputs the results differ:

| Input | Expected output |
| --- | --- |
| `Straße strasse` | `[("strasse", 2)]` |
| `Σ σ ς` | `[("σ", 3)]` |

`str.lower` would leave `straße` and `strasse` as two words, and it would leave final sigma `ς` distinct from `σ`. Articles in this course are English, Russian, or Kazakh; the procedure is still one function for every string, so these inputs are defined too.

### Letters that must not be folded into a neighbor

Russian `ё` is not `е`. Kazakh `і` (U+0456) is not Russian `и` (U+0438) and not Latin `i` (U+0069). Latin `I` casefolds to Latin `i`. Kazakh `І` (U+0406) casefolds to Kazakh `і`. Latin `İ` (U+0130) casefolds to `i` plus U+0307 COMBINING DOT ABOVE, which does not equal `i`.

| Input | Expected output |
| --- | --- |
| `Ёж ёж еж` | `[("ёж", 2), ("еж", 1)]` |
| `іс ис` | `[("ис", 1), ("іс", 1)]` |
| `I І` | `[("i", 1), ("і", 1)]` |
| `ІС іс` | `[("іс", 2)]` |
| `"\u0130 i"` | `[("i", 1), ("i\u0307", 1)]` |

Folding `ё` into `е` would turn the first input into `[("еж", 3)]`. Folding Kazakh `і` into Latin `i` or Russian `и` would collapse the second and third inputs. Casefold runs on each token after the scan, not on the whole string before the scan. Doing it first would expand U+0130 to `i` plus U+0307, the combining mark would be a separator, and `"\u0130 i"` would collapse to `[("i", 2)]`. The second word of the required output is the two characters U+0069, U+0307.

Kazakh letters are letters. They are not stripped and not transliterated.

| Input | Expected output |
| --- | --- |
| `Қазақ қазақ` | `[("қазақ", 2)]` |
| `Ән Ғалым` | `[("ғалым", 1), ("ән", 1)]` |

`ғ` (U+0493) precedes `ә` (U+04D9) in code-point order, so `ғалым` comes first when the counts are equal. A Latin transliteration would return different strings.

### NFC happens before the scan

Accents that Unicode can precompose are precomposed first, then the scan runs. Casefold runs on the finished token, and that result is NFC-normalized again.

| Input | Expected output |
| --- | --- |
| `"cafe\u0301 café"` | `[("café", 2)]` |
| `"cafe\u0301s cafés"` | `[("cafés", 2)]` |
| `"E\u0301 e\u0301 É é"` | `[("é", 4)]` |

Skipping NFC before the scan would split `cafe` + U+0301 + `s` into `cafe` and `s`, and that token would not match `cafés`.

### Apostrophes inside a word

U+0027, U+2019, and U+02BC are the same joiner. The word stored in the result uses U+0027. An apostrophe is kept only when a letter stands on each side of it. A trailing possessive mark, a doubled apostrophe, or an apostrophe used as a quote is a separator.

| Input | Expected output |
| --- | --- |
| `"don't don\u2019t"` | `[("don't", 2)]` |
| `"o\u02bcclock o'clock"` | `[("o'clock", 2)]` |
| `don't dont` | `[("don't", 1), ("dont", 1)]` |
| `students' student` | `[("student", 1), ("students", 1)]` |
| `a''b` | `[("a", 1), ("b", 1)]` |
| `'cat' cat` | `[("cat", 2)]` |
| `rock 'n' roll` | `[("n", 1), ("rock", 1), ("roll", 1)]` |
| `rock'n'roll` | `[("rock'n'roll", 1)]` |
| `"'''"` | `[]` |

Keeping the curly apostrophe as a different character would split the first row into two words of count 1. Treating U+02BC as a letter, which its category `Lm` invites, would make `oʼclock` a different word from `o'clock`. Splitting every apostrophe would turn `don't` into `don` and `t`. Joining across a space would turn `rock 'n' roll` into one word. The two `rock` rows are different inputs and have different outputs.

### Everything else separates words

Digits, underscores, hyphens, dashes, emoji, tags, punctuation, the null character, a non-breaking space, a soft hyphen, and a leading BOM are not part of a word. The scan does not rejoin a word that was broken with a hyphen at the end of a line. It does not treat `_` as a word character the way `\w` does.

| Input | Expected output |
| --- | --- |
| `well-known well known` | `[("known", 2), ("well", 2)]` |
| `"well-\nknown"` | `[("known", 1), ("well", 1)]` |
| `state-of-the-art state` | `[("state", 2), ("art", 1), ("of", 1), ("the", 1)]` |
| `covid19 covid` | `[("covid", 2)]` |
| `foo_bar foo` | `[("foo", 2), ("bar", 1)]` |
| `cat, cat! cat.` | `[("cat", 3)]` |
| `see https://cats.com/cat` | `[("cat", 1), ("cats", 1), ("com", 1), ("https", 1), ("see", 1)]` |
| `cat 🎉 cat` | `[("cat", 2)]` |
| `123 456` | `[]` |
| `"cat\x00dog"` | `[("cat", 1), ("dog", 1)]` |
| `"cat\u00a0cat"` | `[("cat", 2)]` |
| `"cat\u00addog"` | `[("cat", 1), ("dog", 1)]` |
| `"\ufeffcat"` | `[("cat", 1)]` |

Keeping the hyphen inside `well-known` would return `[("well-known", 1), ("well", 1), ("known", 1)]`. Rejoining `well-` and `known` across the newline would return `[("wellknown", 1)]`. Treating digits as letters would make `covid19` different from `covid`. A C-style read that stops at the first null would return `[("cat", 1)]` and drop `dog`. Deleting a soft hyphen and then joining would return `[("catdog", 1)]`.

### The text is not HTML, and common words stay

Tags and entities are ordinary characters. Stopwords stay. One letter is enough to be a word. There is no stemmer and no dictionary split.

| Input | Expected output |
| --- | --- |
| `<b>cat</b> cat` | `[("b", 2), ("cat", 2)]` |
| `"\u0026amp; cat"` | `[("amp", 1), ("cat", 1)]` |
| `the cat the` | `[("the", 2), ("cat", 1)]` |
| `a cat a` | `[("a", 2), ("cat", 1)]` |
| `cats cat` | `[("cat", 1), ("cats", 1)]` |
| `catcat cat` | `[("cat", 1), ("catcat", 1)]` |
| `Cat кот cat` | `[("cat", 2), ("кот", 1)]` |

`<b>` and `</b>` each contain the letter `b`, so `b` has count 2, the same count as `cat`. Equal counts then sort `b` before `cat`. A tag stripper would return `[("cat", 2)]`. `"\u0026amp; cat"` is the characters `&amp; cat`. An entity decoder would not produce `amp`. A stopword list would return `[("cat", 1)]` for `the cat the`. A minimum length of 2 would return `[("cat", 1)]` for `a cat a`. A stemmer would merge `cats` into `cat`. A dictionary segmenter would split `catcat` into two `cat` tokens and return `[("cat", 3)]`.

English and Russian words in one article share one list.

### Order, ties, and the cutoff of 10

Higher count comes first. Equal counts use code-point order, not first-seen order and not dictionary order. `ё` (U+0451) sorts after `я` (U+044F), even though a Russian dictionary places `ё` beside `е`.

| Input | Expected output |
| --- | --- |
| `dog cat` | `[("cat", 1), ("dog", 1)]` |
| `я я а` | `[("я", 2), ("а", 1)]` |
| `яблоко арбуз` | `[("арбуз", 1), ("яблоко", 1)]` |
| `ё я` | `[("я", 1), ("ё", 1)]` |
| `aa a` | `[("a", 1), ("aa", 1)]` |

First-seen order would return `[("dog", 1), ("cat", 1)]`. Sorting the whole list alphabetically, and ignoring counts, would return `[("а", 1), ("я", 2)]`. Dictionary order would put `ё` before `я`.

The list never has more than 10 pairs. A tie at the boundary does not pull in the rest of the tied words. Fewer than 10 distinct words means a shorter list. No words means an empty list.

| Input | Expected output |
| --- | --- |
| `k j i h g f e d c b a` | `[("a", 1), ("b", 1), ("c", 1), ("d", 1), ("e", 1), ("f", 1), ("g", 1), ("h", 1), ("i", 1), ("j", 1)]` |
| `aa aa bb bb cc cc dd dd ee ee ff ff gg gg hh hh ii ii zz yy xx ww vv` | `[("aa", 2), ("bb", 2), ("cc", 2), ("dd", 2), ("ee", 2), ("ff", 2), ("gg", 2), ("hh", 2), ("ii", 2), ("vv", 1)]` |
| `zz zz zz yy yy a b c d e f g h i j k` | `[("zz", 3), ("yy", 2), ("a", 1), ("b", 1), ("c", 1), ("d", 1), ("e", 1), ("f", 1), ("g", 1), ("h", 1)]` |
| `""` | `[]` |
| `"   "`, `"\n"`, and `"\t"` | `[]` for each |

The first of those inputs contains 11 words of count 1, and `k` was seen first. The result is the 10 code-point-earliest words, so `k` is absent. Keeping every tie would return 11 pairs. The second input has nine words of count 2 and five words of count 1. The 10th place goes to `vv`, because `v` precedes `w`, `x`, `y`, and `z`. `ww`, `xx`, `yy`, and `zz` are left out even though each also has count 1. The third input shows that a later word with a higher count (`zz`) outranks earlier rare words, and that `i`, `j`, and `k` fall off the end.

## Out of scope

Stemming, lemmatization, stopword lists, language detection, HTML parsing, transliteration, and a caller-chosen list length are out of scope. The procedure above is the whole function.
