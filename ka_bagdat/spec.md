# Top Words — Specification

## Goal

Write a Python function to find the ten most frequent words in a text that may contain English, Russian, and Kazakh. For a given Python interpreter, the rules below determine one exact result.

## Function contract

```python
def top_words(text: str) -> list[tuple[str, int]]:
    ...
```

- Accept one argument. Use `isinstance(text, str)` to validate it; accept `str` subclasses and raise `TypeError` for other types (including `bytes`).
- Return a list of `(word, count)` tuples, where `word` is a normalized string and `count` is a positive integer. Never print a result. The function has no externally observable side effects.
- Return at most ten distinct words, or fewer if the text has fewer. Return `[]` if there are no words.
- Unicode categories and case folding follow the **running Python interpreter** (`unicodedata` and `str.casefold()`); the result is defined relative to that interpreter's Unicode database. There is no separate pinned Unicode version.

## Exact rules

1. **Normalize first.** Apply NFC to the full input, then `str.casefold()`, then NFC again, in that order. Apply the character and word rules below **to this resulting text**, not to the original spelling. Returned words use this normalized form. Case folding can introduce combining marks: `"İ"` (U+0130) becomes `"i\u0307"` and the dot remains present.
2. **Letters and marks.** Unless explicitly excluded in rule 4, a letter is a character whose Unicode general category begins with `L` (`unicodedata.category(char).startswith("L")`); a combining mark has a category beginning with `M`. A word starts with a letter and can continue with letters or combining marks. A mark belongs to a word only if the immediately preceding character in that same word is a letter or an included mark. A mark without such a preceding character is a separator. Do not discard marks.
3. **Two allowed apostrophes.** ASCII `'` (U+0027) and right curly `’` (U+2019) can occur within a word **only if the characters immediately before and after are both letters**, as defined in rule 2. When included, convert `’` to `'`. Otherwise these apostrophes are separators. An intervening combining mark does not satisfy the immediate-letter condition. This deliberate choice keeps tokenization simple, even in languages that use combining marks.
4. **Everything else separates.** All other characters are word separators, including digits, whitespace, hyphens, underscores, other punctuation, emoji, leading marks, and format characters (`Cf`), including soft hyphen U+00AD, zero-width joiner U+200D and zero-width non-joiner U+200C. They split words; they are **not** silently removed. Also treat modifier letter apostrophe U+02BC (`ʼ`) as a separator even though its Unicode category is `Lm`. Left curly quote U+2018 (`‘`) and prime U+2032 (`′`) are separators, not accepted apostrophes. This explicitly limits special apostrophe handling to U+0027 and U+2019.
5. **Count exactly.** Count every occurrence of every normalized word. Do not remove stop words; do not translate, stem, lemmatize, or merge look-alike letters across scripts. The Russian letters `е` and `ё` remain different.
6. **Rank and limit.** Sort by decreasing count; for equal counts use increasing Python lexicographic order of the normalized strings (Unicode code-point order, not locale collation). **Then** take the first ten tuples.

## Examples (exact outputs)

Each *Input* cell is a Python expression representing the function argument. Each result is the exact Python value returned, except where an exception is specified. Escape sequences such as `\u0307` are used to make invisible characters visible in the table.

| Case | Input | Expected output |
|---|---|---|
| Frequency | `"cat dog cat bird dog cat"` | `[('cat', 3), ('dog', 2), ('bird', 1)]` |
| Case folding | `"Cat CAT cat Dog DOG"` | `[('cat', 3), ('dog', 2)]` |
| Mixed languages | `"Cat кот КОТ қазақ ҚАЗАҚ cat"` | `[('cat', 2), ('кот', 2), ('қазақ', 2)]` |
| Kazakh case pair | `"Ғ ғ Ү ү"` | `[('ғ', 2), ('ү', 2)]` |
| Russian е / ё | `"ел ёл ел"` | `[('ел', 2), ('ёл', 1)]` |
| Punctuation | `"cat,dog.cat!cat?dog"` | `[('cat', 3), ('dog', 2)]` |
| Internal apostrophes | `"don't don’t 'cat' cat"` | `[('cat', 2), ("don't", 2)]` |
| Curly quotes at edges | `"’cat’"` | `[('cat', 1)]` |
| Apostrophe variants | `"aʼb a‘b a′b a’b"` | `[('a', 3), ('b', 3), ("a'b", 1)]` |
| Hyphens | `"state-of-the-art"` | `[('art', 1), ('of', 1), ('state', 1), ('the', 1)]` |
| Digits / underscores | `"word2word word_test 123"` | `[('word', 3), ('test', 1)]` |
| Isolated marks | `"\u0301cat 2\u0301dog"` | `[('cat', 1), ('dog', 1)]` |
| Turkish dotted I | `"İstanbul İstanbul"` | `[("i\u0307stanbul", 2)]` |
| Combining mark before apostrophe | `"q\u0307'b"` | `[('b', 1), ('q\u0307', 1)]` |
| Format characters split | `"пере\u00adход a\u200db a\u200cb"` | `[('a', 2), ('b', 2), ('пере', 1), ('ход', 1)]` |
| Mixed-script lookalikes | `"cat cаt cat"` (`а` in the middle word is Cyrillic) | `[('cat', 2), ('cаt', 1)]` |
| Different forms | `"run runs running"` | `[('run', 1), ('running', 1), ('runs', 1)]` |
| Stop words kept | `"the the cat"` | `[('the', 2), ('cat', 1)]` |
| Alphabetical tie | `"dog cat bird"` | `[('bird', 1), ('cat', 1), ('dog', 1)]` |
| Unicode folding | `"Straße STRASSE"` | `[('strasse', 2)]` |
| Exactly ten | `"a b c d e f g h i j k"` | `[('a', 1), ('b', 1), ('c', 1), ('d', 1), ('e', 1), ('f', 1), ('g', 1), ('h', 1), ('i', 1), ('j', 1)]` |
| Frequency wins at cutoff | `"a b c d e f g h i j k z z"` | `[('z', 2), ('a', 1), ('b', 1), ('c', 1), ('d', 1), ('e', 1), ('f', 1), ('g', 1), ('h', 1), ('i', 1)]` |
| Empty | `""` | `[]` |
| No words | `"123 !!! ---"` | `[]` |
| Invalid None | `None` | raises `TypeError` |
| Invalid bytes | `b"cat"` | raises `TypeError` |

There is no artificial input length limit; processing time and memory may grow with the length of the input and the number of distinct words. The specification does not require a particular algorithm or external package.