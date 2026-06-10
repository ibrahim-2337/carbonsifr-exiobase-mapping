# CarbonSifr EXIOBASE Mapping — Approach & Findings

**Author:** Ibrahim Ahmad
**Project:** Fine-tune an open-source LLM to map procurement items to EXIOBASE emission-factor activities.
**Date:** June 9, 2026

*Main body is the 1–2 page summary; the appendix holds the fuller detail.*

---

## 1. Approach (summary)

*Gold* = the 20 human-verified labels CarbonSifr provided (ground truth). *Silver* = the 600 training labels I generated with LLMs. With only 20 trustworthy examples, I read them by hand and distilled their logic into an explicit rule book (each rule tagged `[GOLD]`), then extended it from EXIOBASE 3.10.1/3.11.1 definitions to cover the other 63 activities (tagged `[EXTENDED]`). That rule book plus three hand-written few-shot examples and the full 83-activity catalog forms the system prompt.

For training data, **Qwen2.5-32B-Instruct** (open-source) labeled 600 stratified items against the rule book. Two independent reviewers checked it: **Claude** re-labeled a random 100 (62% agreement — a quality signal only, never used as training labels), and **Gemini 2.5 Pro** labeled the other 500 (49% agreement), with the 254 disagreements adjudicated by hand. I fine-tuned **Qwen2.5-14B-Instruct** (the student) via LoRA — Unsloth + TRL, r=16/α=16, 2 epochs — on a **frozen 498/51/51 split**, so training and test are provably disjoint. I evaluated four systems on the 20 gold labels and the 51 held-out silver items, scoring exact-match (83-class) and sector-match (16-class). Full detail in Appendix A.

---

## 2. Base vs Fine-Tuned Comparison

| Metric | Embedding NN | Qwen-14B zero-shot | Qwen-14B + rule book | **Qwen-14B + LoRA** |
|---|---|---|---|---|
| Exact-match (gold-20) | 30.0 | 55.0 | 95.0\* | **95.0** |
| Sector-match (gold-20) | 40.0 | 65.0 | 100.0\* | **95.0** |
| Exact-match (silver-51) | 49.0 | 45.1 | 58.8 | **76.5** |
| Sector-match (silver-51) | 56.9 | 58.8 | 68.6 | **84.3** |

\* The gold-20 rule-book scores are optimistic — the rule book was partly written *from* those 20 items, so it nearly encodes their answers. The **silver-51** column is the fair comparison: those items never shaped the rule book, and the fine-tuned model never trained on them.

So I weight the silver column. Reading it left to right, every component earns its place: 45.1% unaided → 58.8% with the rule book in context (+13.7) → 76.5% after fine-tuning (+17.7). Sector accuracy follows the same path, to 84.3%. That **+17.7-point** gain is the headline result, and because the split is frozen and disjoint it reflects genuine generalization, not memorization.

My read is that fine-tuning turns the rule book from an instruction the model must consciously apply into learned behavior — with the rules in context the base already does okay (58.8%) but applies them inconsistently, and 498 worked examples fix that. The sector result matters most in practice: even when the model picks the wrong specific activity, it lands in the right sector ~84% of the time, and an adjacent miss is far cheaper in carbon accounting than a cross-sector one. The fine-tuned model also clears the non-LLM bar by a wide margin — embedding nearest-neighbor manages 49.0% on silver versus 76.5%, a **+27.5-point** gap. One honest caveat: on gold-20 the fine-tuned sector score (95.0) is *below* the rule-book base (100.0) — the single gold item it misses on exact match, it now also misses at sector level. On 20 items that's one example, but it illustrates the small-set tradeoff (sharper common cases, occasional overconfidence on an edge case).

---

## 3. Where the Fine-Tuned Model Still Struggles

Three representative misses (full diagnoses in Appendix B):

- **"Encore AV & Backdrop…"** → predicted `Other business services (74)`, gold `Other services (93)`. Two adjacent service catch-alls; fine-tuning pulled it toward the more frequent bin and committed at 0.85.
- **"Champagne flute PC clear 180 Ml"** → predicted `Glass and glass products`, gold `Rubber and plastic products (25)`. Anchored on the object ("flute" → glass) and missed the "PC" (polycarbonate) qualifier.
- **"MAXIPULL 2PLY 1X6 PURE M800"** (category: CLEANING) → predicted `Chemicals nec`, gold `Paper and paper products`. The cryptic code carried no material cue, so it leaned on the buyer category, which encodes function (cleaning) not material (paper).

