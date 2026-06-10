# AI Usage Log

This project used AI assistance throughout. Below is a phase-by-phase breakdown of what was delegated to AI tools and how outputs were verified, logged for transparency.

---

| Phase | Tool / Model | What I delegated | How I verified / what I overrode |
|---|---|---|---|
| Rule book extraction | Claude (chat) | Drafted initial rule book from the 20 gold examples, then extended it with EXIOBASE 3.10.1/3.11.1 activity definitions for the 63 uncovered classes | Hand-verified all 20 gold examples against the drafted rules; tagged every rule `[GOLD]` or `[EXTENDED]` for provenance; audited the EXTENDED rules against EXIOBASE's published classification logic |
| Few-shot examples | Claude (chat) | Drafted 3 synthetic worked examples to demonstrate the rule book voice (no contamination of the 20 gold) | Hand-reviewed each example for correct rule application and confidence calibration |
| Prompt template | Claude (chat) | Designed the system prompt structure (role → schema → rules → examples → activity list) | Verified the assembled prompt fits within the 8192-token budget on the Qwen tokenizer; smoke-tested on 3 items before bulk labeling |
| Silver labeling — primary teacher | Qwen2.5-32B-Instruct (Alibaba, open-source, run on Colab A100) | Generated `{activity, reason, confidence}` JSON for all 600 stratified procurement items, following the rule book in the system prompt | Spot-checked random labels for rule adherence; 100% JSON parse rate; cross-checked by two independent reviewers (below) |
| Silver labeling — audit reviewer | Claude (chat) | Independently re-labeled a random 100-item subset under the same rule book (no prior viewing of Qwen's labels) | 62% agreement with Qwen, used only as a label-quality signal. Claude's labels did not enter training; those 100 items kept Qwen's labels. |
| Silver labeling — cross-check reviewer | Gemini 2.5 Pro (AI Studio, chat) | Independently labeled the other 500 items under the same rule book | 49% agreement with Qwen. The 254 disagreements were individually adjudicated by me (124 favored Gemini, 130 favored Qwen). The 124 Gemini-favored labels do enter training — disclosed; see integrity notes. |
| LoRA training setup | Claude (chat) | Drafted Unsloth + TRL `SFTTrainer` configuration | Confirmed the Unsloth API matches current docs; pinned library versions; chose `r=16, alpha=16, lr=2e-4, 2 epochs` after considering tradeoffs; verified the train/test split is frozen and disjoint (no leakage) |
| Evaluation harness | Claude (chat) | Drafted the metrics functions (exact-match and sector-match accuracy, robust JSON parsing) | Ran on the 20 CarbonSifr gold labels first as a sanity check; hand-verified the JSON parsing logic |
| README / repo scaffolding | Claude (chat) | Drafted README structure, tech-stack table, AI usage log template | Reviewed and edited |

---

## Notes on integrity

- The 20 CarbonSifr-provided gold labels were never used as training data. They serve as (a) the source for `[GOLD]`-tagged rules in the rule book, and (b) the held-out test set for headline metrics.
- Few-shot examples in the system prompt are synthetic (authored fresh), not drawn from the gold labels, which prevents leakage into evaluation.
- The train/val/test split is frozen and disjoint, so the fine-tuned adapter is evaluated only on items it never trained on and the reported silver-51 numbers are a genuine held-out measurement.
- The fine-tuned model and its primary teacher are open-source. The student (Qwen2.5-14B-Instruct) and the primary silver labeler (Qwen2.5-32B-Instruct) are both open-source (Qwen Community License). The fine-tuned LoRA adapter is published on Hugging Face Hub.
- Proprietary models were used as reviewers, disclosed for transparency. Claude audited 100 items, and its labels did not enter training. Gemini 2.5 Pro cross-checked 500 items; where its label was judged more accurate than Qwen's during adjudication, it was adopted, so 124 of the 600 training labels (~21%) originate from Gemini. The brief permits LLM-assisted labeling; this is noted explicitly rather than hidden. A strictly open-source-only training set is a one-line change (always keep Qwen's label on disagreement) and is listed as a variant.
- Claude was also used for code authoring, prose drafting, and rule-book extension, never as a labeler whose outputs entered the training or evaluation set.
