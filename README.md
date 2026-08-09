# Detecting Medicare Phantom Billing at Scale

**A Post-Mortem Claims, Impossible Service Day & Pre-Mortem Surge Detection Engine on 407 Million Synthetic Records**

I built a four-module detection engine that identifies **$540M in extrapolated Medicare phantom billing exposure** across 407 million synthetic claims records, running end-to-end in **under 15 minutes** on a consumer-grade laptop.

> **Alessandro Oliveira de Sousa** — Healthcare Data Analyst | FWA Detection & Claims Auditing
>
> August 2026

[![Medium](https://img.shields.io/badge/Medium-Deep_Dive-black?style=flat&logo=medium)](https://medium.com/@alessandro.oof/detecting-medicare-phantom-billing-at-scale-building-a-post-mortem-claims-impossible-service-day-c8d57a6b9c3c)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Profile-blue?style=flat&logo=linkedin)](https://linkedin.com/in/aosousa)
[![Companion Project](https://img.shields.io/badge/Companion-Upcoding_%26_Unbundling-green?style=flat)](https://medium.com/@alessandro.oof/detecting-medicare-fraud-at-scale-building-an-upcoding-unbundling-detection-engine-on-230-4de555db568d)

---

## Key Findings

| Metric | Value |
|---|---|
| Deceased beneficiaries analyzed | 107,644 out of 2,326,856 (4.6% mortality) |
| P1: Post-mortem claims detected | 366 records, 100% recovery on controlled injection (0 organic in DE-SynPUF) |
| P2: Impossible-day providers | 194 sustained providers (5+ impossible days each, z > 5.0 or > 100 lines/day) |
| P3: Pre-mortem surge providers | 41 flagged (z > 2.0 on mean surge ratio across deceased patient panel) |
| P4: High/Critical risk providers | 88 out of 1,723 scored (Isolation Forest composite, multi-signal convergence) |
| Combined financial exposure | ~$25M sample / ~$540M extrapolated (DOJ/OIG-calibrated) |

## Platform Context

This is the **second engine** in a Medicare FWA detection platform.

| Engine | Fraud Axis | Modules |
|---|---|---|
| [Engine 1: Billing Code Manipulation](https://medium.com/@alessandro.oof/detecting-medicare-fraud-at-scale-building-an-upcoding-unbundling-detection-engine-on-230-4de555db568d) | Upcoding & Unbundling | M1: DRG Upcoding (CMI z-score) · M2: E&M Upcoding (concentration ratio) · M3: NCCI PTP Unbundling (753,962 edit pairs) · M4: Isolation Forest Composite |
| **Engine 2: Billing Eligibility Fraud** (this repository) | Phantom Billing | P1: Post-Mortem Billing · P2: Impossible Service Days · P3: Pre-Mortem Surge · P4: Isolation Forest Composite |

Together, they cover **four distinct fraud axes** across ~407 million CMS DE-SynPUF records running on a DuckDB analytical datalake — the same data infrastructure and detection logic that production SIU teams deploy against real Medicare claims.

## The Data: 407 Million Records on DuckDB

The pipeline runs on the [CMS DE-SynPUF](https://www.cms.gov/data-research/statistics-trends-and-reports/medicare-claims-synthetic-public-use-files/cms-2008-2010-data-entrepreneurs-synthetic-public-use-file-de-synpuf), a fully synthetic dataset modeled on real 2008–2010 Medicare claims. All 20 samples are loaded into a DuckDB analytical datalake:

| Table | Records | Samples | Cols |
|---|---|---|---|
| beneficiary | 6,873,274 | 20 | 34 |
| carrier_claims | 94,863,452 | 20 | 143 |
| inpatient_claims | 1,332,822 | 20 | 82 |
| outpatient_claims | 15,826,987 | 20 | 77 |
| prescription_drug_events | 111,085,969 | 20 | 9 |
| v_carrier_lines | 177,027,909 | 20 | 8 |
| **TOTAL** | **407,010,413** | | |

**Runtime:** under 15 minutes on a consumer-grade workstation (i5-8250U @ 1.60GHz × 4 cores, 20GB RAM) reading directly from partitioned CSV files.

## Controlled Injection Framework

The DE-SynPUF is synthetic. Its synthesis process scrubbed temporal inconsistencies, which means **zero organic post-mortem claims exist in the dataset**. Running the P1 detection SQL against unmodified data returns an empty table — that proves syntax, not detection.

The controlled injection framework is calibrated against the OIG's actual enforcement findings:

1. **Realistic sampling** — Draws "real" deceased beneficiaries, provider IDs, HCPCS/NDC codes, and realistic payment amounts directly from the datalake.
2. **Two provider archetypes** — "Hot" providers (5–18 deceased patients each, T2/T3 heavy) modeled on the OIG's 400-patient supplier, and "one-off" providers (1 deceased patient, mostly T1 processing lag).
3. **Reproducible and non-destructive** — Uses `random_state=42`, temporary DuckDB tables (datalake never modified), and an exportable manifest CSV.
4. **100% recovery validation** — `assert` confirms every planted record was detected.

P2 (impossible days) and P3 (billing surge) operate on real synthetic patterns. The Isolation Forest (P4) re-ranks the already-flagged providers: **rules detect, models prioritize**.

## Detection Modules

### Module P1: Post-Mortem Billing

*Was a service billed after the patient died?*

For each claim type (inpatient, outpatient, carrier, PDE), the pipeline JOINs service dates against beneficiary death dates and computes `days_post_mortem = service_date - death_date`. Results are classified into severity tiers:

| Tier | Days After Death | Interpretation |
|---|---|---|
| T1 | 1–30 | Potential processing lag. Death notifications take time to propagate. |
| T2 | 31–90 | Suspicious. Exceeds the standard notification window. |
| T3 | 91+ | Hard fraud signal. No legitimate explanation for billing three months after death. |

**Results:** 366 post-mortem records across 154 deceased beneficiaries and 62 providers, $325,770.00 total exposure. 100% recovery. T3 records drive the highest per-record payment amounts, consistent with OIG enforcement patterns.

Under the [False Claims Act](https://www.justice.gov/civil/pages/attachments/2019/02/04/jm-civil-resource-manual-24-false-claims-act.pdf) (31 U.S.C. 3729–3733), these records trigger automatic case referrals with treble damages.

### Module P2: Impossible Service Days

*Can a provider physically perform this many services in one day?*

The pipeline aggregates daily line-item counts per provider across 177 million `v_carrier_lines` records. A **two-pass query architecture** keeps this tractable:

- **Pass 1:** Fast aggregation on raw VARCHAR columns. No date casting, no `COUNT(DISTINCT)`. Produces the full provider-day universe with line counts and payment totals.
- **Pass 2:** Enriches only the flagged subset (provider-days exceeding z > 5.0 or 100+ lines) with beneficiary and claim counts. Turns a 90+ minute single-pass query into under 5 minutes.

A secondary filter retains only providers with **5+ impossible days** in the 3-year window. Single-day outliers are noise; sustained impossible billing is the fraud signal. This mirrors the [Judd case](https://www.acfe.com/acfe-insights-blog/blog-detail?s=impossible-day-health-care-fraud-schemes) pattern: 164 impossible days over 5 years, not one bad Tuesday.

**Results:** 194 providers survived the secondary filter. Calibrated exposure at full Medicare population scale: ~$533M (194 providers × $2.75M DOJ midpoint per case). Sample-level: ~$25M.

### Module P3: Pre-Mortem Billing Intensity Surge

*Is this provider systematically billing dying patients more than the population norm?*

Some increase in billing near end-of-life is clinically expected. This module detects providers whose average surge ratio (billing in the final 90 days / baseline 90-day average over prior 9 months) **systematically exceeds the population norm**.

**Key finding:** The population median surge ratio is 0.92 — most deceased beneficiaries actually have *less* billing in their final 90 days, not more. The 41 flagged providers (z > 2.0) sit in the extreme right tail, well separated from the norm.

**Results:** 41 providers flagged. Top three Critical-tier providers show surge ratios of 70.8×, 63.4×, and 43.8× against the 0.92 population median. Estimated precision at z > 2.0: ~5% (consistent with a first-pass triage filter).

### Module P4: Composite Risk Ranking (Isolation Forest)

*Given all the signals, which providers should an SIU investigate first?*

The Isolation Forest does not detect new fraud. It takes providers already flagged by P1–P3 and produces a **unified investigation priority score** by combining multiple signals.

- Features normalized to [0,1] via MinMaxScaler (without normalization, P3 surge ratios up to 76× would swamp P1 and P2 features entirely).
- Isolation Forest selected over LOF: O(n) vs. O(n²), no distance metric required for sparse mixed-signal feature space.
- Risk tier distribution: Low 1,334 / Moderate 155 / Elevated 146 / High 73 / Critical 15.
- **88 providers in High + Critical tiers** = highest-priority investigation targets.

## Top 20 Investigation Targets

The capstone visualization ranks the top 20 providers by composite phantom billing risk score, annotated with colored badges indicating which detection modules flagged them (P1: Post-Mortem, P2: Impossible Days, P3: Billing Surge). Multi-signal convergence — providers flagged by 2+ modules — represents the highest-confidence SIU referral targets.

![Investigation Priority: Top 20 Providers by Composite Risk](figures/fig5_top20_investigation_targets.png)

*Figure 5 — Top 20 providers ranked by composite phantom billing risk score. Multi-signal convergence (providers flagged by 2+ modules) represents the highest-confidence SIU referral targets.*

The top three Critical-tier providers are all P3-flagged with extreme single-signal magnitude (40–70× the population baseline). Below the P3-dominated top, mid-ranked providers carry P2 badges (impossible days), often paired with P1 (post-mortem) — **less extreme on any single axis, but more damning in combination**.

## Financial Exposure (DOJ/OIG-Calibrated)

The financial summary is anchored to real enforcement data, not raw synthetic output:

| Module | Sample Exposure | Extrapolated (21.5×) | Benchmark |
|---|---|---|---|
| P1: Post-Mortem | $325,770.00 | $7,004,055 | $23,000,000 ([OIG](https://oig.hhs.gov/oei/reports/oei-04-12-00130.asp)) |
| P2: Impossible Days | $24,813,953 (calibrated) | $533M | $500,000 – $5,000,000 per case (DOJ) |
| P3: Pre-Mortem Surge | context-dependent | context-dependent | OIG hospice audits |
| **Combined** | **~$25M** | **~$540M** | |

For context: CMS FY2025 Medicare FFS improper payments totaled [$28.83 billion](https://www.cms.gov/data-research/monitoring-programs/medicare-ffs-compliance-programs/cert/annual-medicare-ffs-improper-payment-rate) (6.55%). The 2026 National Healthcare Fraud Takedown recovered [$6.5 billion](https://www.justice.gov/opa/pr/) across 455 defendants. The extrapolated $540M represents ~8% of the Takedown figure — plausible for phantom billing as one fraud vector among many.

## Cross-Engine Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                   FWA DETECTION PLATFORM                    │
│                                                             │
│  ┌─────────────────────────┐ ┌─────────────────────────┐    │
│  │ ENGINE 1: Billing Code  │ │ ENGINE 2: Billing       │    │
│  │ Manipulation            │ │ Eligibility Fraud       │    │
│  │                         │ │ (this repository)       │    │
│  │  M1: DRG Upcoding      │ │  P1: Post-Mortem        │    │
│  │  M2: E&M Upcoding      │ │  P2: Impossible Days    │    │
│  │  M3: NCCI Unbundling   │ │  P3: Pre-Mortem Surge   │    │
│  │  M4: Composite Risk    │ │  P4: Composite Risk     │    │
│  └────────────┬────────────┘ └────────────┬────────────┘    │
│               │                           │                 │
│               ▼                           ▼                 │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              UNIFIED RISK RANKING                    │   │
│  │   Merge M4 + P4 scores → Multi-axis provider ranking│   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

In a production SIU environment, the unified ranking would feed a case management system where investigators receive prioritized provider dossiers from all eight detection modules.

## Data Source

[CMS 2008–2010 Data Entrepreneurs' Synthetic Public Use File (DE-SynPUF)](https://www.cms.gov/data-research/statistics-trends-and-reports/medicare-claims-synthetic-public-use-files/cms-2008-2010-data-entrepreneurs-synthetic-public-use-file-de-synpuf)

The DE-SynPUF is a fully synthetic dataset modeled on real Medicare claims. It is freely available from CMS for research and development purposes. No real patient or provider data is used in this project.

## Author

**Alessandro Oliveira de Sousa**
Healthcare Data Analyst | FWA Detection & Claims Auditing

- [LinkedIn](https://linkedin.com/in/aosousa)
- [Medium](https://medium.com/@alessandro.oof)
- [GitHub](https://github.com/sousa-a)

*This project is part of a portfolio demonstrating production-grade healthcare claims auditing capabilities for FWA detection, SIU analytics, and program integrity roles. The methodology draws from DOJ prosecution patterns, OIG audit methodologies, and CMS Program Integrity practices.*

---

*Source: CMS DE-SynPUF (2008–2010), all 20 samples.*
*Analysis: Alessandro Oliveira de Sousa (August, 2026).*
