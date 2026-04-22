# Termination Payroll — UI Flow Spec

**Feature:** Termination Payroll Wizard  
**Product:** Netchex — Payroll Domain  
**Priority:** P0  
**Date:** April 2026  
**PM:** James

---

## Step 1 — Terminate Employee

### 1.1 Entry Point: Change Status Modal

The flow begins when a user sets an employee's status to **Terminated**.

> **Screenshot:** Change Status modal — "Terminate Employee"

![Change Status Modal](screenshots/step1_change_status.png)

---

### 1.2 Business Rule — Mutual Exclusivity

```
Paid today? = Yes   ──►  Included in payroll? = No (forced & disabled)
                          └─► Wizard launched on submit

Paid today? = No    ──►  Included in payroll? = Yes/No (free choice)
                          └─► Normal termination, no wizard
```

> "Should this employee be included in payroll?" must be **disabled** — not just defaulted — when "Paid today?" is Yes.

---

### 1.3 On Submit

1. Employee status set to **Terminated**
2. `person_hire.TermPayroll_Status_Cd` → **`InProcess`**
3. Redirect to the Termination Payroll wizard:

```
https://{host}/a/payroll/multi-company/start/termination
  ?companyCode={companyCd}
  &employeeCode={employeeCd}
```

| `TermPayroll_Status_Cd` | Meaning |
|---|---|
| `InProcess` | Wizard launched — check not yet generated |
| `Paid` | Termination check generated successfully |

---

## Step 2 — Termination Payroll Wizard

Wizard progress bar:

```
[1] Earnings  →  [2] Accruals  →  [3] Deductions  →  [4] Review & Submit  →  [5] Confirmation
```

### 2.1 Pre-flight Data Load

When the wizard URL resolves, **before any screen renders**, two calls fire in sequence:

**① Preflight — earnings, accruals, deductions**

```
GET v1/company/{companyCd}/termination/employees/{employeeId}/preflight
```

Returns one entry per paycheck. Two-check scenarios (prior period gap + current partial period) produce two entries.

```json
{
  "checks": [
    {
      "periodStartDate": "2026-03-20",
      "periodEndDate":   "2026-04-05",
      "paycheckDate":    "2026-04-15",
      "earnings":   [ ... ],
      "accruals":   [ ... ],
      "deductions": [ ... ]
    },
    {
      "periodStartDate": "2026-04-06",
      "periodEndDate":   "2026-04-15",
      "paycheckDate":    "2026-04-15",
      "earnings":   [ ... ],
      "accruals":   [ ... ],
      "deductions": [ ... ]
    }
  ]
}
```

> Single-check = 1 entry. Two-check = 2 entries. The wizard renders each screen per period using this data.

**② Active Payroll Run Check — drives the Earnings screen banner**

```
GET v1/company/{companyCd}/termination/employees/{employeeId}/payroll-run/active
```

| Status | Meaning | UI behaviour |
|---|---|---|
| `200 OK` | Employee is in an open payroll run | Show banner |
| `404 Not Found` | No active run — nothing to report | Hide banner |

**200 response:**

```json
{
  "employeeName": "Maria Johnson",
  "runNumber":    2,
  "periodStart": "2026-04-06",
  "periodEnd":   "2026-04-19"
}
```

Once both responses are received, the Earnings screen renders.

---

### ── Wizard Screen 1 of 5 — Earnings ──

> **Screenshot:** Earnings screen

![Earnings Step](screenshots/step2_earnings_wizard.png)

Pre-populated from the preflight response. Earnings are editable — the user can add or remove lines before continuing.

#### Open Payroll Run Banner

Shown when the active payroll run check returns `200`:

> *"[Name] was found in open Payroll Run #[N] ([periodStart] – [periodEnd]). They have been removed — full earnings through today are included here."*

