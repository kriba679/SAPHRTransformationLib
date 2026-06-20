# Employee Migration Scenarios – S4 → EC → S4

## KMZP Phase 2 | Stage 1 Migration Reference

---

```table-of-contents
title: 
style: nestedList
minLevel: 0
maxLevel: 0
includeLinks: true
hideWhenEmpty: false
debugInConsole: false
```

---

## Project Dates (KMZP)

| Parameter | Date | Notes |
|-----------|------|-------|
| Project start | 2026-01-01 | S4 configuration complete; migration prep begins |
| Migration cutoff date | **2027-01-01** | Snapshot date Infoporter uses to read S4 |
| FTSD (Full Transmission Start Date) | **2027-01-01** | BIB replicates EC data dated on or after this date |
| Go-live | **2027-01-01** | EC becomes system of record |
| Terminated employee scope | 2026-01-01 – 2026-12-31 | Terminated within 12 months before cutoff |

---

## Three Scenarios

| # | Employee | Status as of 2027-01-01 | In Scope? |
|---|----------|------------------------|-----------|
| 1 | **John Smith** – PERNR 00001001 | Active | Yes |
| 2 | **Sarah Johnson** – PERNR 00001002 | Leave of Absence (maternity) | Yes |
| 3 | **Mark Chen** – PERNR 00001003 | Terminated 2026-06-01 | Yes – within 12-month window |

---

---

## Scenario 1 – Active Employee: John Smith

### Background

| Field | Value |
|-------|-------|
| PERNR | 00001001 |
| Name | John Smith |
| Hire date | 2020-05-15 |
| Personnel Area | KM10 (Head Office Toronto) |
| Employee Group | 1 (Active) |
| Employee Subgroup | YP (Part-Time Salaried) |
| Payroll Area | YB (Bi-Weekly Salaried) |
| Position | 600003 (Manager – Finance) |
| Status as of 2027-01-01 | **Active** |

---

### Phase 1 — Records in S4 (Before Migration)

#### IT0000 – Actions

| Row | MASSN | Action Description   | BEGDA      | ENDDA      | STAT2      |
| --- | ----- | -------------------- | ---------- | ---------- | ---------- |
| 1   | 01    | Hiring               | 2020-05-15 | 2022-12-31 | 3 (Active) |
| 2   | 11    | Transfer / Promotion | 2023-01-01 | 9999-12-31 | 3 (Active) |

> John was promoted to Manager in January 2023. IT0000 has two rows — original hire, then promotion.

#### IT0001 – Org Assignment (current, valid at 2027-01-01)

| Field | Value |
|-------|-------|
| BEGDA | 2023-01-01 |
| ENDDA | 9999-12-31 |
| BUKRS | KMZP |
| WERKS | KM10 |
| BTRTL | 0001 |
| PERSG | 1 |
| PERSK | YP |
| ABKRS | YB |
| PLANS | 600003 |

#### IT0008 – Basic Pay (current)

| Field | Value |
|-------|-------|
| BEGDA | 2023-01-01 |
| ENDDA | 9999-12-31 |
| LGART | /001 (Basic Salary) |
| BETRG | 95,000 CAD annual |
| TRFAR | SA (Salaried) |
| TRFGB | SO (Salaried Ontario) |

---

### Phase 2 — During Migration (Infoporter, Cutoff 2027-01-01)

Infoporter reads John's infotypes as of the cutoff date and creates two slices in EC.

#### EC Job Information Slices

| Slice | Job Slice # | Event | Event Reason | Begin Date | End Date | EC Employment Status | Notes |
|-------|-------------|-------|-------------|-----------|---------|---------------------|-------|
| **Anchor** | 1 | H (Hire) | MIGRATION | 2020-05-15 | 2026-12-31 | Active | From original hire date to cutoff − 1. Preserves true hire date in EC. |
| **Cutoff slice** | 2 | Data Change | DATALOAD | 2027-01-01 | 9999-12-31 | Active | Represents John's state at go-live. All current IT0001/IT0007/IT0008 values loaded here. |

> **Why two slices and not one per IT0000 row?** Infoporter does not replicate every S4 action history into EC. It collapses all historical changes into a single anchor (preserving the original hire date), then creates one cutoff slice reflecting the current state. EC's job history is kept clean — no 7 years of intermediate events.

#### EC Compensation at Cutoff Slice

| EC Field | Value | Source |
|----------|-------|--------|
| `payComponent` | /001 (Basic Salary) | IT0008 LGART |
| `paycompvalue` | 95,000 | IT0008 BETRG |
| `currencyCode` | CAD | IT0008 WAERS |
| `payScaleType` | SA | IT0008 TRFAR |
| `payScaleArea` | SO | IT0008 TRFGB |

