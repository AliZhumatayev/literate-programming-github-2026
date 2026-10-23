# LLM Review

Prompting Claude Opus 5.5, high effort, with `Review spec.md` produced the following output;
strikethrough indicates LLM output that I deemed incorrect or irrelevant.

---

# Review: Top Words — Specification

**Reviewed:** 2026-10-23

## Summary

The spec is in good shape. I wrote a reference implementation from the rules alone, and it reproduced all 39 example rows plus both `TypeError` cases. Each example's "different plausible reading" column also names a real alternative interpretation.

I also checked the D1 claim that every returned word is already NFC and made only of letters and joiners. It holds for every code point Python considers a letter.

The problems below are about presentation and a few gaps, not contradictions.

## Findings

### 1. Invisible characters in the example inputs (most important)

Several rows depend on characters you can't see:

| Line | Row | Hidden character(s) |
|---|---|---|
| 51, 66 | decomposed `café` | U+0301 combining acute |
| 60, 67 | `моло\u0301ко` | U+0301 combining acute |
| 68 | `ѝ` | U+0300 combining grave |
| 69 | `trending` row | U+00AD soft hyphen, U+200B zero-width space |
| 104 | `pre‐post pre‑post` | U+2010 and U+2011, which look exactly like `-` |
| 142 | expected-wrong `i\u0307stanbul` | U+0307 combining dot above |
| 157 | `"cat сat"` | Cyrillic `с` U+0441, identical to Latin `c` |

Many editors, CMSs and copy-paste paths normalize to NFC or strip soft hyphens. When that happens the tests still pass but no longer test anything: `"café café"` becomes two identical precomposed words.

**Fix:** the inputs are already Python expressions, so write them with escapes:

```python
"cafe\u0301 caf\u00e9"
"trend\u00ading trend\u200bing trending"
"pre\u2010post pre\u2011post pre-post"
"cat \u0441at"
```

### 2. Line 8 is garbled

"(so `U+0301` is one character)" is a bare combining acute sitting on a backtick, so it renders as nothing.

**Suggested wording:** "Inputs are Python expressions: string literals, except the D2 rows. Invisible and look-alike characters are written as `\uXXXX` escapes."

### 3. D10's tie-break has an ambiguity

D10 says "any occurrence of the final form" counts. It doesn't say whether a compound counts as an occurrence of its part.

For `"cat-dog dog cat"` the rules give:

```python
[("cat-dog", 1), ("dog", 1), ("cat", 1)]
```

A reader who treats `cat` inside `cat-dog` as an occurrence would put `cat` before `dog`.

**Fix:** add that row, and say that "occurrence" means a word found by D6.

### 4. No example fixes the D7 order of "find words, then lowercase"

The İstanbul row catches one wrong order. Another wrong order, lowercasing the whole text before splitting, slips through. Greek final-sigma depends on context:

| Input | Spec (split, then lowercase) | Lowercase first |
|---|---|---|
| `"ΟΔΟΣ''Α"` | `[("οδος", 1), ("α", 1)]` | `[("οδοσ", 1), ("α", 1)]` |

This is low priority given the three target languages, but one row would close it.

### 5. The stress-mark handling splits Russian words, by design

- `ми\u0301р` (acute) counts as `мир`.
- `мѝр` (grave) becomes `мѝр` and is counted separately.

Verified: `"ми\u0301р ми\u0300р мир"` gives `[("мир", 2), ("м\u045dр", 1)]`.

The Bulgarian justification doesn't apply to an English/Russian/Kazakh site. If that split is intended, keep it. If not, map `ѝ→и` and `ѐ→е` after NFC.

Separately, add an example for the case where NFC really matters in these languages: `й` or `ё` typed as base letter plus combining mark should become a single letter. For example, `"и\u0306од йод"` should give `[("йод", 2)]`.

### 6. D4 leaves out scripts that don't put spaces between words

D4 warns about Devanagari, Thai and Hebrew because of combining marks. It doesn't mention that Chinese, Japanese, Thai, Lao and Khmer don't separate words with spaces.

- `"我喜欢猫。我喜欢狗。"` gives `[("我喜欢猫", 1), ("我喜欢狗", 1)]`: whole clauses counted as words.
- `isalpha()` is true for `º` and `ª`, so `"1º 2ª"` counts `º` and `ª` as words.

**Fix:** one sentence in D4 would cover both.

### 7. D8 could state its limits up front

The spec uses NFC, not NFKC, and `lower()`, not `casefold()`. So:

- `ｃａｔ ≠ cat` (fullwidth)
- `ﬁnal ≠ final` (ligature)
- `straße ≠ strasse`

All of this is consistent with "exact form", but readers often assume NFKC or casefolding. Listing these next to `ё/е` would prevent bug reports.

### 8. D12 can't be tested as written

A test can't check big-O. A concrete benchmark would be checkable, for example "10 MB of mixed text in under N seconds on the reference machine", plus one input with many distinct words.

Two related points I checked:

- NFC stayed linear on long runs of combining marks: about 0.008 s for 80,000 marks.
- Lone surrogates such as `"a\ud800b"` return normally (`[("a", 1), ("b", 1)]`), so "no other exceptions for any `str`" holds.

## Method

- Reference implementation written from the rules alone (Python 3.13, standard library only).
- All 39 example rows and both D2 rows run against it: all match.
- The D1 final-form claim checked over every Unicode code point that `isalpha()` accepts: `lower()` with `Mn` removed is always NFC and always letters.
- The spec file scanned for `Mn` and `Cf` characters to find the hidden ones listed in finding 1.