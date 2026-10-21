# LLM review of spec.md

Prompting Claude Opus 5.5 (model `claude-opus-5-5`), extra high effort, with
`Review spec.md` produced the output below. Strikethrough indicates LLM output
that I deemed incorrect or irrelevant, and each **Note** gives my reason or
says what I changed.

- **LLM:** Claude Opus 5.5, extra high effort, in a fresh non-interactive
  session started from this directory on 21 October 2026. The session could
  only read files (Read, Grep, and Glob tools), so it could not run Python.
- **Exact prompt:** `Review spec.md`
- **Command:**
  `claude -p "Review spec.md" --model claude-opus-5-5 --effort xhigh --allowedTools "Read,Grep,Glob"`

## Earlier rounds

I ran the same prompt, each time in a new session, after every revision. The
last review is round 8, below. Between rounds, I ran every example row against
a reference implementation to confirm the expected outputs.

| Round | Main findings                                                                                                                                                                                                                                        | What I changed                                                                                                                                                                         |
| ----- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1     | The `\uXXXX` escapes the tables promised were missing: the file had been saved with raw invisible and combining characters, so some rows looked identical. Results depend on the Unicode version. U+2010 and U+2011 hyphens split words. Return types were loose. | Restored every escape. Made U+2010 and U+2011 joiners. Stated the Unicode-version dependence, the stopword trade-off, and exact types. Added rows for a joiner after a mark and for `ё` sorting. |
| 2     | `re.findall(r"[^\W\d_]+", ...)` passed every row although it accepts `²`, `½`, and `Ⅻ`. Three rows used `ё` and `ә` without escapes. "Word" and "key" were mixed up. A claim about full-width letters was false on Python 3.12+. Word joiners and byte-order marks split words. | Added the `x²` row, escaped those rows, renamed the result to `(key, count)`, and corrected the claim. Step 2 now deletes every format character (category Cf).                      |
| 3     | The first NFC step was untested, and its stated reason was wrong. No reasons were given for keeping stress marks and `ʼ`. The Kazakh Latin `I`/`İ` consequence and the cost of letters-only words were understated.                              | Added the Greek U+0345 row with the real reason, reasons in section 5, the costs (`MP3` gives `mp`, `2024-го` gives `го`), and a row for U+034F COMBINING GRAPHEME JOINER.         |
| 4     | `casefold()` merges ligatures such as `ﬁ`, which contradicted the NFKC bullet. `XXI век` and abbreviations with periods had no rows. The supported-versions claim was unchecked.                                                                    | Added the `ﬁle`, `XXI век`, and `U.S. т.е.` rows. Ran every row on CPython 3.10 to 3.14 and changed the requirement to Python 3.10 or newer.                                         |
| 5     | No row had more than one joiner in a word. Look-alike apostrophes and the minus sign were untested.                                                                                                                                                  | Added the `Ростов-на-Дону Кот-д’Ивуар` and `don‘t don´t x−y` rows, and stated that only typographic forms of the same punctuation merge.                                            |
| 6     | No errors. Gaps where a wrong implementation would still pass every row: a joiner followed by a mark, an unnamed format character, Mc and Me marks, case-folding the whole text first, and `str` subclasses.                                       | Added a row for each gap and defined how `str` subclasses are handled.                                                                                                                 |
| 7     | No errors. Superscript letters are letters. `г.` followed by a no-break space and lone surrogates had no rows. The spec did not say why zero-width spaces are deleted. I rejected a `Mock(spec=str)` point and a Python 3.15 point, for the reasons given below. | Added the three rows and the zero-width-space reason.                                                                                                                                  |

## Last review (round 8)

The line numbers in the review refer to the version it reviewed. The fixes
marked "applied after this review" changed only prose: every table row is
byte-for-byte the same as in the reviewed version.

> I found no wrong expected outputs in `spec.md`. It's an unusually precise spec, and the issues below are mostly about where things are stated, not bugs in the rules.
>
> **What I checked:**
> - **Table rows:** I traced every row in all five tables through section 2 by hand, including the tricky ones (U+0345/U+1FB4 Greek, ǰ, İ/ı, `где\u0301-то`, the cutoff rows), and checked the "Rules out" outputs too. I couldn't run Python because shell access is blocked in this session, so this is checked against Unicode data from memory, not executed.
> - **The file itself:** it has no stray invisible characters (NBSP, ZWSP, soft hyphen, BOM, variation selectors) and no mixed Latin/Cyrillic words. The only literal combining marks are U+0301 in the section 5 prose; every table row uses escapes.

**Note:** For the record, I did run them: every row was executed against a reference implementation on CPython 3.10, 3.11, 3.12, 3.13, and 3.14, and all 60 pass.

