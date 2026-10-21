# Top Words — Specification

## Goal

Write a Python 3 function that finds the ten most frequent words in an article. The text may contain English, Russian, and Kazakh. The rules below define one expected result for each input.

## Function contract

```python
def top_words(text: str) -> list[tuple[str, int]]:
    ...
```

- **Input:** exactly one Python string containing the article. An empty string is valid. For any non-string input, raise `TypeError`.
- **Output:** a Python list of `(word, count)` tuples. Each `word` is a string in normalized form, and each `count` is a positive integer. The function does not print or change its input.
- Return at most **10 distinct words**; return fewer when fewer exist. When no words exist, return `[]`.

## Exact word and counting rules

1. Normalize the entire input with Unicode **NFC**, then apply Unicode `casefold()` and normalize with NFC again. Return words in this normalized form. For example, `CAT` and `cat` count together; Unicode case folding also makes `Straße` and `STRASSE` count together as `strasse`.
2. A **letter** is a Unicode character whose general category starts with `L`. A combining mark is a character whose Unicode category starts with `M`. A word is a nonempty sequence beginning with a letter, followed by letters or combining marks, optionally containing internal apostrophes as described below. A combining mark is included only when it follows a letter or another included combining mark in the same word; otherwise it is a separator.
3. The straight apostrophe (`'`) and curly right apostrophe (`’`) may occur **inside** a word only when immediately preceded and followed by letters (not just combining marks). Normalize an accepted curly apostrophe to `'`. Any other apostrophe position is a separator. Thus `don't` is one word, but `'cat'` is `cat`.
4. All other characters are **separators**, including whitespace, digits, underscores, hyphens, emoji, and all other punctuation. Consecutive separators are treated like one separator. `word2word` becomes two occurrences of `word`; `state-of-the-art` becomes four words.
5. Count **every occurrence** of each normalized word, across the whole input. Do not remove stop words, translate, stem, or lemmatize. Thus `run`, `runs`, and `running` are separate words.
6. Sort the distinct words by **descending count**. Break ties by ascending Python lexicographic string order (`<`) of their normalized word strings; do not use locale-aware or natural-language alphabetic ordering. Take the first 10 entries after sorting.

## Examples (exact outputs)

| Case | Input | Expected result |
|---|---|---|
| Frequency | `cat dog cat bird dog cat` | `[('cat', 3), ('dog', 2), ('bird', 1)]` |
| Case-insensitive | `Cat CAT cat Dog DOG` | `[('cat', 3), ('dog', 2)]` |
| Three languages | `Cat кот КОТ қазақ ҚАЗАҚ cat` | `[('cat', 2), ('кот', 2), ('қазақ', 2)]` |
| Punctuation | `cat,dog.cat!cat?dog` | `[('cat', 3), ('dog', 2)]` |
| Apostrophes | `don't don’t 'cat' cat` | `[('cat', 2), ("don't", 2)]` |
| Hyphenation | `state-of-the-art` | `[('art', 1), ('of', 1), ('state', 1), ('the', 1)]` |
| Numbers and underscore | `word2word word_test 123` | `[('word', 3), ('test', 1)]` |
| Separate word forms | `run runs running` | `[('run', 1), ('running', 1), ('runs', 1)]` |
| Keep stop words | `the the cat` | `[('the', 2), ('cat', 1)]` |
| Alphabetical tie | `dog cat bird` | `[('bird', 1), ('cat', 1), ('dog', 1)]` |
| Unicode casefold | `Straße STRASSE` | `[('strasse', 2)]` |
| Exactly top 10 | `a b c d e f g h i j k` | `[('a', 1), ('b', 1), ('c', 1), ('d', 1), ('e', 1), ('f', 1), ('g', 1), ('h', 1), ('i', 1), ('j', 1)]` |
| Empty input | `` (empty string) | `[]` |
| No words | `123 !!! ---` | `[]` |
| Wrong input type | `None` | raises `TypeError` |

**Note:** The specification defines behavior rather than requiring a particular algorithm or third-party package.
