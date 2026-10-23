# LLM review: Top words

## Model and reviewed version

- Model: **GPT-6**, as identified by ChatGPT in this conversation.
- Review date: **2026-10-23**.
- Reviewed document: the revised `spec.md` in this submission, with 15 grouped acceptance rows.
- This is the latest review output for the revised specification; it does not claim that Claude or another model reviewed this version.

## Exact prompt

```text
Review the attached revised spec.md as the latest version after addressing the previous review. Check correctness and internal consistency of the rules and expected outputs. Verify that it addresses all prior feedback: shorten and merge the acceptance table; justify the apostrophe and hyphen policy; explain Kazakh Latin dotted-I behavior under locale-independent case folding; clarify Unicode-version handling; state the word definition declaratively; preserve representative normalization and combining-mark edge cases; and specify fresh-list and interface behavior. Check expected outputs carefully. Distinguish any required corrections from optional improvements, and give the full review in English.
```

## Full GPT-6 review output

**Verdict:** The revised specification is clearer and substantially shorter. It preserves the product's important observable behavior while removing several rare Unicode-specific examples that added length without requiring separate product rules. The word definition is declarative, the normalization pipeline is explicit, and the acceptance table now has 15 grouped rows.

### Previous feedback addressed

- **Length and repetition:** The examples are consolidated from 48 rows to 15 grouped rows. Related counting, separator, sorting, and interface behaviors share rows or concise prose. The separate scope section and repetitive interface checklist are removed.
- **Apostrophes and hyphens:** The specification explicitly chooses to split contractions and hyphenated compounds. This is presented as a deliberate, language-independent tokenization policy, not as an accidental behavior.
- **Kazakh Latin dotted I:** The spec explicitly says that case folding is locale-independent and does not apply Turkish/Azerbaijani-specific tailoring. It explains that `I` maps to `i` while `İ` maps to `i\u0307`, and the table includes an example showing the distinction.
- **Unicode version:** The rules use the Unicode database provided by the Python runtime rather than requiring every Python version to use Unicode 15.0. The document notes that its examples were checked against Unicode 15.0 and recommends Python 3.12 as a convenient baseline.
- **Declarative word definition:** A word is a maximal uninterrupted sequence that starts with a letter and may continue with letters or combining marks. Unattached marks are discarded and cannot attach across separators or start a word.
- **Unicode edge cases:** The table retains representative cases that resolve the real category-versus-appearance ambiguity (U+02BC and U+2139 followed by U+FE0F), plus attached/unattached marks and examples requiring NFC before and after case folding. Several additional rare code-point examples have been removed because the general rules already define their behavior.
- **Interface behavior:** The prose requires a fresh list on every call, including empty results; calls do not accumulate counts, perform I/O, or affect external state. No unenforced performance constraint has been introduced.

### Correctness checks

I checked all **36 concrete input/output examples** in the revised table against a small reference tokenizer implementing the written normalization, Unicode-category, counting, and ordering rules, using **Python 3.13.5 / Unicode 15.1.0**. All 36 matched the expected results, including the three invalid-input cases. This is a check of the examples in this revision, not a claim that the assignment's complete test suite or a production implementation was run.

The ordering examples are consistent: `i` sorts before `i\u0307` when counts tie; `apple`, `кот`, and `қазақ` are in code-point order; T12 returns only `a` through `j`; and T13 ranks `k` first because it occurs twice. The combining-mark cases and both normalization-order cases are consistent with the stated requirement to normalize and case-fold the entire input before tokenization.

The stated product choices are now explicit: contractions and compounds are split, stop words and markup fragments count as plain text, distinctions that remain after normalization are preserved, and case folding is not locale-tailored. These are trade-offs, not contradictions.

### Remaining issues

I found **no required corrections** in this revision. The remaining portability limitation is documented: newly assigned or reclassified characters may behave differently under different runtime Unicode databases. If exact reproducibility across Python versions becomes a grading requirement, the Python and Unicode database versions should then be pinned; that is not necessary under the current wording.
