# Pharma Product Matcher

Match messy medicine names to the exact product in Finland's official Kela medicine list, using multilingual embeddings, blocking, pricing-aware business rules and an LLM second opinion for uncertain cases. Evaluated on held-out data, with every change traced to an error analysis.

| Version | What it adds | Precision | Recall |
|---|---|---|---|
| v1 | embeddings + blocking + rules (held-out: frozen, evaluated once on 285 unseen products) | **96%** | **88%** |
| v1.1 | strength-parser fixes + LLM judge for "not sure" cases (same 285 products\*) | **99.6%** | **99.6%** |

\* v1.1 was designed by studying v1's errors on this set, so the set is no longer truly held-out and these numbers are optimistic. See [Evaluation](#evaluation).

[Open the notebook in Colab](https://colab.research.google.com/github/AadarshKr-11/pharma-product-matcher/blob/main/pharma_matcher.ipynb)

---

## Why this problem

Before you can compare medicine prices across sources or countries, you have to know that two records describe **the same product**: same brand, strength, form and pack size. Names rarely match exactly (`burana 400mg tabl 30` vs `BURANA 400 mg tabletti, kalvopäällysteinen 30 fol läpipainopakkaus`), and a wrong match puts a **wrong price** into the comparison. This project automates that matching step and measures how far it can be trusted.

## How it works

```
messy name → normalise → embed → block by active substance → rank by cosine similarity
          → strength rule → pack rule → name rule → threshold → match / not sure / no match
          → "not sure" only → LLM judge (Gemini, JSON verdict, validated) → final decision
```

| Step | What it does | Why |
|---|---|---|
| Normalise | lowercase, `400mg` → `400 mg`, `tabl` → `tabletti` | puts the query in Kela's vocabulary before embedding |
| Embed | `paraphrase-multilingual-MiniLM-L12-v2` (384-dim) | the data is Finnish; an English-only model was tried first and replaced |
| Block | keep only rows with the same active substance | faster, and makes cross-substance errors impossible |
| Rank | cosine similarity, whole substance block kept | a top-10 cut-off once dropped the correct product (retrieval recall) |
| Strength rule | same strength **and unit**, every number in a combination (`5/160 mg` ≠ `10/160 mg`; `25 mg` ≠ `25 mikrog`) | embeddings scored 400 mg vs 600 mg at 0.98 |
| Pack rule | same pack count | a 10-pack and a 100-pack have different prices |
| Name rule | every word of the product name must appear in the query; prefer the most specific name | embeddings confused brands (Panadol vs Pamol) and variants (Panadol vs Panadol Zapp) |
| Thresholds | ≥ 0.75 match · < 0.50 no match · otherwise not sure | uncertain cases get a second opinion instead of becoming silent errors |
| LLM judge | only "not sure" cases → Gemini returns `{"match", "confidence", "reason"}` | rules handle ~95% of cases cheaply; the LLM is used only where it adds value |

**Key finding:** embeddings handle language (abbreviations, word order, Finnish), but barely distinguish numbers and similar brand names. Business rules handle those, and the LLM handles the uncertain middle.

## The LLM step

- **Only uncertain cases:** 16 calls instead of 344 (cost, speed, and a free-tier limit of 20 calls/day per model).
- **Reproducible:** pinned model (`gemini-3.5-flash-lite`), temperature 0, JSON output mode.
- **Never trusted blindly:** every reply is parsed and validated (valid JSON, `match` is true/false, `confidence` between 0 and 1). Missing or invalid answers keep the case as "not sure"; the system never guesses.
- **Decision rule:** match with confidence ≥ 0.8 → match; LLM says no → no match; otherwise not sure.
- **Robust batch:** server errors are retried with exponential backoff; quota errors stop the batch cleanly; finished rows are skipped on re-run; verdicts are cached in `tests_with_llm.csv`, so the notebook runs end to end without an API key.

## Evaluation

**Test set.** Each Kela row is turned into a messy query (`make_messy()`), so the correct answer is known. A match counts as correct if product, strength, form and pack count agree. Blister and bottle packs with the same count are treated as equal, because the data showed identical prices.

**Dev vs held-out.** The v1 rules were built while looking at errors on a development set (Burana, Panadol, Pamol). The system was then **frozen**, 285 products from 22 new medicines were scraped, and the evaluation was run **once**: 96% precision, 88% recall.

**v1 improvement loop (dev set, 59 products):**

| Version | Change (each from a pattern in the errors) | Precision | Recall |
|---|---|---|---|
| v1 | first run | 88% | 65% |
| v2 | fixed pack parsing; full product names in the tests | 92% | 67% |
| v3 | name rule (brand and variant) | 97% | 67% |
| v4 | retrieve the whole substance block instead of the top 10 | 100% | 67% |
| v4, threshold 0.75 | match threshold chosen before the held-out run | 100% | 97% |

**v1.1 (unseen set, now used for error analysis\*):**

| Version | Change | Precision | Recall | Wrong |
|---|---|---|---|---|
| v1 | frozen, evaluated once | 96% | 88% | 11 |
| v1.1a | combination strengths in the strength parser | 99.6% | 92.3% | 1 |
| v1.1b | units (mikrog, g, elisa u) | 99.6% | 94.7% | 1 |
| v1.1c | LLM judge on the 14 "not sure" cases | 99.6% | 99.6% | 1 |

Dev set after v1.1: 100% precision, 100% recall.

## Error analysis

**v1 (11 wrong matches on unseen data):**
- **10 combination strengths** (`5/160 mg` vs `10/160 mg`). The strength regex read only the number before "mg" (`160`), in both the matcher **and the test generator**, so 5/160 and 10/160 products even produced identical test queries. Fixed in v1.1a.
- **1 device variant** (Nucala pre-filled pen vs syringe, same strength and pack). Still open: the query doesn't say which device.

**After v1.1a:** 16 "no match" cases were all non-mg units (`27,5 mikrog`, `1 g`, `1440 elisa u`), where the pack size was misread. Fixed in v1.1b.

**After v1.1b:** 14 "not sure" cases, all actually correct, scoring 0.73–0.75 (just under the threshold). The LLM judge confirmed all 14, with no new wrong matches.

Lowering the threshold would also have accepted them, but that would mean tuning a number on the set being studied (overfitting). The LLM is a second check rather than blind acceptance.

## Data quality issues found along the way

- VNR codes lost leading zeros in Excel → read as text, `zfill(6)`
- An invisible zero-width space inside product text → found with `repr()`, removed
- Prices scaled ×100 by decimal-comma parsing → caught by checking against Kela
- Product type ("prescription medicine") mixed into product names → moved to its own column

## Limitations and next steps

1. **Rejection test for the LLM judge.** So far it has only seen correct candidates; a judge that always says "yes" would score the same. Next: deliberately wrong pairs (wrong pack, wrong combination strength, wrong variant).
2. **Fresh held-out test set** for honest v1.1 numbers.
3. **Device variants** (pen vs syringe) are not captured by the rules.
4. **Synthetic queries** are tidier than real data. Next: match against a second real source (France, BDPM open data).
5. **Brand typos:** the name rule needs exact spelling; add fuzzy matching.
6. **v1.2: RAG price Q&A** on the same data (e.g. "cheapest 5 mg amlodipine 30-pack?"), answering only from retrieved rows and citing VNR codes.

## How to run

1. Open `pharma_matcher.ipynb` in Google Colab (link above).
2. Upload `finland_KELA.xlsx` and `tests_with_llm.csv` from this repository.
3. **Runtime → Run all.** By default (`RUN_LLM = False`) the notebook loads the cached LLM verdicts, so **no API key is needed**.
4. Optional, to re-run the LLM step: add a Gemini API key to Colab Secrets as `GEMINI_API_KEY` and set `RUN_LLM = True` in Section 13.7.

`requirements.txt` lists the packages if you run it locally.

## Data and disclaimer

Medicine data from Kela's public medicine price database (kela.fi), collected for this personal learning project. This is not affiliated with Kela or any employer, and no employer data or tools were used. Not for clinical or commercial decisions.

*Aadarsh Kumar · [LinkedIn](https://www.linkedin.com/in/adz11) · [GitHub](https://github.com/AadarshKr-11)*
