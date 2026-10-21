# Tableau build guide: Retention Command Center (3 dashboards)

Tool: Tableau Public Desktop (free). Publishing puts the workbook on the public web. The data is synthetic and anonymised (hashed employee keys, no names or emails), so that is fine.

Each step ends with a **Check** so you know it is right before moving on.

## Data files (all in `outputs/`)
| File | Used for | Rows |
|---|---|---|
| `tableau_attrition_history.csv` | Dashboard 1 (KPIs, department, tenure, satisfaction) | 2,865 |
| `tableau_attrition_rates.csv` | Optional: confidence intervals | 36 |
| `tableau_employee_risk.csv` | Dashboard 2 (action list) | 2,544 |
| `tableau_fairness_audit.csv` | Dashboard 3 (traffic lights) | 12 |
| `tableau_fairness_by_department.csv` | Dashboard 3 (job-mix view) | 12 |
| `fairness_accuracy_tradeoff.csv` | Dashboard 3 (trade-off table) | 2 |
| `shap_global_importance.csv` | Dashboard 2 (global drivers) | 20 |

Create **one data source per file** (Data > New Data Source > Text file). Do not join them. Each dashboard uses only its own sources.

## Numbers you must be able to reproduce (use as your Checks)
- Cohort: 2,865 employees, 321 voluntary leavers, **11.2%** attrition.
- By department: Production **14.8%** (287/1,944), Software Engineering **17.9%** (20/112, small n), Sales 3.2%, IT/IS 0.8%, Admin 1.3%, Executive 0%.
- Production Technician II **23.5%** vs Technician I **12.2%**.
- Tenure: <1 yr 9.5%, 1-2 yrs 10.0%, 2-3 yrs 11.5%, 3-4 yrs 13.3%, 4+ yrs 13.7% (gradual rise, no 3-5 year spike).
- Satisfaction 1 to 5: 10.5%, 11.3%, 9.5%, 12.5%, 12.2% (no relationship).
- Current employees scored: 2,544. High risk: 227 (224 Production, 3 Software Eng).
- Model (random forest) test AUC **0.688** (95% CI 0.625-0.750) vs department-only baseline 0.615. Top 20% flagged captures 39% of leavers (1.9x lift).
- Gender flag rate: Female 24.3%, Male 14.5%. Four-fifths ratio on "not flagged": **0.886**. Race minimum 0.955. Age minimum 0.949.

---

## Dashboard 1: Executive Overview ("Where is attrition concentrated?")
Source: `tableau_attrition_history.csv`.

**Step 1.1: Calculated fields** (Analysis > Create Calculated Field)
- `Attrition Rate` = `AVG([left])` (format: percentage, 1 decimal)
- `Leavers` = `SUM([left])`
- `Headcount` = `COUNT([employee_key])`
- `Overall Rate` = `{ FIXED : AVG([left]) }`

**Check:** drag `Attrition Rate` to Text on a blank sheet. It shows 11.2%.

**Step 1.2: Sheet "KPI"**: put `Headcount`, `Leavers`, `Attrition Rate` as three text tiles (Measure Values, or three small sheets).

**Step 1.3: Sheet "By Department"**
- Rows: `DepartmentType`. Columns: `Attrition Rate`. Sort descending.
- Label: `Attrition Rate`; Tooltip: add `Headcount`, `Leavers`.
- Add the reference line: Analytics > Reference Line > Constant on axis, value = `Overall Rate` (average line, label "Overall 11.2%").
- Title: "Attrition is concentrated in Production and Software Engineering".

**Check:** Production 14.8%, Software Engineering 17.9%, IT/IS 0.8%.

**Step 1.4: Sheet "By Tenure"**
- Columns: `tenure_band`. Rows: `Attrition Rate`. Right-click `tenure_band` > Sort > Manual: `<1 yr`, `1-2 yrs`, `2-3 yrs`, `3-4 yrs`, `4+ yrs` (the default alphabetical order is wrong).
- Mark type: Line with circles. Add the same reference line.
- Title: "Attrition rises gradually with tenure (9.5% to 13.7%)".

**Step 1.5: Sheet "Satisfaction"**
- Columns: `Satisfaction Score` (convert to Dimension). Rows: `Attrition Rate`. Bar chart, same reference line.
- Title: "Satisfaction score does not predict exits".

**Step 1.6: Assemble** (New Dashboard > Size: Fixed, 1200 x 800). KPI tiles on top, Department on the left, Tenure and Satisfaction on the right. Add a text box: "Synthetic HR dataset. Cohort excludes terminations for cause and future starters."

