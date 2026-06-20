# Stage 2 – HR Data Replication
## KMZP HR Transformation: SuccessFactors Employee Central → SAP S/4HANA

---
```table-of-contents
title: 
style: nestedList # TOC style (nestedList|nestedOrderedList|inlineFirstLevel)
minLevel: 0 # Include headings from the specified level
maxLevel: 0 # Include headings up to the specified level
include: 
exclude: 
includeLinks: true # Make headings clickable
hideWhenEmpty: false # Hide TOC if no headings are found
debugInConsole: false # Print debug info in Obsidian console
```
## Table of Contents

### Part 1 – Architecture
1. Architecture Overview
2. Integration Flow: How It All Connects

### Part 2 – Design Decisions
3. Position Management and PA/PD Integration
4. Employee Class, Employment Type, Personnel Area, and Personnel Subarea
5. Employee Number and Identifier Strategy
6. FTE – Impact on IT0007-EMPCT and IT0008-BSGRD
7. Wage Type Scope – What to Migrate and Replicate
8. Event and Event Reason Mapping
9. Address Types – Three Address Templates (Home, Office, Mailing)
10. Phone Numbers – Two Types (Personal Cell, Business)
11. Email Addresses – Business and Personal
12. Rehire with New Employment – Special Handling

### Part 3 – Organizational Data Replication
13. Organizational Data Replication (SF → S4)

### Part 4 – Employee Data Replication
14. Employee Data Replication – Compound Employee API
15. Correct CE API Structure: Person Level vs Employment Level
16. Business Integration Builder (BIB) – Transformation Templates
17. Exact Field Mapping Per Template (from SAP Replication Workbook)
18. BIB Configuration – Value Mapping, Secondary Mapping, Conversion Rules
19. BAdI Reference

### Part 5 – Reference
20. Key Tables Reference
21. Key Programs and Transactions Reference

---

---

## PART 1 – Architecture


## 1. Architecture Overview

KMZP is moving to a Core Hybrid model:

- **SAP SuccessFactors Employee Central (EC)** – System of record for HR master data: personal info, job info, org assignments, compensation, time.
- **SAP S/4HANA on-premise (HCM)** – System of record for **Payroll** (Canadian payroll). PA infotypes maintained here; payroll runs on-premise.
- **SAP CPI (Cloud Platform Integration)** – Acts as middleware. Forwards queries from S4 to EC and returns responses. S4 drives all scheduling and parameters. CPI configuration is out of scope for this document.

The **Mini Master** concept means EC holds only the HR data needed for payroll. For KMZP Canada, the infotypes in scope are:

> IT0000, IT0001, IT0002, IT0006, IT0007, IT0008, IT0009, IT0014, IT0015, IT0016, IT0041, IT0105, IT0267

---

## 2. Integration Flow: How It All Connects

S4 is the orchestrator — it initiates all queries to EC. CPI is a stateless pass-through.

**Step 1:** S4 runs a background job that constructs a query (delta or full) and sends it outbound via a SOAP web service consumer proxy to CPI.

**Step 2:** CPI receives the query, forwards it to EC. For employee data: calls the **Compound Employee API** (SOAP). For org data: calls EC OData APIs.

**Step 3:** EC returns the data payload (XML). CPI forwards it to S4 inbound SOAP web services. Data is written to staging tables.

**Step 4:** An SAP event fires automatically, triggering a background job to process staging data. BIB field mapping, value mapping, conversion rules, and BAdIs are applied. Infotypes and OM objects are created/updated.

**Step 5:** S4 sends a confirmation back to EC to mark replication complete.

### Key Web Services

| Service | Proxy Name | Direction | Purpose |
|---------|-----------|-----------|---------|
| Employee Data Outbound | `CO_ECPAOX_EE_MD_ORGAS_BNDL_QRY` | S4 → CPI | Send employee replication query |
| Employee Data Inbound | `II_ECPAOX_EE_MD_ORGAS_BNDL_REQ` | CPI → S4 | Receive employee data response |
| Org Object Outbound | `CO_SFIOMX_ORG_OBJECT_REPL_QRY` | S4 → CPI | Send org object query |
| Org Object Inbound (data) | `II_SFIOMX_ORG_OBJ_REPL_RSP` | CPI → S4 | Receive org object data |
| Org Object Inbound (status) | `II_SFIOMX_ORG_OBJ_REPL_NOTISAP` | CPI → S4 | Receive org replication status/errors |
| Confirmation Outbound | `CO_PAOCF_EC_EMPLOYEE_MASTER_DA` | S4 → CPI | Send replication confirmation to EC |

---

---

## PART 2 – Design Decisions


## 3. Position Management and PA/PD Integration – Detailed Behavior

### 3.1 Position Management Switch: SFSFI PMACT = X

With Position Management turned on in EC:

**What EC owns:**
- Position creation, title, FTE, reporting line, org unit assignment, cost center assignment
- All position-to-position and position-to-org-unit relationships

**What this means for S4:**
- S4 must NEVER auto-create positions. The switch `SFSFI PMACT = X` in T77S0 prevents this.
- Every position in S4 must first be replicated from EC via org object replication (`RH_SFIOM_ORG_OBJ_REPL_QUERY`)
- The position must exist in the key mapping table `SFIOM_KMAP_OSI` before an employee can be assigned to it
- If an employee is assigned to a position in EC that has not yet been replicated to S4, the employee replication will fail with a key mapping error

**Replication order matters:**
Org objects must be replicated BEFORE employee data. The sequence is:
1. Business Units → Divisions → Departments (all as Org Units type O)
2. Jobs (type C)
3. Positions (type S) — must have S-O, S-C relationships from the org object template
4. Employee data (job_information links employee to position)
5. Org assignment replication (creates 1001 relationships P-S, S-O, S-C, S-K, S-S)

**Vacant positions:**
When a position has no incumbent in EC (employee leaves or position is unfilled), EC sets the employee's position to a default "dummy" position (e.g., 9999999). Upon replication to S4, the position-to-org-unit/job/cost-center relationships (S-O, S-C, S-K, S-S) in infotype 1001 are **delimited** automatically. This keeps S4 OM consistent with EC Position Management.

### 3.2 PA/PD Integration: PLOGI ORGA = X

With PA/PD integration on, the 3-step flow described in Section 3.7 applies to every employee event. The critical behavioral points:

**IT0001 updates come in two waves:**
- Wave 1 (Employee Master Data Replication): Updates BUKRS, WERKS, BTRTL, PERSG, PERSK, KOSTL, ABKRS. Position set to default (PLOGI PRELI). Org Unit and Job left blank.
- Wave 2 (Org Assignment Replication): Fills PLANS (position), ORGEH (org unit), STELL (job) in IT0001 by syncing from 1001 relationships.

**Timing is critical:** The org assignment job (`RH_SFIOM_PROC_EE_ORG_ASS_RPRQ`) is triggered by event `SAP_SFIOM_EE_ORGAS_RPPQ_CREATED`. This must be scheduled to fire after each employee replication batch completes. If it runs before the position has been replicated (org replication), IT0001 will be incomplete.

**Operational sequence to schedule:**
1. Org object replication (scheduled, e.g., hourly)
2. Employee data replication (scheduled, e.g., every 15-30 min or triggered by EC push)
3. Org assignment processing (event-triggered: SAP_SFIOM_EE_ORGAS_RPPQ_CREATED)

### 3.3 Multiple Actions on the Same Day

EC can send multiple events for the same employee on the same date. Examples:
- A promotion (job change) and a salary change effective the same day
- A hire and an immediate address data change
- A transfer and a schedule change

**How S4 handles this:**

IT0000 (Actions) has **Time Constraint 1** — only ONE active record per day. When multiple events arrive on the same date:

**Switch `ADMIN EVSUP` in T77S0** must be set to control which action is primary:
- When this switch is active: only the highest-priority action per day goes to IT0000
- All other actions that day are stored in **IT0302 (Additional Actions)**

**Priority List configuration:**
IMG: Personnel Administration → Customizing Procedures → Actions → Set Up Personnel Action Types → Define Priority for Personnel Actions (table T529A, field PRTY)

Lower priority number = higher priority (action that wins in IT0000).

**Recommended KMZP priority (suggested order, adjust per business rules):**

| Priority | Action Type | Reason |
|---------|-------------|--------|
| 1 | 01 – Hiring | Always wins over data changes |
| 2 | 40 – Rehire | |
| 3 | 33 – Termination (Layoff) | |
| 4 | 34 – Termination (Voluntary) | |
| 5 | 14 – Leave of Absence | |
| 6 | 15 – Return from Leave | |
| 7 | 11 – Transfer | |
| 8 | 12 – Promotion/Demotion | |
| 9 | 02 – Data Change | Lowest priority – always goes to IT0302 if combined with above |

**Result for example scenario:**
Employee gets a promotion (action 12) AND salary change (action 02) on the same day:
- IT0000: Shows Promotion (12) — higher priority
- IT0302: Shows Data Change (02) — stored for audit trail

**Important for Payroll:** Payroll reads IT0302 as well as IT0000 for retro-calculation triggers. So the additional actions in IT0302 are not lost — they still participate in payroll processing.

---

## 4. Employee Class, Employment Type, Personnel Area, and Personnel Subarea

### 4.1 EC employeeClass → S4 Employee Group (PERSG)

Yes — EC `employeeClass` maps to S4 **Employee Group (PERSG)**. Configured in BIB WS_4 with value mapping entity `EMPLOYEE_CLASS_WS`.

