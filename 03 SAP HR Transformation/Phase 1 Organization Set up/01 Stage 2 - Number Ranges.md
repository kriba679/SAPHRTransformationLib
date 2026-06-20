1. PA04: **Number range for Personnel Number (PERNR)** — internal & external ranges, e.g. 01 = 00000001–99999999
2. NUMKR feature controls which range is picked at hiring (configured in Phase 4 below)
3. SNRO → object **PERSNO**: Personnel Number range
4. SNRO → object **HRADATA**: HR object number ranges
5. OONR (or `RP_OBJEC` in SPRO): **OM Object Number Ranges** — separate range per object type:
  - O — Organizational Unit (e.g. 50008000–50008999)
  - S — Position (e.g. 50015000–50015999)
  - C — Job (e.g. 50029000–50072999)
  - P — Person (mapped to PERNR)
  - K — Cost Center (external)


--

## 0. Design principles (read first)

There are **two unrelated number-range systems** to set up in Phase 2. Don't conflate them:

| Number-range system | What it numbers | Where it's maintained | Used by |
|---|---|---|---|
| **PERNR (Personnel Number) range** | Employee Personnel Numbers (8-digit) | PA04 (or SNRO object PERSNO) | PA40 hire action |
| **OM object number ranges** | Org Units (O), Positions (S), Jobs (C), Persons (P), Cost Centers (K), etc. | OONR | PPOCE / PPOME / PO13 / PO03 |

The two are linked only through the **Person (P) object** in OM, which is keyed externally by PERNR.

**Decisions for this design:**
1. **Single internal PERNR range** for all four worker types (FT / PT / Union / Contractor). Reason: simplicity and minimum NUMKR maintenance. Reporting still segments worker types via EG/ESG, not PERNR ranges.
2. **Internal numbering for O / S / C** (let SAP assign), **external for P** (inherits from PERNR), **external for K** (inherits from CO cost center key).

---

## 1. PERNR Number Range — PA04

`SPRO →` Personnel Management → Personnel Administration → Basic Settings → **Maintain Number Range Intervals for Personnel Numbers**  
`T-code:` PA04 (shortcut to the same screen — also reachable via `SNRO → object PERSNO`)  
`Number range object:` PERSNO  
`Table:` NRIV (number range intervals — generic across all SAP modules)

### Recipe — single internal range

In the **Maintain Number Range Intervals** screen → click *Intervals* (pencil icon) → **Insert Line**:

| No (range) | Year | From number | To number | Current number | Ext (external)? |
|---|---|---|---|---|---|
| **01** | 9999 | 00000001 | 99999999 | 00000000 | *unchecked* (= internal) |

**Notes:**
- `Year = 9999` means "valid forever" — standard for PERNR. Don't use a real year (you'd have to re-maintain it annually).
- `Ext` checkbox **unchecked** = internal (SAP auto-assigns the next number on PA40 hire). Checked = external (admin types in the PERNR).
- `Current number = 00000000` is fine — SAP will increment from `From number` automatically on first hire.

Save → choose your customizing transport request.

### Verification
- SE16N → table **NRIV** → object = `PERSNO`, sub-object = blank, no = 01 → confirm range exists with from/to as set.
- PA04 → click *Status* → confirms `Current Number = 00000000` (or last assigned).

---

![](20260510175251.png)


## 2. NUMKR Feature — PE03

NUMKR is the decision tree PA40 consults at hire time. It returns a **2-character number range key** (like `01`) that points to the PERNR ranges defined in step 1.

`SPRO →` Personnel Management → Personnel Administration → Basic Settings → **Determine defaults for number ranges**  
`T-code:` PE03 → enter feature `NUMKR` → click *Change*  
`Underlying:` Feature NUMKR (table T549B)

### Decision tree — single range for all worker types

Since FT / PT / Union / Contractor all draw from range 01:

```
NUMKR
   &NUMKR=01.
```

That single line returns "01" regardless of CC / PA / EG / ESG.

### If you ever fork later (reference for future)

If contractors should pull from a separate range (e.g. range 02), the structure would become:

```
NUMKR
   D PERSG               (decide on Employee Group)
     1                   Active Employee
       &NUMKR=01.
     4                   External / Contractor
       &NUMKR=02.
   &NUMKR=01.            (default fallback)
```

Or by Personnel Area:

```
NUMKR
   D WERKS               (decide on Personnel Area)
     1010
       &NUMKR=01.
     1020
       &NUMKR=02.
   &NUMKR=01.
```

But for now: **single line `&NUMKR=01.`**.

![](20260510180006.png)
### Activate the feature

After saving, click the **Activate** button (yellow lightning icon at top). A feature is useless until activated — easy to miss.

