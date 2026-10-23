Prompting Cursor Auto, with `Review spec.md` produced the following output; strikethrough indicates LLM output that I deemed incorrect or irrelevant.

The shortened spec still has one output per listed input. The procedure order is unchanged: NFC, then the two apostrophe replacements, then the scan, then `casefold` per token. The rows that previously sat in prose (`"\u0130 i"`, `<b>cat</b> cat`, the 11-way tie, and the tie at `vv`) still name the other reading, so those outputs are not left implicit.

~~The third column restates the assignment. Delete it and leave only input and output. The file is still too long.~~ The column is the required example of a different plausible output. Removing it would hide which decision each row locks. The cut already removed the paragraphs that repeated those cells.

~~Drop `Straße strasse` and `Σ σ ς`. The course text only mentions English, Russian, and Kazakh.~~ Those two rows are what makes `str.casefold` a different rule from `str.lower`. Without them, an implementation can use `lower` and still pass every other row.

~~Put the URL, emoji, and NBSP rows back. A separator rule with only punctuation is incomplete.~~ Step 4 already says every non-letter is a separator. Those rows would repeat `cat, cat! cat.` They do not change the procedure.

~~Add an `n` parameter defaulting to 10, and drop stopwords, so the result looks like a news widget.~~ The assigned problem asks for the 10 most common words. `top_words("cat", 10)` raises `TypeError`, and `the cat the` keeps `the`.

No listed input still has two results under the procedure.