The recurring weaknesses: **overlapping catch-all categories**, **anchoring on a salient noun** while missing a material qualifier, **cryptic descriptions where category metadata misleads**, and **long-tail activities** — the silver set covers only 56 of 83 activities, so 27 classes have zero training signal. One self-inflicted issue: I trained on a placeholder `reason` string, so the model classifies well but no longer explains itself (fix in §5).

---

## 4. Would I Recommend It for Production?

Conditionally yes, with guardrails (expanded in Appendix C):

1. **Validate JSON with one-shot repair** — schema-check every output; on failure retry once before falling back.
2. **Route `confidence_level < 0.4` to the embedding baseline or a human** — the model's low-confidence calls are honest uncertainty signals.
3. **Monitor `main_category` drift** — accuracy on a brand-new business unit will degrade silently; alert when the incoming category mix diverges from training.
4. **Re-evaluate against a refreshed gold set quarterly** — 20 gold items can't detect a 5% drift.
5. **Restore the reason field, for auditors not end users** — useful once trained on real reasons (§5), but it can be confidently wrong.

With those in place, the model gives a real, reproducible lift over both the embedding baseline (+27.5 exact) and the in-context base LLM (+17.7 exact), at much lower per-item cost than a frontier API.

---

## 5. What I'd Do Differently With More Data or Compute

1. **Train on real reasons, not a placeholder** — Qwen/Gemini already produced a justification for every label; keeping it restores the model's ability to explain itself at zero extra cost. Cheapest win.
2. **Active-learning loop** — label the remaining ~1,400 pool items and send the highest-disagreement cases to a human; disagreements concentrate label noise and the volume attacks the 27 uncovered activities directly.
3. **Larger student** — Qwen2.5-32B would likely close most of the long-tail gap on the same recipe.
4. **More gold** — 20 items is too few to trust small differences; ~200 hand-labeled would make 2–3-point gaps distinguishable from noise. (Further ideas in Appendix D.)

---
---

# Appendix

## A. Full pipeline detail

**Data preparation.** The 20 gold labels served two roles: the source for the rule book, and the held-out headline test set (never used for training). Because 20 examples only touch 20 of 83 activities, I extended the rule book from EXIOBASE's published definitions for the other 63, tagging each rule `[GOLD]` or `[EXTENDED]` for provenance. The three few-shot examples were authored by hand (not drawn from gold) to avoid leaking the test set. The assembled prompt is ~3.5k tokens.

**Silver labeling.** I sampled 600 items from the unlabeled pool (gold IDs excluded), stratified by category, and labeled all of them with Qwen2.5-32B-Instruct (4-bit on a Colab A100) under the rule book. Reviewers: Claude re-labeled a random 100 without seeing Qwen's answers (62% agreement, used only as a label-quality signal; those 100 items kept Qwen's labels), and Gemini 2.5 Pro labeled the other 500 (49.2% agreement, 246/500). I adjudicated the 254 disagreements one by one — Gemini's call won 124 times, Qwen's 130. Final breakdown: 246 agreements, 130 Qwen-favored, 124 Gemini-favored, 100 Claude-audited. Coverage is 56 of 83 activities. This was not the original plan — I'd intended a fully automated two-teacher filter, but a second open-source teacher was too slow on the available time, so it became one teacher plus two reviewers. I record that as a limitation.

**Model + fine-tuning.** Qwen2.5-14B-Instruct as the student: strong structured output, fine-tunes comfortably on one A100 in 4-bit. Teacher/student share a family (a bias risk), guarded by the cross-family reviewers and the family-independent gold test. LoRA via Unsloth + TRL `SFTTrainer`: rank 16, alpha 16, all attention + MLP projections (~0.46% of weights), 2 epochs, lr 2e-4 cosine, batch 2 × grad-accum 4, packing on. Split: 498 train / 51 val / 51 silver-test, rare classes routed to train. I initially re-derived the split each run, which let the saved adapter be evaluated on items it had trained on; I froze the split to disk and retrained. Validation loss: 0.0196 → 0.0184 → 0.0183 over 126 steps, tracking train loss, no overfitting.