**Optional (stretch): confidence intervals.** Use `tableau_attrition_rates.csv`: filter `dimension` = `tenure_band`, plot `ci_low` and `ci_high` as a dual-axis area behind `attrition_rate`.

---

## Dashboard 2: HR Action List ("Who do we talk to first?")
Source: `tableau_employee_risk.csv`.

**Step 2.1: Parameter**: create `Top N` (Integer, range 10 to 300, step 10, default 50). Show the parameter control.

**Step 2.2: Sheet "Priority List"**
- Rows: `employee_key`, `DepartmentType`, `title_group`, `tenure_years` (1 decimal), `driver_1`, `driver_2`, `driver_3`, `suggested_action`.
- Columns: `risk_pct_rank` (format as percentage; this is a **ranking percentile**, not a probability).
- Filter `employee_key` > Top tab > By field > Top `[Top N]` by `Max of risk_score`.
- Filters (show as controls): `risk_band` (default High + Medium), `DepartmentType`, `title_group`.
- Sort by `risk_score` descending.

**Check:** with Top N = 50 and no other filters, the first row is a Production Technician II with Job title / Department / Tenure as drivers.

**Step 2.3: Sheet "Risk Band by Department"**: Columns `DepartmentType`, Rows `COUNT(employee_key)`, colour by `risk_band` (High = dark red, Medium = amber, Low = grey). This view shows that nearly all High/Medium employees are in Production.

**Step 2.4: Sheet "Global Drivers"** (source `shap_global_importance.csv`): bar chart of `mean_abs_shap` by `feature`, top 10 sorted descending. DepartmentType and title_group dominate.

**Step 2.5: Assemble** (1200 x 800). Priority list at the bottom, the other two sheets on top. Add a caveat text box: "Test AUC 0.69 (95% CI 0.63-0.75). Within Production the ranking AUC is 0.60. Use as a prioritisation aid for conversations, not for decisions about individuals. Drivers come from SHAP."

---

## Dashboard 3: Fairness & Governance ("Is the model treating groups equally?")
Source: `tableau_fairness_audit.csv`.

**Step 3.1: Calculated field** `Four-fifths status` (row-level, no aggregation):
```
IF [dir_favourable_vs_best] >= 0.9 THEN "Pass"
ELSEIF [dir_favourable_vs_best] >= 0.8 THEN "Watch"
ELSE "Fail" END
```
Colours: Pass = green, Watch = amber, Fail = red. Always show the text label too, not colour alone.

**Step 3.2: Sheet "Audit Table"**: Rows: `attribute`, `group`. Columns (Measure Names): `n`, `flag_rate`, `actual_attrition_rate`, `dir_favourable_vs_best`, `tpr`, `fpr`. Colour `Four-fifths status`.

**Check:** Female/Male `dir_favourable_vs_best` = 0.886 / 1.000 (Watch). All race and age rows >= 0.9.

**Step 3.3: Sheet "Flag Rate by Group"**: bar chart of `flag_rate` by `group`, panel by `attribute`. Female 24.3% vs Male 14.5% is visible.

**Step 3.4: Sheet "Job Mix"** (source `tableau_fairness_by_department.csv`): Columns `DepartmentType`, Rows `flag_rate`, colour `GenderCode`. Filter out departments with n < 30 per gender (flag with caption). Takeaway: 76% of women work in Production versus 58% of men, but a gap remains inside Production (30.9% vs 24.2%).

**Step 3.5: Sheet "Trade-off"** (source `fairness_accuracy_tradeoff.csv`): text table with `variant`, `oof_auc`, `flag_rate_female`, `flag_rate_male`, `dir_favourable_not_flagged`. Takeaway: removing job-structure features closes the gender gap (0.886 to 0.997) but drops AUC from 0.705 to 0.520, which is chance level.

**Step 3.6: Assemble** (1200 x 800) with a methods text box:
- "Flagged = top 20% of out-of-fold risk scores. Model excludes gender, race, and age."
- "Four-fifths rule applied to the favourable outcome (not flagged). No group-specific thresholds were used."
- "No mitigation applied. Finding: gender gap is mostly job mix; flagged cases need human review."

---

## Publish and capture
1. Sign in to Tableau Public: Server > Tableau Public > Save to Tableau Public As...
2. Copy the public URL into the README.
3. For each dashboard: Dashboard > Export Image. Save as `dashboard_screenshots/01_executive.png`, `02_action_list.png`, `03_fairness.png`.
