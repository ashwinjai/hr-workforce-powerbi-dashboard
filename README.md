# HR & Workforce Analytics Dashboard

## Project Overview:
The "HR & Workforce Analytics" project analyses the workforce of a 1,300-employee organisation operating across the UAE and Saudi Arabia (Dubai, Abu Dhabi, Riyadh and Remote). It tracks how the workforce grows, how well hiring keeps up with the headcount plan, who leaves the organisation, when they leave and why. The project is built end to end in **Power BI Desktop**, using Power Query for data preparation, a star-schema data model, around 60 DAX measures and a two-page interactive report with a custom dark theme. The report is saved as a **Power BI Project (PBIP)**, so the data model (TMDL) and the report layout (PBIR) are stored as plain text and version-controlled in this repository.

> All data in this project is **synthetic** (generated with Python). No real employees or companies are represented.

## High level architecture diagram:
![Architecture](images/architecture.png)

## Data Generation:
In this phase, a synthetic HR dataset is generated with Python (pandas and numpy) to reflect realistic workforce behaviour. It covers 2,085 employees hired between 2002 and 2026, including people who have left the organisation. Business rules are built into the data so that the analysis produces meaningful patterns: employees in their first year are far more likely to leave, employees without a promotion in three or more years and employees working high overtime leave more often, and Sales and Customer Success have higher attrition than Finance and HR. The data is exported to six Excel files:

| Table | Rows | Description |
|---|---|---|
| Employees | 2,085 | Gender, birth date, department, location, manager, job title, level, monthly salary (AED), performance rating, hire date, last promotion, weekly overtime, termination date, exit type, exit reason, regretted exit |
| Departments | 7 | Engineering, Operations, Sales, Finance, Marketing, Customer Success, HR |
| Locations | 4 | Dubai, Abu Dhabi, Riyadh, Remote |
| Requisitions | 507 | Job openings for 2025–2026 with open date, filled date and status |
| HeadcountPlan | 84 | Planned headcount per department per month for 2026 |
| Calendar | 9,862 | Daily date table from 2000 to 2026 |

## Power Query Editor:
In this phase, the six Excel files are connected to Power BI and loaded through the Power Query Editor. Headers are promoted, and every column is given the correct data type (date, whole number, decimal and text) so that date intelligence and aggregations behave correctly. A dedicated **Measure_Tbl** is created with Enter Data to hold all DAX measures in one place, separate from the data tables. Two calculated columns are added to the Employees table: **Age Band** (18–24, 25–34, 35–44, 45–54, 55+) and **Tenure Band** (0–1, 1–3, 3–5, 5+ years, measured at exit or at year end for active employees). These columns group employees for the age-profile and attrition-heatmap visuals.

![Power Query](images/power_query.png)

## Data Modeling:
In the data modelling phase, a **Star Schema** is implemented with Employees as the central fact table and Departments, Locations and Calendar as dimensions. Requisitions and HeadcountPlan share the Department and Location dimensions, so one slicer filters every visual. All relationships are one-to-many and single-directional. The Calendar table is marked as the model's date table to enable time intelligence (SAMEPERIODLASTYEAR, DATESINPERIOD, TOTALYTD).

Because an employee has both a hire date and a termination date, the relationship between Employees[HireDate] and Calendar[Date] is kept **inactive** and switched on only inside the Hires measure with USERELATIONSHIP. Headcount is a **point-in-time** measure: it counts everyone hired on or before the selected date who had not left by that date, so it is calculated from the dates rather than through a relationship.

![Data Model](images/data_model.png)

## DAX Measures:
About 60 measures are grouped into five families:

| Family | Measures |
|---|---|
| Headcount | Headcount, Headcount LY, Headcount YoY %, Avg Headcount, Planned Headcount, Gap to Plan |
| Hiring | Hires, Hires LY, Hires YTD, Open Positions, Open Positions LM |
| Attrition | Exits, Exits LY, Attrition Rate, Attrition Rate LY, Rolling 12M Attrition, Attrition Target, Voluntary %, Regretted Exits |
| Tenure | Avg Tenure (yrs), Avg Tenure LY |
| Display | Card value, Δ label and Δ colour measures for every KPI, heatmap colour measures |

```dax
Headcount =
VAR d = MAX ( 'Calendar'[Date] )
RETURN
    CALCULATE (
        COUNTROWS ( Employees ),
        Employees[HireDate] <= d,
        OR ( ISBLANK ( Employees[TerminationDate] ), Employees[TerminationDate] > d ),
        REMOVEFILTERS ( 'Calendar' )
    )

Hires = CALCULATE ( COUNTROWS ( Employees ), USERELATIONSHIP ( Employees[HireDate], 'Calendar'[Date] ) )

Rolling 12M Attrition =
VAR d = MAX ( 'Calendar'[Date] )
RETURN
    DIVIDE (
        CALCULATE ( [Exits], DATESINPERIOD ( 'Calendar'[Date], d, -12, MONTH ) ),
        CALCULATE ( [Avg Headcount], DATESINPERIOD ( 'Calendar'[Date], d, -12, MONTH ) )
    )

Attrition Δ Label = FORMAT ( [Attrition Δ pts], "▲ 0.0 pts;▼ 0.0 pts;0.0 pts" ) & " vs last year"
Attrition Δ Color = IF ( [Attrition Δ pts] <= 0, "#2DD4BF", "#F59E0B" )
```

The KPI colour measures use **inverted logic** where a fall is good news (attrition, exits, regretted exits, open positions): teal means good, amber means attention needed. Each change is always shown with a ▲/▼ arrow, so it can be read without relying on colour.

## Power BI Dashboard:

