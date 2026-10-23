# LLM review of `spec.md`

## Last review

Prompting **Claude Opus 5.5** (`claude-opus-5-5`, Anthropic, used through
Claude Code) with `Review spec.md`, with the final 120-line `spec.md` as
context, produced the following output; strikethrough indicates LLM output
that I deemed incorrect or irrelevant.

> I checked every example in `spec.md` against a reference implementation
> written from the seven steps: all 29 table rows, the Rule 6 inputs, and the
> Rule 7 `TypeError` and `str`-subclass cases pass on Python 3.13 (Unicode
> 15.1) and 3.14 (Unicode 16.0). The 16 examples that contain combining or
> invisible characters are stored as `\u` escapes, so an editor cannot change
> them. The spec is 120 lines, the step order is given once, and each example
> tests a different behavior.
>
> **Issues**
>
> 1. Rule 4's single sentence ("`ё`/`е` and look-alikes across scripts
>    differ") is terse. The table makes it clear, but a full sentence would
>    read better.
> 2. ~~Rule 3 says a word is "a maximal run of letters, plus any `-` or `'`",
>    which could be read as allowing a leading hyphen.~~
> 3. ~~Rule 5 uses "`a` through `j`, each 1" instead of a literal list, so it
>    cannot be checked mechanically.~~
>
> **Not issues**
>
> - The step order is unambiguous, and joiners are judged on the text after
>   NFC, `lower()` and the `’` replacement, as the `İ-i` and `й-й` rows show.
> - Ties and the cutoff are deterministic.
> - The `str`-subclass case is fully specified.

Notes on this review:

- Issue 1: kept as is to stay within 100–120 lines; the table shows both cases.
- Issue 2: incorrect. The same sentence requires a letter immediately on both
  sides, and the `'cat' -cat- a--b` row shows edge hyphens separate.
- Issue 3: irrelevant. The shorthand is unambiguous and is defined in the spec.

## Response to the previous review (Claude Opus 5.5, medium effort)

1. Length: cut from 275 to 120 lines. Greek sigma, Devanagari nukta,
   T + U+0308, Latin Extended Additional and fullwidth sorting, and the "only
   letter in Unicode 15.1" claim are removed. The look-alike joiner list is
   one sentence in Rule 3.
2. Repetition: the step order is given once; tie-breaking is only in Rule 5;
   the joiner rule is stated once; the meta-commentary is removed.
3. Duplicate examples: removed all the listed duplicates and merged related
   rows; Rule 7 now names only `None` and `b"abc"`.
4. Title: no change needed. Line 1 already began with `#` (no BOM), so it
   rendered as a heading.
5. The "different result under another reading" claim is removed.
6. The floor is raised to Python 3.13, and the spec says which versions the
   examples were checked on.
