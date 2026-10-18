# LLM Review

Prompting Claude Opus 5.5 with `Review spec.md` produced the following output;
strikethrough indicates LLM output that I deemed incorrect or irrelevant.

---

## Review of spec.md (Top Words)

**Overall:** The spec is tight. Each decision (D1–D11) states one rule, and each
rule has at least one example whose expected output differs from a competing
reading. I traced all 35 examples (33 return values, 2 `TypeError` cases)
through the rules in D3 → D4/D5/D6 → D7 → D8–D11 order, and every expected
output follows from the rules as written. I found no contradictions between
the prose and the tables.

### Remaining observations

1. **Curly left quote used as an apostrophe (D3/D5).** Only `’` (U+2019) and
   `ʼ` (U+02BC) are normalized to `'`. Text that uses `‘` (U+2018) or a
   backtick inside a word, e.g. `"don‘t"`, will split into `don` and `t`. The
   spec is unambiguous here because D5 lists `‘` as a separator, so this is a
   deliberate choice, not a gap. Consider adding one example row to make the
   choice visible.

2. ~~**Case folding (D7).** `str.lower()` is not full Unicode case folding.
   `"İ".lower()` produces `i` + U+0307 (combining dot), and because `Mn`
   characters are removed in D3 *before* lowering, the combining dot survives
   in the returned word. `str.casefold()` also differs from `lower()` for `ß`.
   Consider specifying `casefold()` or removing `Mn` characters after
   lowering.~~

3. ~~**Configurable count.** Consider adding an optional parameter
   `n: int = 10` so the box size can change without editing the function.~~

4. ~~**Performance.** The spec does not state time or memory limits. Consider
   requiring O(n) time in the length of the text.~~

5. ~~**Scripts with spacing combining marks (D4).** Vowel signs in scripts
   such as Devanagari have category `Mc`, for which `isalpha()` is `False`, so
   such words will be split. Consider treating `Mc` characters as letters.~~

6. ~~**2017 Kazakh Latin alphabet.** That alphabet used apostrophes as letter
   modifiers (e.g. `a'` for `ә`). A word ending in such a letter loses its
   final apostrophe under D6.~~

7. **Line-break hyphenation (D6).** The example `"trend-\ning trending"`
   correctly documents that soft-wrapped words are not rejoined. This is
   consistent with the formal definition `L+ (J L+)*`. No change needed.

### Verdict

No errors and no underspecified behavior for English, Russian, or Kazakh
(Cyrillic) input. Item 1 is optional polish; the spec already determines the
output.