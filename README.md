# 🗺️ Power BI – Project 1 Rebuild Roadmap

A start-to-finish checklist for recreating the **HR Dashboard** from a blank `.pbix`.
Everything below is reverse-engineered from the original file, so the formulas and steps match what the original author actually did. Work top to bottom — each phase builds on the one before it.

> **The data source:** `HRDataset.xlsx`. It has 7 sheets: `Employee`, `Position`, `Department`, `MaritalStatus`, `Performance`, `Manager`, and `EmployeeNew`.
> You'll use the first 6. `EmployeeNew` is a leftover/practice sheet and is **not** part of the final model — your teacher explicitly says *don't take EmployeeNew*.

---

## 🎯 Read this first — what's actually being graded

Per the assignment brief, this is a **re-skin, not a pixel-perfect clone.** You copy the *bones*, you invent the *skin*:

- ✅ **Match exactly:** data model, DAX formulas (you may copy or rewrite them), layout structure, which chart types go where, canvas size.
- 🎨 **Make your own:** a **new colour theme** (harmonious + aesthetically pleasing), your **own icons**, your own visual identity. Do **not** reuse the sample's colours — using a fresh palette is a graded requirement.
- 📐 **Canvas size = 1920 × 1080** (same as the sample).
- 🆕 **The new technique this project teaches is the Field Parameter** (the `Demographics` column is one — see Phase 6).

The real skills being marked: layout design, chart customisation, colour harmony, using varied chart types, and reading charts. So the **report pages (Phases 8–10) are where the marks are** — give them the most care. The data work (Phases 1–7) just needs to be correct.

---

## 📍 PHASE 0 — Setup (5 min)

- [ ] Open Power BI Desktop → **blank report**.
- [ ] **File → Save As** → name it something like `HR_Dashboard_Rebuild.pbix`.
- [ ] Put `HRDataset.xlsx` in a known folder (e.g. your project folder). You'll point Power BI at *your* copy, not the original author's path.
- [ ] **Plan your colour palette now** (before building visuals). Pick 4–6 harmonious colours that are clearly *different* from the sample, plus a source for icons. Tools like Coolors, Adobe Color, or a Power BI theme generator help. You'll apply this in Phase 8.

---

## 📍 PHASE 1 — Connect & load the data (10 min)

- [ ] **Home → Get Data → Excel workbook** → pick `HRDataset.xlsx`.
- [ ] In the Navigator, tick these 6 sheets: **Employee, Position, Department, MaritalStatus, Performance, Manager**.
- [ ] Click **Transform Data** (NOT "Load"). All cleaning happens in Power Query first.

You should now be in the **Power Query Editor** with 6 queries on the left.

---

## 📍 PHASE 2 — Clean each table in Power Query

The dimension sheets all have a **junk title row** above the real headers (e.g. "Department info"), so each needs **Promote Headers run twice**. The Employee sheet has **two** junk rows, so it needs an extra "remove top row" step. This is the part people find fiddly — follow it carefully.

### 2A. Department, Position, MaritalStatus, Performance (the simple dimensions)

For **each** of these four, the pattern is identical:

- [ ] Select the query → **Transform → Use First Row as Headers** (this promotes the junk title).
- [ ] Power Query auto-adds a "Changed Type" step — fine.
- [ ] **Use First Row as Headers again** (this promotes the *real* header row).
- [ ] Set the final data types:

| Query | Columns & types |
|---|---|
| **Department** | `DeptID` = Whole Number, `Department` = Text |
| **Position** | `PositionID` = Whole Number, `Position` = Text, `DeptID` = Whole Number |
| **MaritalStatus** | `MaritalStatusID` = Whole Number, `MaritalDesc` = Text |
| **Performance** | `PerfScoreID` = Whole Number, `PerformanceScore` = Text |

### 2B. Manager (helper query — connection only)

- [ ] The Manager sheet already has clean headers (`ManagerID`, `ManagerName`) — just confirm types (Whole Number / Text).
- [ ] **Right-click the Manager query → uncheck "Enable load."** It's only used to look up manager names inside the Employee query, so it shouldn't load as its own table.

### 2C. Employee (the main / fact table — most steps)

Do these in order:

- [ ] **Use First Row as Headers** (promotes the "EMPLOYEE DATA" junk row).
- [ ] **Home → Remove Rows → Remove Top Rows → 1** (drops the "Data as of…" subtitle row).
- [ ] **Use First Row as Headers again** (promotes the real header row).
- [ ] Set types: `ManagerID`, `PositionID`, `MaritalStatusID`, `PerformanceScoreID`, `Salary`, `SatisfactionScore`, `AbsenceDays` = Whole Number; `Birthday`, `DateofHire`, `DateofTermination` = Date; `EngagementScore` = Decimal Number; the rest = Text.
- [ ] **Remove the `Location` column.**
- [ ] **Replace Values** on `Gender`: `"M "` → `Male`, then `"F"` → `Female`.
- [ ] Change `Termination` to **Text**, then Replace Values: `"0"` → `No`, `"1"` → `Yes`.
- [ ] **Replace Values** on `TerminationReason`: replace `null` → `NA-StillEmployed`.
- [ ] **Transform → Rounding → Round** `EngagementScore` to **1 decimal**.
- [ ] **Split Column → By Delimiter** on the `Employee` column, delimiter `-`, into two columns.
- [ ] **Rename**: `Employee.1` → `EmployeeID` (Whole Number), `Employee.2` → `EmployeeName` (Text).
- [ ] **Home → Merge Queries**: merge Employee with **Manager** on `ManagerID` (Join kind: **Left Outer**).
- [ ] **Expand** the merged column and keep only **`ManagerName`**.
- [ ] (Optional) Reorder columns so IDs and name come first.

### 2D. Apply

- [ ] **Home → Close & Apply.** Tables load into the model.

> 💡 **Shortcut tip:** If you ever want to check the exact M code, in Power Query use **View → Advanced Editor** on each query. The full M for every step is in the companion file I extracted, if you get stuck.

---

## 📍 PHASE 3 — Build the data model / relationships (10 min)

Go to **Model view**. Create these relationships (drag the first column onto the second). All are **Many-to-One**, single cross-filter direction, active:

- [ ] `Employee[PositionID]` → `Position[PositionID]`
- [ ] `Employee[MaritalStatusID]` → `MaritalStatus[MaritalStatusID]`
- [ ] `Employee[PerformanceScoreID]` → `Performance[PerfScoreID]`
- [ ] `Position[DeptID]` → `Department[DeptID]`

> ⭐ **Note the shape:** Employee is the center (fact) table. Department is **not** linked directly to Employee — it connects *through* Position (`Employee → Position → Department`). This is a snowflake branch and is intentional.

- [ ] Confirm there are **no extra auto-relationships** Power BI guessed wrong. Delete any you didn't create above.

---

## 📍 PHASE 4 — Calculated columns (on the Employee table)

Select the **Employee** table → **Modeling → New Column** for each. Paste these exactly:

```DAX
Age =
    DIVIDE(
        NOW() - Employee[Birthday],
        365
    )
```

```DAX
Age Category =
SWITCH(
    TRUE(),
    [Age] < 25, "Under 25",
    [Age] >= 25 && [Age] < 35, "25-34",
    [Age] >= 35 && [Age] < 45, "35-44",
    [Age] >= 45 && [Age] < 55, "45-54",
    [Age] >= 55 && [Age] < 65, "55-64",
    "65+"
)
```

```DAX
Tenure (Year) =
    DATEDIFF([DateofHire], NOW(), YEAR)
```

```DAX
Absenteeism Rate =
DIVIDE(
    Employee[AbsenceDays],
    Employee[Tenure (Year)] * 365
)
```

```DAX
Cost per Absence =
    Employee[Salary] / 260 * Employee[AbsenceDays]
```

```DAX
Retention Risk =
VAR EmployeeEngagementScore = (Employee[EngagementScore] + Employee[SatisfactionScore]) / 2
RETURN
    SWITCH(
        TRUE(),
        EmployeeEngagementScore <= 3, "High",
        EmployeeEngagementScore > 3 && EmployeeEngagementScore <= 4, "Medium",
        "Low"
    )
```

These three pull text labels from the dimension tables into Employee (so they're easy to use in visuals):

```DAX
Position = RELATED(Position[Position])
```

```DAX
MaritalStatus = RELATED(MaritalStatus[MaritalDesc])
```

```DAX
Performance = RELATED(Performance[PerformanceScore])
```

- [ ] Age
- [ ] Age Category
- [ ] Tenure (Year)
- [ ] Absenteeism Rate
- [ ] Cost per Absence
- [ ] Retention Risk
- [ ] Position (RELATED)
- [ ] MaritalStatus (RELATED)
- [ ] Performance (RELATED)

> ⚠️ `Age`, `Tenure`, and `Absenteeism Rate` use `NOW()`, so their values drift over time — that's expected, not a bug.

---

## 📍 PHASE 5 — Create the Measures (in a dedicated Measure table)

### 5A. Make an empty "Measure" table to hold them

- [ ] **Home → Enter Data** → don't type anything → name the table **`Measure`** → Load. (You'll get an empty table; that's the home for all measures.)

### 5B. Add the 8 measures

Select the `Measure` table → **New Measure** for each:

```DAX
Total Employees = DISTINCTCOUNT(Employee[EmployeeID])
```

```DAX
Total Salary Expenses = SUM(Employee[Salary])
```

```DAX
Avg. Age = AVERAGE(Employee[Age])
```

```DAX
Avg. Tenure = AVERAGE(Employee[Tenure (Year)])
```

```DAX
Avg. Satisfaction Score = AVERAGE(Employee[SatisfactionScore])
```

```DAX
Avg. Engagement Score = AVERAGE(Employee[EngagementScore])
```

```DAX
Avg. Absenteeism Rate = AVERAGE(Employee[Absenteeism Rate])
```

```DAX
Employee Turnover Rate =
VAR TotalEmployees = DISTINCTCOUNT(Employee[EmployeeID])
VAR TerminatedEmployees = COUNTROWS(FILTER(Employee, NOT(ISBLANK(Employee[DateofTermination]))))
RETURN
    DIVIDE(TerminatedEmployees, TotalEmployees, 0)
```

- [ ] Total Employees
- [ ] Total Salary Expenses
- [ ] Avg. Age
- [ ] Avg. Tenure
- [ ] Avg. Satisfaction Score
- [ ] Avg. Engagement Score
- [ ] Avg. Absenteeism Rate
- [ ] Employee Turnover Rate

### 5C. Format the measures

- [ ] `Total Salary Expenses` → **Currency** (whole number).
- [ ] `Employee Turnover Rate` and `Avg. Absenteeism Rate` → **Percentage** (1–2 decimals).
- [ ] `Avg. Age`, `Avg. Tenure` → 1 decimal.

---

## 📍 PHASE 6 — Field parameter (the "Demographics" toggle)

The original has a **field parameter** named `Dimension` that lets a slicer switch which demographic a chart breaks down by.

- [ ] **Modeling → New Parameter → Fields.**
- [ ] Name it **`Dimension`** (it creates columns `Dimension` / `Dimension Fields`; the author renamed the display column to **`Demographics`**).
- [ ] Add the demographic fields users should be able to toggle, e.g. `Employee[Gender]`, `Employee[Age Category]`, `Employee[RaceDesc]`, `Employee[MaritalStatus]`, `Employee[RecruitmentSource]`.
- [ ] Tick "Add slicer to this page." You'll wire it into the demographic chart in Phase 8.

---

## 📍 PHASE 7 — Organize the model into folders (the "folders" step)

This is the tidy-up the brief asks for. In **Model view** or the **Data pane**:

- [ ] Select the measures → in Properties, set a **Display Folder** (e.g. `Key Measures`) to group them inside the `Measure` table.
- [ ] **Hide** the technical key columns from report view (right-click → Hide): `EmployeeID`-style IDs, `ManagerID`, `PositionID`, `MaritalStatusID`, `PerformanceScoreID`, `PerfScoreID`, `DeptID`. (Keep `EmployeeID` visible only if a visual needs it — it's used by `Total Employees` as a measure, not on a visual, so it can be hidden.)
- [ ] Hide the auto-generated date tables / `Manager` helper if visible.
- [ ] (Optional) Mark dimension tables and arrange them neatly around Employee in the diagram.

---

## 📍 PHASE 8 — Build Report Page 1: "Executive Summary"

**Before placing any visual — set up your canvas and theme:**
- [ ] Select the page → **Format pane → Canvas settings → Type = Custom → 1920 × 1080.**
- [ ] **View → Themes → Customize current theme** (or import a theme JSON): apply *your own* palette and fonts. This is what gives every visual your colours by default.
- [ ] Decide where your icons come from (e.g. Flaticon, your own set) and keep them consistent.

Rename Page 1 → **Executive Summary**. Rebuild these visuals (titles match the original — but apply *your* colours/icons, not the sample's):

**KPI cards (7 total)** — one card visual each:
- [ ] Total Employees
- [ ] Avg. Tenure
- [ ] Employee Turnover Rate
- [ ] Total Salary Expenses
- [ ] Avg. Absenteeism Rate
- [ ] Avg. Satisfaction Score
- [ ] Avg. Age

**Charts & tables:**
- [ ] **Matrix** — *"Workforce Metrics Overview by Dimension"*: Rows = `Department`; Values = Total Employees, Total Salary Expenses, Avg. Age, Avg. Tenure, Avg. Satisfaction Score, Avg. Engagement Score, Avg. Absenteeism Rate, Employee Turnover Rate.
- [ ] **Clustered column chart** — *"Turnover Rate by Department"*: Axis = `Department`; Values = Employee Turnover Rate (+ Total Employees).
- [ ] **Donut chart** — *"Gender Distribution"*: Legend = `Gender`; Values = Total Employees.
- [ ] **Bar chart** — *"Age Distribution"*: Axis = `Age Category`; Values = Total Employees.
- [ ] **Funnel** — *"Top Recruitment Sources"*: Category = `RecruitmentSource`; Values = Total Employees (Avg. Engagement / Avg. Tenure as tooltips).
- [ ] **Bar chart** — *"Departure Reason"*: Axis = `TerminationReason`; Values = Total Employees.

**Slicers & interactivity:**
- [ ] Slicer on `Department`
- [ ] Slicer on `Performance[PerformanceScore]`
- [ ] Slicer on `Employee[ManagerName]`
- [ ] The **Demographics field-parameter slicer** (from Phase 6) — connect it to the demographic chart so it toggles the breakdown.

**Page furniture:**
- [ ] Title / header textboxes, logo image, and background shapes to match the layout.

---

## 📍 PHASE 9 — Build Report Page 2: "Workforce Database"

Rename Page 2 → **Workforce Database**.

- [ ] Set this page's canvas to **1920 × 1080** as well, and confirm it uses the same theme/palette as Page 1.

- [ ] **Matrix / Table** — *"Global Workforce Database Overview"*: Rows = `EmployeeName`; Columns/Values = Position, MaritalStatus, Performance, RaceDesc, RecruitmentSource, EmploymentStatus, Gender, Retention Risk, plus measures Avg. Tenure, Avg. Absenteeism Rate, Avg. Satisfaction Score, Total Salary Expenses.
- [ ] Slicer on `Department`
- [ ] Slicer on `Performance[PerformanceScore]`
- [ ] Slicer on `Employee[ManagerName]`
- [ ] Slicer on `Employee[EmployeeName]`

---

## 📍 PHASE 10 — Polish, navigation & QA

- [ ] Add a **Page Navigator** (Insert → Buttons → Navigator) so users move between the two pages.
- [ ] Apply a consistent **theme** (View → Themes) and align visuals on a grid.
- [ ] **Sync slicers** across pages if you want shared filtering (View → Sync slicers).
- [ ] Set **interactions** between visuals where needed (Format → Edit interactions).
- [ ] **Final check:** Total Employees ≈ 300 rows of staff; turnover rate and percentages display correctly; no broken/blank visuals; relationships filter as expected.
- [ ] **Design check (this is what's graded):** colours are harmonious and clearly *your own* (not the sample's); icons are consistent; both pages are 1920×1080; spacing/alignment is clean; chart types match the sample even though styling differs.
- [ ] **Save.** 🎉

---

## ✅ Quick reference — what the finished model contains

| Item | Count | Names |
|---|---|---|
| Loaded tables | 5 | Employee, Position, Department, MaritalStatus, Performance |
| Helper (no load) | 1 | Manager |
| Measure table | 1 | Measure (8 measures) |
| Field parameter | 1 | Dimension (Demographics) |
| Relationships | 4 | Employee→Position, Employee→MaritalStatus, Employee→Performance, Position→Department |
| Calculated columns | 9 | Age, Age Category, Tenure (Year), Absenteeism Rate, Cost per Absence, Retention Risk, Position, MaritalStatus, Performance |
| Report pages | 2 | Executive Summary, Workforce Database |

**Suggested order if you feel overwhelmed:** do one phase per sitting. Phases 1–3 (data) first, take a break, then 4–7 (model), then 8–10 (visuals). Each phase is a clean stopping point.
