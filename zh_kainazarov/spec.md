# Top words: specification

Author: zh_kainazarov

## Problem

> A news site shows a "Trending words" box beside each article. Write a Python
> function that finds the 10 most common words in an article's text, along with
> how many times each one appears. Articles may be in English, Russian, or
> Kazakh.

This spec settles every choice the problem leaves open, so that for a given
Unicode version (`unicodedata.unidata_version`) only one return value is
correct for each input. The prose of [section 1](#1-interface) and
[section 2](#2-procedure), together with the escape convention in "How to read
the tables", is normative. Everything else, including every example table (the
one in section 1 too), is informative: it explains a decision and pins it down
with examples, and it follows from the normative text. Anything informative
that disagrees with the normative text is a bug in this spec, and the
normative text wins.

## How to read the tables

- **Input** is the argument passed to `top_words`, written as a Python literal.
  Where the exact code point matters and the character is invisible, combining,
  or easy to confuse with another, the literal uses a Python `\u` escape: a
  backslash, `u`, and four hex digits that give the code point. For example,
  `"\u0456"` is Kazakh Cyrillic `і`, never Latin `i`.
- **Expected output** is the exact return value as a Python literal, or the
  name of the exception that must be raised.
- **Rules out** names a plausible reading of the original problem that gives a
  different output.

An implementation that follows sections 1 and 2 returns, for every row, a
value that is `==` to the expected output and has exactly the types given in
section 1, item 3, or it raises the named exception.

## 1. Interface

1. The function is defined as `def top_words(text):` in the file
   `top_words.py`, so both `top_words(s)` and `top_words(text=s)` work. It
   takes exactly one argument.
2. If `isinstance(text, str)` is false (for example `bytes` or `None`), the
   function raises `TypeError` itself, not a subclass of it. The message is not
   part of the contract. An instance of a `str` subclass is accepted and is
   treated as the plain string with the same characters, `str.__str__(text)`:
   methods that the subclass overrides are never called, and the keys returned
   are plain `str`. For example, if `class S(str)` overrides `casefold` to
   return `"x"`, `top_words(S("Cat"))` still returns `[("cat", 1)]`.
3. The return value has type exactly `list`. Each element has type exactly
   `tuple` (not a named tuple) and is `(key, count)`, where the key is the
   normalized word defined in [section 2](#2-procedure), step 5, `type(key)` is
   `str`, and `type(count)` is `int`, with `count` at least 1. The return value
   is not a `dict`, a `Counter`, or a formatted string.
4. The list holds the 10 highest-ranked keys, or every key if the text has
   fewer than 10 distinct keys. It is never padded. Text with no words gives
   `[]`.
5. The function is deterministic and has no side effects: it does not print,
   read or write files, or use the network, and no call can observe anything
   left behind by an earlier call.
6. Python 3.10 or newer, standard library only (for example `re`,
   `unicodedata`, `collections`). No third-party packages such as NLTK, spaCy,
   or pymorphy.
7. Character properties (letter, mark, format character, case folding, NFC)
   come from the running interpreter's Unicode database. `str.isalpha`,
   `str.casefold`, and the `unicodedata` module all use the same Unicode
   version, `unicodedata.unidata_version`. A character whose properties differ
   between Unicode versions may therefore be treated differently by different
   interpreters; that is accepted. Every example row in this spec gives the
   same result on CPython 3.10 through 3.14 (Unicode 13.0 through 16.0).

| Input                        | Expected output                         | Rules out                                                                                 |
| ---------------------------- | --------------------------------------- | ----------------------------------------------------------------------------------------- |
| `"cat dog cat bird dog cat"` | `[("cat", 3), ("dog", 2), ("bird", 1)]` | returning `{"cat": 3, ...}`, a `Counter`, or the string `"cat: 3, dog: 2, bird: 1"`       |
| `""`                         | `[]`                                    | returning `None`                                                                          |
| `"123 ... !!!"`              | `[]`                                    | counting numbers or punctuation as words                                                  |
| `"cat\ud800dog"`             | `[("cat", 1), ("dog", 1)]`              | encoding the text to UTF-8 first, which raises `UnicodeEncodeError` on the lone surrogate |
| `None`                       | `TypeError`                             | treating `None` as empty text and returning `[]`                                          |
| `b"cat"`                     | `TypeError`                             | decoding bytes as UTF-8 and returning `[("cat", 1)]`                                      |

## 2. Procedure

Definitions, for a single character `c`:

- A **letter** is a character for which `c.isalpha()` is `True` (Unicode
  general categories Lu, Ll, Lt, Lm, Lo).
- A **mark** is a character whose `unicodedata.category(c)` starts with `"M"`
  (Mn, Mc, Me), such as U+0301 COMBINING ACUTE ACCENT.
- A **format character** is a character whose `unicodedata.category(c)` is
  `"Cf"`, such as U+00AD SOFT HYPHEN, U+200B ZERO WIDTH SPACE, U+2060 WORD
  JOINER, or U+FEFF ZERO WIDTH NO-BREAK SPACE.
- A **joiner** is one of U+0027 APOSTROPHE `'`, U+2019 RIGHT SINGLE QUOTATION
  MARK `’`, U+002D HYPHEN-MINUS `-`, U+2010 HYPHEN, or U+2011 NON-BREAKING
  HYPHEN.
- Every other character (digits and other number characters, whitespace,
  punctuation, symbols, en and em dashes, the minus sign, `_`, control
  characters) is a **separator**.

`top_words(text)` must return the same value as this procedure:

1. If `isinstance(text, str)` is false, raise `TypeError`. Otherwise, continue
   with `str.__str__(text)`, the plain string with the same characters
   (section 1, item 2).
2. Delete every format character from `text`.
3. Normalize `text` to NFC: `unicodedata.normalize("NFC", text)`.
4. Scan `text` from left to right for **words**. A word starts at a letter and
   is the longest run of the form

   ```text
   letter (letter | mark)* ( joiner letter (letter | mark)* )*
   ```

   That is, a word is letters and marks, where a single joiner may sit between
   a letter or mark on its left and a letter on its right. Scanning resumes
   after the end of each word. Every character that is not part of a word,
   including a mark or joiner that cannot extend a word, is discarded.
5. Turn each word into its **key**: replace every U+2019 with U+0027 and every
   U+2010 or U+2011 with U+002D, apply `str.casefold()`, then normalize to NFC
   again.
6. Count how many times each key occurs.
7. Sort the distinct keys by count, highest first. Break ties by comparing the
   keys as Python strings (`<`, which compares code points), smallest first.
8. Return the first 10 entries of that order as `(key, count)` tuples.

## 3. What counts as a word

- A word is made of letters from any script, so Russian and Kazakh Cyrillic
  (including `ә ғ қ ң ө ұ ү һ і`) works the same way as Latin.
- Digits are separators, so a bare number such as `2024` is never a word. The
  same rule splits tokens that mix digits and letters: `5G` gives `g`, `MP3`
  gives `mp`, and Russian `2024-го` gives `го`. That cost is accepted, because
  "letters only" is one short rule that behaves the same in all three
  languages.
- Characters that Unicode classes as numbers, such as superscript digits,
  fractions, and the Roman numeral character `Ⅻ` (U+216B), are not letters
  either. Superscript letters such as `ⁿ` and `ª` are letters. Roman numerals
  typed with ordinary Latin letters, as in Russian `XXI век`, are letters, so
  `xxi` is a word.
- A period is a separator, even inside an abbreviation: `U.S.` gives `u` and
  `s`, and `т.е.` gives `т` and `е`. In Russian news, `г.` (short for "year",
  or "city" as in `г. Алматы`) will often put `г` in the box.
- A joiner joins only when a letter (or a mark on a letter) is on its left and
  a letter is on its right. This keeps `don't`, `кое-что`, `well-known`, and
  `Нұр-Сұлтан` whole. Quotes at the edges of a word are dropped, and dashes
  and line-break hyphens split words.
- Different typographic forms of the same punctuation count as one: `’` as
  `'`, and U+2010 and U+2011 as `-`. News sites and publishing tools often
  insert them automatically. Look-alikes that are not one of these forms are
  never merged: `‘` (U+2018), `´` (U+00B4), and the minus sign `−` (U+2212)
  are separators, and `ʼ` (U+02BC) is a letter (see section 5).
- Most format characters are invisible. Scraped web articles often contain
  soft hyphens, zero-width spaces, word joiners, and byte-order marks inside
  words. All format characters are deleted, so they never split a word. This
  also joins words that a format character separates: `cat` followed by
  U+200B and `dog` gives `catdog`. Unicode's word-boundary rules (UAX #29)
  break at U+200B, but this spec assumes that in English, Russian, and Kazakh
  text, where words are already separated by spaces, a zero-width space is a
  line-break hint like the soft hyphen, not a word separator.
- Invisible characters that are marks rather than format characters, such as
  U+034F COMBINING GRAPHEME JOINER and the variation selectors, follow the rule
  for marks and stay in the key. They carry meaning: the grapheme joiner
  controls how NFC orders marks, and a variation selector picks a glyph. As a
  result, the box can show two entries that look identical, such as `cat` and
  `cat` followed by U+034F.
- A mark (Mn, Mc, or Me) is kept only directly after a letter or after
  another kept mark, so a Russian stress mark does not split a word. Any other
  mark, such as one after a space, a digit, or a joiner, is discarded. A mark
  right after a joiner also stops the joiner from joining.
- The text is plain text. Markup such as HTML is not parsed: tag names are
  scanned like any other letters.

| Input                                               | Expected output                                                                       | Rules out                                                                                                                                  |
| --------------------------------------------------- | ------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| `"Hello, world! Hello?"`                            | `[("hello", 2), ("world", 1)]`                                                        | splitting on whitespace only, which keeps `hello,` and `hello?`                                                                            |
| `"«Астана» — столица."`                             | `[("астана", 1), ("столица", 1)]`                                                     | stripping only ASCII punctuation, which keeps `«астана»` and `—`                                                                           |
| `"5G G7 COVID-19 2024"`                             | `[("g", 2), ("covid", 1)]`                                                            | `re.findall(r"\w+", ...)`, which keeps digits: `5g`, `g7`, `covid`, `19`, `2024`                                                           |
| `"MP3 2024-го"`                                     | `[("mp", 1), ("го", 1)]`                                                              | words of letters and digits that contain a letter, which gives `mp3` and `2024-го`                                                         |
| `"snake_case"`                                      | `[("case", 1), ("snake", 1)]`                                                         | `\w+`, which keeps `_` and gives `[("snake_case", 1)]`                                                                                     |
| `"x\u00b2 \u00bd \u216b"`                           | `[("x", 1)]`                                                                          | `re.findall(r"[^\W\d_]+", ...)`, which gives `[("x\u00b2", 1), ("\u00bd", 1), ("\u217b", 1)]` (`Ⅻ` folds to `ⅻ`)                           |
| `"x\u207f \u00aa"`                                  | `[("x\u207f", 1), ("\u00aa", 1)]`                                                     | treating every superscript as a number, which gives `[("x", 1)]`                                                                           |
| `"XXI век"`                                         | `[("xxi", 1), ("век", 1)]`                                                            | treating Roman numerals as numbers, which gives `[("век", 1)]`                                                                             |
| `"U.S. т.е."`                                       | `[("s", 1), ("u", 1), ("е", 1), ("т", 1)]`                                            | keeping abbreviations with periods whole, which gives `u.s.` and `т.е.`                                                                    |
| `"г.\u00a0Алматы"`                                  | `[("алматы", 1), ("г", 1)]`                                                           | dropping single letters left by abbreviations, which gives `[("алматы", 1)]`                                                               |
| `"cat category cat's"`                              | `[("cat", 1), ("cat's", 1), ("category", 1)]`                                         | counting substrings (`cat`: 3), or stripping the possessive `'s` (`cat`: 2)                                                                |
| `"don't don\u2019t"`                                | `[("don't", 2)]`                                                                      | splitting at the apostrophe, or counting the `’` and `'` spellings as different words                                                      |
| `"don\u2018t don\u00b4t x\u2212y"`                  | `[("don", 2), ("t", 2), ("x", 1), ("y", 1)]`                                          | treating every apostrophe-like or minus-like character as a joiner, which keeps `don‘t`, `don´t`, and `x−y` whole                          |
| `"'cat' dogs'"`                                     | `[("cat", 1), ("dogs", 1)]`                                                           | keeping a leading or trailing apostrophe                                                                                                   |
| `"кое-что Нұр-Сұлтан well-known"`                   | `[("well-known", 1), ("кое-что", 1), ("нұр-сұлтан", 1)]`                              | splitting at every hyphen                                                                                                                  |
| `"Ростов-на-Дону Кот-д\u2019Ивуар"`                 | `[("кот-д'ивуар", 1), ("ростов-на-дону", 1)]`                                         | allowing at most one joiner per word, or only one kind of joiner per word                                                                  |
| `"Нұр\u2011Сұлтан Нұр\u2010Сұлтан"`                 | `[("нұр-сұлтан", 2)]`                                                                 | splitting at U+2010 or U+2011, or keeping them as different spellings from `-`                                                             |
| `"где\u0301-то"`                                    | `[("где\u0301-то", 1)]`                                                               | joining only when a letter is directly on the left, which gives `где\u0301` and `то`                                                       |
| `"a-\u0301b"`                                       | `[("a", 1), ("b", 1)]`                                                                | joining when a mark is on the right of the joiner, which gives `[("a-\u0301b", 1)]`                                                        |
| `"a--b e\u2014mail x\u2013y z - w"`                 | `[("a", 1), ("b", 1), ("e", 1), ("mail", 1), ("w", 1), ("x", 1), ("y", 1), ("z", 1)]` | joining across a run of hyphens, an em dash, an en dash, or a spaced hyphen                                                                |
| `"well-\nknown"`                                    | `[("known", 1), ("well", 1)]`                                                         | rejoining a word hyphenated at a line break                                                                                                |
| `"news\u00adpaper newspaper"`                       | `[("newspaper", 2)]`                                                                  | treating the soft hyphen as a separator, which gives `news` and `paper`                                                                    |
| `"news\u200bpaper news\u2060paper \ufeffnewspaper"` | `[("newspaper", 3)]`                                                                  | treating a zero-width space, word joiner, or byte-order mark as a separator                                                                |
| `"news\u200epaper cat\u200bdog"`                    | `[("catdog", 1), ("newspaper", 1)]`                                                   | deleting only the four format characters named in section 2 (`news`, `paper`), or keeping words apart at a zero-width space (`cat`, `dog`) |
| `"cat\u034f cat"`                                   | `[("cat", 1), ("cat\u034f", 1)]`                                                      | deleting every invisible character, which gives `[("cat", 2)]`                                                                             |
| `"a\u20dd b\u0903"`                                 | `[("a\u20dd", 1), ("b\u0903", 1)]`                                                    | treating only category Mn as marks, which gives `[("a", 1), ("b", 1)]`                                                                     |
| `"за\u0301мок замок"`                               | `[("за\u0301мок", 1), ("замок", 1)]`                                                  | splitting at the mark (`за`, `мок`), or stripping it (`замок`: 2)                                                                          |
| `" \u0301cat"`                                      | `[("cat", 1)]`                                                                        | letting a word start with a mark, which gives `[("\u0301cat", 1)]`                                                                         |
| `"<p>cat</p>"`                                      | `[("p", 2), ("cat", 1)]`                                                              | parsing HTML, which gives `[("cat", 1)]`                                                                                                   |

## 4. Case and Unicode normalization

- Counting ignores case as Unicode's language-independent full case folding
  defines it: a key is `str.casefold()` output, so `ß` folds to `ss`. The box
  shows that folded key, not the spelling first seen in the article.
- Folding is not locale-aware, and in Kazakh Latin script this means case is
  not always ignored. The 2021 Kazakh Latin alphabet pairs `I` with `ı` and `İ`
  with `i`, the Turkish way. This spec folds `I` to `i`, so `I` and `ı` give
  different keys. It folds `İ` to `i` followed by U+0307 COMBINING DOT ABOVE,
  so `İ` and `i` give different keys too, and many fonts draw that key with two
  dots. So a word that starts with `I` or `İ` does not merge with its
  lower-case form wherever it is capitalized: at the start of a sentence, in a
  name, or in an all-caps headline. There is no language-neutral fix, and this
  spec does not detect the language.
- Canonically equivalent spellings of a word give the same key. This is why the
  text is normalized both before case folding (step 3) and after it (step 5),
  the order Unicode itself uses for caseless matching. Unicode's definition
  uses NFD; this spec uses NFC so that keys are shown in composed form. Folding
  can produce a decomposed sequence, which the second NFC recomposes. Folding a
  string that is not yet normalized can give a different result, which the
  first NFC prevents. That happens only when a word contains U+0345 COMBINING
  GREEK YPOGEGRAMMENI, which folds to the Greek letter `ι`, as the row with
  U+0345 shows. Elsewhere the first NFC can change the text but not the key: in
  the `и` + U+0306 row it composes `й`, but the second NFC would compose it
  too.
- Case folding is applied to each word after scanning (step 5), not to the
  whole text before it. The two differ when folding turns a non-letter into a
  letter: U+0345 is a mark that folds to the Greek letter `ι`.
- NFKC is not used, so compatibility characters such as full-width Latin
  letters stay distinct from their plain forms, unless `str.casefold()` itself
  maps them. It does map a few: the ligature `ﬁ` folds to `fi`, the long s `ſ`
  to `s`, and the micro sign `µ` to Greek `μ`. Text copied from PDFs often
  contains such ligatures.

| Input                         | Expected output                             | Rules out                                                                                    |
| ----------------------------- | ------------------------------------------- | -------------------------------------------------------------------------------------------- |
| `"Cat cat CAT"`               | `[("cat", 3)]`                              | counting case-sensitively, or showing the first-seen spelling `Cat`                          |
| `"Москва МОСКВА москва"`      | `[("москва", 3)]`                           | lower-casing only ASCII letters                                                              |
| `"Қазақстан ҚАЗАҚСТАН"`       | `[("қазақстан", 2)]`                        | lower-casing only the letters of the Russian alphabet                                        |
| `"US us"`                     | `[("us", 2)]`                               | keeping all-capital acronyms apart from ordinary words                                       |
| `"Straße STRASSE"`            | `[("strasse", 2)]`                          | `str.lower()`, which gives `[("strasse", 1), ("straße", 1)]`                                 |
| `"\u0130 i I \u0131"`         | `[("i", 2), ("i\u0307", 1), ("\u0131", 1)]` | Turkish or Kazakh Latin folding, which gives `[("i", 2), ("\u0131", 2)]`                     |
| `"\u0439 \u0438\u0306"`       | `[("\u0439", 2)]`                           | skipping both NFC steps, which counts `"\u0438\u0306"` as a different word                   |
| `"\u01f0 J\u030c"`            | `[("\u01f0", 2)]`                           | skipping the second NFC, which gives `[("j\u030c", 2)]`                                      |
| `"\u03b1\u0345\u0301 \u1fb4"` | `[("\u03ac\u03b9", 2)]`                     | skipping the first NFC, which gives `[("\u03ac\u03b9", 1), ("\u03b1\u03af", 1)]`             |
| `" \u0345cat"`                | `[("cat", 1)]`                              | case-folding the whole text before scanning, which gives `[("\u03b9cat", 1)]`                |
| `"\uff43\uff41\uff54 cat"`    | `[("cat", 1), ("\uff43\uff41\uff54", 1)]`   | NFKC, which gives `[("cat", 2)]`                                                             |
| `"\ufb01le file"`             | `[("file", 2)]`                             | keeping every compatibility character distinct, which gives `[("file", 1), ("\ufb01le", 1)]` |

## 5. Spellings that are not merged

After step 5, keys are compared code point by code point. Nothing else is
merged:

- `ё` and `е` are different letters, even though Russian text often writes
  `е` for `ё`.
- Stress marks are kept, so `за́мок` and `замок` are different keys. Stress
  can be the only thing that tells two words apart: `за́мок` is "castle" and
  `замо́к` is "lock". Stripping marks would also need a rule for which marks
  count as stress and which are part of a letter.
- U+02BC MODIFIER LETTER APOSTROPHE `ʼ` is not converted to `'`, although it
  looks the same. Unicode classes it as a letter, not punctuation, and some
  orthographies write it as a letter. Merging it with `'` would be a
  look-alike merge, and this spec makes none (see the next item). Because it
  is a letter, it also stays at the start or end of a word.
- Look-alike letters from different scripts stay different: Latin `i`
  (U+0069) is not Kazakh Cyrillic `і` (U+0456). Mixed-script typing does occur
  in Kazakh text, but merging it needs a confusables table, which this spec
  does not define.
- There is no stemming or lemmatization: inflected forms such as `Алматы` and
  `Алматыда` are different words.
- There is no stopword filtering and no minimum word length: `the`, `и`, `в`,
  and `және` count like any other word. The problem asks for the most common
  words, and a stopword list for three languages would be a separate product
  decision. Because the function also stops at 10 entries, a caller cannot
  filter stopwords out afterwards and still show 10 words. This is intended:
  on real articles the box will often be led by `the`, `и`, `в`, or `және`.

| Input                        | Expected output                                     | Rules out                                                                      |
| ---------------------------- | --------------------------------------------------- | ------------------------------------------------------------------------------ |
| `"ещ\u0451 еще"`             | `[("еще", 1), ("ещ\u0451", 1)]`                     | merging `ё` into `е`, which gives `[("еще", 2)]`                               |
| `"\u02bccat don\u02bct"`     | `[("don\u02bct", 1), ("\u02bccat", 1)]`             | treating `ʼ` as an apostrophe joiner, which gives `[("cat", 1), ("don't", 1)]` |
| `"б\u0069р б\u0456р"`        | `[("б\u0069р", 1), ("б\u0456р", 1)]`                | merging look-alike Latin and Cyrillic letters                                  |
| `"run runs running"`         | `[("run", 1), ("running", 1), ("runs", 1)]`         | stemming, which gives `[("run", 3)]`                                           |
| `"Алматы Алматыда Алматыға"` | `[("алматы", 1), ("алматыда", 1), ("алматыға", 1)]` | lemmatization, which gives `[("алматы", 3)]`                                   |
| `"the cat and the dog"`      | `[("the", 2), ("and", 1), ("cat", 1), ("dog", 1)]`  | removing stopwords such as `the` and `and`                                     |
| `"и в и в и"`                | `[("и", 3), ("в", 2)]`                              | dropping one-letter words, which gives `[]`                                    |

## 6. Ranking and the cutoff

- A higher count ranks first.
- Ties are broken by the code point order of the key (Python's default string
  order), not by first appearance and not by any language's alphabetical
  order. This needs no locale and gives the same result on every machine with
  the same Unicode version. As consequences:
  - basic Latin words (`a` to `z`) come before Cyrillic words, but other Latin
    letters may not: full-width `ｃ` (U+FF43) sorts after every Russian and
    Kazakh letter;
  - Russian `ё` (U+0451) comes after `я` (U+044F), not between `е` and `ж`;
  - Kazakh letters such as `і` (U+0456) and `ә` (U+04D9) also come after `я`,
    unlike in the Kazakh alphabet.
- The list has at most 10 entries. When there are more than 10 distinct keys,
  words tied with the 10th entry but sorted after it are left out.

| Input                       | Expected output                                                                                        | Rules out                                                                          |
| --------------------------- | ------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------- |
| `"b a b a c"`               | `[("a", 2), ("b", 2), ("c", 1)]`                                                                       | breaking ties by first appearance, which puts `b` first                            |
| `"яблоко apple"`            | `[("apple", 1), ("яблоко", 1)]`                                                                        | breaking ties by first appearance, which puts `яблоко` first                       |
| `"\u0451ж яма"`             | `[("яма", 1), ("\u0451ж", 1)]`                                                                         | Russian alphabetical order, or first appearance, which both put `ёж` first         |
| `"я \u04d9 а"`              | `[("а", 1), ("я", 1), ("\u04d9", 1)]`                                                                  | Kazakh alphabetical order `а`, `ә`, `я`, or first appearance `я`, `ә`, `а`         |
| `"k j i h g f e d c b a"`   | `[("a", 1), ("b", 1), ("c", 1), ("d", 1), ("e", 1), ("f", 1), ("g", 1), ("h", 1), ("i", 1), ("j", 1)]` | returning all 11 tied words, or the first 10 seen (`k` to `b`)                     |
| `"a b c d e f g h i j k k"` | `[("k", 2), ("a", 1), ("b", 1), ("c", 1), ("d", 1), ("e", 1), ("f", 1), ("g", 1), ("h", 1), ("i", 1)]` | taking the first 10 distinct words before ranking, which drops `k`, or keeping `j` |

## 7. Out of scope

- Detecting the article's language, transliteration, and any locale-specific
  rule.
- Parsing HTML or decoding entities: `&amp;` gives the word `amp`.
- The 2017 Kazakh Latin alphabet, which wrote some letters with an apostrophe,
  such as `on'` for Cyrillic `оң` ("right"). A trailing apostrophe is
  discarded, so `on'` gives the Latin key `on`, which is also the key for `on`,
  the 2017 spelling of Cyrillic `он` ("ten"). Keys are never transliterated to
  Cyrillic.
- Scripts written without spaces between words, such as Chinese or Thai. The
  procedure still gives a defined result there (a whole run of letters is one
  word), but it is not a useful one.
