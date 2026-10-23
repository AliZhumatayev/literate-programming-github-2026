# LLM review of spec.md

## Setup

- LLM: Claude Opus 5.5 (`claude-opus-5-5`), run as a Claude Code subagent. The agent used its default effort level.
- Prompt:

  ```
  Review /Users/iomiras/uni/literate-programming-github-2026/m_shaltayev/spec.md. Read-only: no edit, no write in repo. Find errors, contradictions, underspecified behavior, ambiguity that lets 2 valid impls give different output. Also flag needless length or repetition. Each finding: quote spec, say problem, suggest fix. Group: Must fix / Should fix / Minor. Reply w/ review text only.
  ```

- Strikethrough marks output that I judged incorrect or irrelevant.

## Output of the last review

Review of /Users/iomiras/uni/literate-programming-github-2026/m_shaltayev/spec.md

How I checked the spec

I wrote a reference implementation and a harness that reads every table row from the spec:

- /private/tmp/claude-501/-Users-iomiras-uni-literate-programming-github-2026/273601c4-0575-4ba6-a3a0-4726e9fe39e2/scratchpad/ref/top_words.py
- /private/tmp/claude-501/-Users-iomiras-uni-literate-programming-github-2026/273601c4-0575-4ba6-a3a0-4726e9fe39e2/scratchpad/ref/check.py

Results:

- All 45 example rows and the realistic-text case pass on Python 3.9.6 (Unicode 13), 3.11 (Unicode 14), 3.13 (Unicode 15.1), and 3.14 (Unicode 16).
- The spec file is already NFC.
- The Unicode-version clause is safe. I compared every character that Unicode 13 assigns, using Python 3.9 and Python 3.14. Only U+1734 and U+171E change, and they change from Mn to Mc. The spec counts both categories as marks, so the output does not change.
- The examples contain no errors. The algorithm in rule 5 is complete. I found only one gap that lets two valid implementations give different output (Should fix, item 1).

MUST FIX

1. "The limit of 10 cuts `россии`, the last word in code point order."
   Problem: This statement is false. `столица` (с, U+0441) comes after `россии` (р, U+0440) in code point order. The limit cuts `россии` because it is the last word among the words with count 1. A reader can think that the tie-break applies to all words, not only within one count.
   Fix: "The limit of 10 cuts `россии`. It is the last of the words with count 1 in code point order."

SHOULD FIX

1. "A `str` subclass is accepted. The function converts it to an exact `str` first"
   Problem: The spec does not say how to convert. `str(text)` calls the `__str__` method of the subclass, and `str.__str__(text)` does not. I tested a subclass with `__str__` that returns 'zzz'. `str(s)` gives 'zzz', and `str.__str__(s)` gives the original text. Two valid implementations then give different output. Also, `unicodedata.normalize` returns the subclass object unchanged on Python 3.9 to 3.13 when the text is already NFC. Thus an implementation that skips the conversion still works, except for the overridden methods.
   Fix: Write "The function converts it with `str.__str__(text)`." As an alternative, write "Behavior is not specified for subclasses that override `str` methods."

2. "6. A mark that does not come right after a letter or a mark is not part of a word."
   Problem: The rule suggests the converse: a mark right after a mark is part of a word. That is false for `3́́d`, where the second mark comes after a mark that is not in a word. Rule 5 already decides all word boundaries, so rule 6 adds nothing and can mislead a reader.
   Fix: Delete rule 6. Alternatively, write "A mark is part of a word only when rule 5 reaches it."

3. "7. Every other character separates words."
   Problem: The rule does not say what "other" refers to. Rule 5 already says that a word "ends at the first character that does not continue it". Rules 6 and 7 thus give a second, weaker description of the same boundary.
   Fix: Merge rules 6 and 7 into one sentence that only explains: "All characters that rule 5 does not include are separators, for example digits, variation selectors, `Cf` characters, and `Me` marks."

4. Section name "Normalization" and the sentence "Do not normalize again after `str.lower()`."
   Problem: The spec uses "normalize" with two meanings: Unicode NFC (rule 1) and the word-key steps (the section). The sentence "do not normalize again" can mean "do not apply NFC" or "do not apply this section again".
   Fix: Rename the section to "Word key". Change the sentence to "Do not apply NFC again after `str.lower()`."

