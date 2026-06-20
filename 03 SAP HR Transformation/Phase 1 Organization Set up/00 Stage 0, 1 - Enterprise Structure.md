
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

**Goal:** Build a working HCM system end-to-end in a single SAP ECC / S/4HANA on-premise system — so you can hire employees, capture time, run payroll, and process the full Hire-to-Retire lifecycle for a fictional company (e.g. KMZP Consulting or BLDP Blood Pact Consulting).

**How to read this:** Each phase has the SPRO path / T-Code, the configuration object, and the dependency. Do them in order — each phase depends on the previous one. Country-specific items here use Canada (MOLGA 07) as the reference, consistent with your KMZP/BLDP notes.

---
## Step 0 — Pre-requisites (System & User Setup)

1. System Access: Confirm SAP GUI access and a development client where customizing is open
2. SU3: Maintain own user parameters
  - `MOL` = 07 (Canada — country grouping / molga)
  - `UGR` = 07 (User group — drives Personnel Action menus in PA40)
  - `PLOG` = 01 (Plan version, after OM is set up)
	![](screenshots/20260505123809.png)

1. SCC4: Verify client settings allow customizing (only in dev)

2. SE16 / SE16N: Familiarize with key tables you will reference (T500P, T501, T503, T527X, PA0001, HRP1000, HRP1001)

3. SPRO: Open **SAP Reference IMG** — this is your single navigation point for all customizing

---
## Step 1 — Create Chart of Accounts (OB13 + OB62)

The Chart of Accounts (COA) is the master list of all GL account numbers used in Financial Accounting. It must exist before a Controlling Area can be configured, and before cost centers can post to GL accounts.

> **Important context for KMZP:** If your SAP system already has a standard chart of accounts configured (e.g., the SAP-delivered `INT` or a country-specific COA for Canada), you should use that and skip COA creation — just verify the assignment to company code KMZP via OB62. Only follow Sub-Step A if you need a custom COA.

---

### KMZP Chart of Accounts Design Decision

| Field | Value | Reason |
|---|---|---|
| Chart of Accounts ID | `KMCA` | 4-char key — KM = KMZP, CA = Canada |
| Description | KMZP Canada Chart of Accounts | Clear, self-documenting |
| Maintenance Language | `EN` | English |
| Length of GL Account No. | `6` | SAP standard; allows accounts 100000–999999 |
| Controlling Integration | ✅ Checked | Required so GL accounts can be used in CO postings (payroll posting to cost centers) |

---

### Sub-Step A — Create the Chart of Accounts (OB13)

**T-Code:** `OB13`

**SPRO Path:** SPRO → Financial Accounting → General Ledger Accounting → GL Accounts → Master Data → Preparations → Edit Chart of Accounts List

1. Enter T-Code `OB13` → the COA list table opens
2. Click **New Entries**
3. Fill in the fields:

| Field | Value |
|---|---|
| Chart of Accts | `KMCA` |
| Description | `KMZP Canada Chart of Accounts` |
| Maint. Language | `EN` |
| Length of GL Account Number | `6` |
| Controlling Integration | ✅ (check the box) |
| Group Chart of Accounts | Leave blank unless you have a corporate group COA |
| Block Indicator | Leave unchecked |

4. **Save (Ctrl+S)**

> **Controlling Integration checkbox:** This is mandatory. Without it, GL accounts in this COA cannot be used as reconciliation accounts in CO, and payroll cost postings to cost centers will fail.

---

### Sub-Step B — Assign Chart of Accounts to Company Code (OB62)

**T-Code:** `OB62`

**SPRO Path:** SPRO → Financial Accounting → General Ledger Accounting → GL Accounts → Master Data → Preparations → Assign Company Code to Chart of Accounts

1. Enter T-Code `OB62`
2. Find company code `KMZP` in the list
3. In the **Chart of Accts** column for row KMZP, enter `KMCA`
4. **Save (Ctrl+S)**

Assigning INT Chat Of accounts to KMZP
![](screenshots/20260525173139.png)


> **One COA per company code:** A company code can only be assigned to one chart of accounts. This assignment is permanent once documents are posted — do not change it after go-live.

---

### Verification

| Check | T-Code | Expected Result |
|---|---|---|
| COA KMCA exists | `OB13` | Row KMCA visible in the list with Controlling Integration checked |
| COA assigned to KMZP | `OB62` | Company code KMZP row shows Chart of Accts = KMCA |

