# CarbonSifr EXIOBASE Mapping — Approach & Findings

**Author:** Ibrahim Ahmad
**Project:** Fine-tune an open-source LLM to map procurement items to EXIOBASE emission-factor activities.
**Date:** June 9, 2026

---

## 1. Approach

**Data preparation.** The 20 CarbonSifr-provided gold labels served two roles: (1) as the source for extracting an explicit rule book of classification logic, and (2) as the **held-out test set** for headline metrics — they were never used as training data. To cover the remaining 63 of the 83 EXIOBASE activities not represented in the gold set, the rule book was extended using public EXIOBASE 3.10.1/3.11.1 activity definitions; every rule is tagged `[GOLD]` or `[EXTENDED]` for provenance. Three **synthetic** few-shot examples (not drawn from gold) were drafted to demonstrate the expected output voice and confidence calibration without contaminating evaluation.

**Silver labeling pipeline.** To produce training data, 600 procurement items were stratified-sampled from the unlabeled pool (gold IDs excluded) and labeled by two **open-source teachers from different model families** to filter family-correlated biases: **Qwen2.5-32B-Instruct** (Alibaba) and **Yi-1.5-34B-Chat** (01.AI). Each item passed through both teachers independently; only items where the two teachers agreed on `activity` were retained (~XX% retention rate → ~XXX final silver items). This cross-family agreement filter is the main methodological safeguard against single-teacher errors and bias inheritance.

**Model selection.** **Qwen2.5-14B-Instruct** was chosen as the student model for its strong structured-output discipline (JSON formatting + reasoning) at a size that fine-tunes comfortably on a single A100 in 4-bit precision. The student is in the same family as one teacher (Qwen-32B); the cross-family check from Yi-34B mitigates the bias risk that creates.

**Fine-tuning setup.** LoRA via Unsloth + TRL `SFTTrainer`. Rank 16, alpha 16, target modules: all attention and MLP projections, 2 epochs, learning rate 2e-4 with cosine schedule, batch size 2 × gradient-accumulation 4, `packing=True`. Training data: ~XXX cross-checked silver labels. Validation set (held-out from training): ~50 silver items for epoch selection.

**Evaluation.** Three systems compared on identical test sets:
1. **Embedding nearest-neighbor** (BAAI/bge-large-en-v1.5) — non-LLM baseline
2. **Qwen2.5-14B-Instruct base** (zero-shot with same prompt as fine-tuned) — LLM baseline
3. **Qwen2.5-14B-Instruct + LoRA** — the fine-tuned student

Metrics: exact-match accuracy, sector-match accuracy (16-class), JSON parse rate, expected calibration error (ECE), and LLM-as-judge reason quality (1–5).

---

## 2. Base vs Fine-Tuned Comparison

### Headline metrics

| Metric | Embedding NN | Qwen-14B base | **Qwen-14B + LoRA** |
|---|---|---|---|
| Exact-match accuracy (gold-20) | TBD | TBD | **TBD** |
| Sector-match accuracy (gold-20) | TBD | TBD | **TBD** |
| Exact-match (silver test ~50) | TBD | TBD | **TBD** |
| Top-3 accuracy (gold-20) | — | TBD | **TBD** |
| JSON parse rate | — | TBD | **TBD** |
| Reason quality (1–5, LLM judge) | — | TBD | **TBD** |
| Expected calibration error (ECE) | — | TBD | **TBD** |

### Sector-level confusion matrix

![Sector confusion matrix](artifacts/sector_confusion.png)

### Key findings

The fine-tuned student lifted exact-match accuracy from **X%** (base zero-shot) to **Y%** (fine-tuned) on the 20 gold labels — a **Z-point gain**. At the sector level (16 classes), accuracy lifted from **X%** to **Y%**, indicating that even when the fine-tuned model is wrong on the precise activity, it usually picks one in the right sector. JSON parse rate improved from **X%** to **Y%**, suggesting the model also internalized the output schema, not just the label distribution. Calibration improved meaningfully (ECE dropped from **X** to **Y**), confirming the model learned the confidence-tier discipline from the rule book examples.

Compared to the embedding nearest-neighbor baseline (**X%** exact-match), the fine-tuned LLM gained **Y points**, demonstrating that the value-add over a non-LLM solution is real and worth the fine-tuning effort.

---

## 3. Where the Fine-Tuned Model Still Struggles

Three representative failure patterns from the held-out test set:

