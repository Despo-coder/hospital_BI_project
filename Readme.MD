# BI Portfolio Project — Patient Retention & Care Insights **hypothetical**

**Project name:** Patient Retention & Care Insights Dashboard

**Origin:** Clinical Director meeting notes and sample visit dataset.

---

## Executive summary

This portfolio project walks a prospective employer through a complete BI engagement: from scoping and stakeholder management through data modeling, visualization, and presentation. The eventual analysis uses a visit-level dataset (patient demographics, visit details, charges, insurance, departments, length of stay, specialist consultations, survey responses) and focuses on improving patient retention, operational efficiency, financial insights, and patient experience.

**Key outcomes:**

* Cleaned, versioned dataset and documented ETL pipeline
* Star schema data warehouse (FactVisits + Dim tables)
* Interactive public dashboard (Tableau Public or Power BI) showcasing retention, charges, satisfaction, and operational KPIs
* Slide deck summarizing findings and action recommendations

---

## Files & data sources (local)

* `Health_Care_Analytics_Modified.csv` — primary visit-level dataset (place in MySQL import folder).
* `CLinical_Director_Meeting_Notes.docx` — project scoping, stakeholders, and directives (source of requirements).

> Replace the placeholder CSV above with the canonical export when loading into MySQL Workbench.

---

## Audience & stakeholders

* **Primary sponsor:** Clinical Director
* **Project manager:** Practice Manager
* **BI Analyst (owner):** You — responsible for ETL, modeling, dashboard
* **IT & Data Systems Lead:** Data ingestion, security, monthly refresh
* **Finance Officer:** Charge data validation
* **Nursing Supervisor / Patient Experience Coordinator:** Clinical interpretation and survey process improvements

### RACI (high level)

* **Responsible:** BI Analyst (ETL, dashboard)
* **Accountable:** Clinical Director
* **Consulted:** Finance, Nursing, IT
* **Informed:** All practice leadership

---

## Project scope & objectives

**Scope (in-scope):**

* Combine Hospital A and B visit records
* Build retention metrics (1-year, 5-year return rates)
* Compute visit- & patient-level charges and compare by hospital/department
* Analyze survey completion vs retention and satisfaction
* Build drillable dashboard with filters (hospital, department, age group, insurance)

**Out-of-scope:**

* Real-time streaming ingestion
* Patient-identifiable PHI sharing beyond the secure environment

**Success criteria:**

* Dashboard published to a public view (Tableau Public / Power BI) with sample anonymized data
* Clear recommendations with at least three priority actions (clinical or operational)

---

## Deliverables

1. Markdown project document (this file) with links to artifacts
2. ETL scripts & SQL (MySQL Workbench SQL file)
3. Data warehouse schema diagram (PNG)
4. Published dashboard (Tableau Public or Power BI public link)
5. Presentation slides (PowerPoint/PDF)
6. README with reproducible steps

---

## Timeline (example — adapt to real dates)

* Week 0: Kickoff & data intake
* Week 1: Data cleaning, profiling, Bronze ingestion
* Week 2: Transformations (Silver), data model design
* Week 3: Gold dataset (views), dashboard mockups
* Week 4: Dashboard build, internal review
* Week 5: Finalize dashboard + presentation

---

## Technical architecture (high-level)

1. **Bronze (Raw)** — CSV loaded into staging table in MySQL Workbench (`stg_visits_raw`). No transformations.
2. **Silver (Transformed)** — Cleaned, typed, deduped tables (`visits_stg`, `visits_parsed`) with standardized codes (dept codes, insurance categories)
3. **Gold (Curated views)** — Business-ready views (not physical tables): `vw_fact_visits`, `vw_dim_patient`, `vw_dim_department`, `vw_dim_insurance` used by BI tool.
4. **Presentation layer** — Tableau Public or Power BI connected to Gold views or extracted dataset.

---

## Data model (star schema suggestion)

**FactVisits** (view)`

* visit_id (PK)
* patient_id
* visit_date
* hospital_id
* department_id
* length_of_stay
* charge_amount
* insurance_id
* specialist_flag (bool)
* survey_completed (bool)
* satisfaction_score (numeric)

## **DimPatient**

* patient_id
* dob (or age_bucket)
* gender
* postal_region

## **DimDepartment**

* department_id
* department_name

## **DimHospital**

* hospital_id
* hospital_name

## **DimInsurance**

* insurance_id
* insurance_type

---

## ETL steps (detailed)

### 1) Bronze — Load raw CSV

* Place `Health_Care_Analytics_Modified.csv` in an accessible folder.
* In MySQL Workbench: use `Table Data Import Wizard` to import to `stg_visits_raw`.

`CREATE TABLE stg_visits_raw ( /* use dynamic column types as imported */ );`

### 2) Silver — Cleaning & Transform

* Remove duplicates: identify by `visit_id` or combination `(patient_id, visit_date, department)`.
* Standardize date formats: `STR_TO_DATE(visit_date, '%Y-%m-%d')`
* Convert numeric fields: `CAST(charge_amount AS DECIMAL(10,2))`
* Normalize categorical values (department names, insurance types)
* Handle missing values: impute or flag `NULL` for sensitive fields

Sample SQL transform:

```sql
-- dedupe into visits_parsed
CREATE TABLE visits_parsed AS
SELECT DISTINCT
  COALESCE(visit_id, UUID()) as visit_id,
  patient_id,
  STR_TO_DATE(visit_date, '%Y-%m-%d') AS visit_date,
  hospital AS hospital_name,
  department AS department_name,
  CAST(charge AS DECIMAL(10,2)) AS charge_amount,
  length_of_stay,
  insurance,
  specialist_flag,
  survey_completed,
  satisfaction_score
