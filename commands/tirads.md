---
description: Score a thyroid nodule with ACR TI-RADS — points, TR level, and FNA/follow-up thresholds
argument-hint: [nodule description, report text, or file path]
---

# TI-RADS Scoring

**Input**: $ARGUMENTS

Delegate to the `tirads-scorer` agent. Pass the input through verbatim — do not paraphrase sonographic findings before the agent sees them, since wording carries the feature distinctions ("lobulated" vs "ill-defined", "punctate echogenic foci" vs "large comet-tail artifacts").

## Input Modes

**Free text** — a dictated or typed description of the nodule. Pass as-is.

**File path** — read the file, then pass the relevant nodule paragraph(s). If the report describes several nodules, score each one separately; ACR TI-RADS levels do not aggregate across nodules.

**No arguments** — ask for the five feature categories and the maximum diameter:

1. Composition (cystic / spongiform / mixed / solid)
2. Echogenicity (anechoic / hyper- or isoechoic / hypoechoic / very hypoechoic)
3. Shape (wider-than-tall / taller-than-wide, on transverse)
4. Margin (smooth / ill-defined / lobulated or irregular / extra-thyroidal extension)
5. Echogenic foci (none or large comet-tail / macrocalcifications / peripheral / punctate — all that apply)

Plus the maximum diameter in cm, which drives the FNA and follow-up thresholds.

## Rules

- A missing feature is reported as missing, never assumed. If the gap changes the recommendation, say so and ask for it.
- Report the point arithmetic alongside the level so it can be checked.
- Size is not optional — a TR level alone yields no recommendation.

Output follows the format in `agents/tirads-scorer.md`, ending with the clinical-correlation note.
