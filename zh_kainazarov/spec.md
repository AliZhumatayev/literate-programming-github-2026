# Top words: specification

Author: zh_kainazarov

> A news site shows a "Trending words" box beside each article. Write a Python
> function that finds the 10 most common words in an article's text, along with
> how many times each one appears. Articles may be in English, Russian, or
> Kazakh.

Sections 1 and 2 are normative; the tables are examples that follow from them.
Inputs and outputs in the tables are Python literals, with `\u` escapes such
as `\u0456` (Cyrillic `і`) for invisible or look-alike characters.

## 1. Interface

- `def top_words(text):` in `top_words.py`. Python 3.10+, standard library
  only (no NLTK, spaCy, or pymorphy). No side effects; deterministic.
- Returns exactly a `list` of at most 10 `(key, count)` tuples (`str`, `int`),
  never padded; `[]` if the text has no words.
- Raises `TypeError` for input that is not a `str` (section 2, step 1).
- Character properties come from the interpreter's Unicode database
  (`unicodedata.unidata_version`); every example holds on CPython 3.10–3.14.

## 2. Procedure

A **letter** has `c.isalpha()`. A **mark** has a `unicodedata.category(c)`
starting with `M`. A **format character** has category `Cf` (soft hyphen,
zero-width space, word joiner, BOM, ...). A **joiner** is `'` (U+0027), `’`
(U+2019), `-` (U+002D), U+2010, or U+2011. Everything else is a separator.

1. Raise `TypeError` unless `isinstance(text, str)`; then use
   `str.__str__(text)`.
2. Delete all format characters.
3. Normalize to NFC.
4. Scan left to right for words. Each word is the longest match of
   `letter (letter | mark)* (joiner letter (letter | mark)*)*`; all other
   characters are discarded.
5. Key: replace `’` with `'` and U+2010/U+2011 with `-`, apply
   `str.casefold()`, then normalize to NFC again.
6. Count keys. Sort by count, highest first, then by key in Python string
   order (code points).
7. Return the first 10 as `(key, count)` tuples.

## 3. Words

Words are letters from any script. Digits, `_`, and all punctuation except the
joiners split words, so numbers never count and `5G` gives `g`. A joiner joins
only when a letter follows it; look-alikes such as `‘` (U+2018) and `−`
(U+2212) are separators, and `ʼ` (U+02BC) is a letter. Format characters
vanish, including U+200B, which is treated as a line-break hint, not a word
break. Marks stay with the letter before them. Markup is not parsed.

| Input | Expected output | Decision |
| --- | --- | --- |
| `"cat dog cat bird dog cat"` | `[("cat", 3), ("dog", 2), ("bird", 1)]` | list of `(key, count)` |
| `""` | `[]` | no words |
| `"2024 — 15%!"` | `[]` | numbers and symbols are not words |
| `None` | `TypeError` | only `str` |
| `b"cat"` | `TypeError` | bytes are not decoded |
| `"Hello, world! Hello?"` | `[("hello", 2), ("world", 1)]` | punctuation splits |
| `"«Астана» — столица."` | `[("астана", 1), ("столица", 1)]` | any script; non-ASCII punctuation |
| `"5G COVID-19 2024 snake_case"` | `[("case", 1), ("covid", 1), ("g", 1), ("snake", 1)]` | digits and `_` split |
| `"x\u00b2 \u00bd \u216b XXI"` | `[("x", 1), ("xxi", 1)]` | number characters are not letters |
| `"U.S. г.\u00a0Алматы"` | `[("s", 1), ("u", 1), ("алматы", 1), ("г", 1)]` | periods split abbreviations |
| `"cat category cat's"` | `[("cat", 1), ("cat's", 1), ("category", 1)]` | whole words; `'s` kept |
| `"don't don\u2019t 'cat' dogs'"` | `[("don't", 2), ("cat", 1), ("dogs", 1)]` | `'` joins only inside; `’` is `'` |
| `"Ростов-на-Дону Кот-д\u2019Ивуар Нұр-Сұлтан Нұр\u2010Сұлтан Нұр\u2011Сұлтан"` | `[("нұр-сұлтан", 3), ("кот-д'ивуар", 1), ("ростов-на-дону", 1)]` | hyphens join; U+2010/U+2011 are `-` |
| `"a--b e\u2014mail x\u2212y well-\nknown"` | `[("a", 1), ("b", 1), ("e", 1), ("known", 1), ("mail", 1), ("well", 1), ("x", 1), ("y", 1)]` | dash runs, dashes, minus, line breaks split |
| `"don\u2018t \u02bccat"` | `[("don", 1), ("t", 1), ("\u02bccat", 1)]` | look-alikes are not joiners |
| `"news\u00adpaper news\u200bpaper news\u2060paper \ufeffnews\u200epaper cat\u200bdog"` | `[("newspaper", 4), ("catdog", 1)]` | format characters are deleted |
| `"за\u0301мок замок где\u0301-то a-\u0301b \u0301x"` | `[("a", 1), ("b", 1), ("x", 1), ("где\u0301-то", 1), ("за\u0301мок", 1), ("замок", 1)]` | marks kept after letters only |
| `"a\u20dd b\u0903 c\u034f"` | `[("a\u20dd", 1), ("b\u0903", 1), ("c\u034f", 1)]` | every mark category, even invisible |
| `"<p>cat</p> cat\ud800dog"` | `[("cat", 2), ("p", 2), ("dog", 1)]` | no HTML parsing; lone surrogates split |