> ## Worth fixing
>
> 1. **The str-subclass rule is missing from the procedure.** Section 1 says to treat subclass instances as `str.__str__(text)` (`spec.md:43-47`). Step 1 of section 2 (`spec.md:99`) only checks `isinstance`. Since section 2 is "the procedure `top_words` must match," make the conversion part of step 1. It matters in practice: `unicodedata.normalize` returns its argument unchanged when the text is already normalized, so a subclass instance can pass through and reach overridden methods.
>    - ~~There's also a small hole in the "only one correct value" promise. `isinstance(unittest.mock.Mock(spec=str), str)` is `True` because it fakes `__class__`. Item 2 only covers real subclasses, so that input has no defined result. Requiring `issubclass(type(text), str)` closes it.~~

**Note:** I accepted the first point and applied it after this review: step 1 of section 2 now continues with `str.__str__(text)`. That moves a rule section 1 already stated into the procedure, so no output changes. I rejected the `Mock` point. I ran `top_words(unittest.mock.Mock(spec=str))` on CPython 3.10 and 3.14, and following item 2 already gives a defined result there: `TypeError`, because `str.__str__` refuses anything that is not really a `str`. A mock object is not article text either.

> 2. ~~**The tested-versions claim will go stale, and nothing re-checks it.** Item 7 says every row was checked on CPython 3.10–3.14 (`spec.md:67-68`), while item 6 allows any version from 3.10 up. Python 3.15 is due around now and likely ships a newer Unicode version. There's no test file in the directory, so the claim is only as good as the last manual run. Turning the tables into a pytest or doctest would let you re-check it and extend the range.~~

**Note:** I rejected this. Item 7 claims only the versions the rows were actually run on (CPython 3.10.16, 3.11.15, 3.12.8, 3.13.1, and 3.14.8, all 60 rows passing), and it already says other Unicode versions may differ. On 21 October 2026 only a beta of Python 3.15 was available. A test file would also put a third file in this directory, and the assignment asks for exactly `spec.md` and `llm-review.md`.

> 3. **It's unclear which parts are normative.** The intro says sections 1–2 are normative and 3–7 follow from them (`spec.md:14-18`). That leaves two gaps:
>    - Section 1 contains a table: is it normative?
>    - The unnumbered "How to read the tables" section defines the escape convention but has no status.
>
>    One sentence such as "all example tables are informative; prose in 1–2 wins" would settle both.

**Note:** I accepted this and applied it after this review. The introduction now says the prose of sections 1 and 2 and the escape convention are normative, and every table, including the one in section 1, is informative.

> ## Wording fixes
>
> 4. **`spec.md:140`** says "quotes … split". Quotes at word edges are actually dropped, not split. A straight `'` with letters on both sides (`said'hi`) does join.
> 5. **`spec.md:161-163`** says "Any other mark is discarded: one at the start of a word, and one right after a joiner". The colon makes this read as a complete list, but marks after a space or digit are dropped too. Also, by the definition, a word can never start with a mark. Something like "any mark not preceded by a letter or kept mark (after a space, digit, joiner, …)" would be accurate.
> 6. **`spec.md:209-210`** says "A word that starts a sentence with I or İ". The same problem hits every capitalised occurrence: names and all-caps headlines too.

**Note:** I accepted all three and applied them after this review, as wording changes only.

> ## Optional additions
>
> - ~~**Emoji that count as letters:** U+2139 ℹ is category Ll, so `"ℹ️ Note"` gives the key `"\u2139\ufe0f"`. Section 2 handles this consistently, but section 3 tells readers symbols are separators. One table row would pin it down; confirm with `unicodedata.category("\u2139")`.~~

**Note:** I rejected this as irrelevant. Section 2 defines a letter by Unicode category, and U+2139 is category Ll, so the spec already says it is a letter. It is not news vocabulary, and a row for it would pin down nothing new.

> - ~~**Arabic-script Kazakh:** Kazakh is also written in Arabic script, in China and Mongolia. The procedure handles it reasonably (letters, marks for vowel signs, spaces between words), but the spec only mentions Cyrillic and Latin. Either a line in section 7 or a sentence in section 3 would cover it. As with full-width Latin, Arabic presentation forms would stay separate from the base letters.~~

**Note:** I rejected this as irrelevant. The procedure already gives a defined result for Arabic-script text, as the review says, and a sentence about it would not change any return value.

> I'd leave the documented trade-offs alone (digits splitting `5G` into `g`, no stopwords, folded keys shown in the box). They're deliberate and the spec already explains them.

## Code check

As the e-book asks, I also gave the final `spec.md`, and nothing else, to a
fresh session of a different model, Claude Sonnet 5.5, with no shell access,
using the prompt `Write top_words.py implementing spec.md.` Its code returned
the expected output for all 60 rows on CPython 3.10 through 3.14. It also gave
the same result as my reference implementation on 200,000 random inputs built
from the characters the spec cares about. The code is not part of this
submission.