| EC employeeClass Value | S4 PERSG | S4 Employee Group |
|----------------------|---------|-----------------|
| Employee | 1 | Active employee |
| Contractor | 3 | External/Contractor |
| Owner_Operator | 3 | External (treat as contractor) |

### 4.2 EC employmentType → S4 Employee Subgroup (PERSK)

Yes — EC `employmentType` maps to S4 **Employee Subgroup (PERSK)**. Configured in BIB WS_4 with value mapping entity `EMPLOYEE_TYPE_WS`.

This is the most important mapping because ESG drives payroll area, pay scale structure, work schedule grouping, and PCR/CAP groupings.

| EC employmentType | S4 PERSK | S4 ESG Description | Payroll Area |
|------------------|---------|-------------------|-------------|
| Full_Time_Salaried | YF | Full-time Salaried (non-union) | YB (bi-weekly) |
| Part_Time_Salaried | YP | Part-time Salaried | YB (bi-weekly) |
| Full_Time_Hourly | YU | Full-time Hourly (union) | YW (weekly) |
| Part_Time_Hourly | YU | Part-time Hourly (union) | YW (weekly) |
| Contractor | YO | Contractor (non-payroll) | 99 |

**Note:** The Payroll Area (ABKRS) does NOT come from the employmentType mapping. It comes from the `payGroup` field in EC Compensation (WS_11 → ABKRS via PAY GROUP WS value mapping). The employmentType only sets the ESG; the payGroup sets the payroll area independently.

### 4.3 Personnel Area (WERKS) – Location from EC

EC `location` field on Job Information → S4 `WERKS` (Personnel Area).

Standard mapping: `location` → WERKS via key mapping or value mapping.

For KMZP (from Phase 1 design):

| EC Location externalCode | S4 WERKS | Description |
|-------------------------|---------|-------------|
| KM10 (or Toronto location code) | KM10 | Toronto, Ontario |
| KM15 (or Ottawa location code) | KM15 | Ottawa, Ontario |
| KM20 (or Montreal location code) | KM20 | Montreal, Quebec |

### 4.4 Personnel Subarea (BTRTL) – Custom EC Field

There is **no standard EC field** for Personnel Subarea. In the replication workbook (WS_4), BTRTL is mapped from a **custom EC field** labeled "Sub Location" using:
- EC field: `custom` (custom field on Job Information)
- S4 field: BTRTL
- Value mapping entity: PERSONNEL_SUBAREA_WS
- Plus a Conversion Rule

**KMZP design decision required:** Create a custom field on EC Job Information (e.g., `custom01` or named `personnelSubarea`) to hold the Personnel Subarea value. This field is maintained by HR in EC and replicated to BTRTL in S4.

**What should Personnel Subarea represent for KMZP?**
Based on Phase 1 design (PA = location/city), PSA could represent:
- Department type within the location (Sales, Operations, Finance)
- Union/Non-union distinction within a location
- A functional grouping needed for time or payroll calculations

| EC custom field value | S4 BTRTL | Description |
|----------------------|---------|-------------|
| 0001 | 0001 | Non-Union office staff |
| 0002 | 0002 | Union production staff |
| 0003 | 0003 | Management |

**Configure the custom field in EC:**
1. In EC Admin Center → Configure Business Rules / Manage Business Configuration: add a custom field to the Job Information portlet
2. Configure picklist values for the custom field
3. Map in BIB WS_4: EC field = `custom`, EC Field Description = "Sub Location" or "Personnel Subarea", S4 field = BTRTL, Value Mapping Entity = PERSONNEL_SUBAREA_WS

---

## 5. Employee Number and Identifier Strategy

### EC Identifiers

| Identifier | Scope | Stability | Description |
|-----------|-------|-----------|-------------|
| PersonID_External | Person | Permanent | Identifies the human being |
| UserID | Employment | Changes on intl. transfer | Login ID; new per employment |
| EmploymentID | Employment | Per employment | Numeric ID for the employment |
| Assignment ID | Employment | Per employment | Work relationship ID; recommended as PERNR |

### KMZP Strategy

- EC Assignment ID → S4 PERNR (constant `ECPAO ASSIGNIDUSED = X`)
- PERNR number range = External (allows Assignment ID to be used)
- Business rules in EC generate Assignment IDs matching S4 PERNR number format
- EC PersonID_External stored in IT0709 for cross-employment person tracking

### S4 Person Linking

- **Central Person (CP):** Auto-created on hire; links multiple PERNRs of the same person in HRP1001
- **IT0709:** Stores EC PersonID_External; populated via BAdI `EX_PAOCF_EC_PROCESS_EMPLOYEE`

---

## 6. FTE – Impact on IT0007-EMPCT and IT0008-BSGRD

### 6.1 What FTE Controls in S4

The `fte` field in EC Job Information is a decimal value from 0 to 1 (e.g., 1.0 = full-time, 0.5 = half-time, 0.6 = 60% part-time).

It impacts TWO critical fields in two different infotypes:

| S4 Field | Infotype | Description | Impact |
|----------|---------|-------------|--------|
| EMPCT | IT0007 | Employment Percentage | Controls time quota generation, absence quota eligibility |
| BSGRD | IT0008 | Capacity Utilization Level | Controls reduction of indirectly valuated wage types in payroll |

### 6.2 IT0007 – EMPCT (Employment Percentage)

**Mapping:** `fte` × 100 → EMPCT

- FTE 1.0 → EMPCT = 100 (full-time)
- FTE 0.5 → EMPCT = 50 (part-time 50%)
- FTE 0.6 → EMPCT = 60 (part-time 60%)

**What it drives:**
- Time quota entitlements (vacation, sick leave) are often calculated as: full entitlement × EMPCT/100
- Work schedule assignment (SCHKZ) combined with EMPCT determines actual hours
- The standard BIB mapping in WS_4 applies a conversion rule (multiply FTE × 100) to populate EMPCT

**Also in IT0007:** `WOSTD` (Standard Weekly Hours) is mapped from EC `standardHours`. These two fields together define the employee's work pattern.

### 6.3 IT0008 – BSGRD (Capacity Utilization Level)

**What BSGRD controls:** When an employee is part-time (FTE < 1), payroll uses BSGRD to calculate the correct partial salary. Specifically, for **indirectly valuated wage types** (where the amount is derived from pay scale tables), BSGRD tells the system to pay only `BSGRD%` of the full-time equivalent amount.

**Example:** Salaried employee at 50% (FTE = 0.5, BSGRD = 50). Annual salary in the pay scale table for their grade is CAD 80,000/year. Payroll calculates: 80,000 × (50/100) = CAD 40,000/year.

**BSGRD is NOT automatically set by IT0007.** You must explicitly set BSGRD = FTE × 100 during replication.

**Standard BIB WS_4 does NOT map FTE → BSGRD** (it only maps FTE → IT0007-EMPCT). BSGRD on IT0008 requires one of these approaches:

**Option A (Recommended): Add BSGRD mapping in BIB WS_4:**
Add a row in the WS_4 primary mapping:
- EC Field: `fte`
- S4 Infotype: P0008
- S4 Field: BSGRD
- Conversion Rule: Multiply by 100 (same as EMPCT)

**Option B: BAdI EX_PAOCF_EC_CHANGE_INFOTYPE_DA:**
After the standard BIB maps IT0008 fields, the BAdI reads the already-mapped EMPCT value from IT0007 structure and sets BSGRD = EMPCT on the IT0008 structure before it's written to the database.

**Option C: Payroll Schema:**
Some SAP payroll schemas have a processing class that reads EMPCT from IT0007 and derives BSGRD automatically. Verify with your payroll consultant whether this is configured for Canada payroll schema in your system.

### 6.4 FTE Impact Summary Table

| FTE Value | IT0007-EMPCT | IT0008-BSGRD | Effect |
|-----------|-------------|-------------|--------|
| 1.0 (full-time) | 100 | 100 | Full pay, full quotas |
| 0.5 (half-time) | 50 | 50 | 50% pay (indirect WT), 50% quotas |
| 0.6 (60%) | 60 | 60 | 60% pay, 60% quotas |
| 0.75 (75%) | 75 | 75 | 75% pay, 75% quotas |

**For directly valuated wage types** (where the amount is stored directly on IT0008 — e.g., a fixed salary amount from EC): BSGRD does NOT reduce the amount. The amount in EC already IS the part-time salary. So for salaried employees where EC sends the actual part-time amount, BSGRD is less critical. For pay scale employees (union), where the amount is derived from T510 tables and then reduced by BSGRD, this is critical.

---

## 7. Wage Type Scope – What to Migrate and Replicate

### 7.1 Governing Principle

The rule: **Wage types that form an employee's "Total Rewards" — components visible to managers and employees in EC — should live in EC and replicate to S4. Payroll-calculated, statutory, and technical wage types stay in S4 only.**

### 7.2 Wage Types TO Migrate to EC (and Replicate to S4)

These go into EC Pay Components (Recurring or Non-Recurring) and are replicated via WS_12_ERP_BASIC_PAY (→ IT0008) or WS_12 (→ IT0014) or WS_13 (→ IT0015):

| Category | Examples | EC Portlet | S4 Infotype |
|----------|---------|-----------|------------|
| Base Salary | 20SA (Monthly salary), 20BI (Bi-weekly salary) | Pay Component Recurring | IT0008 (via WS_12_ERP_BASIC_PAY) |
| Hourly Rate | 20HO (Hourly rate) | Pay Component Recurring | IT0008 (via WS_12_ERP_BASIC_PAY) |
| Regular Allowances (part of total package) | Car allowance, Phone allowance, Housing allowance | Pay Component Recurring | IT0014 (via WS_12) |
| Regular Premiums visible to employee | Shift differential (if part of agreement), Bilingual bonus | Pay Component Recurring | IT0014 (via WS_12) |
| One-time Bonuses | Annual bonus, Sign-on bonus, Spot bonus | Pay Component Non-Recurring | IT0015 (via WS_13) or IT0267 |

