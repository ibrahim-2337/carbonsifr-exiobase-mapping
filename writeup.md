# CarbonSifr EXIOBASE Mapping — Approach & Findings

**Author:** Ibrahim Ahmad
**Project:** Fine-tune an open-source LLM to map procurement items to EXIOBASE emission-factor activities.
**Date:** June 10, 2026

---

## 1. Approach

A brief note on terminology, since it recurs throughout. *Gold* refers to the 20 human-verified labels CarbonSifr provided — the ground truth. *Silver* refers to the 600 training labels I generated with LLMs, of which there are far more but which I trust correspondingly less. The design follows directly from that asymmetry: with only 20 reliable examples, I distilled them into an explicit rule book, used the rule book to mass-produce silver training data, fine-tuned the student model on the silver set, and evaluated everything against the gold I had held out. The substantive work is done by two open-source Qwen models — a 32B teacher and a 14B student — with Claude and Gemini serving as independent reviewers of the silver labels.

**Working from 20 examples.** The gold labels serve two purposes. First, I read through them manually and codified the classification logic they implied into a rule book. Second, I reserved them as the held-out test set and kept them out of training entirely. The difficulty is that 20 examples touch only 20 of the 83 EXIOBASE activities, so for the remaining 63 I extended the rule book using EXIOBASE's own published activity definitions. Every rule is tagged either `[GOLD]` (derivable from one of the 20 labels) or `[EXTENDED]` (added from the EXIOBASE documentation), making each rule's provenance explicit. I also authored three short few-shot examples by hand rather than reusing any gold item, to avoid leaking the test set into the prompt. The assembled system prompt — rule book, few-shot examples, and the full 83-activity catalog — comes to roughly 3.5k tokens.

**Labeling the silver set.** For training data I sampled 600 items from the unlabeled pool (gold IDs excluded), stratified by category, and had **Qwen2.5-32B-Instruct** label all of them against the rule book — open-source, running 4-bit on a Colab A100. Rather than trust a single model's output, I introduced two independent reviewers:

- **Claude** re-labeled a random 100 of those items from scratch under the same rule book, without seeing Qwen's answers. It agreed with Qwen on **62%**. I treated this strictly as a quality signal; Claude's labels never entered training, and those 100 items retained their Qwen labels.
- **Gemini 2.5 Pro** (via AI Studio) labeled the other 500. It agreed with Qwen on **49.2%** (246/500). I adjudicated the 254 disagreements individually: Gemini's call was the better one 124 times, Qwen's 130 times.

The final 600 labels therefore comprise 246 Qwen–Gemini agreements, 130 disagreements resolved in Qwen's favor, 124 resolved in Gemini's, and 100 from the Claude-audited slice. They cover 56 of the 83 activities; the rare classes remain sparsely represented, which becomes a limiting factor later. To be candid, this was not the original design — I had intended a fully automated two-teacher filter, but a second open-source teacher proved too slow within my time budget, so the method became this one-teacher-plus-two-reviewers arrangement. I record it as a limitation rather than a design choice.

**Selecting the student.** I chose **Qwen2.5-14B-Instruct** as the model to fine-tune. It is consistently strong at structured output (clean JSON and coherent reasoning) and small enough to fine-tune comfortably on a single A100 in 4-bit. The evident risk is that teacher and student share a model family and may therefore share systematic blind spots; the cross-family reviewers (Claude, Gemini) and the gold test set — which favors no model family — are the safeguards against that.

**Fine-tuning.** LoRA via Unsloth + TRL's `SFTTrainer`: rank 16, alpha 16, applied to all attention and MLP projections (approximately 0.46% of weights trainable), 2 epochs, learning rate 2e-4 on a cosine schedule, batch size 2 with gradient accumulation 4, packing enabled. I split the 600 silver labels into 498 train / 51 validation / 51 silver-test, routing any class too rare to stratify into the training set so that no split is left invalid. One issue I encountered and corrected is worth recording: I had initially been re-deriving the split on every notebook run, which meant the saved adapter was being evaluated on items it had partly trained on. I froze the split to disk so that train and test are provably disjoint, then retrained. Training itself was uneventful — validation loss moved 0.0196 → 0.0184 → 0.0183 over 126 steps and tracked the training loss throughout, with no sign of overfitting.

**Evaluation protocol.** Four systems, assessed on the same two test sets (the 20 gold and the 51 held-out silver items):

1. Embedding nearest-neighbor (BAAI/bge-large-en-v1.5) — a non-LLM baseline
2. Qwen-14B zero-shot, activity list only, no rule book — the unaided model
3. Qwen-14B with the full rule-book prompt, no fine-tuning
4. Qwen-14B + LoRA — the fine-tuned student, same full prompt

