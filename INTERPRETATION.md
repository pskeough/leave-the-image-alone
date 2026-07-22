# Interpretation

This document reasons from the verified numbers in [RESULTS.md](RESULTS.md) to claims, ties each
to the literature ([../docs/REFERENCES.md](../docs/REFERENCES.md)), and states what the pilot can
and cannot support.

## 1. The central finding and why it is not trivial

For receipts that are already legible to a human, the **unaltered image is the best input** to a
vision-LLM transcriber, and the three encoding manipulations we tested all made transcription
worse, in the order downscale < upscale < binarize. The naive expectation imported from decades of
classic-OCR practice is the opposite: binarization and resolution normalization are standard
pre-OCR cleanup. Our result confirms the regime split that the recent VLM literature implies but
rarely measures directly with gold-text CER: deep multimodal recognizers do not want
classic-OCR cleanup (Tesseract's own LSTM prefers grayscale over binarized input; Usama et al.
2025 find VLMs are hurt by geometric/structural corruption far more than by the kind of
photometric normalization classic engines like).

## 2. Claim-by-claim

**Binarization is actively and robustly harmful (H3).** CER rose +1.16 (BCa CI [0.51, 2.39],
d_z 0.47, Wilcoxon p = 0.004), and the sign held for all four models. Global Otsu thresholding on
photographed thermal receipts — which carry shadows, gradients, and faint low-contrast print —
plausibly erases or fragments glyphs, and the diagnostic supports two distinct downstream failure
modes: gemini drops content under binarize (output shrinks 125→96 chars, deletion errors), while
mistral and gpt-4o-mini emit far more (157→357, 221→256, insertion/hallucination errors). This
matches the DIBCO-era observation that binarization quality is itself a bottleneck (Otsu 1979 is a
baseline precisely because it fails on uneven illumination), now inherited by VLMs that were never
trained to prefer bilevel input.

**Upscaling does not help and tends to hurt (H2 refuted).** This is the most informative negative
result. The TextZoom super-resolution literature (Wang et al. 2020) shows SR helps recognition
*when the text is too small to resolve*. CORD receipts are already legible at native resolution,
so there is no small-text deficit for 2× Lanczos upscaling to recover; the larger image only
enlarges the failure surface, and the harm concentrates in the two models whose output inflates
most (gpt-4o-mini +0.52, mistral +1.10). The actionable reading is conditional: resolution
interventions should be expected to help only when native effective resolution is below the
glyph-legibility threshold, not as a blanket "encoder" step.

**Downscaling is weakly and consistently harmful (H1).** The sign was positive for all four models,
but the pooled effect is small (CER +0.078) with a CI spanning zero and a non-significant Wilcoxon
(p = 0.19). The natural reading is that halving resolution on these receipts mostly stays above the
legibility threshold, so degradation is real but mild. H1 functioned as intended as a sanity
direction-check; it did not produce the clean dose-response a fuller resolution sweep would.

**Effects generalize across vendors (H4).** All twelve model × condition contrasts share the same
sign. The conclusion "raw is best, binarize worst" is not an artifact of one model family; it held
for Google, Alibaba (Qwen), OpenAI, and Mistral checkpoints simultaneously.

**Degradation amplifies non-determinism — a second, methodological finding.** Raw transcription was
near-reproducible across identical temperature-0 repeats (CER SD 0.05), but instability rose 1–2
orders of magnitude under degradation (binarize SD 1.06, larger than the mean effect). Hosted-API
non-determinism (Atil et al. 2024) is not constant; it is *input-dependent*, exploding when the
image is ambiguous. Any benchmark that reports a single deterministic pass on degraded inputs is
reporting one draw from a wide distribution.

## 2b. Confound-free corroboration sharpens the picture

