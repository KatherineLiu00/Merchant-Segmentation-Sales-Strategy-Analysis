#  Merchant Segmentation Sales Strategy Analysis —— BA Case

A business analysis case study by Katherine Liu. The core workflow lives in a Jupyter Notebook: clean, enrich, and segment **Australian merchant lead data**, then build a sales canvassing strategy with an interactive dashboard.

**Main file:** [`BA Case Study.ipynb`](BA Case Study.ipynb)

---

## Data Scope: Australia

All merchant records in this project are based in **Australia**. Key geographic characteristics:

- **Country:** Australia — phone validation, state abbreviations, and suburb tiering all follow Australian rules
- **States/territories covered:** NSW, VIC, QLD, WA, SA, ACT, TAS, NT (8 official jurisdictions)
- **NSW dominates the dataset** — after cleaning, **NSW (New South Wales) accounts for the largest share of merchants (~58%)**, roughly 3× more than the second-largest state (VIC)

**State distribution (cleaned data, `Case Study_cleaned.csv`):**

| State | Merchants | Share |
|-------|-----------|-------|
| **NSW** | ~108,524 | **~58%** |
| VIC | ~32,757 | ~18% |
| QLD | ~21,386 | ~11% |
| WA | ~9,514 | ~5% |
| SA | ~7,751 | ~4% |
| ACT | ~2,253 | ~1% |
| TAS | ~1,681 | ~1% |
| NT | ~576 | <1% |

> NSW-heavy distribution is reflected throughout the pipeline — tiering, suburb analysis, and the Part 3 dashboard are all skewed toward New South Wales merchants.

---

## CSV File Relationships

The project contains **5 CSV files** that form a single data pipeline. Each downstream file is derived from the previous one, and all files are linked by the `id` column for the same merchant record.

```
Case Study.csv                    ← Only required raw input
        │
        │  Part 1: Data cleaning (phone, name, state, suburb, address, industry, dedup)
        ▼
Case Study_cleaned.csv            ← Cleaned dataset (same column structure)
        │
        │  Part 1: Data enrichment (merchant size + location tier)
        ▼
Case Study_cleaned_enriched.csv   ← Adds 4 analytical columns
        │
        │  Google Maps API fetch (Large + High location merchants only)
        │  ├─ Progress saved ──► CaseStudy_google_progress.csv (intermediate)
        │  └─ Ratings merged back into enriched
        ▼
Case Study_cleaned_enriched.csv   ← Updated google_rating / google_rating_count
        │
        │  Part 2: Merchant tiering (Tier 1–6)
        ▼
merchant_with_tier.csv            ← Final dataset for analysis
        │
        │  Part 3: Performance dashboard (charts shown in Notebook only)
        ▼
     Interactive dashboard
```

---

## CSV File Reference

### 1. `Case Study.csv` — Raw Input

| Attribute | Description |
|-----------|-------------|
| **Role** | The only required raw input for the entire project |
| **Geography** | Australian merchants only; **NSW is the largest state** (~51% of raw rows) |
| **Rows** | ~218,557 (including header) |
| **Columns** | 10 |

| Column | Description |
|--------|-------------|
| `id` | Unique merchant ID (primary key across the pipeline) |
| `lead_key` | Lead identifier |
| `phone` | Australian phone number (mobile/landline formats validated in cleaning) |
| `business_name` | Business name |
| `state` | Australian state or territory (NSW, VIC, QLD, WA, SA, ACT, TAS, NT) |
| `suburb` | Australian suburb |
| `address` | Street address |
| `sector_level_1` | Industry level 1 |
| `sector_level_2` | Industry level 2 |
| `sector_level_3` | Industry level 3 |

---

### 2. `Case Study_cleaned.csv` — Cleaned Data

| Attribute | Description |
|-----------|-------------|
| **Source** | `Case Study.csv` after Part 1 cleaning pipeline |
| **Rows** | ~186,332 (~32,000 invalid/duplicate rows removed) |
| **Columns** | 10 (same as raw, no new fields) |

**Cleaning steps:**
- Phone validation (Australian mobile and landline rules)
- Business name: remove nulls, garbled text, numeric-only, punctuation-only values
- State standardization to 8 official abbreviations (NSW, VIC, QLD, etc.)
- Suburb, address, and industry column formatting
- Deduplication by `lead_key`, or `business_name + address` as fallback

---

### 3. `Case Study_cleaned_enriched.csv` — Enriched Data

| Attribute | Description |
|-----------|-------------|
| **Source** | `Case Study_cleaned.csv` + size/location inference + Google rating merge |
| **Rows** | ~186,332 (one-to-one with cleaned) |
| **Columns** | 14 (original 10 + 4 new fields) |

