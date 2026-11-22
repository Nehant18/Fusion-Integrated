# Visitors Hostel Management (VHMS) blueprint

- Module: Visitors Hostel Management System
- Specification set: `VH.zip`, synchronized baseline v1.3 (Problem Statement v1.2; CFR/AFR, UC, BR and WF v1.3)
- Module code: `vh`
- Environment assumption: `module-only`
- Documentation sequence assumption: `09`
- Branch: `module/vh`

## 1. Specification inventory

| Artefact | Inventory |
|---|---:|
| Coherent Functional Requirements (CFR) | 22 |
| Atomic Functional Requirements (AFR) | 115 |
| Active use cases | 22 (UC-VH-001..009, 011..015, 017..025) |
| Active business rules | 38 |
| Basis workflows | BW-VH-01..10 |
| Composite workflows | CW-VH-01..02 |
| Complete workflows | None |

The registry explicitly preserves historical `VH-FR-001..115` as AFR lineage. The active requirements to implement/test are the current `CFR-VH-*` and `AFR-VH-*` identifiers, not the historical `VH-FR-*` aliases. UC-VH-010 and UC-VH-016 are retired; the v1.3 UC synchronization note says UC-VH-010's event behavior is represented by UC-VH-023..025. The v1.3 workflow baseline restores BW-VH-10 (Approved → Room Allotted → Checked In → Checked Out). The BR document has 38 active rules; its legacy/reclassified section must be treated as disposition, not additional active rules.

## 2. Scope and boundaries

Implement booking intake (online and offline), review/routing/decision, modification and cancellation, room allotment and stay lifecycle, visitor categories and effective tariffs, meal records, bill generation and offline settlement records, inventory and replenishment approval, event notifications, reports, and VH-specific operational role reassignment.

Do not implement visitor login, platform-wide account administration, HR master data, procurement/vendor workflows, central stock, online payment gateways, central accounting, or central email infrastructure. Payment and cash settlement are recorded as VHMS operational facts only. Identity references are integer `user_id` values; no cross-module foreign keys or local IAM/role tables.

## 3. Proposed layers and enforcement

| Specification family | Planned enforcement |
|---|---|
| CFR-VH-001..017 | selectors, services, models, API and UI for their corresponding domain capabilities |
| CFR-VH-018..020 | API authentication boundary, permission manifest and module-level tests |
| CFR-VH-021..022 | append-only audit/history records, immutable bill snapshots, effective-dated configuration and database constraints |
| AFR-VH-001..115 | atomic acceptance obligations; map each to a service/domain invariant, selector scope, event/task, API response, or explicit out-of-scope assertion |
| BR-VH-002..050 (38 active rules) | pure domain policies where possible; transaction-safe service enforcement; database constraints for concurrency-sensitive invariants |
| UC-VH-001..025 (22 active UCs) | API/service success and alternative/exception path tests |
| BW-VH-01..10, CW-VH-01..02 | declarative booking/cancellation/stay transition tables and transition tests |

The detailed per-ID mapping is to be expanded alongside the code and spec-test designs; this blueprint does not claim that an identifier is implemented merely because it appears in this table.

## 4. Domain model proposal

All records live in the platform business database. Person references are integer IDs only.

- **Booking**: public reference, indenter user ID, visitor details, purpose, category snapshot/reference, billing responsibility, visit start/end, room/occupant counts, online/offline channel, lifecycle status, created/updated timestamps.
- **BookingDecision**: booking, actor user ID, decision (reject/forward/approve), note, timestamp; append-only.
- **BookingChangeRequest**: booking, requested changes, requester, status, confirmation actor/time; original booking remains auditable.
- **Cancellation**: booking, requester, requested/confirmed timestamps, charge basis, computed amount, confirmation actor; charge is derived from the specified schedule and one-day room rent.
- **Room**: room identifier, suite flag, capacity/occupancy classification and active state. Allotment links approved booking/stay to room and records occupancy.
- **Stay**: booking, allotted room, expected arrival/departure, actual check-in/check-out, no-show/overstay state.
- **VisitorCategory** and **TariffVersion**: institute-defined category and effective-dated category/occupancy tariff; tariff versions are immutable once referenced by bills.
- **MealBooking**: stay, meal date, option (full-day/individual/add-on), meal type/add-on and recorded amount.
- **Bill** and **BillLine**: immutable generated snapshot of tariff, room, meals and cancellation components; billing responsibility and verification facts.
- **Settlement**: bill, offline settlement amount/method/reference, actor and timestamp; no payment gateway.
- **InventoryItem**, **InventoryMovement**, **ThresholdAlert**, **ReplenishmentRequirement**: quantity history and threshold evaluation; approval belongs to VH In-Charge.
- **NotificationRecord**: event, recipient only where specified, template/data snapshot, delivery state and timestamps; transport remains an integration boundary.
- **RoleAssignmentHistory**: VH-specific caretaker/in-charge assignment periods and reassignment audit; does not duplicate IAM roles or grant platform permissions.

Money uses fixed-precision decimals. Historical bills and decision/audit records are append-only. Effective-dated tariffs are superseded by new versions, never overwritten.

## 5. State machines

### Booking