---

## Step 2 — Create Controlling Area and Activate Cost Center Accounting (OKKP)

The Controlling Area is the top-level organizational unit in SAP Controlling (CO). All cost centers, internal orders, and CO postings (including payroll) belong to a controlling area. It must exist and have Cost Center Accounting activated before you can create cost centers in KS01.

> **KMZP design:** One controlling area (`KMZP`) mapped 1:1 to company code `KMZP`. This is the simplest and most common setup for a single-entity company operating in one currency (CAD).

---

### KMZP Controlling Area Design

| Field                      | Value                        | Reason                                                                                                                                                        |
| -------------------------- | ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Controlling Area           | `KMZP`                       | Same key as company code — clear 1:1 mapping                                                                                                                  |
| Name                       | `KMZP Manufacturing CO Area` | Descriptive                                                                                                                                                   |
| Controlling Area Currency  | `CAD`                        | Canadian Dollar — matches company code                                                                                                                        |
| Currency Type              | `10`                         | Company Code Currency — correct when all company codes in the CO area share the same currency (CAD). Simple and standard for a single-entity Canadian company |
| Chart of Accounts          | `KMCA`                       | The COA created in Step 0.3                                                                                                                                   |
| Fiscal Year Variant        | `K4`                         | Calendar year (Jan 1 – Dec 31), 12 periods. SAP-delivered standard. Correct for Canadian fiscal year if KMZP operates Jan–Dec                                 |
| Cost Center Std. Hierarchy | `KMZP_H`                     | Standard cost center hierarchy root node — enter a name; SAP will create it                                                                                   |

> **Currency Type 10 note:** Type `10` (Company Code Currency) means CO postings are always in CAD. This is correct for KMZP since there is only one company code and it operates in CAD. If you ever add a USD or EUR company code to this CO area, you would need Type `20` (Controlling Area Currency). For now, use `10`.

> **Fiscal Year Variant K4:** K4 is SAP-delivered for January–December calendar year. Use F4 to verify it exists in your system. If your company's fiscal year differs (e.g., April–March), select the appropriate variant — but it must match the fiscal year variant assigned to company code KMZP in FI.

---

### Sub-Step A — Create the Controlling Area (OKKP)

**T-Code:** `OKKP`

**SPRO Path:** SPRO → Controlling → General Controlling → Organization → Maintain Controlling Area

1. Enter T-Code `OKKP`
2. Click **New Entries** (or use the **Create** button)
3. Enter the Controlling Area key: `KMZP`
4. Fill in the **Basic Data** tab:

| Field | Value |
|---|---|
| Controlling Area | `KMZP` |
| Name | `KMZP Manufacturing CO Area` |
| Controlling Area Currency | `CAD` |
| Currency Type | `10` |
| Chart of Accounts | `KMCA` |
| Fiscal Year Variant | `K4` |
| Cost Center Standard Hierarchy | `KMZP_H` |
![](screenshots/20260525174500.png)


5. **Save (Ctrl+S)**
   - SAP will create the standard hierarchy node `KMZP_H` automatically

---

### Sub-Step B — Activate Cost Center Accounting Component (OKKP)

After creating the controlling area, you must activate Cost Center Accounting as an active component. Without this, KS01 will not allow cost center creation under this CO area.

**T-Code:** `OKKP`

1. In OKKP, select controlling area `KMZP` and double-click on **Activate Components / Control Indicators** in the left-side navigation tree
2. Set the following:

| Component                | Setting                                                              |
| ------------------------ | -------------------------------------------------------------------- |
| Cost Center Accounting   | **Active** (set to `Active`)                                         |
| Order Management         | Set to Active if you plan to use internal orders (optional for KMZP) |
| Profit Center Accounting | Set to Active if EC-PCA is in scope (optional)                       |
| All other components     | Leave as per your project scope                                      |
|                          |                                                                      |
![](screenshots/20260525174522.png)
3. **Save (Ctrl+S)**

---

### Sub-Step C — Assign Company Code KMZP to Controlling Area (OKKP)

**T-Code:** `OKKP`

1. In OKKP, select controlling area `KMZP`
2. Double-click on **Assignment of Company Code(s)** in the left-side navigation tree
3. Click **New Entries**
4. Enter Company Code: `KMZP`
5. **Save (Ctrl+S)**

![](screenshots/20260525174535.png)