### 7.3 Wage Types to Keep ONLY in S4 (Do NOT migrate to EC)

| Category | Examples | Reason |
|----------|---------|--------|
| Statutory deductions – employee | Canada Pension Plan (CPP), Employment Insurance (EI), Federal Income Tax, Provincial Tax | Calculated by payroll schema; not "compensation" |
| Statutory deductions – employer | Employer CPP, Employer EI | Back-office payroll only |
| Benefits deductions | Extended Health, Dental, Life Insurance premiums | Managed in S4 Benefits or by HR admin |
| Garnishments | Child support, spousal support | Legal obligation; not an EC pay component |
| Union dues | CUPE dues, OPSEU dues | Calculated by union agreement rules in payroll |
| Payroll-calculated adjustments | Retro adjustments (/559, /560), Retro differences | Schema-generated; not maintained by HR |
| Technical/control wage types | /001, /559, /560, /ZZZ | Internal payroll processing |
| Holiday pay | If auto-calculated in schema | No manual maintenance needed |
| Overtime pay | If calculated from time evaluation | Derived from time data, not HR master data |
| Absence-related pay | STD/LTD insurance payouts | Managed by insurance carrier or absence processing |
| T4 year-end wage types | Box 14, Box 52, etc. | Payroll-generated at year-end |

### 7.4 Value Mapping for Wage Types

In BIB, the PAY_COMPONENT_WS value mapping entity maps EC Pay Component IDs to S4 wage types:

| EC Pay Component ID | EC Description | S4 Wage Type | S4 Description |
|--------------------|---------------|-------------|----------------|
| BASE_SALARY | Base Salary | 20SA | Monthly Salary |
| HOURLY_RATE | Hourly Rate | 20HO | Hourly Rate |
| CAR_ALLOW | Car Allowance | 20CA | Car Allowance |
| PHONE_ALLOW | Phone Allowance | 20PH | Phone Allowance |
| ANNUAL_BONUS | Annual Bonus | 20AB | Annual Bonus |
| SPOT_BONUS | Spot Bonus | 20SB | Spot Bonus |

Separate value mapping entities are used for IT0008 (PAY_COMPONENT_WS), IT0014 (PAY_COMPONENT_WS_0014), and IT0015 (PAY_COMPONENT_WS_0015).

---

## 8. Event and Event Reason Mapping

EC event reasons from Job Information drive S4 IT0000 Actions. The EVENT_REASON value mapping entity translates EC codes to S4 Action Types (MASSN).

### Common Canadian HR Events

| EC Event Code | Description | S4 MASSN |
|--------------|-------------|----------|
| HIREMP | New Hire | 01 |
| HIRACQ | Hire via Acquisition | 01 or custom |
| REHEMP | Rehire | 40 or custom |
| TERRTM | Voluntary Resignation | 34 or custom |
| TERORG | Organizational Departure (Layoff) | 33 or custom |
| TERDTH | Death | custom |
| TERBAL | Termination – Balance | custom |
| PROMOT | Promotion | 11 |
| DEMOTN | Demotion | custom |
| XFER | Transfer | 11 |
| LOA | Leave of Absence | 14 or custom |
| RTNLOA | Return from Leave | 15 or custom |

### Multiple Events on Same Date

Priority list in IMG (Define Priority for Personnel Action Types) determines which action goes to IT0000. Others go to IT0302 (Additional Actions). Switch `ADMIN EVSUP` in T77S0 must be set.

---

## 9. Address Types – Three Address Templates (Home, Office, Mailing)

### 9.1 IT0006 Address Subtypes in S4 HCM

Standard SAP-delivered subtypes for IT0006 (Addresses):

| Subtype | Description | Time Constraint | Notes |
|---------|-------------|----------------|-------|
| 1 | Permanent Residence (Home) | 1 (one record at a time) | Primary home address |
| 2 | Alternative Permanent Residence | 1 | Temporary home during assignment |
| 3 | Mailing Address | 1 | PO Box or different mailing address |
| 4 | Emergency Contact Address | 3 (multiple allowed) | Emergency contact location |
| 5 | Temporary Accommodation | 1 | Short-term stay (travel, etc.) |

**For KMZP's 3 address types, use:**
- **Home** → IT0006 Subtype 1 (Permanent Residence)
- **Mailing** → IT0006 Subtype 3 (Mailing Address)
- **Office/Work** → Use a **custom subtype** (e.g., Subtype 8 or Z1) since standard S4 does not have a dedicated "office address" subtype — work location in S4 is typically derived from Personnel Area/Subarea, not stored as a personal address

Configure subtypes in: IMG → Personnel Management → Personnel Administration → Personal Data → Addresses → Define Address Types

### 9.2 EC Address Types and BIB Template Configuration

In EC, the `homeAddress` entity has an `addressType` field that distinguishes different address types (e.g., home, business, mailing). The value mapping entity `ADDRESS_TYPE_WS` maps EC address type codes to S4 IT0006 subtypes.

**Three separate BIB templates for KMZP:**

| BIB Template | Description | EC addressType Value | IT0006 Subtype |
|-------------|-------------|---------------------|----------------|
| ERP_WS_10 | Home Address | home / H | 1 |
| ERP_WS_10_MAIL | Mailing Address | mailing / M | 3 |
| ERP_WS_10_WORK | Office/Work Address | work / W | 8 (or customer-defined) |

**Why separate templates (not cloning):**
Address templates for different subtypes often need **different secondary mappings** by country. Because you cannot use cloning when secondary mapping is present on a template, SAP recommends separate templates per subtype. (This matches the pattern in the replication workbook — see WS_10_DEP_1, WS_10_DEP_2, WS_10_DEP_11 for dependents.)

### 9.3 Canada-Specific Field Mapping (All Three Templates)

Each address template for Canada uses secondary mapping (Secondary Mapping Code = 'CA'):

| EC Field | IT0006 Field | Notes |
|----------|-------------|-------|
| `address1` | STRAS (Street) | Primary street line |
| `address2` | LOCAT (Address Line 2) | Apt/unit/suite — Canada secondary mapping |
| `city` | ORT01 (City) | |
| `state` | STATE (Province) | PROVINCE_CAN value mapping (ON→ON, QC→QC, etc.) |
| `zipCode` | PSTLZ (Postal Code) | Canadian format: A1A 1A1 |
| `country` | LAND1 (Country) | NATIONALITY_WS value mapping (CA→CA) |

### 9.4 Address Type Value Mapping

Configure in BIB: Define Value Mapping Entities → ADDRESS_TYPE_WS

| EC addressType | S4 IT0006 Subtype | Description |
|---------------|------------------|-------------|
| home | 1 | Permanent Residence |
| mailing | 3 | Mailing Address |
| work | 8 (custom) | Office/Work Address |

---

## 10. Phone Numbers – Two Types (Personal Cell, Business)

### 10.1 Where Phone Numbers Live in S4

Phone numbers can be stored in two places in S4:
- **IT0006** (Addresses infotype) – has `TELNR` (Home Telephone), `TELNX` (Extension), `TELMB` (Mobile/Cell) fields
- **IT0105** (Communications) – can store phone numbers using custom subtypes

Standard BIB mapping (WS_8) maps EC `phoneNumber` → IT0006 subtype 1 `TELNR` (the home telephone field on the address infotype). Only the primary phone type is handled by the standard template. Other phone types require BAdI.

### 10.2 KMZP Phone Type Strategy

| Phone Type | EC Source | S4 Target | Method |
|-----------|----------|----------|--------|
| Home phone (primary) | `phoneNumber` where `phoneType = home` and `isPrimary = true` | IT0006 subtype 1, field TELNR | Standard BIB WS_8 template |
| Personal mobile/cell | `phoneNumber` where `phoneType = cell` | IT0006 subtype 1, field TELMB | BAdI EX_PAOCF_EC_PROCESS_EMPLOYEE |
| Business phone | `phoneNumber` where `phoneType = business` | IT0105 custom subtype (e.g., 0020 or 0030) | BAdI EX_PAOCF_EC_PROCESS_EMPLOYEE |

**BAdI logic for multiple phone types:**
In `EX_PAOCF_EC_PROCESS_EMPLOYEE`, loop through the CE API `phone_information` segment, check the `phoneType` value, and write to the appropriate field. Pseudo-logic:

```abap
LOOP AT lt_phone_info ASSIGNING FIELD-SYMBOL(<ls_phone>).
    CASE <ls_phone>-phone_type.
        WHEN 'cell'.
            " Write to IT0006 TELMB (mobile field on address)
        WHEN 'business'.
            " Write to IT0105 custom subtype (e.g., BSPH)
        WHEN 'home'.
            " Handled by standard WS_8 template — skip
    ENDCASE.
ENDLOOP.
```

### 10.3 IT0105 Communication Subtypes for Phone

If storing phone in IT0105, define custom subtypes:

| IT0105 Subtype | Description | EC phoneType |
|---------------|-------------|-------------|
| 0020 (or custom) | Mobile/Cell Phone | cell |
| 0030 (or custom) | Business Phone | business |

Configure subtypes via: IMG → Personnel Administration → Personal Data → Communication → Determine Communication Types

---

## 11. Email Addresses – Business and Personal

### 11.1 Standard IT0105 Email Subtypes