| From | Event/actor | Guard | To |
|---|---|---|---|
| absent | Indenter submits online request | required fields and permitted billing responsibility | Pending |
| absent | Caretaker records offline request | required fields; channel=offline | Pending |
| Pending | Caretaker rejects | — | Rejected |
| Pending | Caretaker forwards | availability review recorded | Forwarded |
| Forwarded | VH In-Charge approves | — | Approved |
| Forwarded | VH In-Charge rejects | — | Rejected |

Offline requests use the same availability review and approval routing.

### Cancellation

Cancellation request → Caretaker review → confirmed cancellation. The charge schedule is exact: at exactly 7 days before arrival, 0%; within 7 days excluding same-day, 25% of one-day room rent; same day/no-show, 50% of one-day room rent. Boundary behavior must be tested at each stated case.

### Stay

Approved → Room Allotted → Checked In → Checked Out (BW-VH-10). Allotment before approval is prohibited. No-show and overstay alerts are event/time evaluations, not invented actor workflows.

## 6. Permission and role assumptions

Permissions use the closed RBAC verbs from the platform guide and the `vh.<singular_resource>.<verb>` form. Proposed resources include booking, room, stay, tariff, meal, bill, settlement, inventory, replenishment, report, and assignment.

- Indenter is a basic-role capability; access to own bookings is scoped by `user_id`.
- Caretaker and VH In-Charge are operational actors in the specification.
- Their exact ERP designation names are not established by the supplied files. For this working blueprint only, assume the institute will map existing designations to those two duties before production. Do not seed invented designations or claim the mapping is confirmed.
- UC-VH-011 (tariff/category configuration) names an unspecified configuration authority and is explicitly provisional.
- UC-VH-019 and CFR-VH-020 constrain VH-specific role reassignment but do not authorize existing operational actors to perform platform-level role administration. The authority to reassign VH roles remains an open decision.

No module-owned Role, Permission, UserRole or ModuleAccess tables will be created.

## 7. Endpoint and client plan

All API routes are under `/api/v1/vh/`; API views call selectors for reads and services for writes.

| Area | Proposed endpoints |
|---|---|
| Bookings | `GET/POST /bookings/`, `GET /bookings/{id}/`, `POST /bookings/{id}/review/`, `POST /bookings/{id}/decision/` |
| Modifications/cancellations | `POST /bookings/{id}/modifications/`, `POST /bookings/{id}/cancellations/`, `POST /cancellations/{id}/confirm/` |
| Rooms/stays | `GET/POST /rooms/`, `POST /bookings/{id}/allotment/`, `POST /stays/{id}/check-in/`, `POST /stays/{id}/check-out/` |
| Categories/tariffs | `GET/POST /categories/`, `GET/POST /tariffs/` |
| Meals/billing | `POST /stays/{id}/meals/`, `POST /bookings/{id}/bills/`, `POST /bills/{id}/settlements/` |
| Inventory | `GET/POST /inventory/`, `POST /inventory/{id}/movements/`, `POST /replenishments/{id}/decision/` |
| Reports/assignments | `GET /reports/{report_type}/`, `GET/POST /assignments/` |

Client pages: Indenter booking list/create/detail; Caretaker booking queue, offline intake, rooms/stays, meals, bills, inventory and operational reports; VH In-Charge approval queue, replenishment approvals and management reports. Record detail should use drawers/modals rather than unregistered detail routes. Every route must be guarded by its corresponding permission and every navigation item must have a route.

Module-only shell handoff entry: `{ id: "vh", label: "Visitors Hostel", icon: "Building2", to: "/vh", section: "Operations" }`. Confirm the final icon/section against the full-platform shell owner before integration.

## 8. Time, money, policy, and contracts

Time-driven no-show, overstay and reminder behavior requires Celery tasks/beat entries as the exact UC/AFR trigger definitions specify. Do not invent reminder lead times or notification recipients. Money is represented by immutable bill snapshots and append-only settlement/adjustment entries. Tariffs are effective-dated immutable versions. No cross-module contract is currently required by the supplied specification; user identity and authorization come from the request principal/session.

## 9. Known gaps and decisions required before production

1. **Caretaker and VH In-Charge designation mapping:** no exact `globals_designation` values are supplied. The role grants cannot be safely finalized until the institute maps them.
2. **Tariff/category configuration authority:** UC-VH-011 is explicitly provisional; do not grant this capability to an assumed actor.
3. **VH role reassignment authority:** the spec bounds the capability but does not identify the authority. Do not grant it to Caretaker or VH In-Charge by inference.
4. **Notification recipient for generic booking-submission notification:** UC-VH-023 notes that the problem statement does not specify a recipient; do not invent one.
5. **Reminder timing and delivery infrastructure:** implement only the triggers/recipient/timing explicitly stated in AFR/UC/BR; central email infrastructure is out of scope.
6. **Room inventory identity/capacity and exact tariff values:** use configured records; do not invent category eligibility, room assignment policy, tariff amounts, or discounts.
7. **External integration and deployment verification:** no local clone, Postgres/Redis/IAM services or test environment were available in this session; runtime gates remain unverified.

## 10. Approval and status

This document is a working blueprint prepared under the user's instruction to make reasonable assumptions. It is not evidence of approval by a second student or institute authority. The unresolved decisions above remain explicit known gaps. No production-readiness or 10/10 claim is made.