**New columns:**

| Column | Values | How it is derived |
|--------|--------|-------------------|
| `merchant_size_estimate` | Small / Medium / Large | Keyword matching on `sector_level_1` |
| `location_tier` | High / Medium / Low | Suburb-to-tier mapping dictionary |
| `google_rating` | Numeric or empty | Google Maps Places API |
| `google_rating_count` | Numeric or empty | Google Maps Places API |

> Only merchants with `merchant_size_estimate = Large` and `location_tier = High` are queried via Google API. All other merchants have empty (NaN) rating fields.

---

### 4. `CaseStudy_google_progress.csv` — Google Fetch Progress (Intermediate)

| Attribute | Description |
|-----------|-------------|
| **Role** | Intermediate checkpoint, not a final deliverable |
| **Source** | Auto-saved every 500 rows during Google API fetch |
| **Rows** | ~2,848 (target merchant subset only, not the full dataset) |
| **Columns** | 14 (same as enriched, without `merchant_tier`) |

**Purpose:**
- Prevents data loss if the API fetch is interrupted
- After fetch completes, `google_rating` and `google_rating_count` are merged back into `Case Study_cleaned_enriched.csv` via `id`

> If this file already exists, you can skip re-running the Google API cell and run the merge cell directly.

---

### 5. `merchant_with_tier.csv` — Final Tiered Data (Analysis Entry Point)

| Attribute | Description |
|-----------|-------------|
| **Source** | `Case Study_cleaned_enriched.csv` + Part 2 merchant tiering |
| **Rows** | ~186,332 (same as enriched) |
| **Columns** | 15 (enriched 14 + 1 new field) |

**New column:**

| Column | Values | Description |
|--------|--------|-------------|
| `merchant_tier` | Tier 1 – Tier 6 | Priority tier based on size, location, and Google ratings |

**Tier rules summary:**

| Tier | Criteria | Priority |
|------|----------|----------|
| Tier 1 | Large + High location + rating ≥ 4 + review count ≥ 20 | Highest |
| Tier 2 | Large + High location + rating or review count below threshold | High |
| Tier 3 | Large + High location + incomplete rating data | High |
| Tier 4 | Large/Medium + Medium/High location combinations | Medium |
| Tier 5 | Large/Low, Medium/Medium, Small/Medium | Low |
| Tier 6 | Medium/Low, Small/Low | Lowest |

> **Lower tier number = higher outreach priority.** Part 3 dashboard reads this file directly.

---

## File Relationship Summary

| File | Required? | Rows | Cols | Upstream | Downstream |
|------|-----------|------|------|----------|------------|
| `Case Study.csv` | ✅ Yes | ~218,557 | 10 | — | → cleaned |
| `Case Study_cleaned.csv` | Auto-generated | ~186,332 | 10 | Case Study.csv | → enriched |
| `Case Study_cleaned_enriched.csv` | Auto-generated | ~186,332 | 14 | cleaned + progress | → merchant_with_tier |
| `CaseStudy_google_progress.csv` | Optional intermediate | ~2,848 | 14 | Google API | → merged into enriched |
| `merchant_with_tier.csv` | Auto-generated | ~186,332 | 15 | enriched | → Part 3 dashboard |

**Join key:** All files link the same merchant record via the `id` column.

---

## Quick Start

### Install dependencies

```bash
pip install -r requirements.txt
```

### Run order

1. Place `Case Study.csv` in the project root directory
2. Open the Notebook and run cells in order:
   - **Part 1 Cleaning** → generates `Case Study_cleaned.csv`
   - **Part 1 Enrichment** → generates `Case Study_cleaned_enriched.csv`
   - **Google API** (optional — use existing `CaseStudy_google_progress.csv` instead)
   - **Part 2 Tiering** → generates `merchant_with_tier.csv`
   - **Part 3 Dashboard** → view charts inside the Notebook

### Just want to see the analysis?

If all CSV files are already present, open the Notebook and start from Part 2 or Part 3 — no need to re-run the cleaning pipeline.

---

## Project Structure

```
.
├── README.md                                      # This file
├── requirements.txt                               # Python dependencies
├── Case Study.csv                                 # Raw input
├── Case Study_cleaned.csv                         # Cleaning output
├── Case Study_cleaned_enriched.csv                # Enrichment output
├── CaseStudy_google_progress.csv                  # Google API progress checkpoint
├── merchant_with_tier.csv                         # Final tiered dataset
└── BA Case Study.ipynb  # Main analysis Notebook
```