> **Prerequisite check before saving:** SAP validates that the company code's fiscal year variant and chart of accounts match those set on the controlling area. If you get an error here, verify that:
> - Company code KMZP in OB62 uses COA = `KMCA` ✅ (done in Step 0.3)
> - Company code KMZP's fiscal year variant in FI matches `K4` ✅ (check in OBY6 if needed)

---

### Verification

| Check | T-Code | Expected Result |
|---|---|---|
| Controlling area KMZP exists | `OKKP` | Row KMZP visible with currency CAD, COA KMCA, FYV K4 |
| Cost Center Accounting active | `OKKP` → Activate Components | Cost Center Accounting = Active |
| Company code KMZP assigned | `OKKP` → Assignment of Company Code(s) | KMZP listed under CO area KMZP |
| No currency mismatch error | Assignment save | Saves without error |

---

## Step 3 — Create Cost Centers in FI/CO (KS01)

Cost centers must exist in FI/CO **before** you can assign them to org units in PPOME. This is a Finance/CO activity but is documented here because it is a hard prerequisite for Step 3C (assigning cost centers to Level 3 org units).

> **Design decision (confirmed):** One cost center per Level 3 department. Level 4 teams and all positions below each department inherit the department's cost center automatically via SAP OM inheritance. No cost centers are needed at Level 1 or Level 2 for payroll posting — Level 2 rollup nodes can be created in KSH1 separately if Finance wants a CO reporting hierarchy.

---

### KMZP Cost Centers to Create

| CC ID     | Description               | Dept (Org Unit) | Location                | Category       |
| --------- | ------------------------- | --------------- | ----------------------- | -------------- |
| KMZP-EXEC | Executive Leadership      | EXEC            | KM10 – Toronto ON       | Administration |
| KMZP-FNCE | Finance & Accounting      | FNCE            | KM10 – Toronto ON       | Administration |
| KMZP-HRDE | Human Resources           | HRDE            | KM10 – Toronto ON       | Administration |
| KMZP-PROD | Production Operations     | PROD            | KM15 – British Columbia | Production     |
| KMZP-QUAL | Quality Assurance         | QUAL            | KM15 – British Columbia | Administration |
| KMZP-MAIN | Maintenance & Engineering | MAIN            | KM15 – British Columbia | Administration |
| KMZP-SLSO | Sales Operations          | SLSO            | KM20 – Quebec           | Administration |
| KMZP-MKTG | Marketing & Business Dev  | MKTG            | KM20 – Quebec           | Administration |

> **Note on CC IDs:** SAP allows up to 10 characters. The `KMZP-EXEC` format is clear and self-documenting in CO reports. Adjust to match your controlling area's naming convention if Finance has a standard already.

---

### Sub-Step A — Create Each Cost Center (KS01)

**T-Code:** `KS01`

Repeat for each of the 8 cost centers in the table above.

**1. Launch KS01:**
- Enter Controlling Area — the CO area assigned to company code KMZP (confirm in FI config; often the same as the company code key)
- Enter **Cost Center** ID (e.g., `KMZP-EXEC`)
- Enter **Valid From** date: `01.01.2026`
- Press **Enter** to open the master data screen

**2. Fill in the Basic Data tab:**

| Field                | Value                                          | Notes                                                |
| -------------------- | ---------------------------------------------- | ---------------------------------------------------- |
| Name                 | e.g., Executive Leadership                     | Up to 40 characters                                  |
| Description          | e.g., KMZP Executive Leadership costs          | Optional but recommended                             |
| Cost Center Category | Use **F4** to select                           | See note below — do NOT hardcode without checking F4 |
| Hierarchy Area       | Assign to standard hierarchy root or KMZP node | Use F4 to see available nodes                        |
| Company Code         | `KMZP`                                         | Must match company code                              |
| Currency             | `CAD`                                          | Matches company code currency                        |
| Person Responsible   | Department head name                           | Optional                                             |

> **Cost Center Category:** SAP delivers standard categories but your system may have customised values. Always press **F4** to see the allowed values. Typical values: `1` or similar for Production (use for KMZP-PROD), and an Administration/Overhead category for all others. Never type a value without confirming it exists via F4 — invalid categories will cause an error on save.

**3. Save (Ctrl+S)**
- SAP confirms the cost center was created
- Record the CC ID for reference during PPOME assignment

---

### Sub-Step B — Assign Cost Centers to Level 3 Org Units (PPOME)

Once all 8 CCs are created in KS01, link them to the corresponding Level 3 org units in PPOME. SAP stores this as a relationship between the Org Unit (O object) and the Cost Center (K object) in HRP1001.

**T-Code:** `PPOME`

1. Open `PPOME` → Plan Version `01`
2. Navigate the tree to the Level 3 org unit (e.g., EXEC — under HOFF)
3. Select the org unit row → the **Detail** panel opens on the right side
4. Look for the **Cost Center** field (on the Account Assignment or Basic Data tab depending on your PPOME layout) → enter the CC ID (e.g., `KMZP-EXEC`) and press Enter to validate
5. Set validity: `01.01.2026 – 31.12.9999`
6. **Save**

Repeat for all 8 Level 3 departments:

| Org Unit | Cost Center |
|---|---|
| EXEC | KMZP-EXEC |
| FNCE | KMZP-FNCE |
| HRDE | KMZP-HRDE |
| PROD | KMZP-PROD |
| QUAL | KMZP-QUAL |
| MAIN | KMZP-MAIN |
| SLSO | KMZP-SLSO |
| MKTG | KMZP-MKTG |

> **Inheritance reminder:** You do NOT need to assign cost centers to Level 4 teams (PRDA, PRDB, CUST) or to individual positions — they automatically inherit the cost center of their parent Level 3 org unit via the OM inheritance principle.

> **Override positions (PlaMgr / ProdSup):** These positions sit under PROD (CC = KMZP-PROD) but have PSA=MANA. If Finance wants management labour costs separated from union floor costs, set an explicit cost center on those positions via `PO13` → IT1008 (override). Otherwise leave them to inherit KMZP-PROD.

---

### Verification

| Check | T-Code | Expected Result |
|---|---|---|
| All 8 CCs exist | `KS03` → enter each CC ID | Master data displays without error |
| CCs are valid from 01.01.2026 | `KS03` → Basic Data tab | Valid-from = 01.01.2026, currency = CAD |
| CC assigned to each L3 org unit | `PPOME` → select org unit → detail panel | Cost Center field shows the assigned CC |
| Inheritance working on positions | `PO13` → select a position → IT1008 | Cost center shows the parent dept CC (inherited, not direct) |

---


Transport
HR: KMZP | P1 | Enterprise Structure Setup
## Step 4 — Enterprise Structure (Org Structures)

### 4.1 Company & Company Code (FI side, prerequisite for HCM)

1. Enterprise Str: SPRO → Enterprise Structure → Definition → Financial Accounting → Define Company *(creates Group Company, e.g. BLDP)*
	![](screenshots/20260505125220.png)

2. Enterprise Str: SPRO → Enterprise Structure → Definition → Financial Accounting → Edit, Copy, Delete, Check **Company Code** *(4-char, e.g. KMZP / BLDP)*

	![](screenshots/20260507120134.png)

	![](screenshots/20260507120115.png)

3. Enterprise Str: SPRO → Enterprise Structure → Assignment → Financial Accounting → **Assign Company Code to Company**
	![](screenshots/20260507120243.png)

4. OX02 / OX15 — verify CC and Company exist

---

![](screenshots/20260507120931.png)
- Union vs Non-union → Personnel Structure (ESG, via CAP grouping). Don't fork PA/PSA on this.
- Hourly vs Salaried → Personnel Structure (ESG, via PCR grouping).
- FT vs PT → Personnel Structure (ESG, via Work Schedule grouping).
- Contractor vs Employee → Both: EG = External (Personnel Structure), and contractors are normally tracked in PA but excluded from payroll groupings.
- Province/legal entity → Enterprise Structure (PA).
- Job category / regulatory zone within a PA → Enterprise Structure (PSA).
### 4.2 Personnel Area (PA) and Personnel Subarea (PSA)

SAP splits workforce design across two structures. Use them correctly

| Concept                                          | Belongs to           | Lever   |
| ------------------------------------------------ | -------------------- | ------- |
| Province / legal entity                          | Enterprise Structure | **PA**  |
| Job-category / regulatory zone within a PA       | Enterprise Structure | **PSA** |
| Active employee vs External worker (Contractor)  | Personnel Structure  | **EG**  |
| FT vs PT, Salaried vs Hourly, Union vs Non-union | Personnel Structure  | **ESG** |

1. Enterprise Str: SPRO → Enterprise Structure → Definition → Human Resources Management → **Personnel Areas** *(4-char, e.g. 1510 Ontario, 1520 BC, 1530 Quebec)*

	![](screenshots/20260507122349.png)

	77 Dunsmuir Street, 17th Floor, Vancouver, BC V7Y 1K4
	934 rue des Églises Est, St Faustin, Quebec, J0T 2G0

2. Enterprise Str: SPRO → Enterprise Structure → Definition → Human Resources Management → **Personnel Subareas** *(4-char per PA, e.g. PROF, MANA, EXEC, DIRE)*

	![](screenshots/20260507122625.png)

	![](screenshots/20260507122943.png)

	![](screenshots/20260508183532.png)


3. Enterprise Str: SPRO → Enterprise Structure → Assignment → Human Resources Management → **Assign Personnel Area to Company Code**

	![](screenshots/20260508183859.png)

4. Verify table T500P (Personnel Areas) and T001P (Personnel Subareas)

### 4.3 Personnel Structure — Employee Group (EG) and Employee Subgroup (ESG)

| Worker type        | Pay frequency | Union?      | Through SAP PY?  | EG              | ESG                      |
| ------------------ | ------------- | ----------- | ---------------- | --------------- | ------------------------ |
| Full-time salaried | Monthly       | No          | Yes              | **1** Active EE | **YF** FT Salaried       |
| Part-time salaried | Monthly       | No          | Yes              | **1** Active EE | **YP** PT Salaried       |
| Union worker       | Hourly        | Yes (CBA-A) | Yes              | **1** Active EE | **YU** Union             |
| Contractor         | Hourly        | No          | **No** (AP-paid) | **D** Contractor | **YO** Contractor        |

1. Enterprise Str: SPRO → Enterprise Structure → Definition → Human Resources Management → **Employee Groups** *(1-char, e.g. 1 Active, 2 Retiree, D Contractor)* — table T501

	1 - Employee
	D - Contractor
	
	![](screenshots/20260508184818.png)


2. Enterprise Str: SPRO → Enterprise Structure → Definition → Human Resources Management → **Employee Subgroups** *(2-char, e.g. YF FT Salaried, YP PT Salaried, YU Union, YO Contractor)* — table T503K

| Worker type        | Short  | ESG |
| ------------------ | ------ | --- |
| Full-time salaried | FT Sal | YF  |
| Part-time salaried | PT Sal | YP  |
| Union worker       | Union  | YU  |
| Contractor         | Contr  | YO  |
	![](screenshots/20260508190427.png)

3. Enterprise Str: SPRO → Enterprise Structure → Assignment → Human Resources Management → **Assign Employee Subgroup to Employee Group** *(determines which ESG combos are valid for each EG)* — table T503Z

![](screenshots/20260521164216.png)

![](screenshots/20260521164309.png)

4. Enterprise Str: SPRO → Personnel Management → Personnel Administration → Organizational Data → Organizational Assignment → Employee Groups/Subgroups → **Define Employee Attributes** *(country grouping, payroll status, employment status — defines whether ESG is "active" type)* — view V_503_B, table T503

	![](screenshots/20260521164443.png)

### 4.4 ESG Groupings (these unlock downstream tables)

1. ESG grouping for **Personnel Calculation Rule (PCR)** *(controls payroll computations — e.g. 1 = hourly, 3 = salaried)*

	SPRO → Personnel Management → Personnel Administration → Payroll Data → Basic Pay → **Employee Subgroup Grouping for the Personnel Calculation Rule** — view V_503_D, table T503 (columns PERMT = PCR grouping, PERTV = CAP grouping). Note: PCR and CAP are maintained in the same SPRO node (V_503_D).
	In this step, you can define the groupings for the personnel calculation rules and collective agreement provisions for all your employee groups and subgroups.

	The employee subgroup grouping for the personnel calculation rule is required in Payroll Accounting. The collective agreement provisions grouping is required for indirect valuation of wage types in the Basic Pay infotype (0008).

	The personnel calculation rule allows one wage type to be processed in different ways in payroll accounting.

	**ESG for PCR**
	The value of the standard pay wage type should be used as a basis of valuation for hourly wage earners. The value of the standard pay wage type should be divided by the planned working hours before being used as a basis of valuation for salaried employees.
	
	1 = hourly wage earners
	2 = periodic payments (e.g. monthly wage earners)
	3 = salaried employees
	4 = payments (public service sector Germany)


	**ESG for CAP**
	A standard agreement designates the same pay scale groups and levels for both hourly wage earners and salaried employees; however, the user must still be able to enter hourly or monthly values in the pay scale table.
	
	1 = Industrial workers/hourly wages
	2 = Industrial workers/monthly wages
	3 = Salaried employees
	4 = Non pay scale employees
	5 = Public servant

	![](screenshots/20260522101258.png)

	**Actual KMZP implementation — PCR and CAP groupings:**

	| EG | ESG | Description        | PCR Grouping | CAP Grouping | Notes                                                              |
	| -- | --- | ------------------ | ------------ | ------------ | ------------------------------------------------------------------ |
	| 1  | YF  | Full-time salaried | **3**        | **3**        | Salaried — reads monthly salary from T510                         |
	| 1  | YP  | Part-time salaried | **3**        | **3**        | Salaried — pro-rata handled by IT0008 percentage                  |
	| 1  | YU  | Union              | **1**        | **1**        | Industrial / hourly — reads hourly rate from T510                 |
	| D  | YO  | Contractor         | **9**        | **9**        | Non-standard placeholder — YO is not payroll-relevant; 9 used as a non-conflicting value since standard values are 1–5 |
	

2. ESG grouping for **Work Schedule** *(1 industrial, 2 salaried)*
	SPRO → Time Management → Work Schedules → Work Schedule Rules and Work Schedules → **Define Employee Subgroup Groupings**

![](screenshots/20260522101221.png)

	**Actual KMZP implementation — Work Schedule groupings:**

	| EG | ESG | Description        | WS Grouping | Notes                                       |
	| -- | --- | ------------------ | ----------- | ------------------------------------------- |
	| 1  | YF  | Full-time salaried | **2**       | Salaried work schedule rules                |
	| 1  | YP  | Part-time salaried | **2**       | Salaried work schedule rules                |
	| 1  | YU  | Union              | **1**       | Industrial work schedule rules              |
	| D  | YO  | Contractor         | **9**       | Non-standard placeholder — not in time mgmt |

3. ESG grouping for **Wage Type permissibility** *(controls which WT can be entered for which ESG)*
	SPRO → Personnel Management → Personnel Administration → Payroll Data → Basic Pay → **Employee Subgroups for Primary Wage Types** *(assigns an ESG grouping number — e.g. 1 for hourly, 3 for salaried — which then controls permissibility in IT0008)*
	Then separately: SPRO → Personnel Management → Personnel Administration → Payroll Data → Basic Pay → Wage Types → **Check Wage Type Permissibility for each Infotype** — view V_T512Z, table T512Z *(sets Entry/Required flags per infotype per WT per ESG grouping)*

		
---

5. PSA grouping for **Work Schedule** *(assigns a grouping number to each PSA so all subareas sharing the same work schedule rules are in the same group)*
	SPRO → Time Management → Work Schedules → Personnel Subarea Groupings → **Group Personnel Subareas for Work Schedules** — view V_001P_N, table T001P

	![](screenshots/20260508215641.png)

6. PSA grouping for **Daily Work Schedule** *(groups subareas that share the same Daily Work Schedule definitions)*
	SPRO → Time Management → Work Schedules → Personnel Subarea Groupings → **Group Personnel Subareas for Daily Work Schedules** — table T508Z

7. PSA grouping for **Time Quota Types** *(determines which absence quota types are valid per PSA group)*
	SPRO → Time Management → Time Data Recording and Administration → Managing Time Accounts Using Attendance/Absence Quotas → **Group Personnel Subareas for Time Quota Types**

8. PSA grouping for **Attendance/Absence Types** *(groups PSAs that share the same attendance and absence catalogue)*
	SPRO → Time Management → Time Data Recording and Administration → Absences → **Group Personnel Subareas for Attendances and Absences**

9. PSA grouping for **Substitution Types** *(groups PSAs that share the same substitution type rules)*
	SPRO → Time Management → Time Data Recording and Administration → Substitutions → **Group Personnel Subareas for Substitutions**

10. PSA grouping for **Premium Wage Types** *(groups PSAs for time wage type selection rules — used for shift premiums, overtime premiums)*
	SPRO → Time Management → Personnel Subarea Groupings → **Group Personnel Subareas for Time Wage Types**

---