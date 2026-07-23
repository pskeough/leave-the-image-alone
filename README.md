# Leave the Image Alone
### Image Preprocessing Degrades Vision-LLM Transcription of Legible Documents

A controlled pilot: 30 CORD-v2 receipts × 4 preprocessing conditions × 4 vision-language
models × 3 repeats = 1,440 transcriptions. The classical OCR reflex of binarizing, normalizing,
and upscaling before recognition is counterproductive for legible documents read by modern VLMs.

📄 **Paper:** [`paper/main.pdf`](paper/main.pdf) · Patrick S. Keough · pilot preprint in preparation

## Headline findings

- **Raw wins everywhere:** pooled mean character error rate raw 0.297 < downscale 0.374 <
  2× upscale 0.728 < Otsu binarize 1.460, and the ordering holds for every model tested.
- **Binarization is the worst offender:** paired +1.163 CER (95% CI [0.512, 2.390],
  d_z = 0.468, p = 0.004).
- **Degradation amplifies non-determinism:** within-cell SD grows from 0.051 (raw) to
  1.060 (binarized) at temperature 0, showing hosted-model instability is input-dependent.
- A structured-field recall check reproduces the ranking without the CER metric, and the
  LLM judge auto-calibrates to κ = 0.70 against hand scoring with zero human labels.

## Repository layout

```
paper/           manuscript source + compiled PDF + figures
figure_scripts/  scripts that generate the figures
RESULTS.md · STATISTICAL_SUPPLEMENT.md · INTERPRETATION.md   verified stats reports
```

Data: CORD-v2 receipts (CC-BY-4.0, cited in-paper). 5/5 independent verification checks
pass; jiwer-vs-hand CER agreement to ~7e-15.

## Context

An afternoon-scale pilot from a research program auditing LLM behavior with psychometric
method, conducted as a live demonstration for Eric Kwon of AI-orchestrated research
practice (see the paper's acknowledgments).
Program index: [Research_Collection_Patrick_Keough](https://github.com/pskeough/Research_Collection_Patrick_Keough).

## Authorship note

Draft prepared with AI assistance under the author's direction; all design, analysis
decisions, and claims are the author's own and independently verified.

## License

Code: Apache-2.0 · Paper text and figures: CC BY 4.0
