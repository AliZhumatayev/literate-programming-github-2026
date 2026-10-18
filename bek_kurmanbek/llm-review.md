# LLM review: Top words

## Model and reviewed version

- Model: **GPT-6**, as identified by the submitter in ChatGPT.
- Review date: **2026-10-19**.
- Reviewed document: `spec.md`, the version containing cases **T01–T39**, before the changes documented below.
- The full review is reproduced below as provided. The response to feedback records changes made afterward; it is separate from the model's output.

## Exact prompt

```text
Review the attached spec.md for ambiguous rules, contradictions, incorrect expected outputs, and missing examples. Distinguish actual problems from optional improvements. Give your full review in English.
```

## Response to the review

The substantive finding is accepted. The revised specification keeps Unicode general categories authoritative and resolves Rule 3's overbroad wording. No review text is rejected, struck through, or omitted.

| Feedback | Action taken in the revised specification |
| --- | --- |
| Rule 3 incorrectly implied that every apostrophe and emoji is a separator. | Rule 3 now explicitly makes Rule 2's categories authoritative. It names U+0027 and U+2019 as separator examples, and explains why U+02BC and U+2139 followed by U+FE0F are retained. T13 and T17 descriptions were also narrowed to their actual characters. |
| Add examples for the two conflicting cases. | Added T40 and T41. |
| Seven optional examples clarify already-defined behavior. | Added all seven as T42–T48, including the initially isolated U+0345 mark that becomes a letter during case folding. |
| Rule 4 could qualify its preservation language. | Rule 4 now preserves distinctions remaining after Rule 1's normalization and case folding. |
| Check that even empty results are new lists. | Added a supplemental identity check and its executable test. |
| Clarify the practical runtime expectation. | Explicitly recommend Python 3.12 because its standard library supplies Unicode 15.0. |
| Add performance constraints only if they will be enforced. | No performance constraints were added: none is required or enforced for this assignment. This is consistent with the review, not a rejection of a required correction. |

Rule 2 was also clarified to state that unattached marks do not attach across separators or become prefixes, and that the complete normalization pipeline runs before tokenization. The counting and ordering policy, return type, and all original expected outputs remain unchanged.

## Validation after applying the changes

The existing implementation already followed the category-based rules, so its behavior did not need to change. The acceptance suite was extended to cover the revised specification and run on **Python 3.12.14 / Unicode 15.0.0**:

```text
python3 -m unittest -v test_top_words.py
Ran 53 tests
OK
```

The 53 tests comprise all 48 table cases and five supplemental interface checks. This is the post-change validation result; the review below describes the earlier 39-case document. The original review is preserved without claiming that GPT-6 reviewed the later changes.

## Full GPT-6 review

The specification is largely precise and implementable. **I found one genuine wording inconsistency and no incorrect expected outputs.** I checked all 39 table cases against a reference implementation of the stated Unicode-category rules using Python 3.12.14 and Unicode 15.0.0; all matched, including the three `TypeError` cases.

The actual issue is in **Rule 3’s blanket statement that apostrophes and emoji separate words**. This conflicts with Rule 2 for some Unicode characters:

| Input            | Result required by Rule 2 | Conflict with Rule 3                                                                                       |
| ---------------- | ------------------------- | ---------------------------------------------------------------------------------------------------------- |
| `"don\u02bct"`   | `[("don\u02bct", 1)]`     | U+02BC is MODIFIER LETTER APOSTROPHE, category `Lm`, so it belongs to the word.                            |
| `"\u2139\ufe0f"` | `[("\u2139\ufe0f", 1)]`   | The information emoji ℹ️ consists of a letter, category `Ll`, followed by a combining mark, category `Mn`. |

