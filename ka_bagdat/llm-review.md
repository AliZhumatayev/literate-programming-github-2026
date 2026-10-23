# Final LLM Review of `spec.md`

## Model used

OpenAI GPT-6 (ChatGPT).

## Exact prompt used

> Review my `spec.md` for the Top Words problem. Check if the requirements are clear and complete. Find any missing details or ambiguous rules that could lead to different results. Pay attention to word counting, punctuation, capitalization, sorting, and edge cases. Explain any problems in simple English and suggest improvements.

## Full output of this final review

The revised specification gives a deterministic result for each input on a particular Python interpreter. It explicitly defines the argument type, normalization sequence, letter and mark categories, word boundaries, counting, sorting, and the top-ten limit.

1. **Python and Unicode versions — addressed.** The text says that Unicode properties and `casefold()` follow the running Python interpreter, rather than claiming the same result on every Unicode database version. That is a valid choice, and the limitation is explicit.
2. **Normalization and case folding — addressed.** The first rule applies NFC, case folding, then NFC. It explicitly says tokenization happens after those steps. The Turkish `İ` example shows that case folding can introduce a combining mark that survives NFC.
3. **Invisible format characters — addressed.** Soft hyphen, zero-width joiner, and zero-width non-joiner are explicitly separators. This can split a word copied from a web page, but it is now a documented policy, not an accidental detail.
4. **Apostrophes — addressed.** Only ASCII `'` and U+2019 `’` are allowed inside a word, and only with letters immediately on both sides. U+02BC, U+2018, and U+2032 are explicitly separators. This avoids treating U+02BC as an ordinary letter just because its Unicode category is `Lm`.
5. **Combining marks near apostrophes — addressed.** An apostrophe after a mark is a separator even if a letter occurs earlier in the same word. The spec states this choice and includes a case with a combining mark that remains decomposed after NFC.
6. **Wording and input type — addressed.** The apostrophe rule uses the words “characters immediately before and after.” The type check is specified as `isinstance(text, str)`, so subclasses are accepted, but `bytes` and `None` are rejected.
7. **Output and side effects — addressed.** The contract says to return a list of tuples, not print it, and to have no externally observable side effects. The unneeded statement that a string input is not modified has been removed.
8. **Examples — addressed.** Inputs are written as Python expressions, including `""`, `None`, and `b"cat"`. Extra examples cover Kazakh-specific case pairs, Russian `е` versus `ё`, isolated marks, curly quotes around words, mixed-script lookalikes, and a frequency count that changes the top-ten cutoff.
9. **Counting and sorting — clear.** Stop words are counted; related word forms are not merged. Results sort by frequency first and by normalized Python string order for ties. The ten-result limit is applied after sorting.
10. **Complexity — stated appropriately.** There is no arbitrary input size limit and no prescribed algorithm. This is reasonable for a behavior specification.

**Final conclusion:** I found no remaining major ambiguous behavior in the revised specification. The rules are specific enough to produce one expected result on a given Python interpreter. A future implementation should still be checked against the examples, particularly Unicode edge cases; this review did not execute the implementation.

## My notes on feedback

- **Accepted:** All five substantive issues, all wording issues, and the requested edge-case examples from the instructor's Claude Opus 5.5 review. The revised `spec.md` addresses them directly.
- **Accepted (optional):** Added a statement that input size has no artificial limit; did not require a particular performance complexity.
- **Rejected feedback:** None. For the policy choices where Claude offered alternatives (e.g., remove format characters or split at them), I chose an explicit rule and documented it.
- **Review method:** The review above is of the revised specification. It is a textual review; it does not claim to have run code.
