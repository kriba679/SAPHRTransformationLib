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

# PHASE 1 — SAP ECC / S/4HANA HCM: Set up & Run a Company

**Goal:** Build a working HCM system end-to-end in a single SAP ECC / S/4HANA on-premise system — so you can hire employees, capture time, run payroll, and process the full Hire-to-Retire lifecycle for a fictional company (e.g. KMZP Consulting or BLDP Blood Pact Consulting).

**How to read this:** Each stage has the SPRO path / T-Code, the configuration object, and the dependency. Do them in order — each stage depends on the previous one. Country-specific items here use Canada (MOLGA 07) as the reference, consistent with your KMZP/BLDP notes.

---

## Note on system flavour: ECC vs. S/4HANA HCM (H4S4)

Phase 1 works equally on **SAP ECC 6.0 EhP8** or **SAP HCM for S/4HANA on-premise (H4S4 — compatibility scope)**. The SPRO menu, T-codes, and configuration objects (PA, OM, PT, PY) are functionally identical — H4S4 is the classic HCM code line packaged for S/4HANA. Two practical differences to keep in mind:

1. **Business Partner integration** — in S/4HANA, an HCM employee may need a CVI (Customer-Vendor Integration) Business Partner; the FI side enforces this by default.
2. **Authoritative configuration source** — the *HCM Local Enterprise Structure Configuration Guide* and *HCM Global Personnel Administration Configuration Guide* on help.sap.com are the SAP-published step-by-step references for both code lines (links in the Sources section at the end).

For this Phase 1, I'll reference the SPRO paths that are valid in both. Where a step differs in S/4HANA, it's flagged inline.

---

## STAGE 0 — Pre-requisites (System & User Setup)

1. System Access: Confirm SAP GUI access and a development client where customizing is open
2. SU3: Maintain own user parameters
  - `MOL` = 07 (Canada — country grouping / molga)
  - `UGR` = 07 (User group — drives Personnel Action menus in PA40)
  - `PLOG` = 01 (Plan version, after OM is set up)
3. SCC4: Verify client settings allow customizing (only in dev)
4. SE16 / SE16N: Familiarize with key tables you will reference (T500P, T501, T503, T527X, PA0001, HRP1000, HRP1001)
5. SPRO: Open **SAP Reference IMG** — this is your single navigation point for all customizing

---

## STAGE 1 — Enterprise Structure (Org Structures)

### 1.1 Company & Company Code (FI side, prerequisite for HCM)

1. Enterprise Str: SPRO → Enterprise Structure → Definition → Financial Accounting → Define Company *(creates Group Company, e.g. BLDP)*
2. Enterprise Str: SPRO → Enterprise Structure → Definition → Financial Accounting → Edit, Copy, Delete, Check **Company Code** *(4-char, e.g. KMZP / BLDP)*
3. Enterprise Str: SPRO → Enterprise Structure → Assignment → Financial Accounting → **Assign Company Code to Company**
4. OX02 / OX15 — verify CC and Company exist

### 1.2 Personnel Area (PA) and Personnel Subarea (PSA)

1. Enterprise Str: SPRO → Enterprise Structure → Definition → Human Resources Management → **Personnel Areas** *(4-char, e.g. 1510 Ontario, 1520 BC, 1530 Quebec)*
2. Enterprise Str: SPRO → Enterprise Structure → Definition → Human Resources Management → **Personnel Subareas** *(4-char per PA, e.g. PROF, MANA, EXEC, DIRE)*
3. Enterprise Str: SPRO → Enterprise Structure → Assignment → Human Resources Management → **Assign Personnel Area to Company Code**
4. Verify table T500P (Personnel Areas) and T001P (Personnel Subareas)

### 1.3 Personnel Structure — Employee Group (EG) and Employee Subgroup (ESG)

1. Enterprise Str: SPRO → **Enterprise Structure → Definition → Human Resources Management → Employee Groups** *(1-char, e.g. 1 Active, 2 Retiree, 4 Contractor, 6 Casual, 7 Temp)* — table T501
2. Enterprise Str: SPRO → **Enterprise Structure → Definition → Human Resources Management → Employee Subgroups** *(2-char, e.g. YL FT Hourly, YM PT Hourly, YN FT Salaried, YO PT Salaried, YK Casual)* — table T503K
3. Enterprise Str: SPRO → **Enterprise Structure → Assignment → Human Resources Management → Assign Employee Subgroup to Employee Group** *(determines which ESG combos are valid for each EG)* — table T503Z
4. Enterprise Str: SPRO → Personnel Management → Personnel Administration → Organizational Data → Organizational Assignment → Employee Groups/Subgroups → **Define Employee Attributes** *(country grouping, payroll status, employment status — defines whether ESG is "active" type)* — table T503 / V_503_B

### 1.4 ESG Groupings (these unlock downstream tables)