FROM stg_visits_raw;
```

### 3) Gold — Curated Business Views

* Create views that implement business rules (e.g., retention windows, derived KPIs).

Example view for normalized fact:

```sql
CREATE VIEW vw_fact_visits AS
SELECT
  v.visit_id,
  v.patient_id,
  v.visit_date,
  h.hospital_id,
  d.department_id,
  v.length_of_stay,
  v.charge_amount,
  i.insurance_id,
  v.specialist_flag,
  v.survey_completed,
  v.satisfaction_score
FROM visits_parsed v
JOIN dim_hospital h ON v.hospital_name = h.hospital_name
JOIN dim_department d ON v.department_name = d.department_name
LEFT JOIN dim_insurance i ON v.insurance = i.insurance_type;
```

---

## Key KPIs & calculations

* **Patient retention (1yr/5yr):** count distinct patients with repeat visits within calendar windows divided by baseline patient count
* **Average charge (mean/median) by hospital / department**
* **Length of stay** (avg, median, distribution)
* **Survey completion rate** = completed_surveys / total_visits
* **Satisfaction distribution** (boxplots, mean by group)
* **Charge outliers** — use IQR or z-score

**Sample SQL for 1-year retention rate (cohort approach):**

```sql
WITH first_visits AS (
  SELECT patient_id, MIN(visit_date) AS first_visit
  FROM vw_fact_visits
  GROUP BY patient_id
),
retained AS (
  SELECT f.patient_id
  FROM first_visits f
  JOIN vw_fact_visits v ON v.patient_id = f.patient_id
  WHERE v.visit_date BETWEEN DATE_ADD(f.first_visit, INTERVAL 1 DAY) AND DATE_ADD(f.first_visit, INTERVAL 365 DAY)
  GROUP BY f.patient_id
)
SELECT (SELECT COUNT(*) FROM retained) / (SELECT COUNT(*) FROM first_visits) AS retention_1yr;
```

---

## Data quality checks

* Row counts at each layer (bronze → silver → gold)
* Null checks on critical fields (patient_id, visit_date, charge)
* Referential integrity (dim keys referenced in fact)
* Distribution checks (histograms) for charges and length_of_stay
* Outlier detection (IQR rule)

---

## Version control & reproducibility

* Store SQL scripts and documentation in a Git repo (include README, migration scripts)
* Tag releases (v0.1, v1.0) when publishing dashboards
* Store QA checkpoints and test cases (example: sample patients to validate retention logic)

---

## Dashboard design & wireframes

1. **Landing / Executive view**

   * High-level KPIs (retention %, avg charge, avg LOS, survey completion)
   * Trend line (visits over time)
2. **Deep-dive: Retention**

   * Cohort analysis by first-visit month
   * Filter by age bucket, hospital, department
3. **Charges & Finance**

   * Mean/median by hospital & department
   * Outlier table with drill-to-visit
4. **Patient Experience**

   * Survey completion vs satisfaction heatmap by department
5. **Operational efficiency**

   * Avg length of stay by department with control limits

**Visualization types:** KPI cards, line charts, cohort heatmap, boxplots, histogram, bar charts, tables with drill-through.

---

## Publishing & sharing

* **Tableau Public:** publish workbook after anonymizing data. Copy public link into this markdown under `Public Dashboards`.
* **Power BI:** publish to Power BI Service and create a public web embed (if allowed) or publish the report as PDF and provide link.

**Markdown link templates:**

* Tableau Public: `[View dashboard — Tableau Public](https://public.tableau.com/views/<workbook>/<view>)`
* Power BI: `[View report — Power BI] (https://app.powerbi.com/view?r=<report_id>)`

Replace `<workbook>/<view>` and `<report_id>` with the actual IDs after publishing.

---

## Presentation / Storytelling structure (PowerPoint)

1. Title slide & TL;DR (1 slide)
2. Problem & dataset description (1 slide)
3. Approach (ETL + modeling) (1 slide)
4. Key findings — top 3 insights (2–3 slides)
5. Recommendations & next steps (1 slide)
6. Appendix: data dictionary, SQL snippets, methodology (2–3 slides)

---

## README checklist for reproducibility

* [ ] CSV imported to `stg_visits_raw`
* [ ] `visits_parsed` created
* [ ] Dim tables populated
* [ ] Views created (`vw_fact_visits`, etc.)
* [ ] Dashboard connected and validated
* [ ] Presentation exported

---

## Appendix: helpful SQL snippets

* Count rows per layer

```sql
SELECT 'bronze' AS layer, COUNT(*) FROM stg_visits_raw
UNION ALL
SELECT 'silver', COUNT(*) FROM visits_parsed
UNION ALL
SELECT 'gold', (SELECT COUNT(*) FROM vw_fact_visits);
```

* Detect duplicates

```sql
SELECT patient_id, visit_date, COUNT(*) c
FROM visits_parsed
GROUP BY patient_id, visit_date
HAVING c > 1;
```

* Basic stats for charges

```sql
SELECT
  COUNT(*) AS n,
  AVG(charge_amount) AS mean_charge,
  MEDIAN(charge_amount) AS median_charge,
  STDDEV_POP(charge_amount) AS stddev_charge,
  MIN(charge_amount) AS min_charge,
  MAX(charge_amount) AS max_charge
FROM vw_fact_visits;
```

---

## What is included in the GitHub portfolio repo

* `README.md` (short project summary + link to this markdown)
* `data/` folder (sample anonymized CSV)
* `sql/` folder (bronze_schemas.sql, silver_transforms.sql, gold_views.sql)
* `docs/` (schema diagram, meeting notes excerpt)
* `dashboards/` (link text files with published URLs)
* `presentation/` (PowerPoint or PDF)

---