## 4. Case and normalization

Case folding is `str.casefold()`, not `lower()` and not locale-aware. NFC
runs before folding and after it; NFKC is not used.

| Input | Expected output | Decision |
| --- | --- | --- |
| `"Cat cat CAT US us"` | `[("cat", 3), ("us", 2)]` | case ignored; folded key shown |
| `"Москва МОСКВА Қазақстан ҚАЗАҚСТАН"` | `[("москва", 2), ("қазақстан", 2)]` | Cyrillic, including Kazakh |
| `"Straße STRASSE \ufb01le file"` | `[("file", 2), ("strasse", 2)]` | full folding: `ß` to `ss`, `\ufb01` to `fi` |
| `"\u0130 i I \u0131"` | `[("i", 2), ("i\u0307", 1), ("\u0131", 1)]` | no Turkish or Kazakh Latin folding |
| `"\u0439 \u0438\u0306 \u01f0 J\u030c"` | `[("\u01f0", 2), ("\u0439", 2)]` | NFC, also after folding |
| `"\u03b1\u0345\u0301 \u1fb4 \u0345x"` | `[("\u03ac\u03b9", 2), ("x", 1)]` | NFC before folding; fold words, not text |
| `"\uff43\uff41\uff54 cat"` | `[("cat", 1), ("\uff43\uff41\uff54", 1)]` | no NFKC |

## 5. No other merging

`ё` is not `е`, Latin `i` is not Kazakh `і`, and there is no stemming,
lemmatization, stopword list, or minimum length. Stopwords such as `the` and
`и` will often lead the box; the problem asks for the most common words.

| Input | Expected output | Decision |
| --- | --- | --- |
| `"ещ\u0451 еще б\u0069р б\u0456р"` | `[("б\u0069р", 1), ("б\u0456р", 1), ("еще", 1), ("ещ\u0451", 1)]` | no letter merging |
| `"Алматы Алматыда run runs"` | `[("run", 1), ("runs", 1), ("алматы", 1), ("алматыда", 1)]` | no stemming |
| `"the cat and the dog и в и"` | `[("the", 2), ("и", 2), ("and", 1), ("cat", 1), ("dog", 1), ("в", 1)]` | no stopwords or minimum length |

## 6. Ranking

Ties use code point order, not first appearance or any alphabet: Latin comes
before Cyrillic, and `ё`, `і`, `ә` come after `я`. Exactly the first 10
entries are kept.

| Input | Expected output | Decision |
| --- | --- | --- |
| `"b a b a c"` | `[("a", 2), ("b", 2), ("c", 1)]` | ties by code point |
| `"\u0451ж яма я \u04d9 \u0430 apple"` | `[("apple", 1), ("\u0430", 1), ("я", 1), ("яма", 1), ("\u0451ж", 1), ("\u04d9", 1)]` | not Russian or Kazakh order |
| `"k j i h g f e d c b a"` | `[("a", 1), ("b", 1), ("c", 1), ("d", 1), ("e", 1), ("f", 1), ("g", 1), ("h", 1), ("i", 1), ("j", 1)]` | ties cut at 10 |
| `"a b c d e f g h i j k k"` | `[("k", 2), ("a", 1), ("b", 1), ("c", 1), ("d", 1), ("e", 1), ("f", 1), ("g", 1), ("h", 1), ("i", 1)]` | count before order |

Out of scope: language detection, transliteration, HTML entities (`&amp;`
gives `amp`), and scripts without spaces between words.
