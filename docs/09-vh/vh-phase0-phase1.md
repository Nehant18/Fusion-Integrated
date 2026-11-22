# VH Phase 0 inventory and Phase 1 findings

**Status:** discovery report; not an approved implementation blueprint  
**Module:** Visitors Hostel Management System (VHMS)  
**Module code:** `vh`  
**Specification set:** `VH.zip`, final baselines v1.2/v1.3  
**Working environment:** `module-only`  
**Documentation folder assumption:** `docs/09-vh/` (the assignment did not supply an NN; confirm before treating this path as final)

## 1. Source inventory

The supplied archive contains the frozen problem statement, CFR/AFR registry, use-case specifications and diagram, business-rule specifications, workflow specifications and diagram, and traceability workbook. No implementation starter code is present in the archive.

| Artifact | Version | Inventory |
|---|---:|---|
| Problem Statement | v1.2 | Frozen baseline |
| Requirement Registry | v1.3 | 22 CFRs and 115 AFRs (historical VH-FR-001..115 lineage) |
| Use Cases | v1.3 | 25 indexed IDs; 20 active actor/event use cases, 3 provisional/needs clarification, 2 retired |
| Business Rules | v1.3 | 38 active rules; 13 legacy/reclassified IDs listed |
| Workflows | v1.3 | 10 basis workflows and 2 composite workflows |

The traceability workbook is present. Its mappings must be audited against the individual specifications; it is not treated as authority where a detailed specification or retirement note says otherwise.

### Retired and reclassified identifiers

- Use cases: `UC-VH-010` is superseded by event-triggered `UC-VH-024`; `UC-VH-016` is retired because notification behavior is represented by `UC-VH-023..025`.
- Business rules reclassified/retired from the active BR set: `BR-VH-001`, `020`, `021`, `022`, `026`, `027`, `028`, `038`, `039`, `040`, `042`, `044` (and legacy dispositions are to be preserved exactly as documented). These are not to be cited as active BRs. Some obligations remain in CFR/AFR/UC scope; reclassification does not mean the behavior is necessarily dropped.
- Workflows: legacy `WF-VH-004..010` are retired/superseded/re-evaluated as documented. The active model is `BW-VH-01..10` plus `CW-VH-01..02`. In particular, the approved-stay lifecycle is `BW-VH-10`; notification/alert services are event-triggered use cases, not state-changing workflows.

## 2. Actors and ownership found in the specification

