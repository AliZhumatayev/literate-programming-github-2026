# LLM review of `spec.md`

## Model and disclosure

Model: **Claude Sonnet 5.5** (effort: _High_) (Anthropic), used through claude.ai. The first
draft of `spec.md` was written with help from Claude; I then revised it, and the
final version is the one reviewed below. The review was done by the same model
family, so it is not an independent second-model review.

## Last review

Prompt used with Claude Sonnet 5.5, with the complete final `spec.md` as context:

> Review spec.md

Prompting Claude Sonnet 5.5 with `Review spec.md` produced the following output;
strikethrough indicates LLM output that I deemed incorrect or irrelevant.

Review output:

> The spec is in good shape. I checked every expected output in the tables against the rules, and they are consistent. I found a few issues worth fixing and some gaps worth covering.
>
> **Verified correct**
>
> * The Unicode-escape examples: NFC composition of `и` + U+0306 into `й`, `е` + U+0308 into `ё`, and the non-composition of `о` + U+0301.
> * The `m²`, `Ⅳ ½` cases. `[^\W\d_]` does keep `²`, `Ⅳ`, and `½`, so the “regex trap” claim holds.
> * The `İstanbul` case, and the claim that U+0130 is the only unconditional multi-character lowercase mapping.
> * The sort orders in Rule 5. `я` (U+044F) < `ё` (U+0451) < Kazakh letters (U+0492 and up), and Latin Extended-A/B sorting before Cyrillic.
> * The tie-and-cutoff examples, including `k k a b c d e f g h i j`, which yields exactly 10 items.
>
> **Issues**
>
> 1. Rule 1 overclaims for Greek. Python’s `str.lower()` applies the Final_Sigma rule, so `ΟΔΟΣ` lowercases to `οδος` (with final ς), while `οδοσ` stays with σ. These differ only by case but are counted as different words. Since the spec says “one set of Unicode rules applies to all text,” either document this as an exception or state explicitly that case-insensitivity means `str.lower()` behavior, final sigma included.
> 2. “Each example would produce a different result under some other plausible reading” isn’t true of every row. For example, `Hello, world! Hello — world…` and `cat dog cat bird dog cat` are basic cases that most readings would pass. Either soften the sentence or audit the tables and cut or label the redundant rows.
> 3. Rule 2’s combining-mark rule and Rule 3’s joiner rule interact, and the spec doesn’t say how. For example, `а́-b` (a letter, then a combining mark, then a hyphen) would not satisfy “a letter immediately before” the hyphen. This is probably intended, but one example would make it explicit.
>
> **Gaps in coverage**
>
> * A word containing a Latin Extended Additional letter (`ạ`), since Rule 5 mentions it but no word test uses it.
> * Unicode whitespace other than ASCII space and tab, such as NBSP (U+00A0) and U+2028, to confirm they separate words.
> * A repeated-apostrophe or hyphen-chain case like `rock-'n'-roll`, to pin down how the joiner rule applies at the edges of a chain.
> * An exactly-10-distinct-words input with no ties, to confirm the cutoff boundary.
> * A single-character input and a string of only combining marks (expected `[]`).
> * Greek final sigma, if you keep item 1 as a documented behavior.
>
> Do you want me to draft the additional test rows, or write a test file that runs these examples against a `top_words` implementation?

## Notes on the feedback

- Issue 1, Greek final sigma in Rule 1:
- Issue 2, "different result under another reading" claim:
- Issue 3, combining mark next to a hyphen (Rules 2 and 3):
- Gap, Latin Extended Additional letter (`ạ`):
- Gap, Unicode whitespace (NBSP, U+2028):
- Gap, hyphen/apostrophe chain (`rock-'n'-roll`):
- Gap, exactly 10 distinct words with no ties:
- Gap, single-character input and only combining marks:
- Gap, Greek final sigma example: