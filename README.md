# GrowthLab: Product Growth & Experimentation Analytics

## Project Overview

An end-to-end product analytics project using **SQL, Python, BigQuery, DuckDB, statistics and Tableau** to find where an online store loses new visitors, measure who comes back, and evaluate whether tested product and marketing changes should ship.

The project covers the full workflow: raw event data in BigQuery, a tested SQL event model, funnel and cohort analysis, behavioural driver modelling, two A/B experiment evaluations, a simulation study of the methods, and an interactive Tableau Public report.

**Tableau Public report:** [add link]

## Business Problem

A digital store gets plenty of traffic, but few visitors buy or return. This project answers:

- Where between first visit and purchase do we lose the most users?
- Does the loss differ by acquisition source, device or country?
- Which weekly cohorts come back, and how does that change over time?
- Which first-day behaviours are linked to returning, and how far can that be trusted?
- Did a game design change improve retention, and what did it cost on guardrail metrics?
- Which marketing email worked, and for which customers?
- What should the team test next, and how large does that test need to be?

## Project Objectives

1. Extract a bounded, cost-controlled sample of GA4 event data from BigQuery.
2. Build a layered SQL event model with automated data-quality checks.
3. Measure the conversion funnel with confidence intervals.
4. Build weekly retention cohorts that handle incomplete follow-up correctly.
5. Model first-day behaviours associated with returning.
6. Evaluate two randomized A/B tests with pre-written analysis plans.
7. Check experiment validity: sample ratio, power and guardrails.
8. Test for subgroup effects with multiple-comparison correction.
9. Validate the statistical methods with simulations.
10. Export validated, Tableau-ready tables.
11. Build an interactive Tableau Public report.

## Technology Stack

| Area | Technology |
|---|---|
| Programming | Python |
| Data Analysis | Pandas, NumPy |
| Statistics | SciPy, statsmodels |
| Machine Learning | Scikit-learn |
| Cloud Data | Google BigQuery |
| Local Database | DuckDB |
| SQL | CTEs, window functions, conditional aggregation, date arithmetic |
| BI | Tableau Public |
| Visualisation | Matplotlib |
| Testing | Pytest |
| Version Control | Git / GitHub |

## Datasets

