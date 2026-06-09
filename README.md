# CarbonSifr — EXIOBASE Emission-Factor Activity Mapper

Fine-tuning **Qwen2.5-14B-Instruct** with **LoRA** to map procurement items to one of 83 EXIOBASE emission-factor activities. Submission for the CarbonSifr AI Engineering Intern case study (June 2026).

For each procurement item, the model outputs structured JSON:

```json
{
  "activity": "Paper and paper products",
  "reason": "The item is a single-material paper product. Per the material rule, single-material paper items map to 'Paper and paper products'.",
  "confidence_level": 0.95
}
```

---

## Quick links

- **Colab notebook (end-to-end):** [open in Colab](#) <!-- TODO: paste public Colab URL here -->
- **LoRA adapter on HF Hub:** [`retroib/qwen2.5-14b-exiobase-lora`](https://huggingface.co/retroib/qwen2.5-14b-exiobase-lora) <!-- TODO: confirm URL after push -->
- **Writeup (Task 2):** [`writeup.pdf`](./writeup.pdf)
- **AI usage log:** [`AI_USAGE.md`](./AI_USAGE.md)

---

## Approach (one-paragraph)

A rule book of ~50 EXIOBASE classification rules was extracted from the 20 provided gold-labeled examples (tagged `[GOLD]`) and extended with EXIOBASE 3.10.1 / 3.11.1 activity definitions (tagged `[EXTENDED]`) to cover the 63 activities not represented in the gold set. The rule book + 3 synthetic few-shot examples + the full 83-activity catalog form the system prompt. Two open-source teachers from different model families — **Qwen2.5-32B-Instruct** (Alibaba) and **Yi-1.5-34B-Chat** (01.AI) — independently labeled 600 stratified procurement items. Only items where both teachers agreed on `activity` were kept as the silver training set (~510 items after filtering), preserving cross-family bias filtering while keeping the pipeline fully open-source. The Qwen2.5-14B-Instruct student was then fine-tuned via LoRA (Unsloth + TRL) for 2 epochs and evaluated against the 20 CarbonSifr-provided gold labels (headline metric) and ~50 silver test items (secondary).

---

## Results

| Metric | Embedding NN | Qwen-14B base (zero-shot) | Qwen-14B + LoRA |
|---|---|---|---|
| Exact-match accuracy (gold-20) | TBD | TBD | TBD |
| Sector-match accuracy (gold-20) | TBD | TBD | TBD |
| Exact-match (silver test) | TBD | TBD | TBD |
| JSON parse rate | — | TBD | TBD |
| Reason quality (LLM judged, 1–5) | — | TBD | TBD |
| Expected calibration error (ECE) | — | TBD | TBD |

<!-- TODO: fill after Phase 7 evaluation -->

See [`writeup.pdf`](./writeup.pdf) for analysis of failure modes, production recommendation, and future work.

---

## How to reproduce

The Colab notebook runs end-to-end on a single A100 (40 GB) instance and produces all artifacts.

1. Open the Colab notebook ([link above](#quick-links))
2. Runtime → Change runtime type → **A100 GPU**
3. Mount Google Drive when prompted (a `carbonsifr/` folder is created in your Drive root)
4. Upload `Data.xlsx` to `MyDrive/carbonsifr/data/`
5. Upload the contents of [`prompts/`](./prompts) to `MyDrive/carbonsifr/prompts/`
6. Run all cells (Runtime → Run all)

Total wall-clock: ~3–4 hours end-to-end.

---

## Repository structure

```
.
├── README.md                  # this file
├── AI_USAGE.md                # documented AI assistance throughout the project
├── writeup.pdf                # Task 2 — 1–2 page analysis
├── CarbonSifr_EXIOBASE_FineTune.ipynb   # the Colab notebook (top-to-bottom reproducible)
├── prompts/
│   ├── rule_book.md           # 50+ rules tagged [GOLD] or [EXTENDED]
│   ├── few_shot.md            # 3 synthetic worked examples
│   ├── system_prompt.txt      # assembled system prompt used at inference
│   └── user_template.txt      # per-item user message template
├── artifacts/
│   ├── comparison_table.csv   # headline metrics across all 3 systems
│   ├── sector_confusion.png   # sector-level confusion matrix (fine-tuned)
│   └── failure_cases.md       # qualitative analysis of 5 failure cases
├── data/
│   └── splits/
│       ├── train.csv          # cross-checked silver labels used for training
│       ├── val.csv            # held-out validation for epoch selection
│       └── test_gold.csv      # the 20 CarbonSifr gold labels (headline test set)
└── requirements.txt
```

---

## Methodology highlights

- **20 provided gold labels are never used for training.** They are the held-out test set + rule-book extraction source.
- **Cross-family two-teacher labeling** (Qwen + Yi) filters single-model labeling errors.
- **Rule book is tagged for provenance** — every rule is either `[GOLD]` (derivable from one of the 20 labels) or `[EXTENDED]` (added from public EXIOBASE definitions). No hidden assumptions.
- **Hierarchical metrics** — accuracy reported at both activity (83-class) and sector (16-class) levels; sector-level errors are far less harmful in production.
- **Embedding-similarity baseline** included as a non-LLM reference point.
- **Calibration reported** — expected calibration error and reliability diagram, not just accuracy.

---

## Tech stack

| Component | Choice |
|---|---|
| Student model (fine-tuned) | Qwen2.5-14B-Instruct (4-bit, LoRA) |
| Teacher 1 (primary labeler) | Qwen2.5-32B-Instruct |
| Teacher 2 (cross-check) | Yi-1.5-34B-Chat (01.AI) |
| Embedding baseline | BAAI/bge-large-en-v1.5 |
| Fine-tuning framework | Unsloth + TRL `SFTTrainer` |
| Compute | Colab Pro, NVIDIA A100 40 GB |

---

## License

Code in this repository is licensed under the MIT License — see `LICENSE` (if present).
The fine-tuned LoRA adapter inherits the base model's license (Qwen2.5 Community License). The underlying CarbonSifr-provided data and rule book are not redistributed.

---

## Acknowledgments

CarbonSifr provided the case study brief, the 20 gold-labeled examples, and the 2,000-item procurement pool. EXIOBASE provides the activity taxonomy used for the classification targets.

This project used AI assistance throughout — see [`AI_USAGE.md`](./AI_USAGE.md) for a full breakdown of what was delegated to AI tools and how outputs were verified.