In standard SAP HCM, IT0105 Communications handles email:

| Subtype | Description | Typical Use |
|---------|-------------|-------------|
| 0001 | SAP system user name | SAP logon ID (from WS_2 logonUserName) |
| 0010 | Business Email Address | Work email (standard BIB template WS_7) |
| 0020 | Personal / Alternative Email | Personal email (via BAdI or cloning) |

**Note from WS_2 in replication workbook:** `logonUserName` → IT0105 subtype ECUS (system user – the EC login). This is separate from the email.

### 11.2 KMZP Email Strategy

| Email Type | EC Source | S4 IT0105 Subtype | S4 Field | Method |
|-----------|----------|------------------|----------|--------|
| Business/Work Email | `emailAddress` where `emailType = work` or `isPrimary = true` | 0010 | USRID | Standard BIB template ERP_WS_7 |
| Personal Email | `emailAddress` where `emailType = personal` | 0020 (or custom) | USRID | BAdI or template cloning |

### 11.3 Handling Two Email Types in BIB

**Option A – BAdI (recommended):**
In `EX_PAOCF_EC_PROCESS_EMPLOYEE`, loop through the CE API `email_information` segment, check `emailType`, and write to appropriate IT0105 subtype:

```abap
LOOP AT lt_email_info ASSIGNING FIELD-SYMBOL(<ls_email>).
    CASE <ls_email>-email_type.
        WHEN 'work'.   " → IT0105 subtype 0010 (handled by WS_7 standard)
        WHEN 'personal'. " → write IT0105 subtype 0020 via HR_INFOTYPE_OPERATION
    ENDCASE.
ENDLOOP.
```

**Option B – Template Cloning:**
Create a second template ERP_WS_7_PERSONAL using the same EC entity but mapped to IT0105 subtype 0020. Use a filter condition in the template (via BIB filter) to only process records where emailType = 'personal'.

**Business email (`isPrimary`) logic:**
The `isPrimary` field in EC determines which email is the "main" one. The BAdI handles this by only replicating the primary work email to IT0105 subtype 0010. If multiple work emails exist, only the primary one is replicated.

---

## 12. Rehire with New Employment – Special Handling

### 12.1 How EC Models Rehire with New Employment

| Identifier | Original Employment | New Employment (Rehire) | Stable? |
|-----------|--------------------|-----------------------|---------|
| PersonID_External | P-00001 | P-00001 (SAME) | Yes – the person |
| UserID | P-00001 | P-00001-2 (NEW) | No |
| EmploymentID | 6001 | 6002 (NEW) | No |
| Assignment ID | A-00001 | A-00002 (NEW) → new PERNR | No |

Because Person-level portlets (name, address, email, phone) share the same PersonID_External, they contain consistent data across both employments. Employment-level portlets (job info, compensation, bank details) are separate per employment.

### 12.2 Key Mapping Table: PAOCFEC_EEKEYMAP

Cross-reference between EC Employment ID / User ID and S4 PERNR.

**Lookup logic:**
1. CE API response arrives for an employee
2. System looks up Employment ID (or User ID) in PAOCFEC_EEKEYMAP
3. **Found** → existing employee → update infotypes
4. **Not found** → call BAdI `EX_PAOCF_EC_DECIDE_HIRE_REHIRE` → decide: new PERNR (new hire) or reuse old PERNR

For New Employment rehire: new Employment ID is NOT in the table → BAdI is called.

### 12.3 BAdI: EX_PAOCF_EC_DECIDE_HIRE_REHIRE

Without BAdI: every new Employment ID = new PERNR (new hire). Correct for normal rehires with new employment.

**When to implement:** Rehire of a "converted term" (employee terminated before FTSD, old PERNR has good payroll history).

Pre-delivered example: `CL_EC_TOOLS_REHIRE_DUPLICATE`
- EC event reason `REHIREDUPL` → reuse old PERNR
- EC event reason `IGNOREDUPL` → create new PERNR

### 12.4 PERNR-Specific FTSD: HRSFEC_PN_FTSD

Controls from which date data is replicated for a specific PERNR, overriding the TTG global FTSD.

Set manually or via BAdI `EX_PAOCF_EC_DECIDE_HIRE_REHIRE` (set `ev_full_trans_start_date` = rehire job info start date).

### 12.5 IT0041 for Rehire

For a rehired employee (new PERNR):
- IT0041 subtype 01 (Hire Date) = new employment start date (from `startDate`)
- IT0041 subtype 16 (Original Start Date) = original first hire date (from `originalStartDate`)
- IT0041 subtype 17 (Seniority Date) = from `seniorityDate`

These are replicated correctly via ERP_WS_3 as long as EC `originalStartDate` is populated correctly on the new employment.

---

---

## PART 3 – Organizational Data Replication


## 13. Organizational Data Replication (SF → S4)

### 13.1 EC Org Objects and Their S/4HANA OM Equivalents

| EC Object | EC Entity Name | S4 OM Object Type | S4 Object |
|-----------|---------------|-------------------|-----------|
| Department | Department | O | Organizational Unit |
| Division | Division | O | Organizational Unit |
| Business Unit | BusinessUnit | O | Organizational Unit |
| Job Classification | JobCode | C | Job |
| Position | Position | S | Position |
| Cost Center | (not replicated – already in FI) | K | Cost Center |

The hierarchy in S4 OM: Business Unit (O) → Division (O) → Department (O), connected via A002 (Reports To) relationships in infotype 1001.

### 13.2 Technical Process Flow

**Step 1 – Query from S4:**
`RH_SFIOM_ORG_OBJ_REPL_QUERY` (TCode: `SFIOM_QRY_ORG_OBJ`) reads the last run timestamp from `SFIOM_QRY_ADM`, constructs a delta query, sends it outbound via `getOrganisationalObjectReplicationQuery_Out`.

**Step 2 – CPI reads EC OData APIs** for changed org objects since the last run.

**Step 3 – Response arrives via two inbound web services:**
- `OrganisationalObjectReplicationResponse_In` – actual org data
- `OrganisationalObjectReplicationNotification_In` – status/error info

**Step 4 – Staging:** Data lands in `SFIOM_GENRQ_HD`, `SFIOM_GENRQ_DP`, `SFIOM_GENRQ_DFLD`. Event `SAP_SFIOM_PROC_ORG_STRUC_RPRQ` fires.

**Step 5 – Processing:** `RH_SFIOM_PROC_ORG_STRUC_RPRQ` reads staging, applies BIB field mapping, creates/updates HRP1000 and HRP1001 records.

### 13.3 Sample Content: OM_WS_1 vs OM_WS_2

**OM_WS_1 (recommended for KMZP):**
- Maps name/description for all 5 org object types
- Maps parent-child A002 hierarchy
- Position relations (S-O, S-C, S-K, S-S) replicated via Employee Org Assignment replication from Job Information

**OM_WS_2 (use only if vacant position relations needed from Position Management):**
- All of OM_WS_1 plus conditional mapping for vacant positions
- Requires switch `SFSFI VCNCY = X`

**For KMZP:** Use **OM_WS_1** with `SFSFI PMACT = X`.

### 13.4 Field Mapping for Org Objects

**Mandatory for all org templates:** `HRP1000-STEXT` (Object Name).

**Department Template – Conditional Mapping:**

| EC Field | S4 Field | Condition |
|----------|----------|-----------|
| `name_defaultValue` | HRP1000-STEXT | Always |
| `cust_parentDepartment/externalCode` | HRP1001-SOBID (A002) | If parentDepartment is NOT null |
| `cust_toDivision/externalCode` | HRP1001-SOBID (A002) | If parentDepartment IS null |

A Department can report to multiple Divisions in EC, but S4 only allows one A002 (time constraint 1). The conditional mapping ensures the right parent is picked.

### 13.5 Key Mapping Table: SFIOM_KMAP_OSI

Cross-references EC external codes ↔ S4 OM object IDs.

If the EC external code is found in the table → update existing S4 object. If not found → create new S4 object and add entry.

Must be pre-loaded during initial migration so existing S4 org objects are updated (not duplicated).

**Cost Center Mapping Tables (separate):**
Cost centers exist in S4 FI and are NOT replicated from EC. Mapping via:
- `PAOCFEC_KMAPCOSC` – Primary key mapping table
- `ODFIN_MAP_KOSTL` – Finance integration mapping

### 13.6 T77S0 Switches for Org Replication

| Switch | Value | Meaning |
|--------|-------|---------|
| SFSFI PMACT | X | EC manages positions; S4 must NOT auto-create |
| SFSFI SPOMP | blank | Use Generic Processing (recommended) |
| SFSFI VCNCY | X | Replicate vacant position relations from Position Mgmt (OM_WS_2 only) |
| PLOGI ORGA | X | PA/PD Integration is active |
| PLOGI NITF | X | New Infotype Framework (required for BIB) |
| PLOGI PRELI | (position ID) | Default position when PA/PD active but org assignment not yet processed |

### 13.7 PA/PD Integration – 3-Step Process

**Step 1 – Employee Master Data Replication:**
IT0001 gets: Cost Center, Payroll Area, Company Code, Personnel Area, Personnel Subarea, Employee Group, Employee Subgroup.
Position set to default (PLOGI PRELI). Org Unit and Job left empty at this stage.

