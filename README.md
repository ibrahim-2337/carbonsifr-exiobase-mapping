# CarbonSifr — EXIOBASE Emission-Factor Activity Mapper

I fine-tuned Qwen2.5-14B-Instruct with LoRA to map messy procurement line items to one of 83 EXIOBASE emission-factor activities. This is my submission for the CarbonSifr AI Engineering Intern case study (June 2026).

For each item, the model returns structured JSON:

```json
{
  "activity": "Paper and paper products",
  "reason": "The item is a single-material paper product. Per the material rule, single-material paper items map to 'Paper and paper products'.",
  "confidence_level": 0.95
}
```

---

## Quick links

- Notebook (end-to-end): [`CarbonSifr_EXIOBASE_FineTune.ipynb`](./CarbonSifr_EXIOBASE_FineTune.ipynb) · [open in Colab](https://colab.research.google.com/github/ibrahim-2337/carbonsifr-exiobase-mapping/blob/main/CarbonSifr_EXIOBASE_FineTune.ipynb)
- LoRA adapter on HF Hub: [`retroib/qwen2.5-14b-exiobase-lora`](https://huggingface.co/retroib/qwen2.5-14b-exiobase-lora)
- Writeup (Task 2): [`writeup.md`](./writeup.md)
- AI usage log: [`AI_USAGE.md`](./AI_USAGE.md)

---

## Approach (one paragraph)

I started from the 20 gold-labeled examples CarbonSifr provided. Reading through them, I wrote down the classification logic they implied as an explicit rule book (each rule tagged `[GOLD]`), then extended it from EXIOBASE 3.10.1 / 3.11.1 activity definitions to cover the 63 activities the gold set never touches (tagged `[EXTENDED]`). That rule book, plus three hand-written few-shot examples and the full 83-activity catalog, became the system prompt. For training data I had Qwen2.5-32B-Instruct (open-source) label 600 stratified items against the rule book, then brought in two independent reviewers: Claude re-labeled a random 100 (62% agreement, used as a quality signal only and never as training labels) and Gemini 2.5 Pro labeled the other 500 (49% agreement), with the 254 disagreements adjudicated by hand. I fine-tuned the Qwen2.5-14B-Instruct student on the result via LoRA (Unsloth + TRL, 2 epochs) on a frozen 498/51/51 split, and graded everything against the 20 gold labels (the headline metric) and 51 held-out silver items (the clean, unleaked referee).

---

## Results

Evaluated on identical test sets. The silver-51 column is the unleaked referee — those items never informed the rule book and the fine-tuned model never trained on them.

| Metric | Embedding NN | Qwen-14B zero-shot | Qwen-14B + rule book | Qwen-14B + LoRA |
|---|---|---|---|---|
| Exact-match (gold-20) | 30.0 | 55.0 | 95.0\* | 95.0 |
| Sector-match (gold-20) | 40.0 | 65.0 | 100.0\* | 95.0 |
| Exact-match (silver-51) | 49.0 | 45.1 | 58.8 | 76.5 |
| Sector-match (silver-51) | 56.9 | 58.8 | 68.6 | 84.3 |

\* Gold rule-book scores are optimistic, since the rule book was partly derived from the gold items. On the clean silver-51 test, fine-tuning lifts exact-match by 17.7 points over the in-context base (58.8 → 76.5) and 27.5 points over the embedding baseline.

See [`writeup.md`](./writeup.md) for failure modes, the production recommendation, and future work.

---

## How to run it

The fastest way to review this is to just read the notebook — it's committed with every cell's output inline, so you can see the whole run (labels, training curve, metrics, demo) without executing anything.

To run it yourself on Colab:

1. Open the notebook in Colab (link above) and set the runtime to A100 (Runtime → Change runtime type → A100 GPU).
2. The notebook works out of a `carbonsifr/` folder in your Google Drive. It expects `Data.xlsx` in `carbonsifr/data/` and the contents of [`prompts/`](./prompts) in `carbonsifr/prompts/` — upload those first.
3. Run all. The first cell mounts Drive (approve the popup), and the install cell auto-restarts the runtime once. This is expected: Colab ships an older `pyarrow` that has to be reloaded. When it reconnects, just Run all again and it flows straight through.

Two flags at the top of the notebook control how much it rebuilds:

- `RETRAIN = False` (default) loads the trained adapter from Drive, so a full pass is about 10 minutes. Set it `True` to retrain the LoRA from the committed split (~40 min on an A100).
- `REGENERATE_SILVER = False` (default) loads the cached silver labels. Set it `True` to regenerate them from scratch — note this needs the Qwen-32B teacher pass and the manual Gemini labeling step (done by hand in AI Studio), so it isn't a single-click path.

The cached silver labels, the splits, and the trained adapter live on my Drive — they're derived from the CarbonSifr data, so I don't redistribute them in the repo. A reviewer with the original `Data.xlsx` can regenerate everything end-to-end via the two flags above (`REGENERATE_SILVER=True` rebuilds the labels and splits; `RETRAIN=True` retrains the adapter).

---

## Repository structure

```
.
├── README.md                            # this file
├── AI_USAGE.md                          # documented AI assistance throughout the project
├── writeup.md                           # Task 2 — approach & findings
├── CarbonSifr_EXIOBASE_FineTune.ipynb   # the notebook, committed with all cell outputs
└── prompts/
    ├── rule_book.md                     # ~50 rules tagged [GOLD] or [EXTENDED]
    └── few_shot.md                      # 3 synthetic worked examples
```

The notebook also generates, into a Google Drive `carbonsifr/` folder, the assembled `system_prompt.txt`, the train/val/test splits, and the prediction/metric CSVs. Those derive from the CarbonSifr-provided `Data.xlsx`, so, like `Data.xlsx` itself, they are not redistributed in this repo; they are visible inline in the committed notebook's outputs.

---

## Methodology highlights

- The 20 provided gold labels are never used for training. They are the held-out test set and the rule-book extraction source.
- One teacher plus two independent reviewers: Qwen-32B labels all 600 items under the rule book; Claude audits a random 100 (62% agreement, label-quality signal only); Gemini 2.5 Pro labels the other 500 (49% agreement), with the 254 disagreements individually adjudicated.
- Frozen, leakage-free split: the 498/51/51 train/val/test partition is frozen to disk, so the adapter is trained and evaluated on disjoint sets and the reported silver numbers are a true held-out measurement, reproducible across runs.
- Rule book tagged for provenance: every rule is either `[GOLD]` (derivable from one of the 20 labels) or `[EXTENDED]` (added from public EXIOBASE definitions).
- Hierarchical metrics: accuracy reported at both activity (83-class) and sector (16-class) levels, since sector-level errors are far less harmful in production.
- Embedding-similarity baseline included as a non-LLM reference point.

---

## Tech stack

| Component | Choice |
|---|---|
| Student model (fine-tuned) | Qwen2.5-14B-Instruct (4-bit, LoRA) |
| Teacher (silver labeler) | Qwen2.5-32B-Instruct |
| Reviewer 1 — audit (100 items) | Claude (independent re-labeling under same rule book) |
| Reviewer 2 — cross-check (500 items) | Gemini 2.5 Pro (independent labeling, disagreements adjudicated) |
| Embedding baseline | BAAI/bge-large-en-v1.5 |
| Fine-tuning framework | Unsloth + TRL `SFTTrainer` |
| Compute | Colab Pro, NVIDIA A100 40 GB |

---

This project used AI assistance throughout — see [`AI_USAGE.md`](./AI_USAGE.md) for a full breakdown of what was delegated to AI tools and how outputs were verified.
