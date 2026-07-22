# Results

All numbers below are produced by `src/04_score.py` / `06_analyze.py` and **independently
re-derived** by `src/07_verify_stats.py` (5/5 checks pass; see
[STATISTICAL_SUPPLEMENT.md](STATISTICAL_SUPPLEMENT.md) and `verification.json`). Unit of
analysis = image (n = 30); each cell is the mean of k = 3 repeats. Primary normalization:
NFC + lowercase + whitespace-collapsed (newlines → spaces) + punctuation retained.

Design executed: **30 receipts × 4 conditions × 4 models × 3 repeats = 1,440 VLM calls, 0 errors.**
Models: `google/gemini-2.5-flash-lite`, `qwen/qwen3-vl-8b-instruct`, `openai/gpt-4o-mini`,
`mistralai/mistral-small-3.2-24b-instruct` (all verified image-capable on OpenRouter 2026-06-25).

---

## 1. Headline: every encoding made transcription worse than raw

Pooled across the 4 models, per-image cell means (primary normalization):

| Condition | Mean CER | Mean WER |
|---|---|---|
| **raw** | **0.297** | **0.457** |
| downscale 0.5× | 0.374 | 0.501 |
| upscale 2× | 0.728 | 1.499 |
| binarize | 1.460 | 2.924 |

Raw is the best input. Error grows monotonically: downscale < upscale < binarize.
(Absolute CER is inflated by gold-text coverage — see §5 — so the **paired differences** below,
not these levels, are the inferential quantity.)

Figure: `fig_cer_by_condition.png`.

---

## 2. Paired contrasts vs raw (the inferential result)

`mean_diff = condition − raw` (positive = worse). Pooled over models (ALL_MODELS), n = 30 images,
bootstrap BCa 95% CI, Wilcoxon signed-rank p (exploratory), Cohen's d_z.

### CER
| Contrast | Δ mean | 95% CI (BCa) | d_z | Wilcoxon p |
|---|---|---|---|---|
| downscale − raw | +0.078 | [−0.031, 0.383] | 0.163 | 0.191 |
| upscale − raw | +0.431 | [0.077, 1.631] | 0.257 | 0.023 |
| binarize − raw | +1.163 | [0.512, 2.390] | 0.468 | 0.004 |

### WER
| Contrast | Δ mean | 95% CI (BCa) | d_z | Wilcoxon p |
|---|---|---|---|---|
| downscale − raw | +0.044 | [−0.205, 0.286] | 0.065 | 0.084 |
| upscale − raw | +1.042 | [0.055, 4.406] | 0.233 | 0.031 |
| binarize − raw | +2.467 | [0.723, 6.442] | 0.353 | 0.002 |

**upscale** and **binarize** raise error with CIs excluding zero on both metrics. **downscale**
is directionally worse but its CI includes zero when pooled.

Figure: `forest_cer.png` (per-model paired differences with CIs).

---

## 3. Generalization across models (H4): perfect sign-consistency

All 12 model × condition contrasts have the **same sign** — every transform increased CER for
every model:

| Model | downscale | upscale | binarize |
|---|---|---|---|
| gemini-2.5-flash-lite | +0.053 | +0.050 | +0.207 |
| qwen3-vl-8b-instruct | +0.095 | +0.054 | +0.698 |
| gpt-4o-mini | +0.145 | +0.517 | +0.915 |
| mistral-small-3.2-24b | +0.018 | +1.102 | +2.833 |

Per-model exploratory Wilcoxon (Holm-corrected across the 12 tests): **binarize** is the most
consistent harm (gemini p_holm = 0.039; qwen and gpt marginal; mistral heavy-tailed). upscale
harm concentrates in gpt-4o-mini and mistral; downscale is small everywhere.

---

## 4. Image degradation explodes output instability (non-determinism)

Mean within-cell SD of CER across the k = 3 identical repeats (temperature 0):

| Condition | CER SD | WER SD |
|---|---|---|
| raw | 0.051 | 0.042 |
| downscale 0.5× | 0.152 | 0.201 |
| upscale 2× | 0.434 | 1.099 |
| binarize | 1.060 | 2.537 |

Raw transcription is highly reproducible (SD ≈ 0.05). Degradation raises run-to-run variance by
**1–2 orders of magnitude**; for binarize the instability (1.06) exceeds the mean effect itself.
This is consistent with hosted-API non-determinism (Atil et al. 2024) being *amplified* by
ambiguous inputs. Figure: `fig_nondeterminism.png`.

---

## 5. Mechanism (diagnostic): degraded inputs induce more erroneous text

Mean output length (characters; gold mean = 118):

| Condition | gemini | qwen | gpt-4o-mini | mistral | all |
|---|---|---|---|---|---|
| raw | 125 | 130 | 221 | 157 | 158 |
| downscale | 127 | 130 | 239 | 153 | 162 |
| upscale | 127 | 135 | 269 | 231 | 191 |
| binarize | 96 | 171 | 256 | 357 | 220 |