Unicode 15.0 confirms these categories, and its emoji data recognizes the second sequence as an emoji. [unicode.org](https://www.unicode.org/Public/15.0.0/ucd/UnicodeData.txt?utm_source=chatgpt.com)

Consequently, one implementer might follow the category rules while another explicitly removes all apostrophes or emoji. They would produce different results.

**Recommended correction:** make the category definition authoritative. Qualify the examples in Rule 3 by naming the straight apostrophe U+0027 and curly apostrophe U+2019, and explain that visual appearance or emoji presentation does not override Unicode categories. Keep the existing statements that separators split words rather than join them and that consecutive separators do not create empty words.

Add the two examples above as acceptance cases if that is the intended policy. If every apostrophe and emoji must instead be excluded, the specification needs explicit exceptions to Rule 2 and a precise definition of those exclusions. That would change the current category-based behavior.

None of the existing table entries needs correction. In particular:

- **T04–T05:** The Kazakh ordering is correct for Unicode code-point order, even where it differs from dictionary order.
- **T13 and T17:** These particular apostrophes and this particular emoji are separators. The problem is the broader wording, not these outputs.
- **T25–T26:** The cutoff correctly follows sorting, and frequency correctly takes priority over lexical order.
- **T35–T36:** Both normalization-order examples are correct and useful.
- **T37:** The fullwidth letter remains distinct from ASCII `a` under the specified pipeline.
- **T38–T39:** The outputs correctly preserve spacing and enclosing combining marks.

The following are **useful missing examples, but optional coverage improvements**. Their answers are already determined by the existing formal rules.

| Behavior to demonstrate                                                                       | Input                     | Expected output              |
| --------------------------------------------------------------------------------------------- | ------------------------- | ---------------------------- |
| A combining mark can occur inside a word, with more letters afterward                         | `"q\u0301b q\u0301b"`     | `[("q\u0301b", 2)]`          |
| A mark after a separator cannot attach backward or become a word prefix                       | `"q,\u0301b"`             | `[("b", 1), ("q", 1)]`       |
| Normalization and case folding happen before deciding whether characters are letters or marks | `"\u0345"`                | `[("\u03b9", 1)]`            |
| Changing scripts without a separator does not split a word                                    | `"catкот catкот"`         | `[("catкот", 2)]`            |
| An invisible format character separates words                                                 | `"a\u200db"`              | `[("a", 1), ("b", 1)]`       |
| Case folding can introduce a combining mark that must remain attached                         | `"İ i\u0307 i"`           | `[("i\u0307", 2), ("i", 1)]` |
| Letters outside the Basic Multilingual Plane are supported                                    | `"\U00010400 \U00010428"` | `[("\U00010428", 2)]`        |

The `"\u0345"` example is particularly valuable: this character starts as a combining mark but becomes Greek iota during case folding. An implementation that discards unattached marks **before** normalization would incorrectly return `[]`. T35 does not fully cover this failure because its input already begins with a letter.

A few additional refinements are optional:

- **Qualify Rule 4’s preservation language.** “Keep accents and differences between letters” is best understood as preserving distinctions *remaining after the required normalization and case folding*. T32 already shows that some original distinctions, such as `ß` versus `ss`, disappear. A short qualification would prevent confusion; the explicit pipeline already resolves the behavior.
- **Test fresh-list identity directly.** The mutation check is helpful, but the requirement says every call returns a new list, including empty results. An additional check could store two results from `top_words("")` and verify that they are equal but are not the same object.
- **Clarify the practical runtime expectation.** “Python 3” and the later Unicode 15.0 requirement are not contradictory: alternative runtimes are allowed only if their behavior matches. A brief note recommending Python 3.12 for its built-in Unicode 15.0 database would help implementers. [Python 3.12.15 documentation](https://docs.python.org/3.12/library/unicodedata.html?utm_source=chatgpt.com)
- **Add performance limits only if they will be enforced.** The absence of a maximum article length or complexity requirement is not a functional defect. Such constraints are needed only if grading or production use depends on them.

I would not classify the fixed limit of ten, splitting contractions and hyphenated words, retaining stop words, counting HTML fragments, or using code-point ordering as problems. They are explicit design decisions. Likewise, accepting letters from other languages does not contradict the purpose section: the specification expressly broadens support beyond English, Russian, and Kazakh.