**Step 2 – Org Assignment Replication (triggered by event SAP_SFIOM_EE_ORGAS_RPPQ_CREATED):**
`RH_SFIOM_PROC_EE_ORG_ASS_RPRQ` creates OM infotype 1001 relationships:
- P ↔ S (Person to Position: A008/B008)
- S ↔ O (Position to Org Unit: A003/B003)
- S ↔ C (Position to Job: A007/B007)
- S ↔ K (Position to Cost Center: A011)
- S ↔ S (Position to Manager's Position: A002/B002)

**Step 3 – PA/PD Sync:**
Since PLOGI ORGA = X, IT0001 fields (Position, Org Unit, Job) are automatically synchronized from the 1001 relationships.

---

---

## PART 4 – Employee Data Replication


## 14. Employee Data Replication – Compound Employee API

### 14.1 What is the Compound Employee API

The Compound Employee (CE) API is a SOAP-based EC API that returns all data for one or more employees in a single structured XML response. One call returns data from multiple portlets (Personal Info, Job Info, Compensation, Bank Details, etc.) grouped under one Person.

### 14.2 Query Modes

| Mode | When to Use | Key Parameters |
|------|-------------|----------------|
| Full | Initial load | (default, no extra params) |
| Snapshot | Point-in-time extract | `queryMode=snapshot` + `SNAPSHOT_DATE` |
| Delta – Effective-dated | Ongoing production replication | `queryMode=delta` + `EFFECTIVE_END_DATE >=` + `last_modified_on >` |
| Delta – Period-based | Changes effective within a period | `queryMode=periodDelta` + `fromDate` + `toDate` + `last_modified_on >` |

**For KMZP ongoing replication (effective-dated delta):**
```sql
WHERE last_modified_on > [last run timestamp]
AND EFFECTIVE_END_DATE >= [today's date]
```
Returns: all employees with any data changed since last run, with current + future effective-dated records.

### 14.3 CE API Filter Parameters

| Parameter | Description | Operators |
|-----------|-------------|-----------|
| `LAST_MODIFIED_ON` | Employees with any data changed since this datetime | >, >= |
| `EFFECTIVE_END_DATE` | Filter which time slices of each employee's data are returned | =, >= |
| `COMPANY` | Filter by EC company external code | =, IN |
| `PERSON_ID_EXTERNAL` | Specific employee(s) | =, IN, NOT IN |
| `isContingentWorker` | Include/exclude contingent workers (default = false) | =, IN |
| `fromDate … toDate` | Period for period-based delta mode | = |

**Important distinction:** `LAST_MODIFIED_ON` selects **which employees** are included. `EFFECTIVE_END_DATE` filters **which time slices** are returned for those employees.

---

## 15. Correct CE API Structure: Person Level vs Employment Level

This is critical and commonly misunderstood. The CE API XML has a strict hierarchy:

```
CompoundEmployee
  └── Person  (one per PersonID_External – represents the human being)
       │
       ├── [PERSON-LEVEL PORTLETS – same for all employments of this person]
       │    ├── biographical_information    → WS_2  → IT0002 (birth data) + IT0709
       │    ├── personal_information        → WS_5  → IT0002 (name, gender, marital status)
       │    ├── email_information           → WS_7  → IT0105 (email address)
       │    ├── phone_information           → WS_8  → IT0006 (phone number)
       │    ├── home_address                → WS_10 → IT0006 (home address)
       │    └── national_id_card            → BAdI  → IT0185 (SIN for Canada)
       │
       └── employment_information  (one block per employment – separate for each PERNR)
            │
            ├── [EMPLOYMENT-LEVEL PORTLETS]
            │    ├── employment_information   → WS_3  → IT0041 (dates: hire, original start, seniority)
            │    ├── job_information          → WS_4  → IT0000, IT0001, IT0007, IT0008 (pay scale), IT0016
            │    ├── compensation_information → WS_11 → IT0001 (payGroup/ABKRS only)
            │    ├── pay_component_recurring  → WS_12 → IT0014 (recurring pay components)
            │    │                         also WS_12_ERP_BASIC_PAY → IT0008 (wage type amounts)
            │    ├── pay_component_non_recurring → WS_13 → IT0015, IT0267
            │    └── payment_information_v3  → WS_14 → IT0009 (bank details)
            │
            └── (cost distribution handled via BAdI → IT0027)
```

### Why This Matters

- **Email, Phone, Personal name/gender, Home Address** are **Person-level** portlets. They do NOT sit inside the employment block. This means:
  - They are replicated once per person, not once per employment
  - For a rehire with new employment, these portlets still carry the same person's data
  - In BIB, these templates are linked to CE API segments at the Person level (not `employment_information`)

- **Job Information, Compensation, Bank Details, Pay Components** are **Employment-level** portlets. They exist once per employment and are different for each employment.

---

## 16. Business Integration Builder (BIB) – Transformation Templates

**IMG Path:** Personnel Management → Integration with SuccessFactors Employee Central → Business Integration Builder

### 16.1 Complete Template List for KMZP Canada Mini Master

| BIB Template ID | SAP Content Name | CE API Level | EC Portlet (CE API Segment) | Target Infotype(s) |
|----------------|-----------------|--------------|----------------------------|-------------------|
| ERP_WS_2 | Biographical Information | **Person** | `biographical_information` | IT0002 (birth data), IT0105 (logon), IT0709 |
| ERP_WS_5 | Personal Information Info | **Person** | `personal_information` | IT0002 (name, gender, marital) |
| ERP_WS_7 | Email Info | **Person** | `email_information` | IT0105 (email) |
| ERP_WS_8 | Phone Info | **Person** | `phone_information` | IT0006 (phone number) |
| ERP_WS_10 | Address Template | **Person** | `home_address` | IT0006 (home address) |
| ERP_WS_2_DEP | Dependents Biographical Info | Person | `biographical_information` (dependent) | IT0021 |
| ERP_WS_5_DEP | Dependents Personal Info | Person | `personal_information` (dependent) | IT0021 |
| ERP_WS_10_DEP | Dependents Address | Person | `home_address` (dependent) | IT0106 |
| ERP_WS_3 | Employment Info | **Employment** | `employment_information` | IT0041 (hire/original start/seniority dates) |
| ERP_WS_4 | Job Info | **Employment** | `job_information` | IT0000, IT0001, IT0007, IT0008 (pay scale), IT0016 |
| ERP_WS_11 | Compensation | **Employment** | `compensation_information` | IT0001 (ABKRS/Payroll Area only) |
| ERP_WS_12 | Pay Component Recurring | **Employment** | `pay_component_recurring` | IT0014 |
| WS_12_ERP_BASIC_PAY | Basic Pay | **Employment** | `pay_component_recurring` | IT0008 (wage type + amounts) |
| ERP_WS_13 | Pay Component Non Recurring | **Employment** | `pay_component_non_recurring` | IT0015 |
| ERP_SPOTBONUS_0267 | Spot Bonus Off-Cycle | **Employment** | `pay_component_non_recurring` | IT0267 |
| ERP_WS_14 | Payment Information Details | **Employment** | `payment_information_v3` | IT0009 |

---

## 17. Exact Field Mapping Per Template (from SAP Replication Workbook)

### 17.1 ERP_WS_2 – Biographical Information (Person Level → IT0002, IT0105, IT0709)

| EC Field | EC Description | S4 Infotype | S4 Subtype | S4 Field | S4 Field Description | Value Mapping |
|----------|---------------|-------------|-----------|----------|---------|---------------|
| `birthName` | Birth Name | 0002 | – | NAME2 | Birth Name / Second Name | – |
| `countryOfBirth` | Country of Birth | 0002 | – | GBLND | Country of Birth | COUNTRY_CODE |
| `dateOfBirth` | Date of Birth | 0002 | – | GBDAT | Date of Birth | – |
| `placeOfBirth` | Place of Birth | 0002 | – | GBORT | Place of Birth | – |
| `regionOfBirth` | Region of Birth | 0002 | – | GBDEP | Region of Birth | Linking field: GBLND |
| `logonUserName` | Logon User Name | 0105 | ECUS | USRID_LONG | Communication ID (Long) | – |
| `PersonId` | PersonId | 0709 | – | PERSONID_EXT | External Person ID (SF PersonID_External) | – |

**Canada-specific note:** No secondary mapping for Canada in WS_2.

### 17.2 ERP_WS_5 – Personal Information Info (Person Level → IT0002)

| EC Field | EC Description | S4 Infotype | S4 Field | S4 Field Description | Value Mapping |
|----------|---------------|-------------|----------|---------|---------------|
| `firstName` | First Name | 0002 | VORNA | First Name | – |
| `lastName` | Last Name | 0002 | NACHN | Last Name | – |
| `middleName` | Middle Name | 0002 | MIDNM | Middle Name | – |
| `gender` | Gender | 0002 | GESCH | Gender | GENDER_CODE |
| `maritalStatus` | Marital Status | 0002 | FAMST | Marital Status | MARITAL_STATUS_WS |
| `since` | Marital Status Since | 0002 | FAMDT | Marital Status Since | – |
| `nationality` | Nationality | 0002 | NATIO | Nationality | NATIONALITY_WS |
| `nativePreferredLang` | Native Preferred Language | 0002 | SPRSL | Communication Language | LANGUAGE_CODE_WS |
| `namePrefix` | Prefix | 0002 | VORSW | Name Prefix | NAME_PREFIX |
| `salutation` | Salutation | 0002 | ANRED | Form of Address / Salutation | SALUTATION |
| `initials` | Initials | 0002 | INITS | Initials | SALUTATION |
| `title` | Title | 0002 | TITEL | Academic Title | TITLE_WS |
| `suffix` | Suffix | 0002 | NAMZU | Name Suffix | SUFFIX WS |
| `nameFormatCode` | Name Format Code | 0002 | KNZNM | Name Format Key | – |
| `partnerNamePrefix` | Second Name Prefix | 0002 | VORS2 | Second Name Prefix | NAME_PREFIX |
| `secondLastName` | Second Last Name | 0002 | NACH2 | Second Last Name | – |
| `secondNationality` | Second Nationality | 0002 | NATI2 | Second Nationality | – |

### 17.3 ERP_WS_7 – Email Info (Person Level → IT0105)

| EC Field | EC Description | S4 Infotype | S4 Subtype | S4 Field | S4 Field Description | Value Mapping / Notes |
|----------|---------------|-------------|-----------|----------|---------|----------------------|
| `emailAddress` | Email Address | 0105 | 0001 or 0010 | USRID | Communication ID | Subtype depends on time constraint settings |
| `emailType` | Email Type | 0105 | 0001 or 0010 | USRTY | Communication Type | EMAIL_TYPE |
| `isPrimary` | Is Primary | – | – | – |  | BAdI Mapping (not standard infotype mapping) |

**Note:** The subtype can be 0001 or 0010 depending on your IT0105 time constraint configuration for KMZP. BAdI is used to handle the isPrimary logic (e.g., only replicate the primary work email).

### 17.4 ERP_WS_8 – Phone Info (Person Level → IT0006)

| EC Field | EC Description | S4 Infotype | S4 Subtype | S4 Field | S4 Field Description | Value Mapping / Notes |
|----------|---------------|-------------|-----------|----------|---------|----------------------|
| `phoneNumber` | Phone Number | 0006 | 1 | TELNR | Telephone Number | Home phone number goes into IT0006 TELNR |
| `phone-type` | Phone Type | 0006 | 1 | ANSSA | Address Type | PHONE_TYPE; only home (TELNR) mapped standard; other types via BAdI |
| `isPrimary` | Is Primary | – | – | – |  | BAdI Mapping |

**Important:** Phone information in EC maps to the phone number field (`TELNR`) on the **Home Address** infotype (IT0006 subtype 1). Only the primary home phone is mapped via standard BIB; other phone types (mobile, work) are handled via BAdI.

### 17.5 ERP_WS_10 – Address Template (Person Level → IT0006 Subtype 1)

This template uses **Secondary Mapping** extensively because address structure differs by country. The primary mapping covers the generic/international fields; secondary mapping for Canada (secondary mapping code 'CA') overrides or supplements for Canadian specifics.

**Primary Mapping (International – applies to all countries unless overridden):**

| EC Field | S4 Infotype | S4 Subtype | S4 Field | S4 Field Description | Notes |
|----------|-------------|-----------|----------|---------|-------|
| `addressType` | 0006 | 1 | ANSSA | Address Type | ADDRESS_TYPE_WS value mapping |
| `address1` | 0006 | 1 | STRAS | Street and House Number | Street and house number |
| `city` | 0006 | 1 | ORT01 | City | City |
| `country` | 0006 | 1 | LAND1 | Country Key | NATIONALITY_WS value mapping |
| `state` | 0006 | 1 | STATE | Region / Province | STATE_WS value mapping; Linking field: LAND1 |
| `zipCode` | 0006 | 1 | PSTLZ | Postal Code | Postal Code |

**Secondary Mapping – Canada (secondary code 'CA'):**

| EC Field | EC Description | S4 Infotype | S4 Subtype | S4 Field | S4 Field Description | Value Mapping |
|----------|---------------|-------------|-----------|----------|---------|---------------|
| `address2` | Address Line 2 | 0006 | 1 | LOCAT | Additional Address Line (c/o, Suite, Apt) | – |
| `state` | Province | 0006 | 1 | STATE | Region / Province | PROVINCE_CAN |

**Note:** For Canada, `state` in EC holds the province code (ON, QC, BC, etc.) and maps to IT0006 STATE field using the PROVINCE_CAN value mapping entity.

### 17.6 ERP_WS_3 – Employment Info (Employment Level → IT0041)

This template maps the key employment dates from the `employment_information` entity to Date Specifications (IT0041).

| EC Field | EC Description | S4 Infotype | S4 Subtype | S4 Field | S4 Field Description |
|----------|---------------|-------------|-----------|----------|---------|
| `startDate` | Hire Date | 0041 | 01 | DAT01 | Date Value |
| `originalStartDate` | Original Start Date | 0041 | 16 | DAT01 | Date Value |
| `seniorityDate` | Seniority Start Date | 0041 | 17 | DAT01 | Date Value |
| `professionalServiceDate` | Professional Service Date | 0041 | 15 | DAT01 | Date Value |

**Key for Rehire:** For a rehired employee with new employment, `startDate` = new employment start date → IT0041 subtype 01. `originalStartDate` = their very first hire date at the company → IT0041 subtype 16. Seniority continuity is preserved via subtype 17.

### 17.7 ERP_WS_4 – Job Info (Employment Level → IT0000, IT0001, IT0007, IT0008, IT0016)

This is the most critical template — Job Information in EC drives multiple infotypes in S4.

**IT0000 – Actions (from Job Info event/eventReason):**

| EC Field | EC Description | S4 Infotype | S4 Field | S4 Field Description | Value Mapping |
|----------|---------------|-------------|----------|---------|---------------|
| `eventReason` | Event Reason | 0000 | MASSN | Action Type | EVENT_REASON (maps both Action Type and optionally Action Reason) |

Note: The `eventReason` field in EC (e.g., HIREMP, REHEMP, TERRTM) maps to S4 Action Type (MASSN) via the EVENT_REASON value mapping entity. A separate action reason (MASSG) mapping can be added if needed.

**IT0001 – Organizational Assignment (from Job Info):**

| EC Field | EC Description | S4 Infotype | S4 Field | S4 Field Description | Value Mapping |
|----------|---------------|-------------|----------|---------|---------------|
| `company` | Company (Legal Entity) | 0001 | BUKRS | Company Code | – (key mapping) |
| `location` | Location | 0001 | WERKS | Personnel Area | – (key mapping to Personnel Area) |
| `custom` | Sub Location | 0001 | BTRTL | Personnel Subarea | PERSONNEL_SUBAREA_WS; Conversion Rule |
| `employeeClass` | Employee Class | 0001 | PERSG | Employee Group | EMPLOYEE_CLASS_WS |
| `employmentType` | Employment Type | 0001 | PERSK | Employee Subgroup | EMPLOYEE_TYPE_WS |

**IT0007 – Planned Working Time (from Job Info):**

| EC Field | EC Description | S4 Infotype | S4 Field | S4 Field Description | Value Mapping |
|----------|---------------|-------------|----------|---------|---------------|
| `standardHours` | Standard Weekly Hours | 0007 | WOSTD | Standard Working Hours per Week | – |
| `workscheduleCode` | Work Schedule | 0007 | SCHKZ | Work Schedule Rule | WORKSCHEDULE_RULE |
| `fte` | FTE | 0007 | EMPCT | Employment Percentage | Conversion Rule (multiply × 100) |

**IT0008 – Basic Pay – Pay Scale Fields Only (from Job Info):**

Note: Pay scale structure fields come from Job Info (WS_4), NOT from the Compensation portlet. The actual wage type amounts come from WS_12_ERP_BASIC_PAY (Pay Component Recurring portlet).

| EC Field | EC Description | S4 Infotype | S4 Subtype | S4 Field | S4 Field Description | Notes |
|----------|---------------|-------------|-----------|----------|---------|-------|
| `payScaleType` | Pay Scale Type | 0008 | 0 | TRFAR | Pay Scale Type | Conversion Rule |
| `payScaleArea` | Pay Scale Area | 0008 | 0 | TRFGB | Pay Scale Area | Conversion Rule |
| `payScaleGroup` | Pay Scale Group | 0008 | 0 | TRFGR | Pay Scale Group | Conversion Rule |
| `payScaleLevel` | Pay Scale Level | 0008 | 0 | TRFST | Pay Scale Level | Conversion Rule |

**IT0041 – Date Specifications (from Job Info):**

| EC Field | EC Description | S4 Infotype | S4 Subtype | S4 Field | S4 Field Description | Notes |
|----------|---------------|-------------|-----------|----------|---------|-------|
| `positionEntryDate` | Position Entry Date | 0041 | 01 | DAT01 | Date Value | Mapping can be adjusted per customer requirement |

**IT0016 – Contract Elements (from Job Info) – Canada Secondary Mapping:**

| EC Field | EC Description | S4 Infotype | S4 Field | S4 Field Description | Secondary Mapping Code | Notes |
|----------|---------------|-------------|----------|---------|----------------------|-------|
| `contractType` | Contract Type | 0016 | CTTYP | Contract Type | International | CONTRACT_TYPE_WS |
| `ContractEndDate` | Contract End Date | 0016 | CTEDT | Contract End Date | International | – |
| `entryIntoGroup` | Entry into Group | 0016 | KONDT | Entry into Group Date | '7' (Canada) | Canada-specific |
| `initialEntryDate` | Initial Entry | 0016 | EINDT | Initial Entry Date | '7' (Canada) | Canada-specific |
| `continuedSicknessPayPeriod` | Continued Sickness Pay Period | 0016 | LFZFR | Continued Sick Pay Period | '7' (Canada) | Canada-specific |
| `continuedSicknessPayMeasure` | Continued Sickness Pay Measure | 0016 | LFZZH | Continued Sick Pay Measure | '7' (Canada) | CONTINUED_PAY_MEASURE |

### 17.8 ERP_WS_11 – Compensation (Employment Level → IT0001)

Note: The Compensation portlet in EC maps only the Payroll Area to IT0001. The wage type amounts go through WS_12_ERP_BASIC_PAY instead.

| EC Field | EC Description | S4 Infotype | S4 Field | S4 Field Description | Value Mapping |
|----------|---------------|-------------|----------|---------|---------------|
| `payGroup` | Pay Group | 0001 | ABKRS | Payroll Area | PAY GROUP WS |

For KMZP: EC payGroup values (e.g., BiWeeklySalaried, WeeklyHourly, Contractors) map to S4 Payroll Areas (YB, YW, 99).

### 17.9 ERP_WS_12 – Pay Component Recurring (Employment Level → IT0014)

| EC Field | EC Description | S4 Infotype | S4 Field | S4 Field Description | Value Mapping |
|----------|---------------|-------------|----------|---------|---------------|
| `payComponent` | Pay Component | 0014 | LGART (Wage Type) |  | PAY_COMPONENT_WS_0014 |
| `paycompvalue` | Amount | 0014 | BETRG | Amount | – |
| `currencyCode` | Currency | 0014 | WAERS | Currency | CURRENCY_WS |
| `numberOfUnits` | Number of Units | 0014 | ANZHL | Number / Percentage | – |
| `unitOfMeasure` | Unit of Measure | 0014 | ZEINH | Unit of Measurement | UNITOFMEASURE_WS |

### 17.10 WS_12_ERP_BASIC_PAY – Basic Pay Wage Types (Employment Level → IT0008)

The actual wage type and amount for IT0008 come from the Pay Component Recurring portlet (same EC entity as WS_12), but via a dedicated template called WS_12_ERP_BASIC_PAY. This template uses **repetitive fields** (LGA01, BET01 → LGA02, BET02 etc.) to handle multiple wage types in one IT0008 record.

| EC Field | EC Description | S4 Infotype | S4 Subtype | S4 Repetitive Field | S4 Field Description | Notes |
|----------|---------------|-------------|-----------|---------------------|---------|-------|
| `payComponent` | Pay Component | 0008 | 0 | LGA01 | Wage Type 1 (Repetitive Field) | PAY_COMPONENT_WS value mapping; Repetitive field |
| `paycompvalue` | Amount | 0008 | 0 | BET01 | Amount 1 (Repetitive Field) | Repetitive field; include in generic value conversion |
| `startDate` | Start Date | 0008 | 0 | BEGDA | Start Date | – |
| `endDate` | End Date | 0008 | 0 | ENDDA | End Date | – |
| `currencyCode` | Currency | 0008 | 0 | WAERS | Currency | – |
| `numberOfUnits` | Number of Units | 0008 | 0 | ANZ01 | Number of Units 1 (Repetitive Field) | Repetitive field |
| `unitOfMeasure` | Unit of Measure | 0008 | 0 | EIN01 | Unit of Measurement 1 (Repetitive Field) | UNITOFMEASURE_WS; Repetitive field |

**Repetitive field mechanism:** The repetitive field setup (L column in BIB = "Repetitive Field") allows multiple EC pay component records (e.g., base salary + car allowance) to fill successive wage type slots (LGA01/BET01, LGA02/BET02) within a single IT0008 record.

### 17.11 ERP_WS_13 – Pay Component Non Recurring (Employment Level → IT0015)

| EC Field | EC Description | S4 Infotype | S4 Field | S4 Field Description | Value Mapping |
|----------|---------------|-------------|----------|---------|---------------|
| `payComponentCode` | Pay Component | 0015 | LGART | Wage Type | PAY_COMPONENT_WS_0015 |
| `value` | Amount | 0015 | BETRG | Amount | – |
| `payDate` | Pay/Issue Date | 0015 | BEGDA | Start Date | – |
| `currencyCode` | Currency | 0015 | WAERS | Currency | CURRENCY_WS |
| `unitOfMeasure` | Unit of Measure | 0015 | ZEINH | Unit of Measurement | UNITOFMEASURE_WS |

### 17.12 ERP_SPOTBONUS_0267 – Spot Bonus Off-Cycle (Employment Level → IT0267)

Uses the same EC entity as WS_13 (pay_component_non_recurring) but maps to IT0267 (Off-Cycle Payments):

| EC Field | EC Description | S4 Infotype | S4 Field | S4 Field Description | Notes |
|----------|---------------|-------------|----------|---------|-------|
| `payComponentCode` | Pay Component | 0267 | LGART | Wage Type | Direct cloning of subtype |
| `Amount` | Amount | 0267 | BETRG | Amount | – |
| `currencyCode` | Currency | 0267 | WAERS | Currency | CURRENCY_WS |

### 17.13 ERP_WS_14 – Payment Information Details (Employment Level → IT0009)

| EC Field | EC Description | S4 Infotype | S4 Subtype | S4 Field | S4 Field Description | Value Mapping |
|----------|---------------|-------------|-----------|----------|---------|---------------|
| `PaymentInformationV3_effectiveStartDate` | Effective Start Date | 0009 | 0 | BEGDA | Start Date | – |
| `accountNumber` | Account Number | 0009 | 0 | BANKN | Bank Account Number | – |
| `routingNumber` | Routing Number | 0009 | 0 | BANKL | Bank Key / Routing Number | – |
| `bankCountry` | Bank Country | 0009 | 0 | BANKS | Bank Country Key | COUNTRY_CODE |
| `paymentMethod` | Payment Method | 0009 | 0 | ZLSCH | Payment Method | PAYMENT METHOD WS (DirectDeposit→T, Cheque→C) |
| `payType` | Pay Type | 0009 | 0 | BNKSA | Bank Account Type | PAY TYPE (Chequing/Savings) |
| `accountOwner` | Account Owner | 0009 | 0 | EMFTX | Account Holder Name | – |
| `amount` | Amount | 0009 | 0 | BETRG | Amount | – |
| `percent` | Percent | 0009 | 0 | ANZHL | Number / Percentage | – |
| `currency` | Currency | 0009 | 0 | WAERS | Currency | – |
| `iban` | IBAN | 0009 | 0 | IBAN | IBAN | – |
| `purpose` | Purpose | 0009 | 0 | ZWECK | Payment Reference / Purpose | – |

---

## 18. BIB Configuration – Value Mapping, Secondary Mapping, Conversion Rules

### 18.1 Setup Sequence

1. Enable BIB (T77S0 switch)
2. Specify EC Instance ID
3. Maintain Constant Values (e.g., `ECPAO ASSIGNIDUSED = X`)
4. Import EC Metadata (program `ECPAO_ECTMPL_METADATA_WRITER` using XML exported from EC Admin Center → OData API Metadata Refresh and Export)
5. Import EC Picklists (program `ECPAO_PICKLIST_WRITER`; set picklist mode: X = MDF without Option ID, Y = MDF with Option ID, blank = legacy)
6. Define Value Mapping Entities (copy SAP sample content)
7. Define Value Mapping Details (fill EC↔S4 value pairs)
8. Create Transformation Template Group (TTG) with EC Instance ID and FTSD
9. Create/activate Transformation Templates (copy sample content, deactivate what's not needed)
10. Define Primary Mapping per template
11. Define Secondary Mapping for Canada-specific fields (IT0006 province, IT0009 bank format)

### 18.2 Transformation Template Group (TTG)

| Property | Description |
|----------|-------------|
| Template Group ID | e.g., KMZP_CA_01 |
| EC Instance ID | Links to EC metadata configuration |
| FTSD (Earliest Transfer Date) | Full Transmission Start Date – data before this date not replicated |
| Enable for Delta | Flag for delta mode |

Multiple TTGs can exist (e.g., one per go-live wave or payroll area). TTGs can be copied.

### 18.3 Primary vs Secondary Mapping

**Primary Mapping:** Standard field mapping applied to ALL employees regardless of country. Each row maps an EC field to an S4 infotype field, optionally with a value mapping entity and conversion rule.

**Secondary Mapping:** Country-specific override. Has an additional "Secondary Mapping Code" filter (e.g., 'CA' for Canada, '7' for Canada MOLGA). Secondary mapping beats primary mapping for employees of that country.

**Processing priority (highest to lowest):**
1. Secondary value mapping (country-specific)
2. Primary value mapping (general)
3. Pass-through of original EC value (no mapping defined)

**Key secondary mappings for KMZP Canada:**
- WS_10 `state` → IT0006 STATE: secondary mapping 'CA' with PROVINCE_CAN value mapping entity
- WS_10 `address2` → IT0006 LOCAT: secondary mapping 'CA'
- WS_4 `entryIntoGroup` → IT0016 KONDT: secondary mapping '7' (Canada MOLGA)
- WS_4 `initialEntryDate` → IT0016 EINDT: secondary mapping '7'
- WS_4 `continuedSicknessPayPeriod` → IT0016 LFZFR: secondary mapping '7'
- WS_4 `continuedSicknessPayMeasure` → IT0016 LFZZH: secondary mapping '7'

### 18.4 Key Value Mapping Entities for KMZP Canada

| Entity Name | Maps | Example |
|-------------|------|---------|
| GENDER_CODE | EC gender → S4 GESCH | M→1, F→2 |
| MARITAL_STATUS_WS | EC maritalStatus → S4 FAMST | single→1, married→2 |
| EMPLOYEE_CLASS_WS | EC employeeClass → S4 PERSG | Employee→1, Contractor→3 |
| EMPLOYEE_TYPE_WS | EC employmentType → S4 PERSK | FT_Salaried→YF, FT_Hourly→YU |
| EVENT_REASON | EC eventReason → S4 MASSN | HIREMP→01, REHEMP→40, TERRTM→34 |
| PAY GROUP WS | EC payGroup → S4 ABKRS | BiWeeklySalaried→YB, WeeklyHourly→YW |
| NATIONALITY_WS | EC country/nationality → S4 LAND1/NATIO | CA→CA, US→US |
| PROVINCE_CAN | EC province code → S4 STATE | ON→ON, QC→QC, BC→BC, AB→AB |
| STATE_WS | EC state/province (international) → S4 STATE | Generic state mapping |
| WORKSCHEDULE_RULE | EC work schedule code → S4 SCHKZ | EC_WS_FULL→NORM etc. |
| PAYMENT METHOD WS | EC paymentType → S4 ZLSCH | DirectDeposit→T, Cheque→C |
| PAY TYPE | EC payType → S4 BNKSA | Chequing→01, Savings→02 |
| COUNTRY_CODE | EC country → S4 LAND1 | CA→CA, US→US |
| CONTRACT_TYPE_WS | EC contractType → S4 CTTYP | |
| PAY_COMPONENT_WS | EC pay component → S4 wage type (IT0008) | Base_Salary→20SA |
| PAY_COMPONENT_WS_0014 | EC pay component → S4 wage type (IT0014) | CarAllowance→20CA |
| PAY_COMPONENT_WS_0015 | EC pay component → S4 wage type (IT0015) | OneTimeBonus→20BN |
| CURRENCY_WS | EC currency code → S4 WAERS | CAD→CAD |
| UNITOFMEASURE_WS | EC unit of measure → S4 ZEINH | HOURS→H, PIECES→ST |

### 18.5 Value Conversion Rules

| Rule Type | KMZP Use Case |
|-----------|--------------|
| Multiplication | FTE 1.0 → 100 for EMPCT |
| Replace using pattern | Canadian SIN: "123-456-789" → "123456789" (strip hyphens) |
| Split before/after string | Parsing combined fields |
| Edit using pattern | Format bank routing number |
| Check and replace | Clear BET01 if pay component is percentage-based |
| Append/Prepend with string | Add prefixes or suffixes to codes |

**BIB processing order:** Conversion Rules → Secondary Value Mapping → Primary Value Mapping → Pass-through

### 18.6 Constant Values and ECPAO Feature

Constants set fixed values used during replication:
- `ECPAO ASSIGNIDUSED = X` → Use EC Assignment ID as PERNR

The ECPAO feature (Decision Tree) returns org-structure-dependent constants (e.g., default payroll area based on company and employee group) for fields that cannot be derived from EC data alone.

### 18.7 Template Cloning (IT0014 / IT0015 Wage Types)

Define ONE template for IT0014 with the field mapping. Then in IMG (Define Infotypes and Subtypes for Cloning Transformation Templates), list all the wage type subtypes. The framework applies the same mapping for every listed subtype, avoiding duplicate template definitions.

---

## 19. BAdI Reference

### 19.1 EX_PAOCF_EC_DECIDE_HIRE_REHIRE

| | Detail |
|-|--------|
| Purpose | Determine if incoming data is new hire or should link to existing PERNR |
| Triggered when | Employment not found in PAOCFEC_EEKEYMAP |
| Key outputs | `cv_new_hire` (SPACE = not new), `ev_new_pernr`, `ev_full_trans_start_date` |
| Pre-delivered | `CL_EC_TOOLS_REHIRE_DUPLICATE` |

### 19.2 EX_PAOCF_EC_EXT_PERNR_MAP

| | Detail |
|-|--------|
| Purpose | Let EC supply the PERNR (Assignment ID = PERNR) |
| Prerequisite | Number range for PERNR must be external |
| Called | On new hire, before PERNR assignment |

### 19.3 EX_PAOCF_EC_CHANGE_INFOTYPE_DA

| | Detail |
|-|--------|
| Purpose | Override/supplement standard BIB mapping for standard infotypes before DB write |
| Called | After BIB mapping, before infotype update |
| Key uses | Derive BTRTL from custom EC field; handle complex pay scale derivation |
| Note | Before/after comparison prevents unnecessary retro |

### 19.4 EX_PAOCF_EC_PROCESS_EMPLOYEE – Person-Level Non-Standard ITs

| | Detail |
|-|--------|
| Purpose | Write non-standard infotypes at the Person level |
| Called after | All standard employment loop processing |
| KMZP uses | IT0185 (SIN from national_id_card portlet); IT0709 (PersonID_External) |

### 19.5 EX_PAOCF_EC_PROCESS_EMPLOYMENT – Employment-Level Non-Standard ITs

**This is the most-used extensibility BAdI for KMZP.**

| | Detail |
|-|--------|
| Purpose | Write non-standard infotypes at the Employment level |
| KMZP uses | IT0194/IT0558 (Canadian tax infotypes when address/job changes); IT0027 (Cost Distribution); IT0016 complex logic; Z-infotypes; IT0041 original hire date on rehire |

**Execution order (all per employment):**
```
1. BIB Primary + Secondary mapping (standard infotypes)
2. BAdI: EX_PAOCF_EC_CHANGE_INFOTYPE_DA
3. BAdI: EX_PAOCF_EC_EXCLUDE_FROM_DELET
4. Infotype DB update (PA function modules)
5. BAdI: EX_PAOCF_EC_PROCESS_EMPLOYEE
6. BAdI: EX_PAOCF_EC_PROCESS_EMPLOYMENT
```

### 19.6 EX_PAOCF_EC_EXCLUDE_FROM_DELET

| | Detail |
|-|--------|
| Purpose | Prevent specific infotype subtypes from deletion during replication |
| KMZP use | IT0014 subtypes for technical/control wage types not in EC (garnishments, union dues deductions managed only in S4) |

---

---

## PART 5 – Reference


## 20. Key Tables Reference

| Table | Description | Used For |
|-------|-------------|---------|
| PAOCFEC_EEKEYMAP | Employee key mapping: EC IDs ↔ S4 PERNR | Hire/rehire determination |
| HRSFEC_D_EEKEYMP | ECP variant of employee key mapping | ECP implementations |
| SFIOM_KMAP_OSI | Org object key mapping: EC code ↔ S4 OBJID | Org object create/update |
| PAOCFEC_KMAPCOSC | Cost center key mapping | Cost center in IT0001 |
| ODFIN_MAP_KOSTL | Finance CC mapping | FI-integrated CC mapping |
| SFIOM_QRY_ADM | Org query admin (last run timestamp/status) | Delta org replication control |
| SFIOM_GENRQ_HD | Org staging – Request header | Org replication staging |
| SFIOM_GENRQ_DP | Org staging – Data points | Org replication staging |
| SFIOM_GENRQ_DFLD | Org staging – Data fields | Org replication staging |
| HRSFEC_PN_FTSD | PERNR-specific FTSD override | Rehire date control |
| T77S0 | System switches (PLOGI, SFSFI, ADMIN) | All integration switches |
| ECPAO_FLD | EC field metadata (from metadata import) | Template field source list |
| V_ECPAO_CONSTANT | BIB constant values (picklist mode, PERNR mode) | BIB setup constants |

---

## 21. Key Programs and Transactions Reference

### Employee Replication

| Program / TCode                                            | Purpose                                                                 |
| ---------------------------------------------------------- | ----------------------------------------------------------------------- |
| ECPAO_EE_ORG_REPL_QUERY (TCode: ECPAO_EE_ORG_QUERY)        | Main employee replication query – fetches from EC via CPI               |
| RH_SFIOM_PROC_EE_ORG_ASS_RPRQ                              | Process org assignment staging (event: SAP_SFIOM_EE_ORGAS_RPPQ_CREATED) |
| RH_SFIOM_VIEW_EE_ORG_ASS_RPRQ (TCode: SFIOM_VIEW_REQUESTS) | View org assignment replication requests                                |
| RH_SFIOM_CHECK_EE_ORG_ASS (TCode: SFIOM_CHK_EE_ORG_ASS)    | Check org assignment consistency                                        |
| RH_SFIOM_DEL_EE_ORG_ASS_RPRQ                               | Delete processed org assignment requests                                |
| RP_HRSFEC_UPLOAD_EEKEY_MAPPING                             | Upload PERNR key mapping table (initial load)                           |

### Org Object Replication

| Program / TCode | Purpose |
|----------------|---------|
| RH_SFIOM_ORG_OBJ_REPL_QUERY (TCode: SFIOM_QRY_ORG_OBJ) | Send org object query to CPI |
| RH_SFIOM_PROC_ORG_STRUC_RPRQ | Process org staging → create/update OM objects |
| SFIOM_VIEW_ORG_REQS | View org object replication requests |
| RH_SFIOM_DEL_ORG_STRUC_RPRQ | Delete processed org staging requests |
| SFIOM_RESET_QRY_ADM | Reset stuck query status |
| SFIOM_ORGOBJ_GENERATE_SPL_CUST | Auto-generate sample BIB config for org replication |

### BIB Setup

| Program | Purpose |
|---------|---------|
| ECPAO_ECTMPL_METADATA_WRITER | Import EC entity/field metadata from EC OData export XML |
| ECPAO_PICKLIST_WRITER | Import EC picklist values |

---

*Document Reference: KMZP Phase 2 – SF to S/4HANA Mini Master Replication Design*
*Scope: Org Data Replication + Employee Data Replication (Canada)*
*Excludes: CPI/iFlow configuration, SOA Manager, Cloud Connector, Logical Ports*
*Source: SAP Replication Workbook (SuccessFactors Data Replication Workbook.xlsx), SAP Help Docs, SCN Community*
*Prepared/Updated: June 2026*

---