| Dataset | Size | Nature | Used for |
|---|---|---|---|
| [GA4 obfuscated e-commerce sample](https://developers.google.com/analytics/bigquery/web-ecommerce-demo-dataset) (Google Merchandise Store, Nov 2020 – Jan 2021) | 4.3M events, 270K users | Observational | Funnel, retention, behaviour |
| [Cookie Cats A/B test](https://www.kaggle.com/datasets/yufengsui/mobile-games-ab-testing) | 90,189 players | Randomized | Game experiment |
| [Hillstrom e-mail test](https://blog.minethatdata.com/2008/03/minethatdata-e-mail-analytics-and-data.html) | 64,000 customers | Randomized | Email experiment |

The GA4 sample has no sign-ups, so `first_visit` stands in for sign-up, `purchase` for conversion, and return visits for retention.

## Project Architecture

```text
GA4 events (BigQuery)          Cookie Cats CSV        Hillstrom CSV
        |                            |                      |
        v                            |                      |
Bounded extract (dry-run + byte cap) |                      |
        |                            |                      |
        v                            |                      |
DuckDB SQL event model               |                      |
        +----> Staging (flattened event params)             |
        +----> Sessions, users, items                       |
        +----> Marts: funnel, returns, cohorts, features    |
        |                            |                      |
        v                            v                      v
Data-quality assertions        Experiment analysis    Experiment analysis
        |                      (SRM, CI, power,       (3 arms, interactions,
        v                       guardrails)            Holm, CUPED, uplift)
Funnel & cohort statistics           |                      |
Retention-driver model               |                      |
        |                            |                      |
        +-------------+--------------+----------------------+
                      v
          Validated Tableau export (23 tables, 259 checks)
                      |
                      v
              Tableau Public report
```

## 1. Data Extraction

GA4 data was queried from BigQuery using Application Default Credentials (no keys stored in the repo).

Cost controls:

- A dry-run estimate before every query
- `maximum_bytes_billed` enforced on every job
- Date filtering with `_TABLE_SUFFIX`
- Explicit columns, no `SELECT *`
- A local Parquet cache to avoid re-querying

The full 92-day extract scanned 2.2 GiB, within the BigQuery free tier.

## 2. SQL Event Model

Numbered DuckDB models in `sql/ga4/` build the analysis layers:

```text
01_stg_events          typed events, placeholder values bucketed
02_items               item grain, aggregated before joining back
03_sessions            session start, duration, landing page
04_users               first-touch channel, device, country
05_funnel_users        stage timestamps, strict and any order
06_activity_cohorts    user-day activity, daily cohorts
07_data_quality_profile
08_new_user_marts      new users, return flags, weekly cohorts, first-24h features
```

Techniques used: CTEs, `ROW_NUMBER()`, `LAG()`, `arg_min`, conditional aggregation, nested array handling and sessionization.

## 3. Data Quality

18 SQL assertions must return zero rows. They check:

- Row counts preserved through each layer
- No row multiplication when joining nested item data
- Unique keys and one row per user
- Funnel stages in valid order
- No feature leakage beyond the first 24 hours
- Retention windows nested correctly (D1 within D7 within D30)

Findings from the full extract: 4,295,584 events, 360,129 sessions, 0 duplicate events and 29 bot-like users removed. 18.5% of users had a source hidden by Google's obfuscation; these were kept as an explicit group.

## 4. Funnel Analysis

New visitors observed for a full 7 days: **238,621**.

| Stage | Visitors | Conversion from previous |
|---|---:|---:|
| Visited site | 238,621 | — |
| Viewed product | 50,087 | 21.0% |
| Added to cart | 9,624 | 19.2% |
| Started checkout | 3,496 | 36.3% |
| Purchased | 1,785 | 51.1% |

- **79% of new visitors leave before viewing a product**, the largest drop by far.
- No source, device or country differs significantly after Holm correction.
- A device comparison within each channel ruled out a channel-mix (Simpson's paradox) explanation.

## 5. Retention Analysis

- **D7 return: 7.9%** (95% CI 7.7–8.0%)
- **D30 return: 10.6%**, observable only for first visits up to Jan 1
- The best week-4 retention was the Nov 8 cohort (1.4%), versus about 0.5% for late-December holiday cohorts
- Cohorts differ significantly at every age; channels and devices do not

Weeks the data couldn't fully observe are masked, never counted as zero retention.

## 6. Retention Drivers

A logistic regression on 168,471 visitors linked first-24-hour behaviour to returning within 31 days.

- Strongest positive association: **pages viewed (OR 1.62 per SD, 95% CI 1.57–1.68)**
- Holdout AUC 0.69 (logistic) vs 0.72 (gradient boosting)
- Adding purchase-intent signals did not explain the effect away

These are **associations, not causes**. A follow-up randomized test was designed: 12,387 visitors per group to detect a 1-point lift.

## 7. Game Experiment (Cookie Cats)

The first game gate moved from level 30 to level 40, with 44,700 vs 45,489 players.

| Metric | Result | Status |
|---|---|---|
| Day-7 retention (primary) | **−0.82 pts** (95% CI −1.33 to −0.31) | Clearly worse |
| Day-1 retention (guardrail) | −0.59 pts (95% CI −1.24 to +0.06) | Not passed |
| Rounds played (guardrail) | −0.1 rounds | Passed |
| Revenue | Not in the data | Not measured |

- Sample-ratio check: p = 0.009, below the pre-set flag threshold, so it was noted but not flagged
- Minimum detectable effect at 80% power: 0.74 pts
- **Recommendation: don't ship. Keep the gate at level 30.**

## 8. Email Experiment (Hillstrom)

| Arm | Bought | Lift vs no email (95% CI) |
|---|---:|---|
| No email | 0.57% | — |
| Men's email | 1.25% | **+0.68 pts** (0.50 to 0.86) |
| Women's email | 0.88% | +0.31 pts (0.15 to 0.48) |

- Groups balanced (sample-ratio check p = 0.90)
- Interaction tests first, then Holm correction across 44 subgroups: 17 pass
- Only one real interaction: the women's email works mainly for new customers
- CUPED variance reduction on spend was under 0.1%, reported as it came out
- **Recommendation: send the men's email; target the women's email at new customers.**

## 9. Simulation Study

Method checks on simulated data, anchored on the Cookie Cats sample size:

| Check | Result |
|---|---|
| A/A false-positive rate | 4.8% (target 5%) |
| Peeking 10 times | False positives rise from 5.3% to 19.1% |
| 44 uncorrected subgroups | 89% chance of at least one false positive; Holm keeps it at 4.6% |
| Underpowered test | A real 1-pt effect missed 84% of the time |

## 10. Tableau Export

`make tableau` produces 23 Tableau-ready tables, one per data grain, validated with **259 checks** against the published results:

- Every total and rate re-adds to the analysis output
- Unique keys, valid ranges, estimates inside their intervals
- No duplicated rows in long tables
- Experiments kept in separate tables
- No causal wording in observational tables

## 11. Tableau Dashboard

| Dashboard | Content |
|---|---|
| Overview | Headline KPIs, visitor and purchase trend, channel mix |
| Visitor Journey | Conversion funnel, biggest drop-off, segment comparison |
| Retention | Cohort heatmap, D7/D30 trend |
| Returning Users | Behaviour comparison, adjusted odds ratios |
| Game Experiment | Effect with confidence interval, guardrails, decision |
| Email Experiment | Arm results, lift intervals, subgroup effects |

## 12. Key Questions Answered

1. Where does the funnel lose the most visitors?
2. Is the loss concentrated in a source, device or country?
3. How many visitors return within 7 and 30 days?
4. Which cohorts retain best?
5. Which first-day behaviours go with returning?
6. Did moving the game gate change retention?
7. Were the guardrail metrics safe?
8. Which email lifted purchases, and for whom?
9. How large does the next test need to be?
10. What can this data not tell us?

## 13. Repository Structure

```text
src/
├── connectors/     BigQuery client, GA4 extract, schema check, dataset downloads
├── pipeline/       runs the SQL models
├── quality/        SQL data-quality assertions
├── analysis/       funnel, cohorts, drivers, experiments, simulations
├── analytics/      experiment statistics
└── reporting/      Tableau export and validation
sql/ga4/            8 numbered SQL models
scripts/            export_tableau_data.py
notebooks/          5 analysis notebooks with outputs
exports/tableau/    23 Tableau-ready CSVs
reports/results/    published statistics
docs/               metric definitions, analysis plans, causation note
tests/              offline and BigQuery integration tests
Makefile
requirements.txt
```

## 14. How to Run

```bash
make install
export GOOGLE_CLOUD_PROJECT=<your-project-id>
make gcloud-auth
make ga4-extract
make all
make test
```

## 15. Limitations

- The GA4 sample is obfuscated by Google, and a visitor is a browser cookie, not a person.
- About three months of data, so D30 return exists only for early cohorts.
- The game data has no revenue, no D30 retention and no pre-test customer attributes.
- The email test is from 2008 and covers two weeks only.
- Store results are observational; only the two experiments show cause and effect.
- The analysis plans were written before the statistics but after the data was public, so they are not true pre-registrations.

## 16. Skills Demonstrated

### SQL
CTEs · window functions · conditional aggregation · sessionization · nested data · data-quality assertions

### Statistics & Experimentation
Confidence intervals · power analysis · minimum detectable effect · sample-ratio checks · guardrail metrics · non-inferiority · Holm correction · interaction tests · CUPED · bootstrap

### Product Analytics
Funnel analysis · cohort retention · censoring · segment comparison · behavioural drivers

### Machine Learning
Logistic regression · gradient boosting · odds ratios · AUC · uplift modelling

### Data Engineering
BigQuery · DuckDB · cost-controlled extraction · reproducible pipelines · automated testing

### Business Intelligence
Tableau · KPI design · decision-oriented reporting

## Author

**Sneha**
Data Analyst | Product Analytics | Experimentation