→ [How earnings data is retrieved](#appendix-a--earnings-data-source)

---

### ── Wizard Screen 2 of 5 — Accruals ──

> **Screenshot:** Accruals screen

![Accruals Step](screenshots/step3_accruals.png)

Each active accrual plan is shown as a card:

| Field | Description |
|---|---|
| Current balance | Balance before this period |
| This period accrual | Hours earned during the earnings period |
| Balance at term | Current balance + this period accrual |
| Payout hours | Editable — defaults to full balance at term |
| Payout amount | Computed: payout hours × pay rate |

> Setting payout hours to **0** excludes the plan from the final check.

→ [How accruals data is retrieved](#appendix-c--accruals-data-source)

---

### ── Wizard Screen 3 of 5 — Deductions ──

> **Screenshot:** Deductions screen

![Deductions Step](screenshots/step4_deductions.png)

| Column | Description |
|---|---|
| Include | Toggle — on by default unless flagged |
| Code / Description | Deduction identifier |
| Type | Pre-tax or Post-tax |
| Amount | Prorated to the partial period |
| Note | `Prorated` / `Stop on termination` (red, auto off) / `Required by order` |

> Deductions flagged **Stop on termination** are toggled off automatically and the note is shown in red.

→ [How deductions data is retrieved](#appendix-b--deductions-data-source)

---

## Step 3 — Review & Submit

### 3.1 Engine Calculate Call

When the user advances from Deductions, a calculate call fires **before the screen renders**:

```
POST v1/company/{companyCd}/termination/employees/{employeeId}/calculate
```

**Request** — one entry per check. Single-check = 1 item, two-check = 2 items, matching the preflight shape:

```json
{
  "checks": [
    {
      "periodStartDate": "2026-03-20",
      "periodEndDate":   "2026-04-05",
      "paycheckDate":    "2026-04-15",
      "earnings":   [ ... ],
      "accruals":   [ ... ],
      "deductions": [ ... ]
    },
    {
      "periodStartDate": "2026-04-06",
      "periodEndDate":   "2026-04-15",
      "paycheckDate":    "2026-04-15",
      "earnings":   [ ... ],
      "accruals":   [ ... ],
      "deductions": [ ... ]
    }
  ]
}
```

> **Accrual → engine merge (server-side):** Each accrual payout is converted into a `PayCalculationInputEarnings` entry (`EarningCalculationBasisId = 2`, `HoursToPay = payoutHours`, `HourlyPayRateAmount = rate`) and appended to that check's earnings list before the engine call. The engine has no accrual concept — this conversion is transparent to the UI.

**Response** — one calculated result per check, mirroring the request array:

```json
{
  "checks": [
    {
      "periodStartDate": "2026-03-20",
      "periodEndDate":   "2026-04-05",
      "paycheckDate":    "2026-04-15",
      "earningsSummary": [ ... ],
      "grossEarnings":   1200.00,
      "deductions": { "preTax": [ ... ], "taxes": [ ... ], "postTax": [ ... ] },
      "totalDeductions": 400.00,
      "netPay":          800.00,
      "taxLiability":    250.00
    },
    {
      "periodStartDate": "2026-04-06",
      "periodEndDate":   "2026-04-15",
      "paycheckDate":    "2026-04-15",
      "earningsSummary": [ ... ],
      "grossEarnings": 2695.10,
      "deductions": { "preTax": [ ... ], "taxes": [ ... ], "postTax": [ ... ] },
      "totalDeductions": 927.32,
      "netPay":          1767.78,
      "taxLiability":     677.82
    }
  ]
}
```

> `taxLiability` on each check drives the notice: *"Tax liability of $[X] will be collected in the next scheduled payroll run."*

Once the response is received, the Review & Submit screen renders fully populated from the calculate response.

### ── Wizard Screen 4 of 5 — Review & Submit ──

> **Screenshot:** Review & Submit screen

![Review & Submit](screenshots/step5_review_submit.png)

**Nothing is committed at this stage.** The screen is a read-only preview.

### 3.2 Generate On Demand Check

Clicking **"Generate On Demand Check"** advances to Step 4 and triggers the commit call.

---

## Step 4 — Confirmation

### ── Wizard Screen 5 of 5 — Confirmation ──

### 4.1 Commit Call

```
POST v1/company/{companyCd}/termination/payroll
```

### 4.2 Server-Side Process Flow

```
"Generate On Demand Check" clicked
        │
        ▼
① Remove employee from active payroll run (if enrolled)
        │
        ▼
② Call Payroll Engine (same inputs as Step 3 — authoritative DB write)
        │
        ▼
③ Insert → Person_CheckHistory       (check header, per check)
        │
        ▼
④ Insert → Person_PayCheckStubs      (E/D/T line items, per check — mirrors PBI)
        │
        ▼
⑤ Insert → Person_PaymentHistory     (labor distribution per earning, per check)
        │
        ▼
⑥ person_hire.TermPayroll_Status_Cd → Paid
        │
        ▼
   Return check summary → Confirmation screen renders
```

**Response** — one entry per check generated:

```json
{
  "checks": [
    {
      "employeeName": "Maria Johnson",
      "checkAmount":   800.00,
      "payrollRun":   "#2 – On Demand",
      "checkDate":    "2026-04-15"
    },
    {
      "employeeName": "Maria Johnson",
      "checkAmount":   1767.78,
      "payrollRun":   "#2 – On Demand",
      "checkDate":    "2026-04-15"
    }
  ]
}
```

### 4.3 Confirmation Screen

> **Screenshot:** Confirmation screen

![Confirmation](screenshots/step6_confirmation.png)

Success state — one card per check generated. Single-check = 1 card, two-check = 2 cards.

**Actions per card:**
- **Print Check** — opens that check for printing
- **View in On Demands** — navigates to the On Demands payroll queue

---

## Appendix A — Earnings Data Source

Inputs required per check: `periodStartDate`, `periodEndDate` (= last day of work for the current period).

```
Read Company_PayCodes.CalculationMethod_Cd
        │
        ├── NOT 'UnitRate*' (Salaried)
        │        │
        │        ▼
        │   BusinessDaysCalculator.CalculateRegularHoursWorked()
        │     · PayPeriod.StartDate = periodStartDate
        │     · PayPeriod.EndDate   = periodEndDate (lastDayOfWork)
        │     · Prorates NormalHoursWorkedPerPayPeriod by actual vs full-period business days
        │     · Also runs for holiday earnings if applicable
        │     · Pull additional scheduled earnings (bonuses, commissions)
        │       from Person_PayData / Company_EarningCodes at standard amounts
        │
        │   → PayCalculationInputEarnings { EarningCalculationBasis = HOURLY_RATE, hours, rate }
        │
        └── StartsWith('UnitRate') AND AutoTimesheet_Ind = 0 (Hourly punch)
                 │
                 ▼
            T&A MTP endpoint
            GET /v1/employees/{employeeId}/termination-hours?startDate=&endDate=
            Returns aggregated punch hours by pay code (T&A team — James action item)
```

**Salaried path output:**
```json
{ "payCode": "REG", "description": "Regular", "hours": 72.00, "rate": 22.00 }
```

**Hourly punch path output (from T&A):**
```json
{ "payCode": "REG", "hoursWorked": 72.00 },
{ "payCode": "OT",  "hoursWorked": 6.00  }
```
Rate resolved from `Person_PayData` by Payroll — T&A supplies hours only.

---

## Appendix B — Deductions Data Source

**Reusable:** `PayrollDeductionQuery` · `IPayrollDeductionQuery`

```sql
SELECT CDC.Company_DeductionCodeId, CDC.Stub_Description, CDC.CalculationMethod_Cd,
       CDC.SubjectToFederalTax_Ind, CDC.SubjectToStateTax_Ind,
       ISNULL(PDD.MonthToDate_Amt, 0) AS MtdAmount, ISNULL(PDD.Maximum_Amt, CDC.Maximum_Amt) AS MaxAmount
FROM   Person_Main PM
JOIN   Company_DeductionCodes CDC ON CDC.Company_Cd = PM.Company_Cd
LEFT JOIN Person_DeductionData PDD ON PDD.Employee_Cd = PM.Employee_Cd
                                  AND PDD.Deduction_Cd = CDC.Deduction_Cd
WHERE  PM.Employee_Id = @employeeId
  AND  CDC.Company_DeductionCodeId = @deductionId
  AND (PDD.Stop_Dt  IS NULL OR PDD.Stop_Dt  > @checkDate)
  AND (PDD.Start_Dt IS NULL OR PDD.Start_Dt <= @checkDate)
```

Called once per deduction ID — the preflight iterates over all active deduction IDs for the employee and calls this query per entry. `StopOnTermination` flag drives auto-toggle-off on the Deductions screen.

---

## Appendix C — Accruals Data Source

**Reusable:** `EmployeeAccrualPlanRepository.ListEmployeeAccrualPlansAsync(employeeId)`

```sql
SELECT CAP.Plan_Desc, PA.CarryOverBalance_Nbr, FUNC.YTDAccrued_Nbr,
       FUNC.BeginningBalance_Nbr + PA.CarryOverBalance_Nbr + FUNC.YTDAccrued_Nbr - FUNC.YTDTaken_Nbr AS Available
FROM   Person_Main PM
JOIN   Person_Accruals PA ON PA.Employee_Cd = PM.Employee_Cd
JOIN   Company_Accrual_Plans CAP ON CAP.Company_Cd = PM.Company_Cd AND CAP.Plan_ID = PA.Plan_ID
CROSS APPLY udf_AccrueBalancesTableDisplay(PM.Employee_Cd, CAP.Pay_Cd) FUNC
WHERE  PM.Employee_Id = @employeeId
  AND  FUNC.Plan_Id = PA.Plan_Id
```

This-period accrual = `annual rate ÷ pay periods per year`, computed server-side.