I report exact-match accuracy across all 83 classes and sector-match accuracy collapsed to the 16 sectors. The JSON parser achieved a 100% parse rate across every system during development, so parse rate is not a meaningful differentiator and I omitted it from the table.

---

## 2. Base vs Fine-Tuned Comparison

| Metric | Embedding NN | Qwen-14B zero-shot | Qwen-14B + rule book | **Qwen-14B + LoRA** |
|---|---|---|---|---|
| Exact-match (gold-20) | 30.0 | 55.0 | 95.0\* | **95.0** |
| Sector-match (gold-20) | 40.0 | 65.0 | 100.0\* | **95.0** |
| Exact-match (silver-51) | 49.0 | 45.1 | 58.8 | **76.5** |
| Sector-match (silver-51) | 56.9 | 58.8 | 68.6 | **84.3** |

\* The gold-20 rule-book scores should be read with caution. The rule book was written in part *from* those 20 items, so evaluating it against them approaches grading it on its own answer key. The silver-51 column is the fair comparison: those items never informed the rule book, and the fine-tuned model never trained on them.

The silver column is therefore the one I weight most heavily. Read left to right, each component I added contributes measurably: 45.1% unaided, 58.8% once the rule book is in the prompt (+13.7 points), and 76.5% after fine-tuning (+17.7 points). Sector accuracy follows the same trajectory, 58.8 → 68.6 → 84.3. The +17.7-point gain from fine-tuning is the central result, and because the split is frozen and disjoint, it reflects genuine generalization rather than memorization of training items.

My interpretation is that fine-tuning converts the rule book from an instruction the model must consciously apply into learned behavior. With the rules in context the base model already performs reasonably (58.8%), but it applies them inconsistently — it possesses the rule and still fails to invoke it. After 498 worked examples it applies them far more reliably. The improvement at sector level (to 84.3%) is the result I would emphasize to a stakeholder: even when the model selects the wrong specific activity, it lands in the correct sector roughly five times out of six, and an adjacent error is considerably less costly in carbon accounting than a cross-sector one.

The fine-tuned model also clearly outperforms the non-LLM option. Embedding nearest-neighbor reaches 49.0% exact on silver against the fine-tuned model's 76.5% — a 27.5-point gap. This is intuitive: the descriptions are short and noisy ("BSFJ057 BbyR BESPOKE Waitress Winter" is an actual example), and a 498-row lookup table cannot find a good neighbor for every irregular item, whereas the LLM can reason from the text together with the rules. The model's value over plain similarity is real.

One result I should not understate: on gold-20 the fine-tuned model's sector score (95.0) is in fact *lower* than the rule-book base (100.0). The single gold item it misses on exact match it now also misses at sector level, where the base model at least remained in the correct neighborhood. On 20 items this is a single example, but it fairly illustrates the tradeoff — fine-tuning on a small, mildly noisy silver set sharpens the common cases while occasionally inducing overconfidence on an edge case.

---

## 3. Where the Fine-Tuned Model Still Struggles

Three misses, each illustrating a distinct error mode:

**"Encore AV & Backdrop for the Long Service Awards Ceremony"** — the single gold-20 miss.
Predicted `Other business services (74)` at 0.85; gold is `Other services (93)`.
EXIOBASE contains two adjacent service catch-alls, (74) and (93), and AV/event production sits precisely on the boundary between them. This is also the sector miss noted in §2 — the in-context base placed it in the correct sector and the fine-tuned model did not. My reading is that fine-tuning drew it toward (74), the bin more frequent in training, after which it committed at 0.85 to a genuinely ambiguous case. That overconfidence on a borderline item is characteristic of the small-set risk.

**"Champagne flute PC clear 180 Ml"**
Predicted `Glass and glass products` at 0.95; gold is `Rubber and plastic products (25)`.
"Champagne flute" strongly evokes glass, and the model fixed on the object while overlooking the "PC clear" qualifier — PC being polycarbonate, i.e. plastic. The same pattern appears in "Gypsum Channel GI 'W' Type", where it predicted `Cement, lime and plaster` from the word "Gypsum" when "GI" (galvanised iron) makes it `Fabricated metal products (28)`. Both are confident misses in which a salient but misleading term outweighed the true material, which sat in a modifier.

**"MAXIPULL 2PLY 1X6 PURE M800"** (buyer sub-category: CLEANING & DISINFECTING SOLUTIONS)
Predicted `Chemicals nec` at 0.85; gold is `Paper and paper products`.
MAXIPULL is a 2-ply industrial paper wiper — a single-material paper product. The description, however, is essentially a product code with no material cue, so the model fell back on the buyer category, which describes the item's *function* (cleaning) rather than its *material* (paper). The category metadata is usually informative; here it pointed in the wrong direction.

