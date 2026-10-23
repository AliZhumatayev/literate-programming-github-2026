# Review of spec.md

Prompting Claude Opus 5.5, extra high effort, with `Review spec.md` produced
the following output; strikethrough indicates LLM output that I deemed
incorrect or irrelevant, and each **Note** says why. The session could read
files but not run code.

> I reviewed the working copy of `spec.md`, including your uncommitted changes. It's a strong spec: sections 1–2 give a short procedure someone could implement directly, and the tables test the hard cases rather than repeating easy ones. I found no errors in any table row. I'd still raise three design points before you finalise it.
>
> **How I checked the examples.** I wasn't allowed to run shell commands here, so I couldn't execute Python. Instead I worked through all 33 table rows by hand against the Unicode data: character categories, case-folding rules and NFC composition. Every expected output follows from section 2. The hardest rows also hold:
> - **`"\u03b1\u0345\u0301 …"` (`spec.md:87`):** NFC first moves the acute accent before the ypogegrammeni and combines everything into U+1FB4. Case-folding that gives `ά` + `ι` (U+03AC U+03B9). The lone `\u0345x` is dropped as an orphan mark; folding the whole text first would wrongly turn it into the word `ιx`.
> - **`ǰ` / `J\u030c` (`spec.md:86`):** both fold to `j` + caron, and NFC rebuilds U+01F0.
> - **`İ` (`spec.md:85`):** it folds to `i\u0307`, NFC leaves that alone, and `ı` doesn't fold at all.
>
> Running the tables as tests on Python 3.10 and 3.14 would still be worth doing.

**Note:** I ran all 33 rows against a reference implementation on CPython 3.10 through 3.14; all pass.

> ## Findings
>
> ### ~~1. Digits leave junk fragments that also inflate real words (most important)~~
> ~~`spec.md:46-47`, row at `:62`~~
>
> ~~The `5G → g` example makes this look harmless, but numbers with attached letters are common in all three languages:~~
>
> | ~~Input~~ | ~~Output under this spec~~ |
> | --- | --- |
> | ~~`"в 1990-х годах"`~~ | ~~`х` is counted~~ |
> | ~~`"3-я неделя"`~~ | ~~`я` is counted, which is also the real word "я" ("I")~~ |
> | ~~`"2-ші орын"`, `"сағат 10-да"`~~ | ~~`ші` and `да` are counted; `да` is also a real Kazakh word~~ |
> | ~~`"the 1990s"`, `"21st"`, `"5pm"`~~ | ~~`s`, `st`, `pm` are counted~~ |
>
> ~~So these aren't just extra noise entries in the box: they add to the counts of genuine words like `я` and `да`, which goes against the spec's own goal of counting real words.~~
>
> ~~**Suggested fix:** let decimal digits (category `Nd`) be part of a word, discard any word with no letter, and keep the digits in the key. That gives `covid-19`, `5g`, `1990-х`, `2-ші`. Pure numbers still never count, and the `x²` row stays the same because `²` isn't `Nd`. The weaker option is to drop letter runs that touch a digit, but then `covid` is lost. Either way, add one Russian or Kazakh row so the result is visible in the tables.~~

**Note:** Letters-only words are a deliberate decision, pinned by the `5G` row; these fragments are its accepted cost.

> ### ~~2. Case-insensitivity breaks for Kazakh written in Latin script~~
> ~~`spec.md:77`, `:85`~~
>
> ~~The 2021 Kazakh Latin alphabet pairs `İ/i` and `I/ı` the way Turkish does. With plain `casefold()`, the same word splits depending on capitalisation:~~
> - ~~`"İs is"` gives `[("is", 1), ("i\u0307s", 1)]`.~~
> - ~~A capital `I` (which means `ы`) folds to `i`, never to `ı`.~~
>
> ~~So a word at the start of a sentence never merges with its lowercase form, and the box would show `i̇` with a stray combining dot. Avoiding locale-specific folding is a reasonable choice, but the spec should say what it costs. Either add "Kazakh in Latin script" to the out-of-scope line, or add one sentence to section 4 saying capitalised Latin-script Kazakh words won't merge.~~

**Note:** Section 4 already says folding is not locale-aware, and the `İ` row shows the result; the longer explanation was cut to keep the spec short.

> ### ~~3. Breaking ties by code point works against Kazakh letters (optional)~~
> ~~`spec.md:104-106`~~
>
> ~~Sorting ties by code point puts every Latin word before every Cyrillic one. It also puts all Kazakh-specific letters (`ә ғ қ ң ө ұ ү һ і`) after `я`. In a short article, where most words appear once or twice, words starting with `қ` (very common in Kazakh, as in `Қазақстан` and `қала`) will lose most ties. Breaking ties by first appearance would be just as deterministic. If you keep code-point order, one clause explaining why you rejected first appearance would help, since the spec currently rules it out without a reason.~~

**Note:** Code-point order is deliberate; the section 6 row ending in `apple` rules out first appearance and alphabetical order.

> ## Minor points
> - ~~**`spec.md:31-32`:** `str.__str__(text)` is a clever way to turn a `str` subclass into a plain `str`, but the spec doesn't say why it's there, and no row tests it (a table can't show a subclass). A sentence such as "`str` subclasses are treated as their plain `str` value; overridden methods are ignored" would make the purpose clear.~~
> - ~~**`spec.md:17` vs `:21-22`:** "deterministic" is only true for a given interpreter. Characters added in newer Unicode versions are letters on Python 3.14 but separators on 3.10. Something like "deterministic for a given Python version" would be accurate.~~

**Note:** Step 1 already defines the behavior exactly, and the next bullet of section 1 ties results to the interpreter's Unicode database.

> ## What works well
> - The "Decision" column explains the reason for each row, so nobody has to guess at intent.
> - Edge cases that most specs miss are covered on purpose: orphan marks, joiners followed by marks, all three mark categories, lone surrogates, NFC both before and after folding, and folding each word rather than the whole text.
> - The claim that every example holds on CPython 3.10–3.14 is safe. Every character used is old and stable in Unicode, and case-folding and NFC don't change between those versions.
>
> ~~`llm-review.md` shows as deleted in your working tree. Do you want me to save this review there?~~

**Note:** This file is that review.
