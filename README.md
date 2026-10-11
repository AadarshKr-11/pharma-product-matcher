# Pharma Product Matcher

Match messy medicine names to the exact product in Finland's official Kela medicine list, using multilingual embeddings, blocking and pricing-aware business rules. Evaluated on held-out data.

**Result on 285 unseen products (22 medicines): 96% precision, 88% recall.**

[Open the notebook in Colab](https://colab.research.google.com/github/AadarshKr-11/pharma-product-matcher/blob/main/pharma_matcher.ipynb)

---

## Why this problem

Before you can compare medicine prices across sources or countries, you have to know that two records describe **the same product**: same brand, strength, form and pack size. Names rarely match exactly (`burana 400mg tabl 30` vs `BURANA 400 mg tabletti, kalvopäällysteinen 30 fol läpipainopakkaus`), and a wrong match puts a **wrong price** into the comparison. This project automates that matching step and measures how far it can be trusted.

## How it works

```
messy name → normalise → embed → block by active substance → rank by cosine similarity
          → strength rule → pack rule → name rule → threshold → match / not sure / no match
```

| Step | What it does | Why |
|---|---|---|
| Normalise | lowercase, `400mg` → `400 mg`, `tabl` → `tabletti` | puts the query in Kela's vocabulary before embedding |
| Embed | `paraphrase-multilingual-MiniLM-L12-v2` (384-dim) | the data is Finnish; an English-only model was tried first and replaced |
| Block | keep only rows with the same active substance | faster, and makes cross-substance errors impossible |
| Rank | cosine similarity, whole substance block kept | a top-10 cut-off once dropped the correct product (retrieval recall) |
| Strength rule | same mg in query and product | embeddings scored 400 mg vs 600 mg at 0.98 |
| Pack rule | same pack count | a 10-pack and a 100-pack have different prices |
| Name rule | every word of the product name must appear in the query; prefer the most specific name | embeddings confused brands (Panadol vs Pamol) and variants (Panadol vs Panadol Zapp) |
| Thresholds | ≥ 0.75 match · < 0.50 no match · otherwise not sure | uncertain cases go to review instead of becoming silent errors |

**Key finding:** embeddings handle language (abbreviations, word order, Finnish), but they barely distinguish numbers and similar brand names. Business rules handle those.

## Evaluation

**Test set.** Each Kela row is turned into a messy query (`make_messy()`), so the correct answer is known. A match counts as correct if product, strength, form and pack count agree. Blister and bottle packs with the same count are treated as equal, because the data showed identical prices.

**Dev vs held-out.** The rules were built while looking at errors on a development set (Burana, Panadol, Pamol). The system was then **frozen**, 285 products from 22 new medicines were scraped, and the evaluation was run **once**.

| Set | Products | Precision | Recall |
|---|---|---|---|
| Development (used to build the rules) | 59 | 100% | 97% |
| **Held-out (never seen)** | **285** | **96%** | **88%** |

**Improvement loop on the dev set:**

| Version | Change (each from a pattern in the errors) | Precision | Recall |
|---|---|---|---|
| v1 | first run | 88% | 65% |
| v2 | fixed pack parsing when pack = strength (`50 mg/g`, `50 g`); full product names in the tests | 92% | 67% |
| v3 | name rule (brand and variant) | 97% | 67% |
| v4 | retrieve the whole substance block instead of the top 10 | 100% | 67% |
| v4, threshold 0.75 | match threshold chosen before the held-out run | 100% | 97% |

Lowering the match threshold from 0.85 to 0.75 roughly doubled recall at a cost of about 2 points of precision.

## Error analysis (held-out)

All 11 wrong matches came from **strength formats the rule was not designed for**:

- **10 combination strengths** (`5/160 mg` vs `10/160 mg`; `100/12,5 mg` vs `50/12,5 mg`). The regex reads one number, so variants cannot be separated. In pricing these are different products with different prices, so this is the most important fix.
- **1 device variant** (pre-filled pen vs pre-filled syringe, same strength and pack).

None came from retrieval, blocking, the pack rule or the name rule. (An earlier run showed 3 extra errors caused by a test-generator bug that wrote non-mg strengths as "None". Fixing the generator, not the matcher, removed them.)

## Data quality issues found along the way

- VNR codes lost leading zeros in Excel → read as text, `zfill(6)`
- An invisible zero-width space (`​`) inside product text → found with `repr()`, removed
- Prices scaled ×100 by decimal-comma parsing → caught by checking against Kela
- Product type ("prescription medicine") mixed into product names → moved to its own column

## Limitations and next steps

1. **Strength parser** for units (mg / mikrog / g / IU), combinations (`a/b`) and decimal commas: regex first, LLM fallback for unusual formats.
2. **LLM second opinion** for "not sure" cases, with structured JSON output and validation.
3. **Brand typos:** the name rule needs exact spelling; add fuzzy matching.
4. **Synthetic queries** are tidier than real data. Next: match against a second real source (France, BDPM open data) for cross-country comparison.
5. **RAG price Q&A** on the same data (e.g. "cheapest 5 mg amlodipine 30-pack?"), answering only from retrieved rows and citing VNR codes.

## How to run

1. Open `pharma_matcher.ipynb` in Google Colab (link above).
2. Upload `finland_KELA.xlsx` from this repository.
3. **Runtime → Run all.** Section 10 prints the dev and held-out results.

`requirements.txt` lists the packages if you run it locally.

## Data and disclaimer

Medicine data from Kela's public medicine price database (kela.fi), collected for this personal learning project. This is not affiliated with Kela or any employer, and no employer data or tools were used. Not for clinical or commercial decisions.

*Aadarsh Kumar · [LinkedIn](https://www.linkedin.com/in/adz11) · [GitHub](https://github.com/AadarshKr-11)*
