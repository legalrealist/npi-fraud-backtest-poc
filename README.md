# Medicare Fraud Backtest — Can You Spot Fraud in Billing Data Before Exclusion?

Excluded Medicare providers bill differently from their peers — and those differences are visible **in the year before they're caught**. This proof-of-concept backtest quantifies those differences and builds a predictive model that flags providers with billing patterns similar to known fraudsters.

The question for compliance teams, qui tam relators, and enforcement agencies: if the signal is this clear in public data, why wait for convictions?

Companion to the [LegalRealist AI Landscape](https://legalrealist.ai) series.

## The headline numbers

- **289 excluded providers** matched to pre-exclusion Medicare Part B billing, compared against **3.39 million peers**
- **14 of 15 billing features** statistically significant after Bonferroni correction
- **Predictive model AUC: 0.79** — reliably distinguishes excluded from non-excluded billing patterns
- **20% of top-scoring peers** had confirmed enforcement actions in public records — cases the exclusion system missed

## What the fraud fingerprint looks like

Excluded providers don't just bill more — they bill *differently*:

| Signal | What it means | Effect size |
|--------|--------------|-------------|
| **Dual-eligible share** (d=+0.78) | Serve far more Medicaid-Medicare patients — a population with less ability to question billing | Large |
| **Top procedure concentration** (d=+0.66) | 30% of billing from a single procedure code vs. 16% for peers | Large |
| **Service mix concentration** (d=+0.52) | Less diverse service offerings overall | Medium |
| **Younger patients** (d=−0.31) | Younger patient panels than specialty peers | Small–Medium |
| **Lower charges, higher volume** (d=−0.23) | More services at lower per-unit charges — the high-volume mill pattern | Small |
| **Higher-risk patients** (d=+0.18) | Higher average risk scores (only visible after filtering to fraud-specific exclusions) | Small |

These signals survive practice-size controls — 11 of 15 features remain significant after binning by patient volume. The core concentration metrics actually get *stronger* after adjustment.

## Who the model catches that the exclusion system misses

A public records search (DOJ, OIG, state medical boards) of the 50 highest-scoring peers — providers flagged by the model but not in the exclusion database. **6 of 30 searched (20%) had confirmed enforcement actions:**

- **Phlebxpress** (CA) — owners convicted of $7M Medicare fraud. Company NPI stayed active because only individuals were excluded.
- **Advanced Clinical Laboratories** (FL) — OIG Corporate Integrity Agreement for False Claims Act violations
- **Hemal Mehta** (TN) — DOJ indictment, 10 counts conspiracy to distribute controlled substances
- **Natera, Inc.** (CA) — qui tam complaint + $8.25M settlement
- **CareDx, Inc.** (CA) — DOJ investigation + whistleblower lawsuit
- **Stephen Dubin** (NV) — two state medical board complaints

All 4 clinical laboratories in the top 50 had enforcement actions. The model finds entity NPIs still active after individual owners are excluded, providers under monitoring instead of exclusion, and indicted providers awaiting conviction.

## DOJ prosecution cross-reference

A search of 43 excluded providers against DOJ press releases found **17 (40%) matched** to published federal prosecutions — sentences from 15 months to 84 months, fraud amounts from $2.5M to $110M.

High match-rate states (TX 80%, NJ 75%, NY 67%) are Medicare Fraud Strike Force jurisdictions. Low match-rate states (OH 14%, CA 0%) are likely prosecuted at the state level by AG offices. All matched prosecutions with DOJ links are in [`report.md`](report.md).

## What exclusion types matter (and why filtering is critical)

The cohort is restricted to fraud-related exclusions under [Section 1128 of the Social Security Act](https://oig.hhs.gov/exclusions/authorities.asp):

| Code | Description | Count |
|------|-------------|-------|
| §1128(a)(1) | Program fraud conviction | 1,221 |
| §1128(a)(3) | Felony healthcare fraud conviction | 300 |
| §1128(b)(7) | Excessive claims / unnecessary services | 78 |

The cohort excludes drug felonies (§1128(a)(4)), license revocations (§1128(b)(4)), and patient abuse (§1128(a)(2)) — these aren't billing-pattern fraud. This filtering matters: with all exclusion types mixed in, the patient risk score showed zero signal (d=0.00). After restricting to fraud types, it became significant (d=+0.18). Non-fraud exclusions were masking the signal.

## Data sources

All publicly available, no API keys required:

| Source | What it provides |
|--------|-----------------|
| [OIG LEIE](https://oig.hhs.gov/exclusions/downloadables/UPDATED.csv) | Excluded provider labels (83K+ records) |
| [CMS Part B Provider](https://data.cms.gov) | Provider-level billing aggregates |
| [CMS Part B Service](https://data.cms.gov) | HCPCS procedure-level billing detail |

## Running the pipeline

```bash
pip install pandas scipy matplotlib requests pyarrow scikit-learn
python backtest.py
```

First run downloads ~4GB of CMS data (cached as parquet). Subsequent runs use cache. Outputs: `report.md`, figures in `figures/`, validation data in `data/`.

The pipeline: download LEIE → build fraud-specific excluded cohort (1,605 providers) → find best pre-exclusion billing year per provider (289 matched) → compute 15 features → statistical comparison (Mann-Whitney U, Welch's t, Cohen's d, Bonferroni) → logistic regression risk model (AUC=0.79) → DOJ cross-reference → visualizations.

## Iteration history

The pipeline went through six iterations to reach the current design — from zero NPI overlap with DMEPOS data, to a Florida-only pilot with 28 providers, to the current all-states fraud-filtered cohort. Each iteration is documented in [`report.md`](report.md).

## Caveats

- **82% of excluded NPIs** didn't match any CMS Part B data — program siloing, suppression thresholds, NPI mismatches
- Peer matching is by state/specialty/year — not by practice size or sub-state geography
- Effect sizes are population-level statistical differences, not individual-level diagnostic criteria
- Billing data spans 2017–2022; COVID-era billing shifts are partially mitigated by year-matching but not eliminated
- The 0.79 AUC means the model is informative but not definitive — it identifies patterns worth investigating, not guilt

## License

All source data is public. Scripts and analysis are open source.