### Verification
- PE03 → NUMKR → *Display* → status icon must be **green** (active).
- The decision tree compiles without errors (red = error, yellow = inactive, green = active).

---

## 3. OM Object Number Ranges — OONR

OM objects need their own number ranges, separate from PERNR. PPOCE / PPOME / PO13 / PO03 will refuse to create objects until these exist.

`SPRO →` Personnel Management → Organizational Management → Basic Settings → **Maintain Number Ranges**  
`T-code:` OONR  
`Tables:` T77NR1, NRIV  
`Number range object:` RP_PLAN

### Subgroup naming convention

In OONR, ranges are defined per **subgroup** (4-character key):
- First two characters = **Plan version** (usually `01` Current Plan)
- Last character = **Object type abbreviation** (single char)

Common object types you need:

| Object type | Meaning | Subgroup (Plan 01) |
|---|---|---|
| **O** | Organizational Unit | `01O` |
| **S** | Position | `01S` |
| **C** | Job (catalog) | `01C` |
| **P** | Person (linked to PERNR — typically external) | `01P` |
| **T** | Task (only if used) | `01T` |
| **A** | Work Center / Workplace | `01A` |
| **K** | Cost Center *(usually external — comes from CO)* | `01K` |

### Recipe — internal for O/S/C/T/A, external for P/K

| Subgroup          | NR  | Year | From     | To       | Ext?              | Why                                                            |
| ----------------- | --- | ---- | -------- | -------- | ----------------- | -------------------------------------------------------------- |
| 01O (Org Unit)    | 01  | 9999 | 50000000 | 59999999 | **IN** (internal) | 10M-object pool — generous                                     |
| 01S (Position)    | 02  | 9999 | 60000000 | 69999999 | **IN**            | 10M positions                                                  |
| 01C (Job)         | 03  | 9999 | 70000000 | 79999999 | **IN**            | 10M jobs                                                       |
| 01P (Person)      | 04  | 9999 | 00000001 | 99999999 | **EX** (external) | Person object is keyed by PERNR; no separate number assignment |
| 01K (Cost Center) | 05  | 9999 | —        | —        | **EX**            | Cost Center comes from CO module via KS01                      |

**Important nuances:**
- Use **IN** (internal) for O, S, C, T, A — let SAP assign numbers automatically.
- Use **EX** (external) for P (Person) — the number IS the PERNR, assigned during PA40, then the OM Person object inherits it.
- Use **EX** for K (Cost Center) — the number IS the CO cost center key, set at CO via KS01.

### How to enter on the OONR screen

1. OONR → **Subgroup Maintenance** view opens
2. *Maintain* → table opens with columns *Subgroup*, *No*, *Year*, *From number*, *To number*, *Current*, *Ext*
3. *Insert Line* — one per subgroup above
4. For Person (01P) and Cost Center (01K): tick the **Ext** checkbox (= `EX`)
5. Save → transport

![](20260510182852.png)

![](20260510182915.png)

![](20260510182942.png)

![](20260510183011.png)

![](20260510183025.png)

![](20260510183037.png)
### Verification
- SE16N → table **NRIV** → object = `RP_PLAN`, sub-object = `01O` → confirm range exists.
- Repeat for `01S`, `01C`, `01P`, `01K`.
- OONR → click *Status* per subgroup — shows `Current Number` (will be 0 until you create the first object).

---

## 4. Optional but recommended — HRADATA range

Some HR adapter / reporting programs (BAdIs, replication to/from SF EC) require this object range. Quick to set; no harm in having it.

`SNRO →` object **HRADATA** → create range 01, internal, 00000001–99999999, year 9999.

Other auxiliary ranges to consider later (not blocking for Phase 2):
- `PD_NUMBER` — used internally by some PD reports
- `BENEFIT` — only if Benefits module is in scope

---

## 5. SPRO execution sequence

```
Phase 2.A — PERNR Range
1. SPRO → PM → PA → Basic Settings → Maintain Number Range Intervals
   for Personnel Numbers   (T-code: PA04)                            → NRIV (PERSNO)
   → Range 01: 00000001-99999999, internal, year 9999

Phase 2.B — NUMKR Feature
2. SPRO → PM → PA → Basic Settings → Determine defaults for number
   ranges    (T-code: PE03 → feature NUMKR)                          → T549B
   → Decision tree: &NUMKR=01.
   → Activate feature (lightning icon)

Phase 2.C — OM Object Number Ranges
3. SPRO → PM → Organizational Management → Basic Settings →
   Maintain Number Ranges    (T-code: OONR)                          → NRIV (RP_PLAN)
   → 01O: 50000000-59999999  internal
   → 01S: 60000000-69999999  internal
   → 01C: 70000000-79999999  internal
   → 01P: 00000001-99999999  external
   → 01K: external

Phase 2.D — Auxiliary (optional)
4. SNRO → object HRADATA → range 01, internal, 00000001-99999999
```

