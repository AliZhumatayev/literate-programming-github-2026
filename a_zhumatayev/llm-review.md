# LLM Review

Prompting Claude Opus 5.5 with the prompt below produced the following output; strikethrough indicates LLM output that I deemed incorrect or irrelevant.

**Prompt:**

> Review spec.md as a specification for the Top words exercise. Check whether two independent Python 3 implementations could reasonably disagree on any valid input. Look for ambiguous interface, Unicode normalization and tokenization problems, case handling, ordering and ties, the 10-word cutoff, edge cases, and contradictions between the rules and the examples. Verify every example row by running a fresh implementation written from the rules alone. Separate real defects from intentional design decisions. Report remaining blockers and suggested improvements; do not rewrite the specification.

After this review I applied suggestions 1–4 to `spec.md` (three example rows added, the rule 11 wording corrected, and the Unicode version pinned). Suggestions 5 and 6 are left as they are.

---

## Review output

### Overall assessment

The specification defines one result for every `str` input and a `TypeError` for every other type. I found no blocking ambiguity and no contradiction between the normative rules and the examples. One explanatory sentence in rule 11 is inaccurate, and three behaviors that independent implementations commonly get wrong have no example. These are listed below.

### Method

1. **Fresh implementation.** I wrote a new `top_words` from the rules alone. It classifies every character as letter, joiner or separator and finds words with the regex `L+(?:JL+)*` over that class string.
2. **Example rows.** I parsed all 36 rows directly from `spec.md` (inputs and expected outputs read with `ast.literal_eval`). All 36 match, including both `TypeError` rows.
3. **Differential test.** I compared it with a second, differently structured implementation (a character-by-character scanner) on 200,000 random strings built from the characters the rules treat specially: Latin, Cyrillic and Kazakh letters, `İ`, Greek sigma, `ß`, every apostrophe and dash variant, combining marks, soft hyphen, zero-width space, digits, superscripts, Roman numerals, underscore and emoji. There were 0 disagreements.
4. **Exhaustive check of rule 6.** For every Unicode code point that is a letter, I checked that `str.lower()` followed by deleting `Mn` characters gives only letters. No exception was found, so lowercasing never changes where words begin or end.
5. **Performance.** 7 million characters of mixed text took 3.4 s, and a single 5-million-letter word took 1.2 s. Both are linear and consistent with the O(n log n) requirement.

### Interface

Unambiguous. The input is checked with `isinstance(text, str)`, so subclasses of `str` are accepted, and every other type raises `TypeError`. The output is a `list` of `(str, int)` tuples with at most 10 entries. The function is pure and uses only the standard library. Lone surrogates such as `"a\ud800b"` return `[("a", 1), ("b", 1)]` without an exception, so "no other exception for any `str` input" holds.

### Unicode normalization

The order of steps (NFC, delete `Mn` and `Cf`, map apostrophes, lowercase, delete `Mn` again) is explicit. Two implementations that follow it agree.

**Defect 1 (wording, non-blocking).** Rule 11 says it "removes Russian stress marks". That is true only for the acute accent. A grave accent on `и` or `е` has a precomposed form (`ѝ` U+045D, `ѐ` U+0450), so NFC merges it into the letter and the deletion step never sees it:

| Input       | Result under the rules     | Result if "remove stress marks" is taken literally |
| ----------- | -------------------------- | -------------------------------------------------- |
| `"мѝр мир"` | `[("мир", 1), ("мѝр", 1)]` | `[("мир", 2)]`                                     |

An implementer who trusts the sentence instead of the steps would disagree. Either correct the sentence or add this row, or both.

**Suggestion 2.** No row shows decomposed `й` and `ё`. The breve U+0306 and diaeresis U+0308 are `Mn`, so an implementation that deletes marks _before_ NFC turns `й` into `и`. The `café` row already catches that wrong order for Latin, but a Russian row is more convincing for this site:

| Input             | Result under the rules    | Marks deleted before NFC                         |
| ----------------- | ------------------------- | ------------------------------------------------ |
| `"йод йод ёж ёж"` | `[("йод", 2), ("ёж", 2)]` | `[("еж", 1), ("иод", 1), ("йод", 1), ("ёж", 1)]` |

### Tokenization

Rules 1, 2, 4 and 5 are unambiguous. A letter is exactly `isalpha()`, a word is the leftmost maximal match of `L+ (J L+)*`, and everything else separates words.

**Suggestion 3.** The most common way to write rule 1 in Python is the regex `[^\W\d_]+`. It is **not** equivalent to `isalpha()`: I found 1,151 code points that it matches but `isalpha()` rejects, for example superscript digits (`²`), vulgar fractions (`½`) and Roman numerals (`Ⅻ`). No row catches this.

| Input      | Result under the rules | Regex `[^\W\d_]+`                 |
| ---------- | ---------------------- | --------------------------------- |
| `"x² ½ Ⅻ"` | `[("x", 1)]`           | `[("x²", 1), ("½", 1), ("ⅻ", 1)]` |

### Case handling

Unambiguous and covered. `str.lower()` (not `casefold()`) is applied to the whole text before words are found, and the `ß`, `İ` and Greek-sigma rows fix all three choices. That `Σ` before `''` becomes `σ`, not `ς`, follows from the order of steps. It is an intentional design decision, not a defect.

### Ordering, ties and the cutoff

Unambiguous and covered. Equal counts are ordered by code point, not by first occurrence and not by locale. The cutoff takes exactly 10 entries after sorting, and the "11 words tied at count 2" row fixes which word is dropped. Code point order puts every Latin word before every Cyrillic word with the same count, and `қ` after all Russian letters. This is intentional and documented in rule 8.

### Edge cases

Covered: empty string, punctuation and digits only, fewer than 10 words, non-`str` input, `bytes` input, line-break hyphenation, double joiners, leading and trailing joiners, invisible characters.

**Suggestion 4 (the only source of disagreement found).** The spec says only "Python 3". The letter, case and category properties come from the Unicode database bundled with the running Python: 15.1.0 in Python 3.13 and 16.0.0 in Python 3.14. Two implementations that are correct line by line can still disagree on letters added in a later Unicode version. Pin the reference version, for example "CPython 3.13, `unicodedata.unidata_version == "15.1.0"`".

### Intentional design decisions (not defects)

- Apostrophes `’` and `ʼ` are mapped to `'` and join words; `‘` and the backtick do not.
- Digits are separators (`a1b` gives `a` and `b`).
- `ё` and `е` are different letters; there is no transliteration, stemming or stop-word list.
- Scripts that use combining vowel signs or no spaces are declared unsupported (rule 14). Their output is still fully determined.

### Remaining blockers

None.

### Minor suggestions

5. The Rule column uses the labels "basic" and "Function" instead of rule numbers.
6. The rules are numbered in an order different from the processing order (rule 11 runs first). The paragraph before the rules gives the real order, so this affects only readability.

### Conclusion

The spec is deterministic and internally consistent. Two independent implementations written from it agreed on all 36 rows and on 200,000 random strings. Apply suggestions 1–4 to remove the remaining wording risk and the Python-version dependence.