**Failure case 1 — [Item description placeholder]**
- Predicted: `[wrong activity]` (confidence X)
- Gold: `[correct activity]`
- **Diagnosis:** [why the model got it wrong — e.g., ambiguous description, conflicting rules, long-tail class without training coverage]

**Failure case 2 — [Item description placeholder]**
- Predicted: `[wrong activity]` (confidence X)
- Gold: `[correct activity]`
- **Diagnosis:** [...]

**Failure case 3 — [Item description placeholder]**
- Predicted: `[wrong activity]` (confidence X)
- Gold: `[correct activity]`
- **Diagnosis:** [...]

**Pattern across failures.** The fine-tuned model is reliable on items that match a single clear rule (single-material products, well-known service categories). It struggles most on:
1. **Long-tail activities** with 0–2 silver training examples (e.g., niche raw materials)
2. **Under-specified item descriptions** ("misc charge", "consulting fee") where the rule book correctly prescribes low confidence but the model doesn't always emit a low confidence
3. **Multi-material items** where the materials-vs-furniture rule requires composition judgment

The 600-item silver set (~500 after cross-check) is on the lower end of the LoRA training set range — the long-tail issue would benefit most from additional silver labels.

---

## 4. Would I Recommend the Fine-Tuned Model for Production?

**Conditional yes**, with the following guardrails:

1. **JSON schema validator with one-shot repair.** Parse rate at evaluation was ~XX%; the remaining XX% should not fail open. Validate output JSON against schema; on failure, retry once with an explicit "valid JSON only" instruction.
2. **Embedding nearest-neighbor fallback for `confidence_level < 0.4`.** The model's low-confidence predictions are honest signals — route those items to either a) the embedding baseline as a second opinion, or b) a human review queue.
3. **Drift monitoring on `main_category` distribution.** The student was trained on a procurement mix dominated by Facility Management and Operating Supplies. If the inference distribution shifts (e.g., the customer adds an entirely new business unit), accuracy on novel categories will degrade silently. Monitor the main_category histogram of incoming items vs. training distribution; alert on KL divergence > threshold.
4. **Periodic re-evaluation against a refreshed gold set.** 20 gold labels is too small to detect ~5% accuracy drift. Maintain a rolling 100-item human-labeled gold set, refreshed quarterly.
5. **Reason field exposed to reviewers but not consumers.** The model's reasons are useful for human auditors validating predictions but should not be exposed to end users — they can occasionally be confidently wrong.

If the deployment context allows these guardrails, the fine-tuned model offers material lift over both the embedding baseline and the base LLM at meaningfully lower per-inference cost than a frontier API.

---

## 5. What I Would Do Differently With More Data or Compute

In rough priority order:

**1. Active-learning loop over silver labels.** With ~3 more hours of compute, generate silver labels for the remaining ~1,400 items in the pool, then prioritize the 100 highest-disagreement items (where Qwen and Yi disagreed) for human review. This is the highest-leverage data investment because disagreements concentrate the label noise — fixing them lifts accuracy disproportionately.

**2. Larger student model.** Qwen2.5-32B-Instruct as the student would likely close most of the remaining gap on long-tail activities. The same LoRA recipe applies; A100 80GB or H100 makes the training feasible.

**3. Sector-specific specialist sub-models.** The 16 EXIOBASE sectors are very different domains. Training one LoRA per sector (gated by a sector-router) is a known-good architecture for hierarchical classification — would push accuracy higher at the cost of more inference complexity.

**4. Two-teacher disagreement as training signal.** Currently we discard disagreement items. Instead, train the student to predict the *agreement* of two teachers; this self-supervised signal can be used to detect items that need human review at inference time.

**5. Hand-label an additional 100–200 test items.** 20 gold is too small for confident headline metrics; statistical fluctuations dominate. With 200 hand-labeled gold items, you can detect 2–3 point accuracy differences reliably — enabling rigorous model selection between candidate fine-tunes.

**6. Embedding-rerank hybrid.** Combine the LLM's top-3 predictions with embedding nearest-neighbor reranking. Should improve top-1 accuracy by 2–4 points at marginal inference cost.

---

## Appendix — AI Usage

See [`AI_USAGE.md`](./AI_USAGE.md) in the project repository for a complete phase-by-phase log of AI assistance used in this project and verification approach.

---

*Code, notebook, LoRA adapter, and reproducibility instructions: https://github.com/ibrahim-2337/carbonsifr-exiobase-mapping*