1. ESG grouping for **Personnel Calculation Rule (PCR)** *(controls payroll computations — e.g. 1 = hourly, 3 = salaried)*
2. ESG grouping for **Collective Agreement Provision (CAP)** *(controls pay scale structure)*
3. ESG grouping for **Work Schedule** *(1 industrial, 2 salaried)*
4. ESG grouping for **Wage Type permissibility** *(controls which WT can be entered for which ESG)*
5. PSA grouping for **Public Holiday Calendar** *(country grouping — usually 07 for Canada)*
6. PSA grouping for **Daily Work Schedule**
7. PSA grouping for **Time Quota Types**
8. PSA grouping for **Attendance/Absence Types**
9. PSA grouping for **Substitution Types**
10. PSA grouping for **Premium Wage Types**

---

## STAGE 2 — Number Ranges

1. PA04: **Number range for Personnel Number (PERNR)** — internal & external ranges, e.g. 01 = 00000001–99999999
2. NUMKR feature controls which range is picked at hiring (configured in Stage 4 below)
3. SNRO → object **PERSNO**: Personnel Number range
4. SNRO → object **HRADATA**: HR object number ranges
5. OONR (or `RP_OBJEC` in SPRO): **OM Object Number Ranges** — separate range per object type:
  - O — Organizational Unit (e.g. 50008000–50008999)
  - S — Position (e.g. 50015000–50015999)
  - C — Job (e.g. 50029000–50072999)
  - P — Person (mapped to PERNR)
  - K — Cost Center (external)

---

## STAGE 3 — Payroll Area & Control Record

1. SPRO → Personnel Management → Personnel Administration → Organizational Data → Organizational Assignment → Create Payroll Area *(2-char, e.g. YA = Weekly Canada, YB = BiWeekly Canada)*
2. SPRO → **Check Default Payroll Area** *(value used as fallback)*
3. SPRO → Period Parameters: **Define Period Parameters** *(03 Weekly, 04 Bi-weekly, etc.)*
4. SPRO → Date Modifiers: **Define Date Modifiers**
5. SPRO → Period Parameters → **Generate Payroll Periods** *(creates the actual period start/end dates by year)*
6. SPRO → **Generate Cumulation Calendar** *(monthly, quarterly, annual cumulations)*
7. PA03: Create **Payroll Control Record** for each payroll area *(controls retro accounting earliest date, period status — Released/Released for Correction/Exit Payroll)*

---

## STAGE 4 — Features (PE03)

Features are decision trees that default field values on infotypes during hiring.

1. PE03 — **NUMKR** → defaults Number Range for PERNR (IT0000) based on CC/PA
2. PE03 — **ABKRS** → defaults Payroll Area on IT0001 *(based on EG/ESG/PA)*
3. PE03 — **PINCH** → defaults Administrator Group on IT0001 *(based on PA/PSA)*
4. PE03 — **SCHKZ** → defaults Work Schedule Rule on IT0007 *(based on EG/ESG/PA/PSA)*
5. PE03 — **TARIF** → defaults Pay Scale Type & Area on IT0008
6. PE03 — **LGMST** → defaults Wage Type model (basic pay wage types) on IT0008
7. PE03 — **TMSTA** → defaults Time Management Status on IT0007 *(0 No time, 1 Actual times, 2 PDC, 7 Time eval w/o clock, 8 External services, 9 Time eval planned)*
8. PE03 — **WRKHR** → defaults Working Hours on IT0007
9. PE03 — **VTART** → default Substitution Type on IT2003
10. PE03 — **IGMOD** → Infogroup Modifier *(makes Personnel Action infogroups dependent on org data)*
11. PE03 — **PFREQ** → Payroll periodicity in feature evaluation
12. SM31 / SM30 → Maintain table T526 (Administrators) — used by PINCH (PER, TIM, PAY admins)

---

## STAGE 5 — Pay Scale Structure (the basis of base pay)

1. SPRO → Payroll → Payroll Canada → Basic Settings → Environment of Wage Type Maintenance → **Check Pay Scale Type** *(e.g. MA Managerial, EX Executive, U5 Union 1)*
2. SPRO → **Check Pay Scale Area** *(e.g. NU Non-Union, WN Canada-Winnipeg)*
3. SPRO → **Assign Pay Scale Type & Area to Personnel Area & Personnel Subarea**
4. SPRO → **Determine Default for Pay Scale Data** *(TARIF feature, configured above)*
5. SPRO → **Pay Scale Group & Pay Scale Level** *(creates the actual table — e.g. UU51 / 01 = $15.00 CAD)*
6. SPRO → **Define Pay Grade Type & Area** (if doing Pay Grades for managers)
7. SPRO → **Define Pay Grades and Levels** with Min / Max / Reference rate
8. SPRO → **Standard Pay Increase** (annual increase batch — t-code OOPC or in IMG)

---

## STAGE 6 — Wage Types

### 6.1 Wage Type Catalogue

1. SPRO → Payroll → Payroll Canada → Basic Settings → Environment of Wage Type Maintenance → **Create Wage Type Catalog (OH11)** — copy from model wage type to your custom WT
2. SPRO → **Check Wage Type Group** *(e.g. 0008 Basic Pay, 0014 Recurring, 0015 Additional)*
3. Common WTs to create: 1010 Base hourly rate, 1020 Pay scale salary, 1030 Hourly bonus, 1040 Performance bonus %, 9000s for taxes, 9100s for deductions

### 6.2 Wage Type Characteristics

1. SPRO → **Check Wage Type Characteristics (V_T511)** *(amount/number indicators, min/max, currency)*
2. SPRO → **Maintain Processing Classes, Cumulation Classes, Evaluation Classes** *(V_512W_D / V_512W_O)*
3. SPRO → **Wage Type Permissibility per PSA & ESG (V_511_B / V_511_M)**
4. SPRO → **Define Default Wage Types (LGMST feature)** — which basic pay wage types appear automatically on IT0008
5. SPRO → **Maintain Pay Scale Group / Level Wage Types (V_T510)** — base hourly & shift premium values per PS Group/Level
6. SPRO → **Wage Type Valuation (V_T539J / V_T539A)** — indirect valuation modules (TARIF, PRZNT, ANSAL)

---

## STAGE 7 — Organizational Management (OM)

### 7.1 OM Foundation Settings

1. OOPL or T77S0 → **Set Plan Version** *(usually 01 = Current Plan; activate via PLOGI-PLOGI)*
2. SPRO → Personnel Management → Organizational Management → Basic Settings → **Maintain Number Ranges** for OM objects (per object O/S/C/P/K)
3. SPRO → OM → Basic Settings → **Maintain Object Types** (which object types are active)
4. SPRO → OM → Basic Settings → **Data Model Enhancements → Infotype Maintenance → Maintain Infotypes** (which 1xxx infotypes are active for which objects)
5. SPRO → OM → Basic Settings → **Maintain Subtypes**
6. SPRO → OM → Basic Settings → **Maintain Country-Specific Infotypes** *(e.g. 1610 for US Vacancy)*
7. SPRO → OM → Basic Settings → **Maintain Relationships** (e.g. A002 Reports to, A003 Belongs to, A007 Describes, A008 Holder, B003 Incorporates)
8. SPRO → OM → Basic Settings → **Set Up Transport Connection** (T77S0 → CORR-TRSP)

### 7.2 Build the Org Structure

1. PPOCE: **Create Root Organization Unit** *(e.g. KMZP / BLDP at the top)*
2. PPOME: Build the hierarchy — Org Units (Business Unit → Division → Department), Positions, Jobs
3. PO03: Maintain **Job Catalog** *(e.g. Architect, Manager, Director, Clerk, Forklift Operator)*
4. PO13: Maintain **Positions** — describe each by Job (A007), assign to Org Unit (A003), set up "Reports to" (A002)
5. PO10: Maintain **Organizational Unit** infotypes (1000 Object, 1001 Relationships, 1002 Description, 1008 Account Assignment, 1018 Cost Distribution)
6. **Account Assignment (IT1008)** — link Org Unit to Personnel Area + Personnel Subarea (gets inherited down to Position)
7. **Cost Center Assignment (IT1001 K-A011)** — link Org Unit to Cost Center *(prereq: cost centers exist in CO via KS01)*
8. **Planned Compensation (IT1005)** — pay grade range for the Position
9. **Work Center (IT1010)** — physical/cost work center for Position
10. **Vacancy (IT1007)** — mark Position as vacant if open
11. PPOSE: View/verify the structure
12. PSO5/PFAL: Test the structure with simulations *(optional ALE if multi-system)*

### 7.3 Organizational Management Personnel Action

1. SPRO → OM → Procedures → **Define Personnel Actions for OM** *(e.g. create new Org Unit, create Position, transfer Position)*

---

## STAGE 8 — PA / OM Integration

1. T77S0: **PLOGI-ORGA = X** *(switches on PA↔OM integration — drives IT0001 ↔ HRP1001 sync)*
2. T77S0: **PLOGI-PLOGI = 01** *(active plan version)*
3. T77S0: **PLOGI-PRELI = 99999999** *(default position number used during hiring before a real position is selected)*
4. T77S0: **PLOGI-EVENT = X** *(triggers OM event when PA changes)*
5. T77S0: **PPABT-EVENT / PPVAC** *(integrate vacancy)*
6. **Run integration consistency programs** (in this order — direction matters):
  - **RHINTE00** — transfer data **PA → OM** (creates OM objects from existing IT0001 records)
  - **RHINTE10** — initial setup of OM tables in PA infotypes
  - **RHINTE20** — **check** consistency of all objects between PA and OM (read-only)
  - **RHINTE30** — transfer data **OM → PA** (updates IT0001 with current OM relationships)
7. After the runs, verify with PPOSE on the org side and PA20 on the EE side that they match

---

## STAGE 9 — Personnel Administration (Infotype Setup)

### 9.1 Infotype Configuration

1. SPRO → PA → **Customizing User Interface → Infotype Menu (V_T588B)** — define PA30 menu (tabs)
  - Examples: Hiring tab, Termination tab, LOA tab, Comp tab — each restricted by user group
2. SPRO → PA → **Infotype Menu Order** (V_T582A)
3. SPRO → PA → **Set Up Personnel Actions Menu (V_T588B for action menu type)** — what shows in PA40
4. SPRO → PA → **Maintain Subtypes** for each infotype (e.g. IT0006 subtype 1 = Permanent, 2 = Temp, 4 = Emergency contact)
5. SPRO → PA → **Time Constraints** (V_T582A) — uniqueness rules for each infotype
6. SPRO → PA → **Country-Specific Settings** *(IT0461 Tax CA, IT0462 Tax US, IT0009 banking — ensure proper screen versions)*
7. SE38 → Run program **RPDLGA20** to verify wage type usage report

### 9.2 Infogroups

1. SPRO → PA → **Personnel Actions → Define Infogroups** *(e.g. BH Hire, BO Org Reassignment, BC Change of Pay, BT Termination, BR Rehire, BL LOA, BW RTW, BM Transfer, BD Data Change)*
2. SPRO → PA → **Assign Infotypes to Infogroups** — per User Group (Operation INS / COP / MOD / DEL)
  - Sample BH Hire: 0000 Actions → 0001 Org Assignment → 0002 Personal Data → 0006 Address → 0007 Planned Working Time → 0008 Basic Pay → 0009 Bank → 0014 Recurring → 0021 Family → 0041 Date Specifications
3. SPRO → PA → **Maintain User Groups (UGR)** — link UGR to infogroup variant

### 9.3 Personnel Actions

1. SPRO → PA → **Define Personnel Action Types (V_T529A)** — link each action to its infogroup
  - Functional Character: 1 = First Hiring, 7 = Hiring from Recruitment
  - Employee Status (after action): 0 Withdrawn, 1 Inactive, 2 Retired, 3 Active
  - Special Payment Status: 0 No, 1 Standard, 2 Special
  - Field flags: PA / P / EG / ESG ready-for-input on PA40 initial screen
2. SPRO → PA → **Reasons for Personnel Actions (V_T530)** *(e.g. for Termination: TR Resignation, TD Deceased, TI Dismissal)*
3. SPRO → PA → **Action Menus per User Group** — restrict who sees which actions
4. SPRO → PA → **Check Status Features** — MSN20 (leaving), MSN21 (hiring), MSN32 (retirement)

### 9.4 Master Data Defaults & Screen Modifications

1. SPRO → PA → **Screen Modifications (T588M)** — hide/grey out fields by user group
2. SPRO → PA → **Default Wage Types per ESG** (LGMST built earlier)
3. SPRO → PA → **Date Specifications (IT0041)** — codes for Technical entry, Hire date, Service date, Vesting date

---

## STAGE 10 — Time Management Setup

### 10.1 Calendars

1. SCAL: Maintain **Public Holiday Calendar** (e.g. CA = Canada — define each public holiday)
2. SCAL: Maintain **Holiday Calendar** for the country/PSA combination
3. SCAL: Maintain **Factory Calendar** (which days are working days)
4. SPRO → PA → Personnel Time Management → Work Schedules → **Group Personnel Subareas for Holiday Calendar**
5. SPRO → **Group Personnel Subareas for Work Schedule**

### 10.2 Daily Work Schedule (DWS)

1. SPRO → Time Management → Work Schedules → **Define Break Schedules** *(e.g. 01 HRLN 12:00–13:30 = 1.0 hr unpaid)*
2. SPRO → Time Management → Work Schedules → **Daily Work Schedule Classes** (1–9 — use to differentiate normal/overtime/holiday)
3. SPRO → Time Management → Work Schedules → **Define Daily Work Schedule** (e.g. KZ00 Normal 08:00–17:00 with break 01)
4. SPRO → Time Management → Work Schedules → **Define Day Types** *(0 Work/paid, 1 TO/paid, 2 TO/Not paid, 3 TO/Special)*
5. SPRO → Time Management → Work Schedules → **Selection Rules for Day Types**

### 10.3 Period Work Schedule (PWS) and Work Schedule Rule (WSR)

1. SPRO → **Define Period Work Schedule** *(e.g. KZ00 = M-T-W-T-F = KZ00 / S-S = FREI)*
2. SPRO → **Set Work Schedule Rules and Work Schedules** (combo of ESG grouping + Holiday Calendar + PSA grouping + PWS = WSR)
3. SPRO → **Generate Work Schedules in Batch (RPTSHE10)** — generates IT2003 and CATS-relevant data per period
4. SPRO → **Set Default Value for Work Schedule (SCHKZ feature)**

### 10.4 Absence and Attendance Types

1. SPRO → Time Management → Time Data Recording → Absences → **Group Personnel Subareas for Attendances and Absences**
2. SPRO → Absences → **Define Absence Types (V_T554S)** *(e.g. LEHA Leave 1/2 day, SICK Sick leave)*
3. SPRO → Absences → **Define Absence Counting Classes** (how absence reduces planned/working hours)
4. SPRO → Absences → **Define Counting Rules**
5. SPRO → Absences → **Assign Counting Rules to Absence Types**
6. SPRO → Attendances → **Define Attendance Types** *(ATAT Attendance, ATBT Business trip, ATOS Off-site, ATOV Overtime)*
7. SPRO → Attendances → **Counting Classes for Attendances**
8. SPRO → Substitutions → **Personnel Subarea Grouping for Substitutions**
9. SPRO → Substitutions → **Define Substitution Types (V_T554S sub)**

### 10.5 Time Quotas

1. SPRO → Time Management → Time Data Recording → Managing Time Accounts → **Define Absence Quota Types (V_T556A)** *(e.g. 10 Vacation, 20 Sick)*
2. SPRO → **Quota Type Selection Rule Groups**
3. SPRO → **Quota Selection Rules** (V_T559L)
4. SPRO → **Generate Absence Quotas** — RPTQTA00 batch program
5. SPRO → **Define Attendance Quotas** if applicable
6. SPRO → **Quota Deduction Rules** — link Absence Type → Quota Type

### 10.6 Time Evaluation Configuration (only if running TM)

1. PE01 → **Time Evaluation Schema** *(TM00 = positive time w/clock, TM04 = negative time, TM01 = positive time)*
2. PE02 → **Personnel Calculation Rules (PCRs)** that the schema calls
3. SPRO → **Time Wage Type Selection (V_T510S)** — generate time wage types by DWS class / day type
4. SPRO → **Day Processing Functions** in schema (P2000, ZL, TIP, TES tables)
5. SPRO → **Time Types** (V_T555A) — accumulators (e.g. ZES Productive hours)
6. SPRO → **Time Evaluation Messages**
7. T77S0: **TIME-IGNAB / IT2002 / cluster B2** — switches for behaviour

### 10.7 Run Time Evaluation

1. PT60: Run **time evaluation** for selected pernrs/period
2. PT_CLSTB2 / PT66: View time evaluation results from cluster B2
3. PT64: Lock/unlock time data
4. PA61: Maintain time data (admin view)
5. PA63: Maintain time data (employee view)

---

## STAGE 11 — Payroll Configuration

### 11.1 Country-Specific Payroll Setup (Canada example, MOLGA 07)

1. SPRO → Payroll → Payroll Canada → **Basic Settings → Payroll Organization** *(Payroll Areas, Period parameters — already done in Stage 3)*
2. SPRO → Payroll Canada → **Tax → Define Tax Authorities** *(Federal, Provincial)*
3. SPRO → Payroll Canada → **Tax → Maintain Tax Forms** (T4, T4A, RL-1)
4. SPRO → Payroll Canada → **Statutory Deductions** — CPP, EI, QPP, QPIP
5. SPRO → Payroll Canada → **Garnishments** *(if applicable)*
6. SPRO → Payroll Canada → **Benefits Integration** *(if RPP/group insurance)*
7. SPRO → Payroll Canada → **ROE — Records of Employment**

### 11.2 Payroll Schema and PCRs

1. PE01: Open country payroll schema *(e.g. KC00 for Canada, U000 for USA, X000 model)*
2. PE01: Copy `KC00` → `ZC00` (custom schema)
3. PE02: Inspect/copy standard PCRs that you will modify *(e.g. K013 base pay, K023 averaging)*
4. PE02: Modify PCRs to handle company-specific logic — bonuses, custom deductions, retro logic
5. PE04: Maintain custom **Functions/Operations** if needed
6. SPRO → Payroll → **Set Modifiers — MODIF 2 (Wage Type modifier)**, MODIF W (working time), MODIF T (time wage type)
7. SPRO → Payroll → **Cumulation Wage Types** (e.g. /101 Total Gross, /5U0 Net, /559 Bank Transfer)
8. SPRO → Payroll → **Average Bases** *(IT0008 valuation for averaged wage types)*

### 11.3 Retroactive Accounting

1. PA03: Set **Earliest Master Data Change Date** in payroll control record
2. SPRO → Payroll → **Retroactive Accounting → Set Earliest Recalculation Date** at PA / EE Subgroup level
3. Decide which infotypes are retro-relevant (V_T588Z) — IT0008 is, IT0006 is not (for example)

### 11.4 Payment Methods & Banks

1. SPRO → Payroll → **Personnel Management → Personnel Administration → Payroll Data → Bank Details** — define IT0009 subtypes
2. SPRO → **Payment Methods per Country (FBZP-style)** — Cheque, ACH/EFT, Wire
3. SPRO → **House Banks** — your company's bank accounts

### 11.5 FI/CO Posting

1. SPRO → Payroll → **Reporting → Posting to Accounting → Define Symbolic Accounts**
2. SPRO → **Assign Wage Types to Symbolic Accounts** (V_T52EZ)
3. SPRO → **Assign Symbolic Accounts to GL Accounts** (table T030)
4. SPRO → **Posting Document Types**
5. SPRO → **Cost Element / Cost Center mapping** (CO integration)
6. PCP0: Run posting → review → release → post to FI

---

## STAGE 12 — Master Data Maintenance (Get Employees In)

1. PA40: **Hire** sample employees using BH Hire action — runs through the infogroup sequence (IT0000 → 0001 → 0002 → 0006 → 0007 → 0008 → 0009 → 0014 → 0041)
2. PA30: Maintain individual infotypes for an existing employee
3. PA20: Display master data (read-only)
4. PA70 / PA71: Fast Entry / Fast Entry for time data
5. PU22 / PU12: Mass infotype upload (LSMW or SECATT for bulk hire)
6. PA40: Run **Organizational Reassignment** (BO), **Change of Pay** (BC), **Transfer** (BM), **LOA** (BL), **Return to Work** (BW), **Termination** (BT), **Rehire** (BR) — to test full lifecycle

---

## STAGE 13 — Authorization Setup (Optional but Realistic)

1. PFCG: Create roles for Personnel Admin, Time Admin, Payroll Admin
2. SPRO → PA → **Authorization Main Switches (T77S0 — AUTSW)** — turn on structural authorizations if needed
3. P_PERNR (own data), P_ORGIN (master data per IT/subtype/auth level), P_ORGINCON (context-sensitive)
4. SU01: Assign roles to test users (e.g. KMZP_PERS_ADMIN, KMZP_TIME_ADMIN, KMZP_PAY_ADMIN)

---

## STAGE 14 — Run Day-to-Day Business Processes (the "Run the Company" stage)

This is where you exercise the system end-to-end. Each item is a real-world HR transaction.

### 14.1 Hire-to-Retire Lifecycle Events

1. **Hire (PA40 / BH)** — create new employee, assign Position, fill IT0000–IT0008+
2. **Promotion** — Org Reassignment (BO) with reason "Promotion" — change Position + Pay
3. **Demotion** (BP/OD) — change Position + Pay downward
4. **Transfer / Lateral Move (BM)** — Position move within / across PA
5. **Position Reclassification** — change Position attributes via PO13
6. **Pay Rate Change (BC)** — pay scale revision (CP) or pay progression (CR)
7. **Leave of Absence (BL)** — STD, LTD, Maternity, Paternity, Parental, Elder Care, Sick
8. **Return to Work (BW / WR)**
9. **Termination (BT)** — Resignation (TR), Dismissal (TI), Deceased (TD)
10. **Retirement** — change Employee Group to Retiree, status 2
11. **Rehire (BR)** — re-activate ex-employee, status 3
12. **Data Change (BD)** — non-status changes

### 14.2 Time Operations (per pay period)

1. CAT2 / CATS: Employee/manager time entry
2. PA61: HR admin maintains absences/attendances
3. PT60: Run time evaluation for the period
4. PT_QTA10 / PT_QTA00: Generate or display absence quotas
5. PT_BAL00: Display time balances (cumulated)
6. CADO: Cross-application time approvals

### 14.3 Payroll Cycle (per pay period)

1. PA03: Move control record into **"Released for Payroll"**
2. PC00_M07_CALC (Canada) / PC00_M99_CALC: **Simulation run**
3. Resolve errors → fix master data → re-simulate
4. PC00_M07_CALC: **Live run**
5. PC_PAYRESULT or PC00_M99_CWTR: View payroll results (cluster RT, CRT, BT, ST)
6. PC00_M99_CEDT or country-specific: **Print remuneration statements (pay slips)**
7. PC00_M07_CDTA / PC00_M99_DTA: **Bank transfer file (EFT)**
8. PC00_M07_CDTC: **Cheque printing**
9. PCP0: Post to FI/CO
10. PA03: Move control record to **"Exit Payroll"** for the period
11. PA03: Move to **next period** (Released for Correction → Released for Payroll)
12. PC_PAYRESULT: Inspect on individual employees

### 14.4 Period-End / Year-End

1. PC00_M07_CT4 (Canada): T4 generation
2. ROE generation (Canada, Record of Employment) when EE leaves
3. Year-end balance carryover (vacation, sick quota carryover via RPTQTA10/RPTLEAACC00 or feature QUOMO)
4. SPRO → Year-end customizing per country

---

## STAGE 15 — Reports & Verification

1. **PA20 / PA30** — display individual employees
2. **S_PH0_48000510 / S_AHR_61016369** — HR Information System reports
3. **HRBEN0085 / HRBEN0099** — benefits reports (if benefits configured)
4. **PC_PAYRESULT** — payroll results
5. **HR Forms (HRFORMS / PE51)** — pay slip layout
6. **Ad-hoc query (SQ01/SQ02/SQ03 + LDB PNP)** — custom HR queries
7. **PA_SE16 on PA0001** — verify org assignments
8. **HRP1000 / HRP1001** — verify OM objects and relationships

---

## Quick Reference: Most-Used T-Codes


| T-Code             | Purpose                              |
| ------------------ | ------------------------------------ |
| SPRO               | All customizing                      |
| SU3                | Maintain own user parameters         |
| PE03               | Features                             |
| PE01               | Schemas                              |
| PE02               | PCRs                                 |
| PE04               | Functions/Operations                 |
| PA40               | Personnel Action (hire/term/etc.)    |
| PA30               | Maintain HR master data              |
| PA20               | Display HR master data               |
| PA61/63            | Time data maintenance                |
| PA03               | Payroll control record               |
| PA04               | PERNR number ranges                  |
| PPOCE              | Create root org unit                 |
| PPOME              | Manage org plan                      |
| PPOSE              | Display org plan                     |
| PO10 / PO13 / PO03 | Maintain Org Unit / Position / Job   |
| OONR               | OM number ranges                     |
| SCAL               | Holiday/factory calendar             |
| PT60               | Run time evaluation                  |
| CAT2               | Time entry                           |
| PC00_M99_CALC      | Cross-country payroll driver         |
| PC00_M07_CALC      | Canada payroll driver                |
| PCP0               | Posting run management               |
| RHINTE00–30        | PA-OM integration consistency        |
| T77S0              | System table — switches (PLOGI etc.) |


---

## Switches & Tables to Know


| Switch / Table               | Meaning                                            |
| ---------------------------- | -------------------------------------------------- |
| T77S0 PLOGI-ORGA = X         | Activate PA/OM integration                         |
| T77S0 PLOGI-PLOGI = 01       | Active plan version                                |
| T77S0 PLOGI-PRELI = 99999999 | Default position for new hire                      |
| T500P                        | Personnel Areas                                    |
| T001P                        | Personnel Subareas                                 |
| T503                         | EG/ESG and attributes                              |
| T527X                        | Org Units (legacy lookup)                          |
| T526                         | Administrators                                     |
| T549A                        | Payroll Areas                                      |
| T549Q                        | Payroll Periods                                    |
| T510                         | Pay Scale Wage Types                               |
| T511                         | Wage Type Characteristics                          |
| T512W                        | Processing/Cumulation/Evaluation classes           |
| HRP1000                      | OM Object header                                   |
| HRP1001                      | OM Relationships                                   |
| PA0000–PA9999                | Infotype data tables                               |
| PCL1 / PCL2 / PCL3           | HR cluster tables (B1/B2 = time, RT/CRT = payroll) |


---

## Final Checkpoint — When can you say "Phase 1 is done"?

You should be able to demonstrate, end-to-end, the following without errors:

1. Hire a new employee in PA40 → all infotypes populate with sensible defaults
2. The org chart in PPOSE shows the new hire under their Position
3. Maintain time entry in CAT2 / PA61 for one pay period (mix attendance + absence)
4. Run time evaluation (PT60) without rejections
5. Run a payroll simulation (PC00_M07_CALC) → review payslip in PC_PAYRESULT
6. Post payroll to FI in PCP0 → verify GL posting document
7. Run a Termination (PA40 / BT) → confirm IT0000 status = 0, payroll exit logic
8. Run an Org Reassignment (BO) → confirm IT0001 + Position change reflect
9. Run a Rehire (BR) → status returns to 3
10. Generate reports (PC_PAYRESULT, PA_SE16) and reconcile

When all 10 work cleanly, you have a fully functioning ECC HCM system ready for **Phase 2 — HR Transformation simulation** (migrating to SuccessFactors EC, configuring Time, and replicating back for ECP).

---

## Sources & References

### SAP Help Portal — Authoritative Configuration Guides (PDFs)

- [HCM Local Enterprise Structure: Configuration Guide (2311)](https://help.sap.com/doc/3449018718a1497a94f73ff42d7a70d1/2311/en-US/HCM_Local_Enterprise_Structure_Configuration_Guide_10M.pdf) — step-by-step for Stages 1–3 (Company → CC → PA → PSA → EG → ESG)
- [HCM Global Personnel Administration: Configuration Guide (2305)](https://help.sap.com/doc/0c77e45a7117450b8123b5bc4fcc2414/2305/en-US/HCM_Global_Personnel_Administration_Configuration_Guide.pdf) — Stages 9 (PA infotype setup, infogroups, actions)
- [Local Payroll Administration: Configuration Guide (2305)](https://help.sap.com/doc/0019b42ad1414b3eb9a9383591dc643e/2305/en-US/Local_Payroll_Administration_Configuration_Guide.pdf) — Stages 5–6, 11 (pay scale, wage types, payroll)
- [Migrating Data from ERP HCM to EC: Configuration Guide (2311)](https://help.sap.com/doc/39b5eeb5766d4a76bf585ef81ef153ab/2311/en-US/Migrating-Data-from-ERP-HCM-to-EC-Config-Guide-4BE.pdf) — relevant for Phase 2

### SAP Help Portal — Conceptual Documentation

- [Personnel Subarea — SAP ERP HCM Help](https://help.sap.com/docs/ERP_HCM/73275d5d354843ecad62cbe3e4c243e8/adc2dc53b5ef424de10000000a174cb4.html)
- [Personnel Area — SAP ERP HCM SPV Help](https://help.sap.com/docs/ERP_HCM_SPV/fba4ead9d03848c3b0945464436ff0bf/45cadc53b5ef424de10000000a174cb4.html)
- [Personnel Subarea Groupings — SAP S/4HANA Help](https://help.sap.com/docs/SAP_S4HANA_ON-PREMISE/c6c3ffd90792427a9fee1a19df5b0925/df38e153038a424de10000000a174cb4.html)
- [Elements of the Enterprise Structure — SAP ERP HCM](https://help.sap.com/erp_hcm_ias2_2014_03/helpdata/en/ce/36e153a217424de10000000a174cb4/content.htm)
- [SAP S/4HANA Help Portal — On-Premise](https://help.sap.com/docs/SAP_S4HANA_ON-PREMISE)

### SAP Learning — Step-by-step courses

- [Master Data Configuration in SAP HCM for S/4HANA (course)](https://learning.sap.com/courses/master-data-configuration-in-sap-hcm-for-s-4hana)
- [Modifying the Enterprise Structure (lesson)](https://learning.sap.com/courses/master-data-configuration-in-sap-hcm-for-s-4hana/modifying-the-enterprise-structure)
- [Enhancing the Personnel Structure (lesson)](https://learning.sap.com/courses/master-data-configuration-in-sap-hcm-for-s-4hana-es/enhancing-the-personnel-structure)
- [Configuring Personnel Actions (lesson)](https://learning.sap.com/courses/master-data-configuration-in-sap-hcm-for-s-4hana/configuring-personnel-actions)
- [Creating Personnel Actions (lesson)](https://learning.sap.com/courses/master-data-configuration-in-sap-hcm-for-s-4hana/creating-personnel-actions)
- [Business Processes in SAP HCM on S/4HANA — Analyzing HCM Structures](https://learning.sap.com/courses/business-processes-in-sap-hcm-on-s-4hana/analyzing-hcm-structures)
- [Get Started with SAP HCM Payroll — Running Payroll](https://learning.sap.com/learning-journeys/get-started-with-sap-hcm-payroll/running-payroll_e1629f8b-5236-4a89-af07-6531cb316d9b)

### SAP Community Blogs (used to validate Stage paths and feature behaviour)

- [Configuration steps – Enterprise structure and Personnel Structure (SAP Community blog)](https://community.sap.com/t5/enterprise-resource-planning-blog-posts-by-members/configuration-steps-enterprise-structure-and-personnel-structure/ba-p/13543971)
- [SAP HRM structure and configuration (SAP Community blog)](https://community.sap.com/t5/enterprise-resource-planning-blog-posts-by-members/sap-hrm-structure-and-configuration/ba-p/13286423)
- [Organizational Management in SAP S/4HANA HCM (SAP Community blog)](https://community.sap.com/t5/technology-blog-posts-by-members/organizational-management-in-sap-s-4hana-hcm/ba-p/14234285)
- [What is SAP HCM for S/4HANA On-Premise (H4S4)? (SAP Community blog)](https://community.sap.com/t5/enterprise-resource-planning-blog-posts-by-members/what-is-sap-hcm-for-s-4hana-on-premise-h4s4/ba-p/13493317)
- [The 5 Commonly Used SAP HCM Features (NUMKR, ABKRS, PINCH, SCHKZ, TARIF, LGMST) — Community blog](https://community.sap.com/t5/enterprise-resource-planning-blog-posts-by-members/the-5-commonly-used-sap-hcm-features/ba-p/13441668)
- [Wage Type Creation in SAP HCM (SAP Community blog)](https://community.sap.com/t5/enterprise-resource-planning-blog-posts-by-members/wage-type-creation-in-sap-hcm/ba-p/13286746)
- [Beginner's Guide to Payroll Personnel Calculation Rules (PCR) — I](https://community.sap.com/t5/human-capital-management-blog-posts-by-members/beginner-s-guide-to-payroll-personnel-calculation-rules-pcr-i/ba-p/14304844)
- [Beginner's Guide to Payroll Personnel Calculation Rules (PCR) — II](https://community.sap.com/t5/human-capital-management-blog-posts-by-members/beginner-s-guide-to-payroll-personnel-calculation-rules-pcr-ii/ba-p/14370343)
- [Work Schedule Creation (SAP Community blog)](https://community.sap.com/t5/enterprise-resource-planning-blog-posts-by-members/work-schedule-creation/ba-p/13250026)
- [Quick Start Config Guide to Planned Working Time Replication (SAP Community)](https://community.sap.com/t5/human-capital-management-blog-posts-by-sap/quick-start-config-guide-to-planned-working-time-replication/ba-p/13515185)
- [Feature LGMST – Planned payment specification (SAP Community blog)](https://community.sap.com/t5/enterprise-resource-planning-blog-posts-by-members/feature-lgmst-planned-payment-specification/ba-p/13031052)
- [Step-by-step procedure to set up organisational key (SAP Community blog)](https://community.sap.com/t5/enterprise-resource-planning-blog-posts-by-members/step-by-step-procedure-to-set-up-organisational-key/ba-p/13216488)

### SCN Wiki

- [Payroll Schemas and Personnel Calculation Rules (PCRs) — SCN Wiki](https://wiki.scn.sap.com/wiki/pages/viewpage.action?pageId=72405)
- [Understanding Relationship PT and PY — SCN Wiki](https://wiki.scn.sap.com/wiki/display/ERPHCM/Understanding+Relationship+PT+and+PY)

### Your Project Notes (used as the structural backbone)

- `2 Step-by-Step.md` — the seed checklist
- `1 Full Config.md` — full PA/OM/PT/PY high-level config
- `0 KMZP Consulting.md` & `3 BLDP Blood Pact Consulting.md` — the company examples used in the worked configurations
- `1 OM SPRO Settings.md` — OM specifics
- `4 Personnel Actions.md` — action types, infogroups, status flags
- `3 Features.md` — NUMKR/ABKRS/PINCH/SCHKZ/TARIF/LGMST/TMSTA

