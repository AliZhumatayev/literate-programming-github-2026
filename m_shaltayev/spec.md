# Top words specification

## Purpose

A news site shows a "Trending words" box beside each article. The function finds the 10 most common words in the text of one article. Articles can be in English, Russian, or Kazakh.

## Interface

- Language: Python 3.9 or later. Use only the standard library.
- The Unicode data is the data of the Python interpreter that runs the function. The tests use only characters that exist in Unicode 13.
- File: `top_words.py`.
- Signature: `def top_words(text: str) -> list[tuple[str, int]]`.
- The function takes exactly one parameter. It does not print anything.
- The function does not change global state and does not read files or the network.
- If `text` is not an instance of `str` (for example `None` or `bytes`), the function raises `TypeError`. The message is not specified. A `str` subclass is accepted. The function converts it to an exact `str` first, and every `word` it returns is an exact `str`.

## Output

- The function returns a list of `(word, count)` tuples.
- `word` is the lowercase form of the word (see "Normalization").
- `count` is the number of words in `text` whose normalized form equals `word`.
- The list holds at most 10 `(word, count)` pairs: the pairs with the smallest `(-count, word)`, in that order. Word order is Python `str` comparison with `<`.
- If the text has no words (empty, only spaces, only digits, only punctuation), the list is empty.

## What is a word

1. Normalize the text to Unicode form NFC (`unicodedata.normalize("NFC", text)`). Do this first, before any other step.
2. A letter is any character that Python `str.isalpha()` accepts.
3. A mark is any character in Unicode category `Mn` or `Mc`, except the variation selectors. The variation selectors are U+180B to U+180D, U+180F, U+FE00 to U+FE0F, and U+E0100 to U+E01EF.
4. A joiner is an apostrophe (`'` U+0027 or `’` U+2019) or a hyphen-minus (`-` U+002D).
5. A word starts at a letter. Read from left to right. The word continues with each letter and each mark. It also continues over a joiner if a letter comes right after the joiner, even when the joiner comes after a mark. The word keeps the joiner. The word ends at the first character that does not continue it.
6. A mark that does not come right after a letter or a mark is not part of a word. It separates words.
7. Every other character separates words. This includes digits, variation selectors, and invisible format characters (Unicode category `Cf`).
8. The rules use `str.isalpha()` for all scripts. Latin, Cyrillic, and Kazakh text is the target. Other scripts follow the same rules, but the tests do not check them.
9. Characters that look like joiners but are not in rule 4 are not joiners. This is intended. `ʼ` (U+02BC) is a letter, so `donʼt` is one word that differs from `don't`. `‘` (U+2018), `´` (U+00B4), `‐` (U+2010), `‑` (U+2011), and `–` (U+2013) separate words.

## Normalization

Apply these steps to each word after the word rules above. Use this order.

1. Replace `’` (U+2019) with `'` (U+0027), so `don't` and `don’t` count as the same word.
2. Call `str.lower()` on the whole word string. Do not lowercase the whole text, and do not lowercase each character alone.
- Do not normalize again after `str.lower()`. Two words that are canonically equal but not identical stay different words. This is intended.
- Do not remove accents or stress marks. Do not map `ё` to `е` or `й` to `и`.
- Do not stem or lemmatize. `cat` and `cats` are different words.
- Do not remove stop words. `the`, `и`, and `және` count like any other word.
- Do not detect the language. The same rules apply to all text.
- Latin `i` and Cyrillic `і` (Kazakh) are different characters. They are different words.

## Performance

The expected running time is O(n + d log d), where n is the text length and d is the number of different words. The bound ignores the cost of comparing two words. Any method with this bound is allowed.

## Examples

Each row is a call `top_words(input)` and the expected result. The input is the text between the backticks. In the input and the output, `\n`, `\t`, `\\`, `\xXX`, `\uXXXX`, and `\UXXXXXXXX` stand for one character each, as in a Python string literal. The output is a Python list.

### Counting and order

| Input                                | Expected output                  | Decision                           |
| ------------------------------------ | -------------------------------- | ---------------------------------- |
| `cat dog cat bird dog cat`           | `[("cat", 3), ("dog", 2), ("bird", 1)]` | Sort by count, highest first.     |
| `b a c a b`                          | `[("a", 2), ("b", 2), ("c", 1)]` | Equal counts: code point order.    |
| `а б в а б`                          | `[("а", 2), ("б", 2), ("в", 1)]` | Tie-break also works for Cyrillic. |
| `a b c d e f g h i j k l` | `[("a", 1), ("b", 1), ("c", 1), ("d", 1), ("e", 1), ("f", 1), ("g", 1), ("h", 1), ("i", 1), ("j", 1)]` | Limit is 10. The tie-break picks which 10. |
| `a b c`                              | `[("a", 1), ("b", 1), ("c", 1)]` | Fewer than 10 words: return all.   |
| empty string                         | `[]`                             | No words: empty list.              |
| `\x20\n\t`                           | `[]`                             | Only whitespace: empty list.       |

### Case

| Input                      | Expected output       | Decision                      |
| -------------------------- | --------------------- | ----------------------------- |
| `Cat cat CAT`              | `[("cat", 3)]`        | Words are case-insensitive.   |
| `Алматы алматы АЛМАТЫ`     | `[("алматы", 3)]`     | Lowercase works for Cyrillic. |
| `Ұлы ұлы`                  | `[("ұлы", 2)]`        | Kazakh `Ұ` lowercases to `ұ`. |

### Punctuation and separators