#### Key Mapping Entry (PAOCFEC_EEKEYMAP) — Pre-loaded Before Go-Live

| EC Employment ID | EC User ID | S4 PERNR | Company Code |
|-----------------|-----------|---------|-------------|
| EMP00001001 | jsmith@kmzp.com | 00001001 | KMZP |

---

### Phase 3 — After Migration: BIB Replication

#### Initial BIB Run (Go-live day, 2027-01-01)

| Step | What Happens |
|------|-------------|
| BIB reads John's EC job_information from FTSD (2027-01-01) onwards | Cutoff slice (Slice 2) is in scope |
| BIB checks `PAOCFEC_EEKEYMAP` | Finds PERNR 00001001 → treats John as **existing employee** |
| BIB compares EC Slice 2 values against S4 IT0001 | If values match (migration was clean), no update needed |
| IT0000 | **No new action created.** John's IT0000 history is preserved from S4. |
| IT0001, IT0007, IT0008 | Updated only where EC data differs from S4 data |

#### Post-Go-Live Delta — Example: Promotion on 2027-03-15

John is promoted in EC on 2027-03-15. BIB picks this up on the next delta run.

**S4 IT0000 after BIB replication:**

| Row | MASSN | Action Description | BEGDA | ENDDA | STAT2 |
|-----|-------|--------------------|-------|-------|-------|
| 1 | 01 | Hiring | 2020-05-15 | 2022-12-31 | 3 (Active) |
| 2 | 11 | Transfer / Promotion | 2023-01-01 | 2027-03-14 | 3 (Active) |
| **3** | **11** | **Transfer / Promotion** | **2027-03-15** | **9999-12-31** | **3 (Active)** |

**S4 IT0001 after BIB replication (new row from 2027-03-15):**

| Field | Old Value (to 2027-03-14) | New Value (from 2027-03-15) |
|-------|--------------------------|----------------------------|
| PLANS | 600003 (Manager – Finance) | 600005 (Senior Manager – Finance) |
| PERSK | YP | YF (Full-Time Salaried, if applicable) |

---

---

## Scenario 2 – Leave of Absence Employee: Sarah Johnson

### Background

| Field | Value |
|-------|-------|
| PERNR | 00001002 |
| Name | Sarah Johnson |
| Hire date | 2019-08-01 |
| Personnel Area | KM10 (Head Office Toronto) |
| Employee Group | 1 (Active) |
| Employee Subgroup | YF (Full-Time Salaried) |
| Payroll Area | YB (Bi-Weekly Salaried) |
| Position | 600007 (Specialist – HR) |
| LOA start | 2026-10-01 |
| Expected return | 2027-04-01 |
| LOA type | Maternity Leave |
| Status as of 2027-01-01 | **Leave of Absence (Payroll Status 4 – Inactive)** |

---

### Phase 1 — Records in S4 (Before Migration)

#### IT0000 – Actions

| Row | MASSN | Action Description | BEGDA | ENDDA | STAT2 |
|-----|-------|--------------------|-------|-------|-------|
| 1 | 01 | Hiring | 2019-08-01 | 2026-09-30 | 3 (Active) |
| 2 | 14 | Leave of Absence | 2026-10-01 | 9999-12-31 | **4 (Inactive – No Payroll)** |

> STAT2 = 4 means payroll is NOT triggered for Sarah's payroll area YB during the LOA period.

#### IT0001 – Org Assignment (current)

| Field | Value |
|-------|-------|
| BEGDA | 2024-06-01 (last org change — she was reallocated to a new position) |
| ENDDA | 9999-12-31 |
| BUKRS | KMZP |
| WERKS | KM10 |
| PLANS | 600007 |
| PERSG | 1 |
| PERSK | YF |
| ABKRS | YB |

> IT0001 remains active even during LOA. The position is still held by Sarah.

#### IT0005 / Absence Record – Leave of Absence

| Field | Value |
|-------|-------|
| Absence type | Maternity Leave |
| BEGDA | 2026-10-01 |
| ENDDA | 2027-03-31 |
| Expected return | 2027-04-01 |

---

### Phase 2 — During Migration (Infoporter, Cutoff 2027-01-01)

Sarah is on LOA as of the cutoff date (2027-01-01). Her employment is still active in an employment sense — only payroll is inactive. Infoporter creates two slices in EC.

#### EC Job Information Slices

