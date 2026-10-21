# Final LLM Review of `spec.md`

## Model used

OpenAI GPT-6 (ChatGPT).

## Exact prompt used

> Review my `spec.md` for the Top Words problem. Check if the requirements are clear and complete. Find any missing details or ambiguous rules that could lead to different results. Pay attention to word counting, punctuation, capitalization, sorting, and edge cases. Explain any problems in simple English and suggest improvements.

## Full output of the latest review

**Overall:** The specification is clear and detailed. It defines the function's input and output, word boundaries, normalization, counting, sorting, and edge cases. A developer should be able to implement it without choosing different rules for common inputs.

**1. Word counting — Clear.** Every occurrence is counted, including common words. Different forms such as `run`, `runs`, and `running` remain separate. There is no stemming or stop-word removal.

**2. Punctuation — Clear.** Punctuation separates words, with a precise exception for straight and curly apostrophes placed directly between letters. Hyphens, numbers, and underscores are separators. The specification gives examples for these rules.

**3. Capitalization and languages — Clear.** The NFC → `casefold()` → NFC process is stated explicitly. This handles uppercase and lowercase characters and supports the English, Russian, and Kazakh text required by the assignment. The `Straße` example also checks a nontrivial case-folding behavior.

**4. Sorting — Clear.** Results are sorted by decreasing count, followed by Python string order when counts tie. The first ten entries are kept only after sorting. This removes ambiguity about ties at the tenth position.

**5. Edge cases — Clear.** Empty text and text without words return `[]`. Inputs that are not strings raise `TypeError`. The output is a list of `(word, count)` tuples, not printed text.

**6. Unicode combining marks — Small wording improvement suggested.** The rule for a combining mark being “attached” to a letter could be expressed more precisely. For example, a word may have multiple consecutive combining marks, and each should remain part of the same word. Suggested revision: a combining mark belongs to a word if it follows a letter or another included combining mark in that word; an isolated combining mark is a separator.

**7. Examples — Consistent.** The listed expected outputs agree with the counting, sorting, and tokenization rules, including the mixed-language and ten-word examples.

**Conclusion:** I found one minor wording issue about consecutive combining marks, not a major contradiction. With that wording clarified, the specification is sufficiently precise to guide one predictable result for the tested cases. No additional changes to the function interface or ranking rules are necessary.

## My notes on the review

- **Accepted:** The comment about combining marks. The final `spec.md` now explicitly allows a combining mark after another included combining mark in the same word.
- **Rejected or irrelevant feedback:** None.
- **Clarification:** This document records the final review and the corresponding final specification. The review is about the `Top Words` specification, not the unrelated SIS1 assignment.
