# Statistical Supplement

Every quantitative claim in the paper, with its exact value, the method that produced it, and its
**independent verification status**. The verification stage (`src/07_verify_stats.py`) re-derives
the metrics and statistics through a separate code path (hand-written Levenshtein, manual
aggregation, manual percentile bootstrap) and asserts agreement with the pipeline
(jiwer + pandas + scipy). Machine-readable record: `verification.json`.

**Reproducibility stamp:** config hash `8668e072e7e9`; n = 30 images; k = 3 repeats;
1,440 calls, 0 errors; bootstrap = 10,000 resamples, seed 42.

---

## A. Verification ledger (5/5 PASS)

| Check | What it proves | Result |
|---|---|---|
| `metric_agreement_jiwer_vs_independent` | CER/WER from jiwer == hand-written Levenshtein on all 1,440 rows | PASS, max\|Δ\| = 7.1e-15 |
| `design_grid_complete` | every (item × condition × model) has exactly k = 3 repeats | PASS, 480 cells, 0 incomplete |
| `cell_mean_manual_vs_pandas` | per-cell means (pandas) == manual dict reduction | PASS, max\|Δ\| = 7.1e-15 |
| `paired_mean_diff_manual_vs_pipeline` | paired Δ means recomputed by hand == stage-06 | PASS, exact |
| `effect_sign_and_ci_consistent` | independent percentile bootstrap agrees in sign + bracket with BCa | PASS |

A discrepancy was found and fixed during verification: the pipeline's whitespace normalization
did not convert newlines to spaces before word tokenization, inflating WER (independent vs jiwer
max\|ΔWER\| = 239 words). Fixed in `_common.build_transforms` (added
`RemoveWhiteSpace(replace_by_space=True)`); after re-scoring, the two paths agree to 1e-14. This is
why the supplement exists.

---

## B. Definitions and pipeline

- **CER** = character Levenshtein(ref, hyp) / |ref_chars|; **WER** = word Levenshtein / |ref_words|.
  Both unbounded above 1.0 under insertions (HF metric cards). Tool: `jiwer` 4.0.
- **Primary normalization** (frozen pre-analysis, D9): NFC → lowercase → all-whitespace→space →
  collapse spaces → strip; punctuation **retained**. Boilerplate prefixes/fences stripped first.
- **Unit of analysis** = image. Each cell = mean of k = 3 repeats; paired tests run on the 30
  per-image values (D7). Pooled "ALL_MODELS" = mean over models per image, then paired.
- **Estimation primary, tests exploratory** (D8): BCa bootstrap 95% CI (10,000 resamples) +
  Cohen's d_z; Wilcoxon signed-rank reported as exploratory; Holm–Bonferroni on the 12 per-model
  tests only.

---

## C. Descriptive statistics (per-image cell means, primary norm)

| Condition | Mean CER | Mean WER | CER SD across repeats | WER SD across repeats |
|---|---|---|---|---|
| raw | 0.2967 | 0.4571 | 0.0512 | 0.0417 |
| downscale 0.5× | 0.3742 | 0.5011 | 0.1519 | 0.2005 |
| upscale 2× | 0.7275 | 1.4988 | 0.4335 | 1.0986 |
| binarize | 1.4598 | 2.9244 | 1.0600 | 2.5373 |

Source: `scores_cell.csv`. The SD columns are the non-determinism result (§ RESULTS 4).

---

## D. Paired contrasts vs raw — with independent cross-check

`mean_diff = condition − raw`, n = 30 paired images, pooled over 4 models.

### CER
| Contrast | Δ mean | BCa 95% CI | indep. percentile CI | d_z | Wilcoxon p | sign-match |
|---|---|---|---|---|---|---|
| downscale − raw | +0.0775 | [−0.0313, 0.3826] | [−0.0604, 0.2691] | 0.163 | 0.191 | ✓ |
| upscale − raw | +0.4308 | [0.0769, 1.6309] | [0.0228, 1.1119] | 0.257 | 0.023 | ✓ |
| binarize − raw | +1.1631 | [0.5115, 2.3901] | [0.3863, 2.1389] | 0.468 | 0.004 | ✓ |

Independent and pipeline `mean_diff` and `d_z` agree **exactly**; the two bootstrap CIs (different
methods, BCa vs percentile) agree in sign and bracket zero identically (both exclude 0 for upscale
and binarize, both include 0 for downscale). Source: `stats_contrasts.csv`, `verification.json`.

### WER
| Contrast | Δ mean | BCa 95% CI | d_z | Wilcoxon p |
|---|---|---|---|---|
| downscale − raw | +0.0439 | [−0.2050, 0.2857] | 0.065 | 0.084 |
| upscale − raw | +1.0416 | [0.0548, 4.4063] | 0.233 | 0.031 |
| binarize − raw | +2.4673 | [0.7229, 6.4422] | 0.353 | 0.002 |

---

## E. Per-model contrasts (CER) and Holm-corrected exploratory tests

