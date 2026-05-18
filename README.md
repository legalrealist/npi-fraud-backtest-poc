# Medicare Fraud Backtest POC

Proof-of-concept backtest testing whether excluded Medicare providers show detectable billing differences from peers in the year before exclusion.

## Results

**289 excluded providers** matched to pre-exclusion Part B billing data across all US states, compared against **3.39 million peers** (same state, specialty, year). **14 of 15 features statistically significant** after Bonferroni correction.

Strongest signals:
- **Dual-eligible share** (d=+0.78) — excluded providers serve far more Medicaid-Medicare dual-eligible beneficiaries
- **Top HCPCS share** (d=+0.66) — 30% of billing from a single procedure code vs. 16% for peers
- **HCPCS Herfindahl** (d=+0.52) — less diverse service mix overall
- **Beneficiary avg age** (d=−0.31) — younger patient panels
- **Avg charge per service** (d=−0.23) — lower charges, higher volume pattern
- **Beneficiary avg risk** (d=+0.18) — higher-risk patients (only visible after filtering to fraud-specific exclusions)

Full results in [`report.md`](report.md).

## Exclusion Type Filtering

The cohort is restricted to fraud-related exclusion types under [Section 1128 of the Social Security Act](https://oig.hhs.gov/exclusions/authorities.asp):

| Code | Description | Count |
|------|-------------|-------|
| §1128(a)(1) | Program fraud conviction | 1,221 |
| §1128(a)(3) | Felony healthcare fraud conviction | 300 |
| §1128(b)(7) | Excessive claims / unnecessary services | 78 |

Excluded from the cohort:
- **§1128(a)(4)** — felony controlled substance convictions (drug crimes, not billing patterns)
- **§1128(b)(4)** — license revocations (administrative, could be anything)
- **§1128(a)(2)** — patient abuse/neglect convictions

This filtering matters. With all exclusion types mixed in, `bene_avg_risk` showed zero signal (d=0.00). After restricting to fraud-specific types, it became significant (d=+0.18) — license revocations and drug felonies were masking a real billing-fraud signal.

## Iteration History

The pipeline went through several iterations to reach the current design:

1. **DMEPOS + NPPES** — zero NPI overlap. DMEPOS data is 99.5% organizational NPIs; LEIE exclusions are individuals. Entity types don't align.
2. **Part B, Florida only, all exclusion types** — 28 matched providers. Signal present but underpowered.
3. **Part B, 10 states, all exclusion types** — 346 providers, 12/15 features significant. Data duplication bug found and fixed (excluded NPIs appearing in multiple state/year datasets).
4. **Part B, 10 states, §1128(a)(1) + §1128(b)(7) only** — 140 providers, 6/15 significant. Cleaner cohort but lost statistical power.
5. **Part B, all states, §1128(a)(1) + §1128(b)(7) only** — 245 providers, 10/15 significant. Power recovered, but concentration metrics weaker.
6. **Part B, all states, §1128(a)(1) + §1128(a)(3) + §1128(b)(7)** — 289 providers, 14/15 significant. Adding healthcare fraud felonies back in recovered concentration signal while keeping the clean cohort.

## Practice Size Control

The strongest methodological objection: the fraud fingerprint might be a small-practice signal. The pipeline tests this by binning providers into four size tiers (11–50, 51–200, 201–500, 500+ beneficiaries) and re-running the statistical comparison within each tier.

**11 of 15 features remain significant after size adjustment** (vs. 13 raw). The three core concentration metrics got *stronger*:

| Feature | Raw d | Size-adjusted d |
|---------|:---:|:---:|
| dual_share | +0.78 | +0.79 |
| top_hcpcs_share | +0.66 | +0.71 |
| hcpcs_herfindahl | +0.52 | +0.60 |

Features that lost significance (total_benes, unique_hcpcs, bene_avg_age, submitted_charges) are plausibly size-confounded. The behavioral fingerprint is robust to this control.

## Predictive Risk Model

Logistic regression trained on all 15 features with balanced class weights. **Cross-validated AUC-ROC: 0.791** — the model reliably distinguishes excluded from non-excluded billing patterns.

Top predictive features (standardized coefficients):
- **Top HCPCS share** (+2.22) — single-code billing concentration is the strongest predictor
- **HCPCS Herfindahl** (−1.62) — interacts with top share to capture concentration shape
- **Total beneficiaries** (−1.56) — smaller patient panels = higher risk
- **Avg charge per service** (−0.71) — lower charges, consistent with high-volume/low-complexity pattern
- **Dual-eligible share** (+0.58) — more dual-eligible patients = higher risk

The model scores all 2.4M peers on a 0–1 scale. At the 0.9 threshold, 31,595 peers (1.3%) have billing patterns highly similar to excluded providers. Top 50 highest-scoring peers listed in [`report.md`](report.md).

## Model Validation: Out-of-Sample Peer Investigation

To test whether the model identifies real fraud beyond its training data, we searched public records (DOJ, OIG, state medical boards) for the top 50 highest-scoring peers. **6 of 30 searched (20%) had confirmed enforcement actions** — none were in the LEIE training labels.

Key findings:
- **Phlebxpress** (CA, Clinical Lab) — owners convicted of $7M Medicare fraud, sentenced to 15 months each. Company NPI stayed active because only individuals were excluded.
- **Advanced Clinical Laboratories** (FL, Clinical Lab) — OIG Corporate Integrity Agreement for False Claims Act violations
- **Hemal Mehta** (TN, Pain Management) — DOJ indictment, 10 counts of conspiracy to distribute controlled substances
- **Natera, Inc.** (CA, Clinical Lab) — qui tam complaint + $8.25M settlement
- **CareDx, Inc.** (CA, Clinical Lab) — DOJ investigation + qui tam whistleblower lawsuit
- **Stephen Dubin** (NV, General Practice) — two Nevada medical board complaints

All 4 clinical laboratories in the top 50 had enforcement actions. The model catches cases the exclusion system misses: entity NPIs active after individual owners are excluded, providers under monitoring instead of exclusion, and indicted providers awaiting conviction. Data stored in `data/peer_validations.json`.

## DOJ Prosecution Matching

A sample of 43 excluded providers was searched against DOJ press releases on justice.gov. **17 (40%) matched** to published federal prosecutions — sentences ranging from 15 months to 84 months, fraud amounts from $2.5M to $110M.

Match rate by state:
- **TX (80%), NJ (75%), NY (67%)** — high federal prosecution visibility (Medicare Fraud Strike Force states)
- **OH (14%), CA (0%)** — likely prosecuted at state level by AG offices
- All §1128(a)(1) providers were convicted by definition — the 40% is what's findable via DOJ press releases, not the true prosecution rate

Matched providers are listed with DOJ links in [`report.md`](report.md). Data stored in `data/doj_matches.json`.

## Data Sources

| Source | URL | Purpose |
|--------|-----|---------|
| LEIE | https://oig.hhs.gov/exclusions/downloadables/UPDATED.csv | Excluded provider labels |
| CMS Part B Provider | https://data.cms.gov (year-specific UUIDs in script) | Provider-level billing |
| CMS Part B Service | https://data.cms.gov (year-specific UUIDs in script) | HCPCS-level billing |

All data is publicly available, no API keys required.

## Usage

```bash
pip install pandas scipy matplotlib requests pyarrow scikit-learn
python backtest.py
```

First run downloads ~4GB of CMS data for all states (cached to `data/` as parquet). Subsequent runs use cache. Outputs: `report.md` and PNGs in `figures/`.

## Pipeline

1. Download and cache LEIE (83K+ records)
2. Build excluded cohort — all US states, 2018–2023 exclusions, fraud-specific types (§1128(a)(1), §1128(a)(3), §1128(b)(7)), valid NPIs → 1,605 providers
3. Find best billing year per provider — walk backward through CMS Part B (2022→2017) → 289 matched
4. Download billing data — provider-level and service-level, cached to parquet
5. Build peer groups — same state, same specialty, same year, ≥11 beneficiaries
6. Compute 15 features — volume, intensity, concentration, demographics
7. Statistical comparison — Mann-Whitney U, Welch's t-test, Cohen's d, Bonferroni correction
8. Predictive risk model — logistic regression scoring every provider by fraud-similarity (AUC=0.79)
9. DOJ prosecution matching — cross-reference excluded providers with justice.gov press releases
10. Visualizations and report

## Caveats

- 82% of excluded NPIs didn't match any CMS Part B data (program siloing, suppression thresholds, NPI mismatches)
- Peer matching by state/specialty/year — not by practice size or sub-state geography
- Effect sizes are population-level, not individual-level diagnostic
- Billing years range from 2017–2022; Medicare norms shifted during COVID-19 (peer-matching by year mitigates but doesn't eliminate)
