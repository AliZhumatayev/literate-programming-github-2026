# LLM review of spec.md

## Setup

- LLM: Claude, run as a Claude Code subagent with the model sonnet 5.5. The agent used its default effort level.
- Prompt (the third and last review run):

  ```
  Review /Users/iomiras/uni/literate-programming-github-2026/m_shaltayev/spec.md. Read-only: no edit, no write, no run code. Reply w/ review text only. Find errors, contradictions, underspecified behavior, ambiguity that lets 2 valid impls give different output. Each finding: quote spec, say problem.
  ```

- I ran the same prompt three times. After rounds 1 and 2, I changed the spec to fix the findings. Round 1 found 14 points, round 2 found 15, and round 3 found 12. Round 3 is the last review. Strikethrough marks output that I judged incorrect or irrelevant.
- After round 3, I changed the spec to fix the findings. I did not run a fourth review on the final text.

## Output of the last review

Review of spec.md

I found no hard contradiction in the examples. I checked each expected output by hand against the rules, and all of them are correct. The findings below are in order of impact.

1. Variation selectors: the range is inconsistent (real ambiguity).
- Rule 3: "any character in Unicode category `Mn` or `Mc`, except the variation selectors U+FE00 to U+FE0F."
- Rule 7: "variation selectors ... separate words."
- Problem: Rule 7 does not give a range. Other variation selectors are category `Mn`: U+E0100 to U+E01EF, and Mongolian U+180B to U+180D and U+180F.
- Under rule 3 they are marks and attach to the previous letter. Under rule 7 they separate words.
- Two valid implementations give different words for `a\U000E0100b`.
- Fix: name the range in rule 7 as U+FE00 to U+FE0F only, or exclude all variation selectors from rule 3.

2. Enclosing marks (`Me`) are not named.
- Rule 3: marks are `Mn` or `Mc`.
- `Me`, for example U+20E3 combining keycap, falls under rule 7 "Every other character". That is correct but never stated.
- A reader can think `Me` is a mark. Fix: add `Me` to the list in rule 7.

3. Circular wording in rule 6.
- "A joiner at the start of a word ... do not join."
- Rule 5 says "A word starts at a letter", so a joiner can never be at the start of a word.
- The outcome is clear from the examples. The sentence is still circular.
- Fix: write "A joiner that has no letter before it, or no letter after it, separates words."

4. Rule 6 repeats the mark logic and hides a case.
- "A mark that does not follow a letter, a mark, or a joiner-letter pair"
- A joiner-letter pair ends in a letter, so the clause is redundant.
- The rule does not say that a mark after a joiner is dropped because the joiner ends the word. Only the example `a-́` shows this.
- Fix: state it in the rule.

5. "Single" joiner in rule 4.
- "a single apostrophe ... or a single hyphen-minus"
- "Single" can mean "one character" or "one joiner in a row". Rule 6 uses the second meaning.
- Fix: delete "single" from rule 4.

6. The `str` subclass is underspecified.
- "A `str` subclass is accepted."
- A subclass can override `lower`, `__iter__`, `__getitem__` and similar methods. An implementation that uses them can differ from one that first converts to plain `str`.
- The type of the returned `word` is not stated. It is probably a plain `str`.
- Fix: say "Convert to plain `str` first" and "`word` is an exact `str`".

7. The Unicode data depends on the interpreter.
- "The Unicode data is the data of the Python interpreter that runs the function."
- Python 3.9 uses Unicode 13. Python 3.13 uses Unicode 15.1 or later. Characters added later change `isalpha`, category and `lower()` results.
- The examples use only old characters, so they are safe.
- Fix: say the tests use only characters that exist in Unicode 13.

8. The performance claim is imprecise.
- "The running time is O(n + d log d)"
- Hash counting is expected O(1) per word, not worst case. String compare and hash cost O(word length), so the sort is not strictly O(d log d).
- Fix: say "expected" and "ignoring word length in comparisons", or drop the bound.

9. ~~Non-target scripts are underspecified.~~
- ~~"Other scripts follow the same rules, but the tests do not check them."~~
- ~~Under `isalpha`, a run of CJK characters is one word, because no spaces cut it.~~
- ~~For Arabic-script Kazakh, U+200C ZWNJ is `Cf` and separates words.~~
- ~~This is stated as untested. A reader should still know that the result is probably not what a user wants.~~

10. Escape notation in the examples.
- "`\n`, `\t`, `\xXX`, and `\uXXXX` stand for one character each"
- No notation exists for a literal backslash, and `\U` for characters above U+FFFF is not listed.
- The current examples do not need either, but a test author who writes new cases will.
- Fix: add `\\` and `\UXXXXXXXX`.

11. Minor wording issues.
- Line 117: "The digits and the hyphen next to them are dropped." The hyphen is dropped because it trails the word, not because of the digits. The sentence suggests a special rule for digits.
- ~~Line 27 repeats lines 25 and 26. It is consistent, so this is only redundancy.~~
- Normalization step 3 and the bullet "Do not normalize again" say the same thing twice. Step 3 is not an action.

12. Per-word lowercasing and Greek final sigma.
- "Lowercase each word alone, not the whole text."
- This is clear. But `ΟΔΟΣ` gives `οδος` with the final `ς`, because `str.lower()` handles the final sigma inside the word.
- An implementation that lowercases each character alone gives `οδοσ`. Both can be called "lowercase".
- Fix: add an example, or say "call `str.lower()` on the whole word string".

## What I did with the output

- I accepted findings 1 to 8, 10, 11 (first and third points), and 12. I changed the spec for each one. For finding 12, I changed the wording and did not add a Greek example, because the tests do not check Greek.
- I rejected finding 9. The spec says the target scripts are Latin, Cyrillic, and Kazakh. It also says that other scripts are not checked. A longer note adds no behavior that a test can check.
- I rejected the redundancy point in finding 11 (line 27). The sentence gives a general rule, and the example after it only shows one case.
