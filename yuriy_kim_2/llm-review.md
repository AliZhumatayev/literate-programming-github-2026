# LLM review of `spec.md`

Prompting **Claude Opus 5.5** (`claude-opus-5-5`, Anthropic, used through
Claude Code) with `Review spec.md`, with the revised `spec.md` as context,
produced the following output; strikethrough indicates LLM output that I deemed
incorrect or irrelevant.

> I checked every example in `spec.md` against a reference implementation
> written from the seven steps. All 43 table rows, the two `TypeError` cases and
> the `str` subclass case pass on Python 3.13 (Unicode 15.1) and 3.14 (Unicode
> 16.0). The spec is now 147 lines (down from 275), the step order is given once, and each
> example tests a different behavior.
>
> **Issues**
>
> 1. The Rule 1 row `Қала`, newline, `қала`, tab, `ҚАЛА` is written in prose.
>    The spec says examples with invisible characters are written as Python
>    literals with escapes, and newline and tab are invisible. Write it as
>    `"Қала\nқала\tҚАЛА"`.
> 2. Rule 7 accepts `str` subclasses but doesn't say whether a subclass's
>    overridden methods (for example a custom `lower()`) are used. The example
>    `class S(str): pass` overrides nothing, so it doesn't decide this. State
>    that the text is treated as a plain `str`, for example by converting it
>    with `str(text)` first.
> 3. ~~Rule 2 says everything that is not a letter separates words, but Rule 3
>    then lets `-` and `'` join words. Rule 2 should mention the exception.~~
>
> **Not issues**
>
> - The step order is unambiguous, and the joiner check uses the text after
>   NFC, `lower()` and the `’` replacement, as the `İ-i` and `й-й` rows show.
> - Ties and the cutoff are deterministic.
> - "Python 3.13 or later" matches the versions the examples were checked on.

## Notes on the feedback

- Issue 1: fixed. The row now uses a Python literal.
- Issue 2: fixed. Rule 7 now says the input is processed as `str(text)`, so
  overridden methods are not used.
- Issue 3: not changed. Rule 3 defines a word explicitly as a run of letters
  plus joiners, and Step 5 already points to both rules.
