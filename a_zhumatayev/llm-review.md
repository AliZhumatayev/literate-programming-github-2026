# LLM Review

Prompting Claude Opus 5.5 (model ID `claude-opus-5-5`, in the Claude app with a Python code-execution sandbox) with the prompt below produced the following output; strikethrough indicates LLM output that I deemed incorrect or irrelevant.

**Prompt:**

> Review spec.md as a specification for the Top words exercise. Check whether two independent Python 3 implementations could reasonably disagree on any valid input. Look for ambiguous interface, Unicode normalization and tokenization problems, case handling, ordering and ties, the 10-word cutoff, edge cases, and contradictions between the rules and the examples. Verify every example row by running a fresh implementation written from the rules alone. Separate real defects from intentional design decisions. Report remaining blockers and suggested improvements; do not rewrite the specification.

**History.** This is the last of several review rounds. The previous round (in my first commit) suggested fixes 1–4, which are now applied: rule 11's wording about stress marks was corrected, the Unicode version is pinned, and rows 37–39 were added. A later round with the same prompt found that lowercasing character by character instead of the whole text (Greek final sigma) was not caught by any row, that my first wording of the final-sigma rule was wrong (case-ignorable characters include `.`, `:`, `^` and the backtick, not only `'`), and that U+02BC from rule 3 had no example. I fixed rule 6 and added rows 40–43. The output below is the review of the resulting spec.

---

## Review output

### Overall assessment

The specification defines exactly one result for every `str` input under the pinned reference environment, and a `TypeError` for every other type. I found no blocking ambiguity, no contradiction between the rules and the examples, and no untested decision. Only cosmetic suggestions remain.

### Method

1. **Two independent implementations.** I wrote implementation A from the rules alone: it classifies each character as letter, joiner or separator and finds words with the regex `L+(?:JL+)*` over that class string. I wrote implementation B independently as a character-by-character scanner with no regular expressions.
2. **Example rows.** I parsed all 43 rows directly from `spec.md` (inputs and expected outputs read with `ast.literal_eval`). Both implementations match all 43 rows, including both `TypeError` rows.
3. **Differential test.** I compared A and B on 200,000 random strings built from the characters the rules treat specially: Latin, Cyrillic and Kazakh letters, `İ`, Greek `Σ`/`σ`, `ß`, every apostrophe and dash variant, combining marks (U+0300, U+0301, U+0306, U+0308), soft hyphen, zero-width characters, digits, `²`, `½`, `Ⅻ`, underscore, emoji, Devanagari, an enclosing mark, a spacing mark and a lone surrogate. There were 0 disagreements.
4. **Mutation test.** For each plausible wrong reading of the spec, I built a variant implementation and checked that at least one example row rejects it:

   | Wrong reading                                   | Rejected by rows       |
   | ----------------------------------------------- | ---------------------- |
   | Marks deleted before NFC                        | 29, 37, 38             |
   | No NFC normalization                            | 29, 37, 38             |
   | U+02BC not mapped to `'`                        | 43                     |
   | `str.casefold()` instead of `str.lower()`       | 19, 40, 41             |
   | Lowercasing character by character              | 40, 41                 |
   | Letters matched with the regex `[^\W\d_]`       | 39                     |
   | All dash characters join words                  | 17                     |
   | Ties ordered by first occurrence                | 5, 7, 23, 25 and more  |
   | Ties at the 10th place kept                     | 24, 25                 |

   Every wrong reading is caught by at least one row.

5. **Claims in the prose.** I checked the factual claims in rules 6 and 11 against CPython 3.13 (Unicode 15.1.0): `'ΟΔΟΣ'.lower()` gives `οδος`; `'ΟΣ-Α'.lower()` gives `ος-α`; `'ΟΣ.Α'`, `'ΟΣ:Α'`, `'ΟΣ^Α'` and `` 'ΟΣ`Α' `` keep `σ`; `'ΟΣ0Α'` and `'ΟΣ Α'` give `ς`; `и` + U+0300 composes to `ѝ` (U+045D) under NFC; decomposed `й` and `ё` compose before marks are deleted; U+02BC is category `Lm` (a letter), so without rule 3 it would stay inside a word. All claims hold.
6. **Performance.** 7 million characters of mixed text took about 2.5 s, and a single 5-million-letter word took about 1.3 s. Both scale linearly, consistent with the O(n log n) requirement.

### Interface

Unambiguous. `isinstance(text, str)` accepts `str` subclasses; every other type raises `TypeError`. The output is a `list` of at most 10 `(str, int)` tuples. A lone surrogate (`"a\ud800b"`) returns `[("a", 1), ("b", 1)]` without an exception, so "no other exception for any `str` input" holds. The reference environment (CPython 3.13, Unicode 15.1.0) removes the earlier dependence on the Python version.

### Unicode normalization

Unambiguous and covered. The step order (NFC, delete `Mn` and `Cf`, map apostrophes, lowercase the whole text, delete `Mn` again) is explicit. Rule 11 now correctly says that only marks without a precomposed form are removed, and rows 37 and 38 show the two consequences (`й`/`ё` survive; grave-accented `ѝ` is not merged with `и`).

### Tokenization

Unambiguous and covered. A letter is exactly `str.isalpha()`, the difference from `[^\W\d_]` is stated and tested (row 39), and a word is the leftmost maximal match of `L+ (J L+)*`. Rule 3 maps `’` and `ʼ`, and both are now tested (rows 6 and 43).

### Case handling

Unambiguous and covered. Rule 6 now states the Final_Sigma condition correctly, including which characters are skipped, and rows 20, 40, 41 and 42 fix the behavior for an apostrophe, the end of a word, a hyphen and a period.

### Ordering, ties and the cutoff

Unambiguous and covered by rows 23–25: code point order for ties, exactly 10 entries after sorting.

### Edge cases

Covered: empty string, punctuation and digits only, fewer than 10 words, `None` and `bytes`, line-break hyphenation, double, leading and trailing joiners, invisible characters, unsupported scripts.

### Intentional design decisions (not defects)

- `‘` and the backtick are separators; only `’` and `ʼ` are mapped to `'`.
- Digits are separators (`a1b` gives `a` and `b`).
- `ё` and `е`, `ѝ` and `и`, and Latin and Cyrillic look-alikes are all different; there is no transliteration, stemming or stop-word list.
- A Greek `Σ` before a hyphen inside a compound becomes `ς` (`ος-α`); this follows from lowercasing the whole text with `str.lower()`.
- Scripts with combining vowel signs or without spaces are declared unsupported (rule 14); their output is still fully determined.

### Remaining blockers

None.

### Minor suggestions

1. The Rule column uses the labels "basic" and "Function" instead of rule numbers.
2. The rules are numbered in an order different from the processing order (rule 11 runs first). The paragraph before the rules gives the real order, so this affects only readability.
3. The raw Markdown of rows 37–43 is not padded to the same column widths as rows 1–36. It renders identically.

### Conclusion

The spec is deterministic, internally consistent and fully illustrated: two independently written implementations agree on all 43 rows and on 200,000 random strings, and every plausible wrong reading I tried is rejected by at least one row. No further changes are required.