- **Indenter:** submits online requests; views own booking status/history/details and printable information; requests pre-arrival modification/cancellation.
- **Caretaker:** records offline requests; reviews requests, checks availability, rejects or forwards; confirms cancellation; allots rooms; records check-in/check-out; manages meal bookings, billing/settlement records, inventory and operational reports; receives/records operational alerts as specified.
- **VH In-Charge:** decides forwarded booking requests; reviews/approves inventory replenishment requirements; views management reports.
- **Visitors:** no direct VHMS access.
- **Unspecified authority actors:** tariff/category configuration (`UC-VH-011)); VHMS role assignment/reassignment (`UC-VH-019)); confirmed modification outcome (`UC-VH-022)). These are explicitly provisional. Do not silently assign them to Caretaker, VH In-Charge, a generic administrator, or a newly invented designation.

## 3. Phase 1 architecture findings

1. **Bounded context:** VHMS is one bounded context for lodging/boarding operations, booking lifecycle, room/stay records, meal records, billing records, VH inventory, notifications and VH reports. It must not absorb HR, platform account administration, procurement, online payments, accounting ledger or central email infrastructure.
2. **External dependencies:** identity/designations and user directory are owned by IAM/ERP. VHMS should retain integer user identifiers and use existing IAM/directory contracts where needed. No cross-module foreign keys and no direct reads/writes to `fusionlab`.
3. **Identity:** no local User/Role/Permission/ModuleAccess tables. The module code is `vh`; permission names should use `vh.<singular_noun>.<closed_verb>`.
4. **Role mapping:** Caretaker and VH In-Charge are named domain roles, but their exact IAM designation names and existence in the target designation catalogue have not been verified. The three provisional authority actors have no established designation at all. This blocks final role grants and production authorization design.
5. **Scoping:** Indenter read access is restricted to own submitted bookings. Caretaker and VH In-Charge read access is limited to their explicitly assigned operational/reporting responsibilities. Cross-role access must be denied at the selector/queryset layer; foreign records should resolve as 404.
6. **Time-driven behavior:** UC-VH-024 includes no-show and overstay alert evaluation; UC-VH-025 includes inventory threshold alerts. Exact clock schedules, timezone policy, reminder lead times, and no-show/overstay thresholds must be read from the detailed AFR/UC/BR text; no cadence or deadline should be invented.
7. **Money:** billing is a VHMS billing-record concern only. Offline cash settlement is recorded, not processed online. Historical billing records must be immutable; corrections should be represented as auditable adjustments/reversals, not mutation of finalized history.
8. **Policy:** category eligibility, category/occupancy tariff values, room availability/allotment policy, and modification deadlines are not supplied as fixed values. Configuration must be effective-dated and historical bills must retain the tariff snapshot actually used.

## 4. Specification constraints that must remain intact

- Booking paths: online Indenter and offline Caretaker entry; both use the same availability review and approval routing.
- Caretaker may directly reject or forward a pending request. VH In-Charge may approve/reject only forwarded requests.
- Booking states include Pending, Forwarded, Approved, Rejected, Confirmed and operational stay progression as supported by the final UC/WF state tables. Do not collapse distinct lifecycle concepts without mapping the source transitions.
- Room allotment is only after approval; supported occupancy values are Single/Double and room types Suite/Non-suite.
- Cancellation schedule: exactly 7 days before arrival = no charge; within 7 days excluding the separate same-day case = 25% of one-day room rent; same day/no-show = 50% of one-day room rent. Same-day is the more specific case. The baseline does not define a >7-day charge policy.
- Meals: Full-day, Individual meal (Breakfast/Lunch/Dinner), Add-on (Tea/Milk); linked to guest stay duration and reflected in VHMS billing.
- Bills use room rent, meal consumption and applicable cancellation charges; settlement is offline cash recording; billing records are immutable.
- Inventory threshold alert is generated only when quantity is below threshold, not at or above it.
- No visitor login, online payment gateway, accounting ledger posting, procurement execution, vendor/purchase-order workflow, central stock management, HR master-data administration, or central email infrastructure.

## 5. Blocking questions before an implementation blueprint can be approved

1. What exact IAM designation names represent **Caretaker** and **VH In-Charge** in the target ERP, and are both already present in `globals_designation`?
2. Who is the authorized actor for visitor-category/tariff configuration in `UC-VH-011`? The UC marks the actor unspecified/provisional.
3. Who is authorized to perform VHMS role assignment/reassignment in `UC-VH-019`? The problem statement explicitly says this is not a routine Indenter, Caretaker or VH In-Charge operation.
4. Who confirms a booking-modification outcome in `UC-VH-022`? The actor is explicitly unresolved.
5. Does the institute want these unspecified authority functions represented by existing IAM designations, or are they outside this module's current implementation scope pending a later specification revision? Do not invent a generic admin role.
6. What documentation number replaces the working assumption `09` for `docs/09-vh/`?
7. Confirm the intended project owner/approver for the Phase 2 blueprint. The assignment guide says Student 2 approves it in writing; that independent approval has not been supplied here.

## 6. Proposed bounded implementation plan (not yet approved)

Once the blockers above are resolved, the likely module aggregates are:

- Booking and booking audit/transition history.
- Room and room-allotment records.
- Guest stay records (check-in/check-out, no-show/overstay evaluation).
- Visitor category and effective-dated tariff versions.
- Meal booking records.
- Bills, immutable bill lines/tariff snapshots, and offline settlement records.
- VH inventory items, threshold alerts, replenishment requirement records and approval history.
- Notification outbox/event records and report selectors/read models.

This list is a design hypothesis, not a final schema. In particular, tariff configuration, role reassignment and modification confirmation remain blocked by actor/policy ambiguity. The implementation should follow the Placement module's layering and registration conventions only after the full reference files and the rest of the required platform rules have been reviewed.

## 7. Evidence and limitations

- Read the supplied VH archive's extracted text artifacts and assignment PDFs.
- Read the repository's new-module prompt, Assignment 9 guide, spec-test prompt guide, student guide, platform structure, RBAC model, ADR-0013, and representative Placement registry/state-machine/model/API/frontend registration files.
- The fork branch `module/vh` exists and is based on commit `5595be6628b4cdc6f8b47cf74bac267a2470f780`, matching upstream `main` at the time checked.
- Local clone/build/test execution remains unavailable in this session because the container could not resolve `github.com`. No test, migration, lint, frontend build, browser, or deployment gate is claimed as run.


## 8. Traceability workbook audit

The workbook is structurally present and contains the expected Summary, CFR_Summary, AFR_RTM, UC_Index, BR_Index and WF_Transition_Index sheets. However, the mapping columns in the CFR_Summary, AFR_RTM and WF_Transition_Index rows inspected are blank (for example CFR mapped UC/BR/WF columns; AFR mapped UC, exact UC location, BR, WF and coverage columns; WF related CFR/key AFR columns). The UC index's Related CFRs/AFRs column is also blank. Therefore the workbook is not a populated source of cross-layer traceability; mapping must be derived from the actual UC/BR/WF specifications and recorded in the implementation traceability matrix. Do not claim matrix coverage from its existence alone.

The active UC count is 23 non-retired entries, of which 3 are provisional/needs clarification (UC-VH-011, UC-VH-019, UC-VH-022) and 20 are currently implementable without resolving those provisional actors. UC-VH-010 and UC-VH-016 are retired.