Scoring against CORD's structured fields (`gt_parse`), which is immune to the header-coverage
confound because it checks only specific gold amounts, both confirms and refines the edit-distance
result. Binarization again collapses accuracy (field recall 0.95 → 0.67, total hit-rate 0.96 → 0.66,
both p < 0.003), and upscaling again hurts modestly (−0.033, p = 0.023). The refinement concerns
downscaling: on actual receipt content, halving resolution caused **no** measurable harm
(+0.009 field recall, p = 0.67), which means the small CER cost of downscaling was largely the
header/formatting noise that the field metric removes. The honest reading is that the two genuinely
harmful levers are upscaling and, overwhelmingly, binarization, while downscaling to one half stayed
above the legibility threshold for these receipts. That two independent ground-truth constructions
(a full-text edit distance and a structured field oracle) agree on the ranking and on the two
significant harms is the strongest internal-consistency evidence the pilot offers.

The judge calibration completed without human labels by adopting the structured-gold oracle as an
objective reference, and the resulting agreement (Cohen's κ = 0.70, 85 percent) sits at the
human-level mark reported for strong judges (Zheng et al. 2023). The judge is therefore admissible
for the pilot, with the caveat that an objective field oracle certifies content equivalence rather
than the full semantic nuance a human adjudicator would supply.

## 3. The practical "tool" implication

The original motivation was a sellable encoder ("our method does this better"). The honest pilot
implication inverts the pitch: for legible document images, the highest-ROI "encoder" is a
**pass-through**, and a tool that defaults to binarization would *degrade* downstream VLM accuracy.
A defensible encoder would be *conditional* — intervene (e.g. upscale) only when measured input
resolution falls below a legibility threshold, and never binarize for VLM consumption. The value
proposition is a gating policy, not a fixed transform.

## 4. What this pilot cannot claim (threats to validity)

1. **Absolute error is not recognition error.** CORD gold covers itemized fields, not
   header/address/footer, so "transcribe all text" outputs are penalized for correctly read header
   content (D11). Raw CER ≈ 0.30 is largely this offset. We therefore rely on **paired contrasts**,
   which difference out a per-image offset that is *approximately* condition-invariant. The residual
   risk is interaction: binarize could drop the faint header and spuriously *lower* insertion
   penalty, which would bias *against* H3 — yet H3 was still strongly positive, so this confound, if
   present, only makes the binarize harm a conservative estimate.
2. **Small, single-domain sample.** n = 30, receipts only, printed Latin script. No claim extends
   to handwriting, scene text, dense documents, or non-Latin scripts. This is a feasibility +
   effect-size pilot, not a powered confirmatory study (Leon et al. 2011); p-values are exploratory.
3. **Heavy-tailed effects.** Several CIs are very wide (e.g. mistral binarize) because catastrophic
   failures create heavy tails. Medians/Wilcoxon are more trustworthy than means here, and we report
   both.
4. **Provider opacity.** Upscaling produced very large images; server-side resizing/retiling is
   undocumented and may contribute to the upscale harm. We logged returned model routes but cannot
   fully attribute provider-side handling.
5. **No executed judge calibration.** The semantic-equivalence judge (stage 05) requires
   human-labeled pairs; those were not produced in this run, so no judge-adjusted numbers are
   reported. The deterministic CER/WER results stand on their own.

## 5. What a full study should do next

- Obtain **full-page gold** (hand-transcribed or header-inclusive) so absolute CER is meaningful
  and the D11 confound is removed.
- Add **regimes**: handwriting (IAM), scene text, dense multi-column documents, low-native-resolution
  inputs (where H2 should finally show SR benefit).
- Replace aggregate-then-pair with a **mixed-effects model** (CER ~ condition + (1|image) +
  (1|model), repeats nested) to use the data efficiently and model the variance structure that §4
  revealed.
- Add a **resolution sweep** (0.25×–4×) to map the dose-response and locate the legibility threshold
  the conditional-encoder policy needs.
- Increase **k** under degraded conditions specifically, since that is where variance lives.

## 6. One-paragraph synthesis

Across 1,440 transcriptions of 30 receipts by four vision-LLMs, the raw image was the best input;
downscaling mildly degraded accuracy, 2× upscaling degraded it more, and Otsu binarization degraded
it most, with the ranking holding for every model and every normalization variant. Image
degradation also sharply increased run-to-run instability, showing that hosted-VLM non-determinism
is input-dependent. The classic-OCR instinct to binarize and normalize before recognition is
counterproductive for modern VLMs; the right preprocessing default is to leave a legible image
alone and intervene only when measured resolution is too low to read.