| Slice | Job Slice # | Event | Event Reason | Begin Date | End Date | EC Employment Status | Payroll Status Equivalent | Notes |
|-------|-------------|-------|-------------|-----------|---------|---------------------|--------------------------|-------|
| **Anchor** | 1 | H (Hire) | MIGRATION | 2019-08-01 | 2026-12-31 | Active | Active | Hire date to cutoff − 1. No detail of intermediate changes. |
| **Cutoff slice** | 2 | Leave of Absence | DATALOAD | 2027-01-01 | 9999-12-31 | **Active (on Leave)** | Inactive – No Payroll | Reflects Sarah's state at cutoff: still employed, position held, but on leave. STAT2 = 4 from IT0000 is represented here. |

> **Why is EC Employment Status "Active" for an LOA employee?** In EC, Leave of Absence does not change the employment status to "Terminated" or "Inactive." The employment is still active — the employee just has a leave record in the Time Off portlet. EC uses a separate Leave of Absence entity to track the leave period. This mirrors how S4 handles it: IT0000 MASSN 14 does not end the employment, it only changes STAT2 to 4.

#### EC Leave of Absence Record (Time Off portlet — separate from Job Info)

| EC Field | Value | Source |
|----------|-------|--------|
| Leave Type | Maternity Leave | IT0005 absence type |
| Start Date | 2026-10-01 | IT0005 BEGDA |
| End Date | 2027-03-31 | IT0005 ENDDA |
| Expected Return | 2027-04-01 | IT0005 ENDDA + 1 |

#### Key Mapping Entry (PAOCFEC_EEKEYMAP)

| EC Employment ID | EC User ID | S4 PERNR | Company Code |
|-----------------|-----------|---------|-------------|
| EMP00001002 | sjohnson@kmzp.com | 00001002 | KMZP |

---

### Phase 3 — After Migration: BIB Replication

#### Initial BIB Run (Go-live day, 2027-01-01)

| Step | What Happens |
|------|-------------|
| BIB reads Sarah's EC data from FTSD (2027-01-01) | Cutoff Slice 2 (LOA state) is in scope |
| Key mapping check | Finds PERNR 00001002 → existing employee |
| IT0000 | **No new action.** IT0000 retains original S4 history (MASSN 14 from 2026-10-01 remains). |
| IT0001 | Refreshed from EC if any field differs |
| LOA record | BIB confirms absence record in S4. If IT0005/IT2001 differs from EC leave record, it is updated. |
| Payroll | Payroll area YB remains, but STAT2 = 4 → payroll not triggered until return from leave. |

#### Post-Go-Live Delta — Example: Sarah Returns from Leave on 2027-04-01

HR records Sarah's return in EC on 2027-04-01. BIB picks it up.

**S4 IT0000 after BIB replication:**

| Row | MASSN | Action Description | BEGDA | ENDDA | STAT2 |
|-----|-------|--------------------|-------|-------|-------|
| 1 | 01 | Hiring | 2019-08-01 | 2026-09-30 | 3 (Active) |
| 2 | 14 | Leave of Absence | 2026-10-01 | 2027-03-31 | 4 (Inactive) |
| **3** | **15** | **Return from Leave** | **2027-04-01** | **9999-12-31** | **3 (Active)** |

> MASSN 15 (Return from Leave) is created by BIB in IT0000 on 2027-04-01. Payroll area YB is reactivated — Sarah starts appearing in the next YB payroll run.

---

---

## Scenario 3 – Terminated Employee: Mark Chen

### Background

| Field | Value |
|-------|-------|
| PERNR | 00001003 |
| Name | Mark Chen |
| Hire date | 2015-11-20 |
| Personnel Area | KM15 (Mississauga Plant) |
| Employee Group | 1 (Active, until termination) |
| Employee Subgroup | YU (Full-Time Hourly) |
| Payroll Area | YW (Weekly Hourly) |
| Position | 600010 (Analyst – Operations) |
| Termination date | **2026-06-01** |
| Termination type | Involuntary – Layoff (MASSN 33) |
| Status as of 2027-01-01 | **Terminated** |
| In scope for migration? | **Yes** — 2026-06-01 is within the 12-month window (2026-01-01 to 2026-12-31) |

---

### Phase 1 — Records in S4 (Before Migration)

#### IT0000 – Actions

| Row | MASSN | Action Description | BEGDA | ENDDA | STAT2 |
|-----|-------|--------------------|-------|-------|-------|
| 1 | 01 | Hiring | 2015-11-20 | 2022-06-30 | 3 (Active) |
| 2 | 11 | Transfer | 2022-07-01 | 2026-05-31 | 3 (Active) |
| 3 | 33 | Involuntary Termination – Layoff | 2026-06-01 | 9999-12-31 | **0 (Withdrawn)** |