**Evaluation.** Four systems on the same two test sets: embedding nearest-neighbor (BAAI/bge-large-en-v1.5), Qwen-14B zero-shot (no rule book), Qwen-14B + rule book, and Qwen-14B + LoRA. Metrics: exact-match (83-class) and sector-match (16-class). JSON parse rate was 100% across all systems, so it isn't a differentiator.

## B. Full failure-case diagnoses

**"Encore AV & Backdrop for the Long Service Awards Ceremony"** — the single gold-20 miss. Predicted `Other business services (74)` at 0.85; gold `Other services (93)`. EXIOBASE has two adjacent service catch-alls, (74) and (93), and AV/event production sits on the boundary. This is also the gold sector miss from §2 — the in-context base placed it in the correct sector and the fine-tuned model didn't. Fine-tuning drew it toward (74), the bin more frequent in training, and it then committed confidently to a genuinely ambiguous case.

**"Champagne flute PC clear 180 Ml"** — predicted `Glass and glass products` at 0.95; gold `Rubber and plastic products (25)`. "Champagne flute" strongly evokes glass, and the model fixed on the object while overlooking "PC clear" (polycarbonate = plastic). The same pattern appears in "Gypsum Channel GI 'W' Type" (predicted `Cement, lime and plaster` from "Gypsum"; gold `Fabricated metal products (28)` — "GI" = galvanised iron). Both are confident misses where a salient but misleading term beat the true material in a modifier.

**"MAXIPULL 2PLY 1X6 PURE M800"** (buyer sub-category: CLEANING & DISINFECTING SOLUTIONS) — predicted `Chemicals nec` at 0.85; gold `Paper and paper products`. MAXIPULL is a 2-ply industrial paper wiper. The description is essentially a product code with no material cue, so the model fell back on the buyer category, which describes the item's function (cleaning) rather than its material (paper).

Also seen: "Sewing Trimming Scissor" → `Furniture; other manufactured goods (36)` instead of `Fabricated metal products (28)` — over-use of the manufactured-goods catch-all over a specific material bin.

## C. Full production guardrails

1. **JSON schema validator with one-shot repair.** Parse rate was 100% in evaluation, but production traffic is messier; validate every output against the schema and, on failure, retry once with an explicit "valid JSON only" instruction before falling back.
2. **Embedding nearest-neighbor (or human) fallback for `confidence_level < 0.4`.** Low-confidence predictions are honest uncertainty signals and should be routed to a safer path rather than trusted.
3. **Drift monitoring on `main_category`.** The student learned on a mix weighted toward Facility Management and Operating Supplies; a new business unit will degrade accuracy silently. Compare the incoming category histogram to the training mix and alert on divergence.
4. **Periodic re-evaluation against a refreshed gold set.** 20 gold labels can't detect a 5% drift (one item is 5%). Maintain a rolling 100-item human-labeled set, refreshed quarterly.
5. **Restore and surface the reason field for auditors, not consumers.** Once trained on real reasons, the justifications help a human reviewer verify a prediction, but shouldn't be shown to end users — they can be confidently wrong.

## D. Full future-work list

1. **Train on real reasons** instead of the placeholder string (cheapest win; no extra labeling).
2. **Active-learning loop** over the remaining ~1,400 pool items, prioritizing the highest-disagreement cases for human review.
3. **Larger or longer-trained student** — Qwen2.5-32B on an 80GB A100 / H100, same recipe.
4. **Teacher disagreement as a training signal** — train the model to predict whether the two teachers would agree, as a cheap self-supervised "needs human review" flag.
5. **Hand-label 100–200 more test items** so 2–3-point differences become distinguishable from noise.
6. **Embedding rerank** on the LLM's top-3 — likely a couple of top-1 points at marginal cost.

---

*See [`AI_USAGE.md`](./AI_USAGE.md) for a full log of AI tool usage and verification. Code, notebook, and adapter: https://github.com/ibrahim-2337/carbonsifr-exiobase-mapping*
