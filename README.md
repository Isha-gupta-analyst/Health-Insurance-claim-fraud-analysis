# Health-Insurance-claim-fraud-analysis


A rule-based system that ranks 8,000 Indian health insurance claims by suspicion level, so an investigation team can review the riskiest claims first. Built with **Python (pandas)** and **Power BI**. No machine learning.

---

## 1. Business problem

Special Investigation Units (SIUs) can manually review only a small share of claims, and there is usually no systematic way to choose which ones. This project builds an explainable scoring system that answers:

> **Which claims should an investigator look at first, and why?**

**Important:** the dataset has **no fraud labels**. This is a *triage* tool. It flags claims worth investigating; it does not prove fraud.

## 2. Headline results

| Metric | Result |
|---|---|
| Claims analysed | 8,000 |
| High-risk claims | **148** (1.85% of claims) |
| High-risk exposure | **₹4.22 Cr** (4.3% of total payout) |
| Avg High-risk claim vs Low-risk | ₹2.85 lakh vs ₹1.20 lakh (about 2.4x) |
| Claims paid above their own components | **80** claims, **₹2.27 Cr** excess (2.33% of payout) |
| High-risk claims backed by a billing-integrity flag | **93 of 148** (63%) |

## 3. Dataset

- Source: [Indian Health Insurance Claims (Kaggle)](https://www.kaggle.com/datasets/shubham001official/indian-health-insurance-claims-analysis)
- 8,000 rows, 38 columns: demographics, lifestyle, policy, hospital and claim financials.
- One claim per policy. No premium column, no claim status, no fraud label.
- The data looks **synthetic** (see Limitations).

From the 38 columns, 18 were kept for this analysis: policy and claim dates, hospital bill, pre/post-hospitalization expense, non-payable items, deductible, co-pay, claim amount, sum insured, length of stay, hospital name/tier, room category, policy type, state, agent channel. Personal and lifestyle columns were dropped because they do not feed any fraud rule (and names/Aadhaar should not sit in a portfolio project).

## 4. Approach

```
Profile -> Clean -> Engineer features -> Define "normal" -> Apply red-flag rules -> Score -> Tier -> Validate -> Dashboard
```

1. **Profiling:** shape, nulls (none), duplicates (none), distributions, categorical spelling (clean).
2. **Cleaning:** dates converted; `₹` and commas stripped from bill and claim columns; negative expenses and zero-day stays **flagged, not deleted** (a bad record can itself be a signal).
3. **Claim formula discovery:** the paid claim reconciles to
   `Bill + Pre-hosp + Post-hosp − Non-payable − Deductible − Co-pay`
   for **7,909 of 8,000 claims (99%)** within ₹1. Deviations from this became a rule.
4. **Features:** `days_to_claim`, `bill_per_day`, `claim_to_si_ratio`, `non_payable_ratio`, `expected_claim`, `claim_vs_expected`, `bpd_vs_peer`.
5. **Peer baseline:** bill per day is compared against the claim's own **hospital tier** median (about ₹19k Govt, ₹28-29k Private Small, ₹35-36k Private Corporate). Room category showed no effect, so it was not used.
6. **Red flags, score and tiers** (below).

### Red-flag rules

Thresholds were chosen from the data distribution, targeting roughly 1-10% flag rate per rule.

| Flag | Rule | Weight | Claims flagged |
|---|---|---|---|
| Amount mismatch | Claim > 1.2x the sum of its own components | 3 | 80 (1.0%) |
| Claim above sum insured | Claim > sum insured | 3 | 22 (0.3%) |
| Early claim | Claim within 90 days of policy start | 2 | 253 (3.2%) |
| Peer outlier | Bill per day > 2x hospital-tier median | 2 | 465 (5.8%) |
| Near limit | Claim > 80% of sum insured | 1 | 67 (0.8%) |
| High non-payable share | Non-payable ratio in top 5% | 1 | 400 (5.0%) |
| Data issue | Negative expense or zero-day stay | 1 | 146 (1.8%) |

**Risk score** = sum of (flag x weight). Weights reflect how direct the signal is: billing-integrity breaks (3) > behavioural signals (2) > weak or data-quality signals (1).

**Tiers:** High = score 3+, Medium = score 2, Low = 0-1.

| Tier | Claims | Share | Total claim | % of payout |
|---|---|---|---|---|
| High | 148 | 1.9% | ₹4.22 Cr | 4.3% |
| Medium | 649 | 8.1% | ₹6.92 Cr | 7.1% |
| Low | 7,203 | 90.0% | ₹86.12 Cr | 88.6% |

Every claim also gets a plain-text **reasons** column (e.g. *"Claim amount mismatch; Claim above sum insured; Near sum insured limit"*), so an investigator can see why it was flagged.

## 5. Validation

- **IQR cross-check:** a per-tier IQR outlier test flagged 451 claims vs 465 from the 2x rule, with **94% overlap** (438 common).
- **Sensitivity test:** with stricter thresholds (early claim 60 days, peer outlier 2.5x, near limit 0.9, non-payable top 1%), **104 of 148** High-tier claims (70%) stayed High, and no new claims entered.
- **Confidence split:** all **93** claims flagged by billing-integrity rules stayed High under stricter thresholds. Of the **55** flagged only by behavioural rules, just 11 did. These 55 are labelled *Borderline* in the dashboard, and the 93 *Confident*.
- **Tested and dropped:** "high bill with short stay" flagged almost nothing (at most 0.2%), and hospital-level concentration was not feasible because there are 7,188 unique hospital names across 8,000 claims.

## 6. Power BI dashboard

| Page | Purpose |
|---|---|
| **Executive Summary** | KPIs (claims, High-risk count, exposure, excess payout), tier distribution, High-risk trend by year |
| **Flag Analysis** | Rule-wise flag counts, rules and weights table, High-risk rate by state and agent channel, confident vs borderline split |
| **Investigator Worklist** | The 148 High-risk claims sorted by score, with reasons, a confidence label and slicers for state, tier and policy type |

Screenshots: `powerbi/screenshots/` (add `01_executive_summary.png`, `02_flag_analysis.png`, `03_investigator_worklist.png`)

## 7. Key findings and recommendations

| Finding | Recommendation |
|---|---|
| 80 claims were paid more than 1.2x what their own bill components justify, about **₹2.27 Cr** above the reconciled amount. | Add an automated **claim-vs-components reconciliation check** before payout approval. |
| 22 claims exceed the sum insured. | Enforce a **hard cap** at the payout stage and audit those 22. |
| 93 of 148 High-risk claims carry a billing-integrity flag and held up under stricter thresholds. | Send these **93 first** to investigators; treat the other 55 as lower-confidence leads. |
| 465 claims bill more than 2x their hospital tier's typical daily cost. | Review these against tier benchmarks and **negotiate package rates** where patterns repeat. |
| 253 claims arrive within 90 days of policy start. | Consider **pre-authorisation review** for early claims (see caveat below). |
| High-risk rate is similar across states (1.6%-2.3%) and channels (1.7%-2.1%). | Do not target by geography or channel on this evidence; risk is driven by claim characteristics. |

## 8. Limitations

- **No fraud labels.** Results are triage signals, not proof, and accuracy cannot be measured.
- **Likely synthetic data.** `days_to_claim` is almost perfectly flat from 30 to 2,000 days, room category has no effect on cost, and nearly every claim has its own hospital. The early-claim flag may therefore reflect chance rather than real waiting-period abuse.
- **Future claim dates.** Claim dates run to May 2029; 866 claims (10.8%) fall after the analysis date. Time trends are unreliable, so the dashboard trend is limited to 2018-2025. Scores and flags do not depend on calendar time.
- **Judgement-based thresholds and weights.** They were justified from distributions, but the sensitivity test shows borderline claims move when thresholds change.
- **Differences across states and channels are small** (a few claims) and should not be over-read.
- **No premium or claim-status columns**, so loss ratio and rejection analysis were not possible.
- **No manual review** of top-ranked claims was done as part of this project.

## 9. Repository structure

```
.
├── data/
│   ├── raw/                 # original CSV (never edited)
│   └── clean/
│       └── claims_scored.csv   # cleaned data with flags, score, tier, reasons
├── notebooks/
│   └── Healthcare_analysis.ipynb
├── powerbi/
│   ├── claims_fraud_dashboard.pbix
│   └── screenshots/
└── README.md
```

## 10. How to reproduce

1. Download the dataset from Kaggle and place it in `data/raw/`.
2. Install dependencies: `pip install pandas numpy matplotlib seaborn jupyter`
3. Run `notebooks/Healthcare_analysis.ipynb` top to bottom (**Kernel > Restart & Run All**). It writes `data/clean/claims_scored.csv`.
4. Open `powerbi/claims_fraud_dashboard.pbix` and refresh the data source to the CSV path on your machine.

## 11. Tools

Python (pandas, numpy, matplotlib, seaborn) · Power BI (DAX measures, conditional formatting, slicers) · Jupyter

---

*Author: [Your Name] · [LinkedIn] · [GitHub]*
