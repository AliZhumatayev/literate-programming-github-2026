# Final LLM Review of `spec.md`

**Model:** OpenAI GPT-6 (ChatGPT), reasoning mode.

**Exact prompt:**

> Review the complete contents of `spec.md` as a specification for the Top Words exercise. Check whether two independent Python 3 implementations could reasonably disagree on any valid input. Look for ambiguous interfaces, Unicode normalization and tokenization problems, case handling, ordering and ties, cutoff rules, edge cases, and contradictions between the rules and the input/output examples. Distinguish actual defects from intentional design decisions. Report remaining blockers and suggested improvements; do not rewrite the specification.

## Review output — last review

**Overall assessment:** The revised specification addresses the professor's substantive Unicode concerns and defines deterministic results for the stated CPython 3.13.5 environment. No unresolved blocking inconsistency was found in the current text or its examples. Its choices are sometimes linguistic compromises, but are stated explicitly.

### Interface
The input must be a `str` or `str` subclass (`isinstance(text, str)`); other values raise `TypeError`. The output is a list of up to ten `(str, int)` pairs. The function name, module name, and non-file-based interpretation of input are explicit. The Python version is fixed to CPython 3.13.5, which is compatible with the stated annotation form. Although runtime type annotations are optional, the observable behavior is not.

### Unicode normalization and case
The specified sequence NFD → `casefold()` → NFC is unambiguous, and addresses the canonical-equivalence problem caused by applying `casefold()` before decomposing. It does not use compatibility normalization, transliteration, stemming, or language-dependent case rules. The output term “case-folded” is accurate: Unicode case folding is not the same as lowercasing.

### Word tokenization
A letter starts a word; subsequent letters and Unicode `M*` marks remain in that word. Marks at the beginning of the text, or after a separator, cannot start words. This handles the Turkish/Kazakh-Latin dotted capital I, Russian stress marks, and Devanagari combining marks without splitting the word. A stress-marked Russian word and its unmarked equivalent remain *different tokens*. That is a deliberate design choice, not a contradiction; if accent-insensitive matching were desired, different rules would be needed. The same applies to `İstanbul` and `istanbul`, whose case-folded forms contain different combining-mark sequences. The apostrophe-like modifier letter U+02BC remains alphabetic under the pinned Unicode tables; ordinary punctuation apostrophes separate words. Adjacent letters from different scripts can form a single word.

### Counting, ordering and cutoff
Identical normalized tokens are counted, with no stop-word filtering. The result is sorted first by count descending, then by Python string order ascending. The limit is applied only after sorting, selecting exactly ten words at most, even when the tenth and eleventh words are tied. The exact-ten and tie-across-cutoff tests cover the boundary.

### Edge cases and examples
The rules specify empty and punctuation-only inputs, one-character words, unrecognized types, marks without an initial letter, canonical equivalence, decomposed Russian `й`/`ё`, and multilingual text. Every one of the **26 example rows** was checked with a fresh Python implementation using the specified NFD → `casefold()` → NFC pipeline, Unicode mark categories, counting, and sorting in **Python 3.13.5**. All 26 matched, including the `None` → `TypeError` example. This is implementation-based validation, not a proof of correctness for every Unicode input.

### Remaining blockers
**None identified.**

### Suggested improvements (non-blocking)
1. If deployment across multiple Python versions is required, document or test the applicable Unicode database version on those versions; the specification intentionally pins CPython 3.13.5 to avoid version-dependent interpretation.
2. For a real news product, consider whether stress-marked and unstressed Russian words, and dotted-I vs plain-i spellings, should be grouped. The specification currently keeps marks and distinguishes those spellings intentionally. Changing that is a product requirement change, not necessary to eliminate ambiguity.
3. Additional property-based testing over randomly generated Unicode strings could reveal implementation errors beyond the 26 examples. No conflicting outputs were found in the stated tests.

**Conclusion:** The revised specification is internally consistent and testable under its pinned Python environment. No further changes are required to resolve the issues identified in this review.