5. The row with `кто́-то` contains a literal U+0301 character.
   Problem: The other rows write marks as `́`. Here the mark is invisible, so a copy-and-paste operation or an editor can remove it, and the row then becomes the same as `кто-то`. The invisible character also breaks the alignment of the table column.
   Fix: Write the input as `кто́-то` and the output as `[("кто́-то", 1)]`.

6. "Normalize the text to Unicode form NFC ... Do this first, before any other step."
   Problem: The type check and the conversion of a `str` subclass (Interface section) must come before NFC. The word "first" contradicts that order.
   Fix: Write "Do this before you find the words."

MINOR

1. "`word` is the lowercase form of the word"
   Problem: The word key also replaces `’` with `'`.
   Fix: Write "`word` is the normalized form of the word".

2. "the pairs with the smallest `(-count, word)`, in that order."
   Problem: "In that order" is vague.
   Fix: Write "sorted by `(-count, word)` in ascending order. Keep the first 10."

3. Example decision text: "Any non-letter separates words." (row `cat;dog/cat\x7cdog`).
   Problem: This contradicts rule 5, because marks and joiners are non-letters that can continue a word.
   Fix: Write "Punctuation and symbols separate words."

4. "The input is the text between the backticks."
   Problem: The "empty string" row has no backticks, and the realistic input is in a code block.
   Fix: Add "or as stated in words, or in a code block".

5. "invisible format characters (Unicode category `Cf`)"
   Problem: Some `Cf` characters are visible, for example U+0600 and U+06DD.
   Fix: Write "format characters (Unicode category `Cf`)".

6. Rule 8: "Latin, Cyrillic, and Kazakh text is the target."
   Problem: Kazakh is a language that uses the Cyrillic script. The sentence mixes scripts and languages, and the Purpose section names languages (English, Russian, Kazakh).
   Fix: Write "English, Russian, and Kazakh text is the target."

7. "Call `str.lower()` on the whole word string. Do not lowercase the whole text, and do not lowercase each character alone."
   Problem: In Python, these three methods give different results only for the Greek final sigma. The tests do not check Greek, so the rule has no tested effect.
   Fix: Keep the rule and add the reason in a few words ("this affects the Greek final sigma"). Alternatively, cut the rule to "Call `str.lower()` on each word."

8. Design risks to confirm. The spec marks these as intended, but they can give poor results for news text:
   - A soft hyphen (U+00AD) from a CMS splits a word.
   - `кто‑то` with U+2011 gives `кто` and `то`.
   - `Москва́` and `Москва` count as two different words.

   If you keep these decisions, no change is necessary. Otherwise, remove `Cf` characters inside a word, and treat U+2010 and U+2011 as joiners.

9. The examples do not test some parts of the spec. Add rows for these cases:
   - `TypeError` for `None` and `bytes`.
   - A `str` subclass.
   - A joiner after a mark with no letter after it, for example `кто́-`.
   - A variation selector outside U+FE00 to U+FE0F.

10. Length and repetition:
    - The "Do not ..." bullets in the Normalization section (accents, `ё`/`й`, stemming, stop words, language detection, Latin `i` and Cyrillic `і`) all follow from "apply only these steps". Several of them also repeat example rows. Change them to one sentence, for example: "Apply no other change: no accent removal, letter mapping, stemming, stop-word removal, or language detection."
    - "This is intended." occurs twice. Keep it one time or delete it.
    - The output bullet "If the text has no words (empty, only spaces, only digits, only punctuation), the list is empty" repeats three example rows. Cut it to "If the text has no words, the list is empty."
    - Some example rows test the same behavior: `cat_dog`, `cat…dog`, `кот—пёс`, and `cat;dog/cat\x7cdog` all show that punctuation separates words. `a--b` and `a-'b` overlap. `кто-то` repeats `well-known`. You can merge them, unless each row is a separate test that you want to keep.
    - The Performance section cannot be tested. A plain sort already satisfies the bound. You can delete the section or cut it to one sentence.

11. The table row `a b c d e f g h i j k l` and the `кто́` row are not aligned with the other rows in the Markdown source. This is cosmetic only.
