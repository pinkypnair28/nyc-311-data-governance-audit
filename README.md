# nyc-311-data-governance-audit

# NYC 311 Service Requests — Data Governance & Quality Audit

**A self-directed audit demonstrating data profiling, standardization, and governance framework design using SQL, Python, and Google Cloud — built entirely with free, browser-based tools.**

## Live Artifacts

- 📊 **Live dashboard:** [NYC 311 Data Quality Scorecard](https://datastudio.google.com/reporting/79acf87a-73c5-467b-a797-4159d31a385e)
- 💻 **Repository (includes the notebook):** [github.com/pinkypnair28/nyc-311-data-governance-audit](https://github.com/pinkypnair28/nyc-311-data-governance-audit)

## Executive Summary

This project audits a real, large-scale public dataset (NYC's 311 service request records, hosted via Google's BigQuery Public Dataset Program) to identify and document data quality and governance gaps, then defines the framework — data dictionary, quality rules, and a canonical taxonomy — an organization would need to bring the dataset under control.

**Headline findings:** of 484 raw category labels in the dataset's `complaint_type` field, only 461 are genuine values. The remaining categories collapse into 23 duplicate clusters once normalized — but 43% of those clusters turned out not to be naming inconsistencies at all: they were non-conforming, injection-style strings that had been accepted directly into a production categorical field, with no validation at ingestion. Separately, over 400,000 records show a closed date earlier than their created date, a direct threat to any SLA or turnaround-time reporting built on this data.

## 1. Objective & Business Context

Enterprise data assets are only as trustworthy as the governance controls behind them. A categorical field with no enforced taxonomy, or a timestamp field with no validity checks, can silently corrupt everything downstream of it — customer-facing reporting, operational KPIs, or risk and compliance metrics. This is especially true in domains like credit and fraud risk, where unvalidated or adversarial input entering a production data pipeline isn't just a cosmetic problem — it's a control gap.

This project simulates exactly that kind of first-pass audit: given a live, unfamiliar dataset, profile it, quantify its quality gaps, and produce the governance artifacts (data dictionary, quality rules, standardization mapping) a data owner would need to act on.

## 2. Methodology

| | |
|---|---|
| **Dataset** | `bigquery-public-data.new_york.311_service_requests` — NYC's 311 non-emergency service request records, sourced from NYC Open Data (DoITT), mirrored into BigQuery's public dataset program |
| **Tooling** | Google BigQuery Sandbox (SQL profiling), Google Colab (Python/pandas automation), Looker Studio (dashboard) — 100% browser-based, no billing account or local install required |
| **Approach** | (1) Profile the dataset across 4 quality dimensions → (2) diagnose root causes, not just symptoms → (3) build the governance artifacts (dictionary, rules, canonical mapping) → (4) document corrective actions and business impact |

## 3. Key Findings at a Glance

| Metric | Result |
|---|---|
| Raw distinct `complaint_type` values | 484 |
| Duplicate clusters after normalization | 23 |
| — genuine naming-variant clusters | 13 (57%) |
| — non-conforming / injected-payload clusters | 10 (43%) |
| Zip code completeness | 95.2% |
| Borough completeness | 95.8% |
| Closed-date completeness | 97.2% |
| Descriptor completeness | 98.4% |
| Primary key (`unique_key`) duplicate rate | 0% |
| Records with `closed_date` before `created_date` | 402,189 |

## 4. Data Dictionary

| Field | Type | Description | Expected Format / Rule | Illustrative Owner |
|---|---|---|---|---|
| `unique_key` | INTEGER | Unique identifier for each service request | Must be unique, non-null | NYC311 intake system |
| `created_date` | TIMESTAMP | Date/time the request was logged | UTC timestamp; must be ≤ `closed_date` | NYC311 intake system |
| `closed_date` | TIMESTAMP | Date/time the request was resolved | UTC timestamp; NULL only while open | Responding agency |
| `complaint_type` | STRING (categorical) | High-level category of the complaint | Must match an approved, versioned taxonomy | Agency taxonomy owner |
| `descriptor` | STRING | Sub-category / detail within `complaint_type` | Agency-defined free text | Agency taxonomy owner |
| `incident_zip` / `borough` | STRING | Geographic location of the request | 5-digit US zip / one of 5 boroughs | Geocoding service |

## 5. Data Quality Rules, Findings & Why Each Matters

### Rule 1 — Completeness
**Rule:** Key operational fields should be ≥98% populated.
**Finding:** zip ~95.2%, borough ~95.8%, closed_date ~97.2%, descriptor ~98.4% — **fails threshold on zip and borough.**
**Why it matters:** Missing location data breaks any geographic rollup — you can't accurately route requests, plan resourcing, or report by neighborhood if roughly 1 in 20 records has no usable location.
**Fix & benefit:** Make zip/borough conditionally required at intake based on request type (some request types genuinely have no fixed location). This closes the completeness gap without forcing bad data into records where a location legitimately doesn't apply.

### Rule 2 — Uniqueness
**Rule:** `unique_key` must have zero duplicates.
**Finding:** 0 duplicates across the full table — **passes.**
**Why it matters:** A clean primary key means downstream joins and counts by `unique_key` won't silently double-count records.
**Fix & benefit:** No fix needed now, but this should be re-verified on a schedule — a single future ingestion bug could break it silently.

### Rule 3 — Validity (temporal)
**Rule:** `closed_date` must be ≥ `created_date`.
**Finding:** 402,189 records violate this — **fails.**
**Why it matters:** Any "average time to resolve" or SLA metric computed naively over this data is wrong wherever this holds — you can't have a negative resolution duration, and at this volume it's not a handful of edge cases.
**Fix & benefit:** Quarantine violating records from time-based KPIs rather than deleting or silently including them, while the likely root cause (time zone handling, or bulk backfilled closures) is investigated with the source agency. This protects reporting accuracy immediately without losing the underlying records.

### Rule 4 — Conformity (domain / taxonomy)
**Rule:** `complaint_type` must belong to an approved, versioned taxonomy.
**Finding:** 484 raw values collapse to 23 duplicate clusters — but only 13 of those are genuine naming drift (case/punctuation only). The other 10 (43%) are non-conforming, injection-style payloads, several appended directly onto an otherwise-legitimate "Misc. Comments" category — **fails.**
**Why it matters:** This is not just a cosmetic naming problem. A categorical field accepting arbitrary injected text means there was no enum-level validation at ingestion at all — the same gap that would let genuinely malicious or malformed input reach a production system undetected.
**Fix & benefit:** Implement allowlist validation at ingestion, with non-conforming values routed to a quarantine table instead of the production field. This both cleans up reporting and closes a real input-validation gap.

## 6. Canonical Category Mapping

### 6a. Genuine naming variants → standardize
Folding these into one canonical value directly improves the accuracy of any count, trend, or report grouped by category.

| Raw Variants | Canonical Value |
|---|---|
| "Outside Building", "OUTSIDE BUILDING" | Outside Building |
| "PLUMBING", "Plumbing" | Plumbing |
| "Building Marshal's Office", "Building Marshals office" | Building Marshal's Office |
| "ELECTRIC", "Electric" | Electric |
| "APPLIANCE", "Appliance" | Appliance |
| "WATER LEAK", "Water Leak" | Water Leak |
| "CONSTRUCTION", "Construction" | Construction |
| "SAFETY", "Safety" | Safety |
| "LEAD", "Lead" | Lead |
| "Elevator", "ELEVATOR" | Elevator |
| "UNSANITARY CONDITION", "Unsanitary Condition" | Unsanitary Condition |
| "General", "GENERAL" | General |
| "Mold", "MOLD" | Mold |

### 6b. Non-conforming values → quarantine, do not standardize
Folding these into a canonical bucket would hide the real signal instead of surfacing it. The correct governance action is to exclude and investigate the ingestion gap — not disguise it as a naming variant.

| Cluster (raw variant count) | Pattern | Action |
|---|---|---|
| 8 variants of repeated path-traversal-style segments | Path traversal probe | Exclude from category field entirely |
| "Misc. Comments" + appended payload (4 variants) | Injection payload appended onto a legitimate category value | Quarantine; the clean "Misc. Comments" value likely exists elsewhere and is itself valid |
| 4 variants referencing a system config file path | Path / config-disclosure probe | Exclude from category field entirely |
| "Misc. Comments" + SQL-style payload (3 variants) | SQL-injection-style payload appended onto a legitimate category value | Quarantine |
| 3 variants of a config-file-path pattern | Path / config-disclosure probe | Exclude from category field entirely |
| "Misc. Comments" + timing-based payload (2 variants) | Command/timing-injection-style payload appended onto a legitimate category value | Quarantine |
| 2 variants of a shell-command-style string | Command-injection-style payload | Exclude from category field entirely |
| 2 variants referencing a form-handler file path | Path / file-disclosure probe | Exclude from category field entirely |
| 2 variants of a directory-query-style string | LDAP-injection-style payload | Exclude from category field entirely |
| "Misc. Comments" + SQL-select payload (2 variants) | SQL-injection-style payload appended onto a legitimate category value | Quarantine |

## 7. Recommendations & Roadmap

**Completed as part of this project:**
- Ported the 4 quality checks into a Python notebook (pandas + BigQuery client) — results independently matched the manual SQL findings exactly, validating the automation
- Built a one-page Looker Studio quality scorecard tracking all 7 metrics, with conditional formatting flagging failing checks at a glance

**Recommended for production adoption:**
- Schedule the 4 quality rules as a recurring BigQuery scheduled query, so regressions are caught automatically rather than discovered ad hoc
- Implement allowlist validation on `complaint_type` at ingestion, with a quarantine path for anything that doesn't match

**Ownership:**
- Assign explicit field-level stewards per the data dictionary in Section 4, so each quality rule has a clear owner accountable for remediation

## 8. Tools & Reproducibility

Built entirely with free, browser-based tools — no billing account, no local installs. Fully reproducible by anyone with a Google account via BigQuery Sandbox mode.

## 9. Skills Demonstrated

- **SQL:** aggregation, regex-based normalization, deduplication logic, data profiling
- **Python:** pandas, automated data quality checks, BigQuery client library
- **Data governance:** data dictionary design, quality rule definition, controlled-vocabulary/taxonomy design
- **Google Cloud:** BigQuery (public datasets, Sandbox mode), Looker Studio (dashboarding)
- **Analytical judgment:** distinguishing genuine data-quality drift from adversarial/non-conforming input, and translating technical findings into business-risk narrative
