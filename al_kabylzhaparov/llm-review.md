# LLM review of spec.md

Prompting Cursor Auto (the agent in this Cursor session) on 21 October 2026, with the prompt below, produced the review that follows. Strikethrough marks review text I judged incorrect or irrelevant.

## Prompt

> Review spec.md for the Top words assignment. Find any input that still has two legal outputs, any example that contradicts the procedure, and any rule an implementer could apply in two orders. Quote the rule you are objecting to. Do not propose new features.

The same prompt was run twice. The first pass found the four defects below. The submitted `spec.md` already includes the repairs. The second pass is the last review.

## Defects fixed before the last review

1. Some inputs were prose: "three spaces, or a newline, or a tab", and "`cafe` + U+0301". An implementer could tokenize that prose, or mis-count the combining mark as its own visible character. Those cells are now Python literals, and the introduction says how to read `\n` and `\uXXXX`.
2. The U+0130 row did not name the other reading it rules out. Casefolding the whole string before the scan expands U+0130 to `i` plus U+0307, then the combining mark is a separator, and `"\u0130 i"` becomes `[("i", 2)]`. The procedure already casefolds each token after the scan. The row now says so.
3. A sentence claimed the tables had been checked on CPython 3.11 through 3.13. Only CPython 3.13.15 was run. That claim is removed. The contract is still "use `unicodedata` and `str.casefold`," and the tables name the interpreter that produced them.
4. The entity input was written as the raw characters `&amp;` in a table cell. A renderer that expands HTML entities would show `& cat`, and the expected word `amp` would look like a contradiction. The cell is now `"\u0026amp; cat"`.

## Last review

The procedure in `spec.md` is ordered, and the tables match that order. I did not find a listed input that the procedure can still map to two outputs.

- Token boundaries are "letters, plus U+0027 only when it sits between two letters." Hyphen, digit, underscore, tag, null, soft hyphen, and BOM fall out of that one rule, and each has a row.
- Case folding is `str.casefold` on the finished token, then NFC. The `Straße`, `Σ σ ς`, and `"\u0130 i"` rows reject `str.lower`, Greek-sigma folding differences, and casefold-before-scan.
- `ё` / `е`, Kazakh `і` / Russian `и` / Latin `i`, and the code-point tie `ё я` reject dictionary folding and dictionary sort.
- The cutoff rows reject "keep every tie" and "keep first-seen order." `k` is absent from an 11-way tie. `vv` takes the 10th place ahead of `ww`, `xx`, `yy`, and `zz`.
- `<b>cat</b> cat` is `[("b", 2), ("cat", 2)]` because `</b>` contains a second `b`. That matches the scan. It is not an off-by-one in the table.

~~Drop the `Straße` and `Σ σ ς` rows. The problem only mentions English, Russian, and Kazakh, so those rows are out of scope and will confuse the grader.~~ They are the rows that make `casefold` a different function from `lower`. Deleting them would leave that choice untested. The problem statement says the articles may be in those three languages; it does not say the function may do something else on other characters.

~~Remove stopwords. A "Trending words" box that ranks `the` above `cat` is not what a news site would ship.~~ The assigned sentence says "the 10 most common words," and the spec repeats that. A stopword list would be a second, unnamed product decision.

~~Give the function an `n` parameter with default 10.~~ The assignment fixes the length at 10. `top_words("cat", 10)` is specified to raise `TypeError`.

~~Rejoin words split with a hyphen at a newline. Newspaper text does that, so `"well-\nknown"` should be `[("wellknown", 1)]`.~~ That is a different product. The spec chooses the simpler scan and shows the output it produces.

~~Parse the HTML and decode entities, because the input is an article.~~ The function receives text, not a document. The `<b>` and `\u0026amp;` rows exist to keep those readings out.

~~Reject subclasses of `str`. `isinstance(text, str)` is too loose.~~ A `str` subclass is a `str`. The spec says what `top_words(S("Cat cat"))` returns. Tightening that further does not change any article.

~~Require the `TypeError` message to be exactly `text must be a str`.~~ The spec says the message is not part of the contract. Callers are required to see the exception type only.

No remaining comment from this pass describes an input with two legal outputs.

## Implementation check

An implementation of this procedure was run against every table in `spec.md` on CPython 3.13.15 (Unicode 15.1.0). Every listed output matched, including the cutoff rows, `"\u0130 i"`, `"cafe\u0301s cafés"`, `"<b>cat</b> cat"`, and `"\u0026amp; cat"`.
