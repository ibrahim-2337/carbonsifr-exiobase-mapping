# Few-Shot Examples for EXIOBASE Mapping

These 3 examples are **synthetic** (not drawn from the 20 CarbonSifr gold labels) to avoid contaminating the held-out test set. Each demonstrates a different aspect of the rule book: a clear material rule, a service vs. material judgment call, and a low-confidence under-specified case.

---

### Example
Item: A4 White Copy Paper, 80gsm, 500 sheets per ream
Buyer category: Operating Supplies & Equipments / Sub-category: OFFICE STATIONERY

Output:
{
  "activity": "Paper and paper products",
  "reason": "The item is a single-material paper product. Per the material rule, single-material paper items map to 'Paper and paper products'.",
  "confidence_level": 0.95
}

---

### Example
Item: Office interior repainting service - 4 walls, neutral white finish
Buyer category: Facility Management / Sub-category: BUILDING MAINTENANCE

Output:
{
  "activity": "Construction work (45)",
  "reason": "The item describes a painting service (labor), not paint material. Per the materials-vs-work rule, construction and renovation services map to 'Construction work (45)'. The paint material itself would map to 'Chemicals nec', but this item is the service.",
  "confidence_level": 0.85
}

---

### Example
Item: Miscellaneous expense reimbursement
Buyer category: Operating Supplies & Equipments / Sub-category: unspecified

Output:
{
  "activity": "Other services (93)",
  "reason": "Description is materially under-specified - 'miscellaneous expense' provides no product or service detail. Per the fallback hierarchy, with no specific match available, 'Other services (93)' is used as a last resort. Confidence is low per the under-specified rule.",
  "confidence_level": 0.3
}