Under binarize/upscale, gpt-4o-mini and mistral emit substantially **more** text (hallucinated or
misread content → insertion errors), while gemini emits **less** under binarize (dropped content →
deletion errors). Both failure modes raise edit distance. Otsu thresholding on photographed
thermal receipts (shadows, low contrast) plausibly destroys faint glyphs; 2× upscaling adds no
legibility to already-readable receipts and only enlarges the failure surface. Only 1/1,440
outputs was empty (an upscale case).

---

## 6. Robustness to normalization (sensitivity)

Pooled mean CER under the three frozen normalization variants:

| Condition | case_sensitive | primary | punct_stripped |
|---|---|---|---|
| raw | 0.297 | 0.297 | 0.210 |
| downscale | 0.376 | 0.374 | 0.325 |
| upscale | 0.728 | 0.728 | 0.317 |
| binarize | 1.464 | 1.460 | 0.635 |

The condition ordering (raw < downscale < upscale < binarize) is **identical under all three**.
Stripping punctuation lowers absolute CER (receipts are punctuation/number-dense) but does not
change the ranking or the qualitative conclusion.

---

## 6b. Confound-free corroboration: structured field accuracy (CORD `gt_parse`)

To remove the gold-coverage confound (§5, D11) without any human labeling, we also score against
CORD's **structured** annotations — the line-item prices and receipt total in `gt_parse`, the
standard receipt key-information-extraction ground truth. `field_recall` = fraction of gold numeric
amounts present in the output (thousands-separators normalized); `total_hit` = was the total
transcribed. Both are **higher = better** and immune to header/footer insertions.

| Condition | field_recall | total_hit |
|---|---|---|
| raw | 0.945 | 0.964 |
| downscale 0.5× | 0.954 | 0.967 |
| upscale 2× | 0.913 | 0.939 |
| binarize | 0.667 | 0.656 |

Paired contrasts vs raw (negative = worse), n = 30:

| Metric | Contrast | Δ mean | 95% CI (BCa) | d_z | Wilcoxon p |
|---|---|---|---|---|---|
| field_recall | downscale − raw | +0.009 | [−0.033, 0.056] | 0.072 | 0.666 |
| field_recall | upscale − raw | −0.033 | [−0.065, −0.011] | −0.424 | 0.023 |
| field_recall | binarize − raw | −0.278 | [−0.432, −0.145] | −0.684 | 0.002 |
| total_hit | binarize − raw | −0.308 | [−0.479, −0.164] | −0.686 | 0.002 |

This **refines** the CER story: on the actual receipt content, **downscale 0.5× causes no
measurable harm** (the CER downscale signal was largely header/formatting noise), **upscale modestly
hurts**, and **binarize is catastrophic** — a 28–31 percentage-point drop in correctly reading the
amounts and the total. The confound-free metric and the deterministic CER metric agree on the
ranking and on the binarize/upscale conclusions. Figure: `fig_field_accuracy.png`.

## 6c. Automated LLM-judge calibration (no human labels)

The semantic-equivalence judge was calibrated against a **deterministic structured-gold oracle**
(a pair is equivalent iff `field_recall == 1.0`) on a balanced 40-pair sample, requiring no human
action. The judge (`gemini-2.5-flash`, distinct from the scored set) reached **85.0% agreement** and
**Cohen's κ = 0.70 (substantial; Landis–Koch)**, clearing the pre-set admissibility threshold of
0.61. The value matches the human-level agreement reported for strong LLM judges (Zheng et al. 2023,
~85%). Per D5/D12 the judge is therefore admissible for this pilot; the calibration is reported as
judge-vs-objective-oracle, not judge-vs-human.

## 7. Hypothesis verdicts

| # | Prediction | Verdict |
|---|---|---|
| **H1** | downscale increases error | **Not supported on content** — sign-consistent but tiny on CER (CI includes 0); on the confound-free field metric, downscale 0.5× caused no harm (+0.009, p = 0.67). Halving resolution stayed above the legibility threshold. |
| **H2** | upscale *reduces* error | **Refuted** — upscale *increased* error (CER Δ +0.43, p = 0.023; WER Δ +1.04, p = 0.031). |
| **H3** | binarize does not help (may hurt) | **Strongly supported, and exceeded** — large, robust harm (CER Δ +1.16, p = 0.004). |
| **H4** | effect sign consistent across vendors | **Supported** — all 12 contrasts same sign. |

Overarching result: **for already-legible document images, the raw image is the best VLM input;
common "encoding/cleanup" transforms degrade transcription, binarization catastrophically.**
This is the opposite of the classic-OCR preprocessing playbook.

*See [INTERPRETATION.md](INTERPRETATION.md) for discussion and [STATISTICAL_SUPPLEMENT.md] for
the per-claim verification ledger.*