---

## 6. Validation — proof Phase 2 works

You don't need a full hire to verify. Two quick smoke tests:

| Test | Steps | Expected |
|---|---|---|
| **PERNR allocation** | PA40 → date / action type Hire / leave Personnel Number blank → Enter | SAP auto-fills PERNR `00000001` (or next available). Press F12 to cancel — don't save the hire. |
| **OM allocation** | PPOCE → create root Org Unit → leave Object ID blank → save (give it a temp name like ZTEST) | SAP auto-assigns Org ID `50000000` (or next from 01O). Delete dummy via PP01 afterward. |

Common failure modes if either test fails:
- NUMKR feature not **activated** (status not green) → PE03 → NUMKR → click Activate
- OONR range marked external when it should be internal → re-edit the subgroup
- Customizing not saved to a valid transport → check transport request status in SE10

---

## 7. Decisions logged (so future-you knows why)

1. **Single PERNR range (01) for all worker types.** Reason: simplest NUMKR feature; reporting can already segment by EG/ESG. Multi-range adds maintenance cost without operational benefit at this scale.
2. **Internal numbering for O / S / C** (10M each). Reason: standard SAP recommendation; avoids manual ID conflicts. From-number starting at 50/60/70M is to leave room below for special-purpose objects.
3. **External numbering for P** (Person). Reason: the OM Person object inherits the PERNR; it's not a separate number system.
4. **Year = 9999** on all ranges. Reason: standard for HR — PERNR/object IDs are not year-segmented.
5. **HRADATA range created** as auxiliary. Reason: cheap to set now; saves a service request later if SF EC replication or any HR API gets activated.

---

## 8. What comes after Phase 2

| Phase | Title | Why next |
|---|---|---|
| **3** | Payroll Area & Control Record (PA03) | Required *before* the ABKRS feature (Phase 4) can default a payroll area on IT0001 |
| **4** | Features (PE03) — ABKRS, PINCH, SCHKZ, TARIF, LGMST, TMSTA | Defaults during hire — depends on Phases 1, 2, 3 |
| 5 | Pay Scale Structure | Depends on CAP grouping (✓ Phase 1) and Phase 4 features (TARIF, LGMST) |

Recommended next: **Phase 3 (Payroll Area & Control Record)** — small phase (~20 min) and Phase 4 features become productive once Payroll Areas exist.

---

## 9. Sources & References

### SAP Community

- [Maintaining Number Range for Personnel Numbers (SAP Community blog)](https://community.sap.com/t5/enterprise-resource-planning-blog-posts-by-members/maintaining-number-range-for-personnel-numbers/ba-p/12995870)
- [Personnel Administration Number Range PA04 (SAP Community Q&A)](https://community.sap.com/t5/enterprise-resource-planning-q-a/personnel-administration-number-range-pa04/qaq-p/9845063)
- [Number Ranges in Personnel Planning (SAP Community blog)](https://community.sap.com/t5/enterprise-resource-planning-blog-posts-by-members/number-ranges-in-personnel-planning/ba-p/13230068)
- [Organizational Management — Maintain Number Ranges (SAP Community Q&A)](https://community.sap.com/t5/enterprise-resource-planning-q-a/organizational-management-maintain-number-ranges/qaq-p/8555611)
- [Internal vs External Number Range in OM (SAP Community Q&A)](https://community.sap.com/t5/enterprise-resource-planning-q-a/internal-external-number-range-in-om/qaq-p/8284327)
- [Guide for numbering of HCM business partners (SAP Community blog)](https://community.sap.com/t5/enterprise-resource-planning-blog-posts-by-sap/guide-for-numbering-of-hcm-business-partners-what-you-need-to-know/ba-p/13548975)
- [The 5 Commonly Used SAP HCM Features (NUMKR / ABKRS / PINCH / SCHKZ / TARIF / LGMST)](https://community.sap.com/t5/enterprise-resource-planning-blog-posts-by-members/the-5-commonly-used-sap-hcm-features/ba-p/13441668)

### SAP Learning

- [Master Data Configuration in SAP HCM for S/4HANA (course)](https://learning.sap.com/courses/master-data-configuration-in-sap-hcm-for-s-4hana)

### Project notes

- `Stage 1 - SAP ECC HCM Setup Checklist.md` (Phase 2 outline)
- `Notes/3 Features.md` (NUMKR / ABKRS feature mechanics)
