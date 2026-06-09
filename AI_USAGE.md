# AI Usage Log

This project used AI assistance throughout. Below is a phase-by-phase breakdown of what was delegated to AI tools and how outputs were verified. Logged for transparency.

---

| Phase | Tool / Model | What I delegated | How I verified / what I overrode |
|---|---|---|---|
| **Project planning** | Claude (chat) | Drafted the 11-phase execution playbook (`plan.md`) — time budget, tech stack tradeoffs, "exceed expectations" list | Reviewed every phase decision; chose Qwen2.5-14B-Instruct over Llama-3.1-8B as student model after weighing reasoning quality vs. tutorial support |
| **Rule book extraction** | Claude (chat) | Drafted initial rule book from the 20 gold examples, then extended it with EXIOBASE 3.10.1/3.11.1 activity definitions for the 63 uncovered classes | Hand-verified all 20 gold examples against the drafted rules; tagged every rule `[GOLD]` or `[EXTENDED]` for provenance; audited the EXTENDED rules against EXIOBASE's published classification logic |
| **Few-shot examples** | Claude (chat) | Drafted 3 synthetic worked examples to demonstrate the rule book voice (no contamination of the 20 gold) | Hand-reviewed each example for correct rule application and confidence calibration |
| **Prompt template** | Claude (chat) | Designed the system prompt structure (role → schema → rules → examples → activity list) | Verified the assembled prompt fits within 8192-token budget on Qwen tokenizer; smoke-tested on 3 items before bulk labeling |
| **Silver labeling — primary** | Qwen2.5-32B-Instruct (Alibaba, open-source, run on Colab A100) | Generated `{activity, reason, confidence}` JSON for 600 stratified procurement items, following the rule book in the system prompt | Spot-checked random labels for rule adherence; cross-checked all labels with Yi-1.5-34B (see below); filtered items where the two teachers disagreed on activity |
| **Silver labeling — cross-check** | Yi-1.5-34B-Chat (01.AI, open-source) | Independently labeled the same 600 items | Different model family from Qwen (Yi vs. Qwen) → independent biases; kept only items where Qwen and Yi agreed on `activity` (~85% retention) |
| **LoRA training setup** | Claude (chat) | Drafted Unsloth + TRL `SFTTrainer` configuration | Confirmed Unsloth API matches current docs; pinned library versions; chose `r=16, alpha=16, lr=2e-4, 2 epochs` after considering tradeoffs |
| **Evaluation harness** | Claude (chat) | Drafted the metrics functions (exact-match, sector-match, ECE, top-K, confusion matrix plotting) | Ran on the 20 CarbonSifr gold labels first as a sanity check; hand-verified the JSON parsing logic |
| **Reason quality judging** | Qwen2.5-32B-Instruct (LLM-as-judge) | Rated 30 sample model reasons 1–5 against a rubric (rule citation, brevity, correctness) | Hand-checked 5 ratings to confirm rubric was being applied consistently |
| **Writeup drafting** | Claude (chat) | Drafted prose around the metrics table and confusion matrix | Rewrote the production recommendation section based on my own judgment; verified every number in the writeup against `artifacts/` files |
| **README / repo scaffolding** | Claude (chat) | Drafted README structure, tech-stack table, AI usage log template | Reviewed and edited |

---

## Notes on integrity

- The 20 CarbonSifr-provided gold labels were **never used as training data**. They serve as (a) the source for `[GOLD]`-tagged rules in the rule book, and (b) the held-out test set for headline metrics.
- Few-shot examples in the system prompt are **synthetic** (authored fresh), not drawn from the gold labels — preventing leakage to evaluation.
- All teacher and student models used in the published pipeline are **open-source** (Apache 2.0 / Qwen Community License). The student LoRA adapter is published on Hugging Face Hub.
- Claude was used only for code authoring, planning, prose drafting, and rule book extension — **never as a labeler whose outputs entered the training data or evaluation set**.