#### IT0001 – Org Assignment (last active record)

| Field | Value |
|-------|-------|
| BEGDA | 2022-07-01 (last org change) |
| ENDDA | 9999-12-31 |
| BUKRS | KMZP |
| WERKS | KM15 |
| PLANS | 600010 |
| PERSG | 1 |
| PERSK | YU |
| ABKRS | YW |

> In S4, IT0001 is typically not end-dated at the termination date — it runs to 9999. The termination is tracked purely in IT0000 via STAT2 = 0 (Withdrawn).

#### IT0008 – Basic Pay (last record)

| Field | Value |
|-------|-------|
| BEGDA | 2022-07-01 |
| ENDDA | 9999-12-31 |
| LGART | /001 |
| BETRG | 62,000 CAD annual |
| TRFAR | UN (Union) |
| TRFGB | UO (Union Ontario) |

---

### Phase 2 — During Migration (Infoporter, Cutoff 2027-01-01)

Mark was terminated 2026-06-01 — **before the cutoff date (2027-01-01)**. His termination IT0000 record (BEGDA 2026-06-01, ENDDA 9999) spans the cutoff. However, because Infoporter recognises him as a terminated employee, it does NOT create a DATALOAD cutoff slice. Instead it creates an anchor up to the day before his termination, then a termination event.

#### EC Job Information Slices

| Slice | Job Slice # | Event | Event Reason | Begin Date | End Date | EC Employment Status | Notes |
|-------|-------------|-------|-------------|-----------|---------|---------------------|-------|
| **Anchor** | 1 | H (Hire) | MIGRATION | 2015-11-20 | 2026-05-31 (termination − 1) | Active | Anchor ends the day before termination — NOT at cutoff − 1, because Mark was not active at cutoff. |
| **Termination** | 2 | T (Termination) | TERORG (Involuntary – Layoff) | 2026-06-01 | 9999-12-31 | **Inactive / Terminated** | Mapped from MASSN 33 via EVENT_REASON value mapping. |

> **Key difference from active employee:** An active employee's anchor ends at cutoff − 1 (2026-12-31), followed by a DATALOAD cutoff slice. Mark's anchor ends at termination − 1 (2026-05-31), followed by the termination event. There is no DATALOAD cutoff slice because Mark had no active employment at the cutoff date.

#### EC Compensation at Anchor Slice (final active compensation)

| EC Field | Value | Source |
|----------|-------|--------|
| `payComponent` | /001 | IT0008 LGART |
| `paycompvalue` | 62,000 | IT0008 BETRG |
| `currencyCode` | CAD | IT0008 WAERS |
| `payScaleType` | UN | IT0008 TRFAR |
| `payScaleArea` | UO | IT0008 TRFGB |

#### Key Mapping Entry (PAOCFEC_EEKEYMAP) — Pre-loaded Before Go-Live

| EC Employment ID | EC User ID | S4 PERNR | Company Code |
|-----------------|-----------|---------|-------------|
| EMP00001003 | mchen@kmzp.com | 00001003 | KMZP |

> This entry is critical even though Mark is terminated. Without it, if Mark is rehired post-go-live, the system would create a new PERNR instead of rehiring him.

---

### Phase 3 — After Migration: BIB Replication

#### Initial BIB Run (Go-live day, 2027-01-01)

| Step | What Happens |
|------|-------------|
| BIB reads Mark's EC data from FTSD (2027-01-01) | Mark's termination date (2026-06-01) < FTSD (2027-01-01). **No EC records fall on or after FTSD.** |
| BIB action | **Nothing replicated.** BIB ignores all EC data for Mark since all his slices predate the FTSD. |
| IT0000 in S4 | **Unchanged.** MASSN 33 from 2026-06-01 remains exactly as it was in S4 history. |

> Mark's terminated status in S4 comes entirely from the original S4 history, not from BIB replication. BIB's FTSD boundary (2027-01-01) means it will never touch pre-go-live data.

#### Post-Go-Live Scenarios

**Scenario A — T4 Amendment (Feb 2027): HR discovers Mark's final pay was miscalculated**

| Step | What Happens |
|------|-------------|
| HR corrects Mark's compensation in EC, effective 2027-02-15 (post-FTSD) | New EC compensation record created, dated after FTSD |
| BIB delta run picks up the change | Finds Mark's PERNR in key mapping → updates IT0008 in S4 with effective date 2027-02-15 |
| Payroll retro | S4 payroll retro runs back to the period in error; corrected payroll results and T4 amendment generated |