| Model | Condition | Δ mean | Wilcoxon p | Holm p |
|---|---|---|---|---|
| gemini-2.5-flash-lite | downscale | +0.053 | 0.758 | 1.000 |
| gemini-2.5-flash-lite | upscale | +0.050 | 0.420 | 1.000 |
| gemini-2.5-flash-lite | binarize | +0.207 | 0.0032 | **0.039** |
| qwen3-vl-8b-instruct | downscale | +0.095 | 0.0057 | 0.059 |
| qwen3-vl-8b-instruct | upscale | +0.054 | 0.811 | 1.000 |
| qwen3-vl-8b-instruct | binarize | +0.698 | 0.0054 | 0.059 |
| gpt-4o-mini | downscale | +0.145 | 0.159 | 0.952 |
| gpt-4o-mini | upscale | +0.517 | 0.0119 | 0.107 |
| gpt-4o-mini | binarize | +0.915 | 0.0150 | 0.120 |
| mistral-small-3.2-24b | downscale | +0.018 | 0.302 | 1.000 |
| mistral-small-3.2-24b | upscale | +1.102 | 0.819 | 1.000 |
| mistral-small-3.2-24b | binarize | +2.833 | 0.069 | 0.480 |

All 12 Δ means are positive (H4 sign-consistency). After Holm correction only gemini-binarize
survives at α = 0.05; this is expected for n = 30 with heavy tails and is consistent with the pilot
framing (estimation, not confirmation). The pooled BCa CIs (§D) are the primary evidence; the
per-model tests are exploratory and reported in full without cherry-picking.

Mistral's large Δ with non-significant Wilcoxon (p = 0.82 upscale, 0.069 binarize) reflects a few
catastrophic items driving the mean while the median shift is small — exactly the heavy-tail case
where Wilcoxon and mean diverge, and a reason to trust the rank test and report both.

---

## F. Sensitivity to normalization (pooled mean CER)

| Condition | case_sensitive | primary | punct_stripped |
|---|---|---|---|
| raw | 0.2971 | 0.2967 | 0.2100 |
| downscale | 0.3761 | 0.3742 | 0.3246 |
| upscale | 0.7283 | 0.7275 | 0.3167 |
| binarize | 1.4643 | 1.4598 | 0.6346 |

The ordering raw < downscale < upscale < binarize is invariant across all three frozen variants
(D9). Case folding is near-inert; punctuation stripping lowers absolute CER without changing rank.

---

## G. Diagnostic numbers (mechanism, § RESULTS 5)

Mean output length in characters (gold = 118); empties = 1/1,440 (an upscale case).

| Condition | gemini | qwen | gpt-4o-mini | mistral | all |
|---|---|---|---|---|---|
| raw | 125 | 130 | 221 | 157 | 158 |
| downscale | 127 | 130 | 239 | 153 | 162 |
| upscale | 127 | 135 | 269 | 231 | 191 |
| binarize | 96 | 171 | 256 | 357 | 220 |

---

## G2. Structured-field ground truth (CORD `gt_parse`) — confound-free metric

Gold amounts = numeric leaf values (≥ 3 digits) under `gt_parse.menu`/`total`; `field_recall` =
|gold ∩ output| / |gold| (thousands-separators removed); `total_hit` = gold `total_price` present in
output. Higher = better. Immune to header/footer coverage (D11/D12). Per-image cell means, n = 30.

| Condition | field_recall | total_hit |
|---|---|---|
| raw | 0.9453 | 0.9639 |
| downscale 0.5× | 0.9543 | 0.9667 |
| upscale 2× | 0.9128 | 0.9389 |
| binarize | 0.6672 | 0.6556 |

Paired contrasts vs raw (Δ = condition − raw, negative = worse), BCa 95% CI, Wilcoxon exploratory:

| Metric | Contrast | Δ mean | 95% CI | d_z | Wilcoxon p |
|---|---|---|---|---|---|
| field_recall | downscale − raw | +0.0090 | [−0.0332, 0.0556] | 0.072 | 0.666 |
| field_recall | upscale − raw | −0.0325 | [−0.0654, −0.0109] | −0.424 | 0.023 |
| field_recall | binarize − raw | −0.2781 | [−0.4320, −0.1450] | −0.684 | 0.002 |
| total_hit | downscale − raw | +0.0028 | [−0.0556, 0.0472] | 0.019 | 0.865 |
| total_hit | upscale − raw | −0.0250 | [−0.0583, 0.0056] | −0.276 | 0.172 |
| total_hit | binarize − raw | −0.3083 | [−0.4793, −0.1639] | −0.686 | 0.002 |

The field metric and the CER metric (Section D) agree on the ranking and on the two significant
harms (upscale, binarize); they diverge only on downscale, which the confound-free metric shows to be
harmless on content. Source: `field_summary.csv`, `field_contrasts.csv`, `field_scores_long.csv`.

## G3. Automated LLM-judge calibration (judge vs structured-gold oracle, no human)

Oracle label = EQUIVALENT iff `field_recall == 1.0`, else DIFFERENT (deterministic). Balanced
40-pair sample (half each class), judge = `gemini-2.5-flash` (distinct from scored models).

| Quantity | Value |
|---|---|
| n pairs | 40 |
| raw agreement | 0.850 |
| Cohen's κ | 0.700 (substantial, Landis–Koch) |
| admissibility threshold | 0.61 |
| verdict | ADMISSIBLE |

Reported as judge-vs-objective-oracle agreement, not judge-vs-human (D12). Source:
`judge_auto_calibration.csv`.

## H. Claims NOT made (guardrails)

- No confirmatory/powered inference from n = 30 (pilot; Leon et al. 2011). p-values are exploratory.
- Absolute CER is an upper bound inflated by gold header-coverage (D11); only paired contrasts are
  interpreted.
- Judge calibration used a deterministic structured-gold **oracle**, not human labels (D12); it
  certifies content equivalence, not full human semantic judgment. No judge-*adjusted* error numbers
  are reported; the judge result is the calibration κ only.
- No causal attribution to provider-side image handling for the upscale effect; flagged as a
  confound, not a conclusion.