Taken together, the model is dependable when there is one clear material or a well-known service type. Its difficulties cluster as follows:

1. **Overlapping catch-all categories** — the two "nec" service bins, and "manufactured goods n.e.c. (36)" versus a specific material bin (for instance, "Sewing Trimming Scissor" was assigned to (36) rather than `Fabricated metal products (28)`).
2. **Anchoring on the salient noun** while missing a material qualifier such as "PC" or "GI" elsewhere in the description.
3. **Cryptic descriptions in which category metadata misleads** — a bare product code pushes the model onto `main_category`/`category_3`, which encodes function rather than material.
4. **Long-tail activities** with few or no silver examples — I covered only 56 of 83 activities, so 27 classes carry zero training signal and exist for the model solely through the rule book in the prompt.
5. **The reason field, which I degraded myself** — I trained on a placeholder reason string, so the model now produces a generic reason even when the activity and confidence are correct. It learned what to classify but not how to justify it; the fix is in §5.

The 600-item set is on the small side for LoRA, and the two largest levers are simply enlarging it and restoring genuine reasons.

---

## 4. Would I Recommend the Fine-Tuned Model for Production?

Conditionally, yes — provided the following guardrails are in place:

1. **Validate the JSON with a one-shot repair.** Parse rate was 100% in evaluation, but production traffic is messier than a test set. Each output should be checked against the schema, and on a failure the system should retry once with an explicit "valid JSON only" instruction before proceeding.
2. **Fall back to the embedding baseline (or a human) when `confidence_level < 0.4`.** The model's low-confidence predictions are an honest signal of uncertainty and should be routed to a safer path rather than trusted.
3. **Monitor the `main_category` distribution for drift.** The student learned on a mix weighted toward Facility Management and Operating Supplies. If a customer introduces an entirely new business unit, accuracy on those novel items will degrade silently. Comparing the incoming category histogram against the training mix, with an alert on divergence, would catch this.
4. **Re-evaluate against a refreshed gold set on a schedule.** Twenty gold labels cannot detect a 5% drift, since one item is already 5%. I would maintain a rolling 100-item human-labeled set and refresh it quarterly.
5. **Restore the reason field, for auditors only.** Once trained on real reasons (§5), the explanations are genuinely useful to a human reviewer verifying a prediction, but I would not surface them to end users — they can be confidently wrong.

With those measures in place, the model delivers a real and reproducible improvement over both the embedding baseline (+27.5 exact) and the in-context base LLM (+17.7 exact) on the clean test, at meaningfully lower per-item cost than a frontier API.

---

## 5. What I Would Do Differently With More Data or Compute

Approximately in the order I would pursue them:

1. **Train on real reasons rather than a placeholder.** This is the cheapest improvement by a wide margin. Qwen and Gemini already produced a genuine justification for every label; I need only retain it in the training target instead of the placeholder string I used. It requires no additional labeling and restores the model's ability to explain itself.
2. **Run an active-learning loop to fill the long tail.** With a few additional hours of compute I would label the remaining ~1,400 pool items, then route the highest-disagreement cases (I have already adjudicated 254 Qwen-vs-Gemini splits) to a human. Disagreements concentrate the label noise, so resolving them yields gains out of proportion to their number, and the added volume targets the 27 currently uncovered activities directly.
3. **Evaluate a larger or longer-trained student.** Qwen2.5-32B as the student would likely close most of the long-tail gap; the same LoRA recipe runs on an 80GB A100 or an H100.
4. **Use teacher disagreement as a signal rather than discarding it.** Instead of only adjudicating disagreements, I could train the model to predict *whether the two teachers would agree* — an inexpensive self-supervised flag for items that warrant human review.
5. **Hand-label a further 100–200 test items.** Twenty gold items are too few to trust small differences (again, one item is 5%). At roughly 200 gold items, a 2–3 point improvement becomes distinguishable from noise, which would allow principled selection between candidate fine-tunes.
6. **Add an embedding rerank on top.** Combining the LLM's top-3 with embedding nearest-neighbor reranking should be worth a few top-1 points at marginal cost.

---

## Appendix — AI Usage

See [`AI_USAGE.md`](./AI_USAGE.md) for a complete phase-by-phase log of how I used AI tools on this project and how I verified their output.

---

*Code, notebook, LoRA adapter, and reproduction instructions: https://github.com/ibrahim-2337/carbonsifr-exiobase-mapping*