| Input                          | Expected output                  | Decision                                   |
| ------------------------------ | -------------------------------- | ------------------------------------------ |
| `cat, cat. cat! "cat" (cat)`   | `[("cat", 5)]`                   | Punctuation separates words.               |
| `cat;dog/cat\x7cdog`           | `[("cat", 2), ("dog", 2)]`       | Any non-letter separates words.            |
| `кот—пёс`                      | `[("кот", 1), ("пёс", 1)]`       | Em dash separates words.                   |
| `кот - пёс`                    | `[("кот", 1), ("пёс", 1)]`       | A hyphen with spaces does not join.        |
| `cat_dog`                      | `[("cat", 1), ("dog", 1)]`       | Underscore separates words.                |
| `cat…dog`                      | `[("cat", 1), ("dog", 1)]`       | Ellipsis separates words.                  |
| `😀 cat 😀`                    | `[("cat", 1)]`                   | Emoji are not letters.                     |

### Apostrophes and hyphens

| Input                       | Expected output                       | Decision                                   |
| --------------------------- | ------------------------------------- | ------------------------------------------ |
| `don't Don't`               | `[("don't", 2)]`                      | Inner apostrophe joins.                    |
| `don't don’t`               | `[("don't", 2)]`                      | Curly apostrophe equals straight one.      |
| `'cat' dogs'`               | `[("cat", 1), ("dogs", 1)]`           | Edge apostrophes are not part of a word.   |
| `well-known well-known`     | `[("well-known", 2)]`                 | Inner hyphen joins.                        |
| `-cat- --dog--`             | `[("cat", 1), ("dog", 1)]`            | Edge hyphens are not part of a word.       |
| `a--b`                      | `[("a", 1), ("b", 1)]`                | Two joiners in a row do not join.          |
| `a-'b`                      | `[("a", 1), ("b", 1)]`                | Two different joiners in a row do not join. |
| `кто-то`                    | `[("кто-то", 1)]`                     | Hyphen rule also works for Cyrillic.       |
| `кто\u0301-то`              | `[("кто\u0301-то", 1)]`               | A mark before a joiner does not stop the join. |

### Digits

| Input               | Expected output                    | Decision                                 |
| ------------------- | ---------------------------------- | ---------------------------------------- |
| `2024 100 3.14`     | `[]`                               | Numbers are not words.                   |
| `abc123def`         | `[("abc", 1), ("def", 1)]`         | Digits separate words.                   |
| `covid-19`          | `[("covid", 1)]`                   | A hyphen with no letter after it does not join. |
| `3d 3D`             | `[("d", 2)]`                       | The digit is dropped. The letter stays.  |

### Unicode

| Input                                   | Expected output                           | Decision                                          |
| --------------------------------------- | ----------------------------------------- | ------------------------------------------------- |
| `Ёж ёж`                                 | `[("ёж", 2)]`                             | `ё` is a letter and lowercases to `ё`.            |
| `еж ёж`                                 | `[("еж", 1), ("ёж", 1)]`                  | `е` and `ё` are different. `е` (U+0435) comes before `ё` (U+0451). |
| `и\u0306 \u0439` (`и` + U+0306, a space, then `й` U+0439) | `[("й", 2)]`           | NFC makes both forms the same word.               |
| `Москва\u0301 Москва` (U+0301 is a stress mark) | `[("москва", 1), ("москва\u0301", 1)]`          | The stress mark stays. These are different words. |
| `\u0306 cat` (lone combining mark)    | `[("cat", 1)]`                            | A word starts with a letter. A lone mark is dropped. |
| `3\u0301d a-\u0301`                     | `[("a", 1), ("d", 1)]`                    | A mark after a digit or after a joiner is not part of a word. |
| `cat\ufe0f \u2764\ufe0fdog`             | `[("cat", 1), ("dog", 1)]`                 | Variation selectors (U+FE00 to U+FE0F) separate words. |
| `soft\u00adhyphen`                    | `[("hyphen", 1), ("soft", 1)]`            | Format characters (`Cf`) separate words.          |
| `don\u02bct don't`                    | `[("don't", 1), ("don\u02bct", 1)]`       | `ʼ` (U+02BC) is a letter, not a joiner.           |
| `a\u2010b a\u2018b`                   | `[("a", 2), ("b", 2)]`                    | U+2010 and U+2018 separate words.                 |
| `i і`                                   | `[("i", 1), ("і", 1)]`                    | Latin `i` and Cyrillic `і` are different.         |
| `\u0130` (`İ`)                       | `[("i\u0307", 1)]`                              | `str.lower()` gives `i` + U+0307. No special case. |
| `Straße strasse`                        | `[("strasse", 1), ("straße", 1)]`         | No case folding beyond `str.lower()`.             |
| `Cat Кот Қой`                           | `[("cat", 1), ("кот", 1), ("қой", 1)]`    | Mixed scripts in one text work. Code point order: Latin, then Cyrillic. |
| `саt` (Cyrillic `с`, Cyrillic `а`, Latin `t`) | `[("саt", 1)]`                     | A run of mixed Latin and Cyrillic letters is one word. No script check. |

### Realistic text

Input:

```
Москва — столица России. Москва большая, и в Москве много людей. Астана — столица Казахстана.
```

Expected output:

```
[("москва", 2), ("столица", 2), ("астана", 1), ("большая", 1), ("в", 1), ("и", 1), ("казахстана", 1), ("людей", 1), ("много", 1), ("москве", 1)]
```

The text has 11 different words. The limit of 10 cuts `россии`, the last word in code point order. The words `москва` and `москве` are different. The words `столица` and `москва` both have count 2. Code point order puts `москва` before `столица`, and `много` before `москве`.