### Page 1 – Workforce Overview
![Workforce Overview](images/overview.png)

### Page 2 – Attrition Analysis
![Attrition Analysis](images/attrition.png)

Both pages share Year, Department and Location slicers, and the page tabs at the top switch between them. All figures below are for **FY 2026**.

## Key Metrics:

**1. Total Headcount, New Hires, Attrition Rate, Average Tenure, Open Positions**

![KPI cards](images/kpi_cards.png)

The organisation closed 2026 with **1,294 employees**, up **10.4%** from 1,172 a year earlier. **242 people were hired** during the year, 14 more than in 2025. The attrition rate fell to **9.7%** from 11.9% (▼ 2.2 pts), and average tenure rose to **4.4 years** (▲ 0.3). **37 positions** were still open at year end, 10 fewer than at the end of November.

**2. Headcount Trend vs Plan**

![Headcount trend vs plan](images/headcount_trend.png)

Headcount grew every month, from 1,180 in January to 1,294 in December. However, it stayed **behind the hiring plan all year**: the gap widened to around 20 people by mid-year and the year closed **21 below plan** (1,294 vs 1,315). This shows that hiring capacity, rather than demand, is the constraint on growth.

**3. Headcount by Department**

![Headcount by department](images/headcount_by_department.png)

Engineering is the largest department with 412 employees (32% of the workforce), followed by Operations (286), Sales (214), Finance (128), Marketing (96), Customer Success (84) and HR (74).

**4. Workforce Profile – Gender, Age and Location**

![Workforce profile](images/workforce_profile.png)

The workforce is **46% female and 54% male**. It is young: the 25–34 age band is the largest group (494 employees), followed by 35–44 (397), 45–54 (221), 55+ (93) and 18–24 (89). Dubai is the main hub with 548 employees (42%), followed by Abu Dhabi (312), Riyadh (236) and Remote (198).

**5. Hires vs Exits by Quarter**

![Hires vs exits](images/hires_vs_exits.png)

Hiring outpaced exits in every quarter. Q3 was the busiest quarter on both sides (71 hires, 38 exits), while Q4 was the quietest (49 hires, 22 exits). Over the year, 242 hires against 120 exits gave a net gain of 122 employees.

**6. Exits, Voluntary Exits and Regretted Exits**

![Attrition KPIs](images/attrition_kpis.png)

**120 employees left** in 2026, 14 fewer than in 2025. **78%** of exits were voluntary resignations. **25 were regretted exits** (high performers who chose to leave), down from 44 the year before.

**7. Exits by Department and Tenure**

![Exits heatmap](images/exits_heatmap.png)

About **half of all exits (59 of 120) happened in the first year** of employment, making onboarding and early engagement the biggest retention lever. Engineering (14), Operations (12) and Sales (10) lose the most new joiners. Relative to their size, Marketing (17%), Customer Success (13%) and Sales (13%) lose the most people.

**8. Top Voluntary Exit Reasons**

![Exit reasons](images/exit_reasons.png)

**Career growth** is the most common reason for resigning (31 exits), ahead of relocation (24) and compensation (15). Manager relationship (11), work-life balance (7) and other reasons (6) follow. Clear career paths and internal mobility would address the largest share of voluntary attrition.

**9. Rolling 12-Month Attrition**

![Rolling attrition](images/rolling_attrition.png)

Rolling 12-month attrition improved steadily from **12.1% in January to 9.7% in December**, but it is still above the **9% target**.

## Business Process Description:
![HR process](images/hr_process.png)

◾ **Workforce Planning:** Each department sets a monthly headcount plan for the year based on business growth targets. This plan becomes the benchmark that actual headcount is measured against.

◾ **Requisition Opened:** When a new role is approved, or an employee leaves and needs to be replaced, HR opens a requisition. Open requisitions are tracked as Open Positions until they are filled.

◾ **Recruit & Select:** Talent Acquisition sources, screens and interviews candidates. The time between opening and filling a requisition is the time-to-fill.

◾ **Hire & Onboard:** The selected candidate joins, and the hire date is recorded. The first year is the highest-risk period for attrition, so onboarding quality directly affects retention.

◾ **Develop & Promote:** Employees are reviewed for performance (rated 1–5) and considered for promotion. A long gap since the last promotion is one of the main drivers of resignation.

◾ **Retain & Engage:** Managers monitor workload (including overtime), engagement and career conversations to keep high performers.

◾ **Exit:** When an employee leaves, the exit is classified as:
- **Voluntary:** a resignation, with the reason captured (career growth, compensation, relocation, work-life balance, manager relationship or other)
- **Involuntary:** termination for performance, restructuring or end of contract

◾ **Exit Analysis:** HR records whether the exit was **regretted** (a high performer leaving voluntarily) and the employee's tenure at exit, and uses these patterns to target retention actions.

◾ **Backfill:** A replacement requisition is opened, and the cycle starts again.

## Tools & Skills:
Python (pandas, numpy) · Power BI Desktop · Power Query · DAX (time intelligence, inactive relationships, point-in-time measures, conditional-formatting measures) · Star-schema data modelling · PBIP / TMDL / PBIR source control · Report UX design

## How to Open:
1. Install the latest **Power BI Desktop**.
2. Clone or download this repository.
3. Open `HR_WorkForce_Dashboard.pbip` (or the `.pbix` file).
4. The Excel sources use local file paths. Go to **Transform data → Data source settings → Change Source** and point each file to your download folder.
5. Click **Refresh**.

## About
HR & Workforce analytics using Python, Power Query, DAX and Power BI. Built by **Ashwin Jai** · [GitHub](https://github.com/ashwinjai)