**S4 IT0008 after T4 amendment (new row added by BIB):**

| BEGDA | ENDDA | LGART | BETRG | Notes |
|-------|-------|-------|-------|-------|
| 2022-07-01 | 2027-02-14 | /001 | 62,000 | Original |
| **2027-02-15** | **9999-12-31** | **/001** | **62,500** | **Corrected by BIB retro** |

**Scenario B — Rehire (Mark is rehired on 2027-08-01)**

| Step | What Happens |
|------|-------------|
| HR creates new hire event for Mark in EC, BEGDA 2027-08-01 | EC event: H (Hire), reason: REHEMP (Rehire) |
| BIB delta run triggered | Reads new hire event from EC |
| BIB checks `PAOCFEC_EEKEYMAP` | **Finds Mark's PERNR 00001003.** Since an entry exists, BAdI `EX_PAOCF_EC_DECIDE_HIRE_REHIRE` returns: REHIRE. |
| IT0000 updated in S4 | New row: MASSN 40 (Rehire), BEGDA 2027-08-01. Mark gets the same PERNR 00001003 — service history preserved. |

**S4 IT0000 after rehire:**

| Row | MASSN | Action Description | BEGDA | ENDDA | STAT2 |
|-----|-------|--------------------|-------|-------|-------|
| 1 | 01 | Hiring | 2015-11-20 | 2022-06-30 | 3 |
| 2 | 11 | Transfer | 2022-07-01 | 2026-05-31 | 3 |
| 3 | 33 | Involuntary Termination – Layoff | 2026-06-01 | 2027-07-31 | 0 |
| **4** | **40** | **Rehire** | **2027-08-01** | **9999-12-31** | **3** |

---

---

## Summary Comparison: All Three Scenarios

### EC Job Information Slices at Migration

| Employee | Slice | Event | Event Reason | Begin Date | End Date | EC Status |
|----------|-------|-------|-------------|-----------|---------|-----------|
| **John (Active)** | 1 – Anchor | H | MIGRATION | 2020-05-15 | 2026-12-31 | Active |
| | 2 – Cutoff | Data Change | DATALOAD | 2027-01-01 | 9999-12-31 | Active |
| **Sarah (LOA)** | 1 – Anchor | H | MIGRATION | 2019-08-01 | 2026-12-31 | Active |
| | 2 – Cutoff (LOA state) | Leave of Absence | DATALOAD | 2027-01-01 | 9999-12-31 | Active (on Leave) |
| **Mark (Terminated)** | 1 – Anchor | H | MIGRATION | 2015-11-20 | 2026-05-31 | Active |
| | 2 – Termination | T | TERORG | 2026-06-01 | 9999-12-31 | Inactive / Terminated |

> **Pattern rule:**
> - Active and LOA employees → Anchor ends at **cutoff − 1 (2026-12-31)**, followed by a DATALOAD cutoff slice at **2027-01-01**
> - Terminated employees → Anchor ends at **termination date − 1**, followed by the termination event. No DATALOAD cutoff slice.

### IT0000 Behaviour in S4

| Employee | IT0000 During Migration | IT0000 After Initial BIB Run | IT0000 After First Post-Go-Live Event |
|----------|------------------------|------------------------------|--------------------------------------|
| John (Active) | Unchanged | Unchanged (key mapping match → no new action) | New row added when EC sends promotion/transfer |
| Sarah (LOA) | Unchanged | Unchanged (LOA MASSN 14 preserved) | New row MASSN 15 when EC records return from leave |
| Mark (Terminated) | Unchanged | Unchanged (termination before FTSD; BIB ignores) | New row MASSN 40 only if Mark is rehired from EC |

### BIB Replication Behaviour

| Employee | Initial BIB Run | Ongoing Delta | Trigger for new IT0000 action |
|----------|----------------|--------------|-------------------------------|
| John (Active) | Updates IT0001/IT0008 where data differs | Yes — every EC change replicates to S4 | Promotion, transfer, termination in EC |
| Sarah (LOA) | Updates infotypes; confirms LOA record | Yes | Return from leave recorded in EC → MASSN 15 |
| Mark (Terminated) | Nothing (all data pre-FTSD) | Nothing, until a post-FTSD correction or rehire | T4 amendment or rehire in EC → BIB fires |

---

*Document Reference: KMZP Phase 2 – Employee Migration Scenarios*
*Covers: Active, LOA, and Terminated (within 12 months) employee patterns*
*Updated: June 2026*
*Cutoff / FTSD / Go-live: 2027-01-01*
