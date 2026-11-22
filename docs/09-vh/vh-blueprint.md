# VH blueprint — draft pending authority decisions

**Status: NOT APPROVED — Phase 2 draft only. No application code may be written from this document until the authority/designation blockers in §9 are resolved and the approver records written approval.**

- Module: Visitors Hostel Management System (VHMS)
- Code/folder/URL segment: `vh`
- Specification baseline: PS v1.2, CFR/AFR v1.3, UC v1.3, BR v1.3, WF v1.3
- Proposed docs folder: `docs/09-vh/` (NN is an assumption pending confirmation)
- Environment: `module-only`

## (a) Traceability approach

The source inventory and retirement register are in [vh-phase0-phase1.md](vh-phase0-phase1.md). The frozen baseline has 22 CFRs, 115 AFRs, 38 active BRs, 23 non-retired UCs (20 active and 3 provisional), and 12 active workflows. UC-VH-011, UC-VH-019 and UC-VH-022 are provisional/needs clarification; they cannot be treated as fully specified operational features until their authority actors are resolved.

| ID | Requirement | Layer | Planned enforcement |
|---|---|---|---|
| CFR-VH-001 / AFR-VH-001..003 | Indenter's own booking status, history, details and print view | selectors/API | Indenter-scoped selector and read DTO |
| CFR-VH-002 / AFR-VH-005..013 | Online booking requisition and validation | domain/services | booking command and validation |
| CFR-VH-003 / AFR-VH-014..020 | Caretaker review, availability, reject/forward, In-Charge decision and notification | domain/services | declarative booking transition table and transactional transition service |
| CFR-VH-004 / AFR-VH-021..025 | Offline booking capture and common approval routing | services | offline booking command using the same transition service |
| CFR-VH-005 / AFR-VH-026, 034 | Pre-arrival modification request and confirmed outcome | services/domain | modification request and confirmation choke points; confirming actor unresolved |
| CFR-VH-006 / AFR-VH-027..033, 035 | Cancellation request, charge schedule, Caretaker confirmation and billing adjustment | domain/services | pure charge calculator and locked cancellation service |
| CFR-VH-007 / AFR-VH-036..040 | Approved-only room allotment, occupancy and room type | domain/services/models | allotment eligibility rule and DB constraints |
| CFR-VH-008 / AFR-VH-041..044 | Check-in/out, no-show and overstay | services/tasks | stay transitions and scheduled evaluator |
| CFR-VH-009 / AFR-VH-045..050, 108 | Visitor category, effective tariffs, tariff selection and historical bill protection | domain/models/services | effective-dated tariff version and bill tariff snapshot; config authority unresolved |
| CFR-VH-010 / AFR-VH-051..058 | Guest meal booking, allowed options, stay linkage and bill inclusion | domain/services | meal validation and booking command |
| CFR-VH-011 / AFR-VH-059..066 | Bill calculation, offline settlement, verification and audit immutability | domain/models/services | bill calculator, immutable bill snapshot and append-only settlement/adjustment |
| CFR-VH-012 / AFR-VH-068..072 | Inventory quantity/threshold and threshold alert | models/services/tasks | inventory update service and below-threshold evaluator |
| Remaining CFR-VH-013..022 and child AFRs | Remaining report, notification, role/access and scope obligations | selectors/services/tasks/registry | Expand each ID into a one-row traceability matrix after final actor mapping; role reassignment stays blocked |
| Active BR-VH rules | Explicit domain invariants, thresholds, calculations and eligibility | domain/services/models | Enforce at domain policy, transaction choke point or DB constraint |
| Active UC-VH-001..025 | Actor/event-triggered use-case flows | services/selectors/tasks/API | One service per use-case family; event UCs dispatched through outbox/task handlers |
| BW-VH-01..10, CW-VH-01..02 | Booking, cancellation and approved-stay transitions | domain/services | Declarative transition table and transactional transition service |

This is a traceability skeleton, not the final one-row-per-id matrix required by Step 3.1. Each live CFR, AFR, BR, UC and WF must have its own row before blueprint approval. Retired IDs must not be cited as live requirements.

## (b) Proposed domain model

All tables live in the module's own Django app/database. All person references are integer IAM `user_id` values; no FK to IAM/ERP or another module. Same-module FKs are allowed.

- **Booking:** public/reference number; source (online/offline); indenter_user_id where applicable; created_by_user_id; visitor details; purpose; category; arrival/departure; room/occupant counts; billing responsibility; status; timestamps.
- **BookingTransition:** booking FK; from/to state; actor_user_id; event; reason/notes; timestamp; immutable audit row.
- **BookingModificationRequest:** booking FK; requested change payload; requested_by_user_id; status; submitted_at; confirmed outcome; confirming authority unresolved.
- **CancellationRequest:** booking FK; requester; requested_at; computed charge snapshot; review/confirmation actor and timestamps; outcome.
- **Room / RoomAllotment:** room identity/type; booking/stay FK; occupancy Single/Double; allotted_by_user_id; allotted_at; release information.
- **GuestStay:** booking FK; expected and actual arrival/departure; check-in/out; no-show/overstay alert markers.
- **VisitorCategory / TariffVersion:** category and effective-dated category/occupancy tariff; immutable version and amount snapshot. Configuration authority unresolved.
- **MealBooking:** stay FK; date; Full-day/Individual/Add-on; Breakfast/Lunch/Dinner or Tea/Milk; amount snapshot where required.
- **Bill / BillLine:** immutable generated bill snapshot for room, meals and cancellation components; billing responsibility and verification facts.
- **BillAdjustment / SettlementRecord:** append-only adjustment and offline cash settlement facts; no online payment processing.
- **InventoryItem / InventoryThresholdEvent:** description/type; quantity; minimum threshold; observed below-threshold event and notification state.
- **ReplenishmentRequirement:** requirement and approval history only to the extent creation/approval actors are explicitly supported.
- **NotificationOutbox/record:** event payload and delivery state using platform outbox infrastructure; central email transport remains external.

Candidate DB invariants: unique booking reference and room identifier; non-negative quantities and monetary values; valid enumerations; effective-date ordering; immutable transition and finalized bill/settlement history. Room overlap and tariff uniqueness constraints require the detailed AFR policy dimensions before implementation. Do not invent room capacity, allocation priority or category eligibility.

## (c) Proposed state machines

Booking routing (must be reconciled against the full UC/WF state-transition tables before code):

| From | Event / actor | To |
|---|---|---|
| none | Indenter submits online request | Pending |
| none | Caretaker records offline request | Pending (source=offline) |
| Pending | Caretaker rejects | Rejected |
| Pending | Caretaker forwards | Forwarded |
| Forwarded | VH In-Charge approves | Approved |
| Forwarded | VH In-Charge rejects | Rejected |
| Approved | Caretaker allots room | Confirmed / room-allotted status as specified by the final source state table |

Offline requests use the same availability review and approval routing. Approved stay progression (BW-VH-10): Approved → Room Allotted → Checked In → Checked Out. Keep booking decision state distinct from stay state. Cancellation and modification are separate request/outcome lifecycles; billing changes only after the confirmed outcome. All state changes go through one locked service with actor, source-state, timestamp and audit recording.

## (d) Permission design — blocked pending IAM role mapping

No local role model. Candidate permissions use the `vh.<singular_resource>.<closed_verb>` convention and must be checked against the closed verb list:

- booking: view/create/review/approve/manage
- room and stay: view/manage
- meal: view/create/manage
- bill: view/create/manage
- settlement: view/create
- inventory: view/manage
- replenishment: view/approve
- report: view
- tariff/category configuration and VH role assignment: not grantable until their actors are resolved.

The exact `MODULE`, `PERMISSIONS`, `SYSTEM_PERMISSIONS`, `SCOPE_PERMISSIONS` and `ROLE_GRANTS` cannot be finalized until Caretaker and VH In-Charge are mapped to actual IAM designations and provisional authority UCs are resolved. These are candidate capabilities only, not a publishable registry.

## (e) Candidate API surface

All routes are under `/api/v1/vh/`; every endpoint requires module grant and permission. Reads use role-aware scoped selectors; Indenter records are own-only. Candidate routes:

| Method | Path | Purpose |
|---|---|---|
| GET/POST | `bookings` | own list / online booking submission |
| GET | `bookings/<id>` | scoped booking detail |
| POST | `bookings/<id>/review` | Caretaker reject/forward |
| POST | `bookings/<id>/decision` | In-Charge approve/reject forwarded request |
| POST | `offline-bookings` | Caretaker records telephonic/walk-in request |
| POST | `bookings/<id>/modifications` | Indenter requests pre-arrival modification |
| POST | `bookings/<id>/cancellations` | Indenter requests cancellation |
| POST | `cancellations/<id>/confirm` | Caretaker reviews/confirms cancellation |
| GET/POST | `rooms`, `bookings/<id>/allotment` | room records and approved booking allotment |
| POST | `stays/<id>/check-in`, `stays/<id>/check-out` | record guest arrival/departure |
| GET/POST | `stays/<id>/meals` | meal bookings |
| GET/POST | `bookings/<id>/bills` | generate/read billing records |
| POST | `bills/<id>/settlements` | record offline cash settlement |
| GET/POST | `inventory` | inventory records and threshold |
| GET | `reports/summary`, `reports/operations` | role-scoped reports |

Record details should use drawers/modals rather than unpaired client routes. Every route needs a permission guard and a matching nav item. Module-only shell handoff candidate: `{ id: "vh", label: "Visitors Hostel", icon: "House", to: "/vh", section: "Operations" }`; verify the icon/section with the shell owner before handoff.

## (f) Contracts

No external domain contract is currently evidenced as necessary. Identity and authorization come from the request principal/session. For names/basic user kinds, use existing plural IAM/directory contracts. Do not query `fusionlab`, join auth tables, or add cross-module FKs. Do not invent a new IAM endpoint.

## (g) Scheduled work

- No-show evaluation for approved stays with no check-in after the specified deadline.
- Overstay evaluation for checked-in stays with no checkout after approved departure.
- Inventory threshold evaluation when quantity changes; scheduled reconciliation only if required by AFR.
- Booking lifecycle and inventory notification events dispatched through platform outbox/Celery.
- Exact cadence, timezone, reminder lead times and recipients must be sourced from detailed AFR/UC/BR text; do not invent them.

## (h) Open questions and approval gates

1. Exact IAM designation strings for Caretaker and VH In-Charge; verify they exist in `globals_designation`.
2. Authority actor for UC-VH-011 category/tariff maintenance.
3. Authority actor for UC-VH-019 VHMS role reassignment and record transfer.
4. Authority actor for UC-VH-022 confirmed modification outcome.
5. Exact creation/trigger/approver semantics for replenishment requirements where not explicit.
6. Exact no-show/overstay thresholds, timezone, cadence and recipients where not specified.
7. Category eligibility and tariff values are institutional inputs; provide effective-dated seed/configuration data or explicitly accept a first-use “policy not configured” response.
8. Room availability/allotment policy and overlap behavior must be confirmed from source; no allocation policy is invented.
9. Confirm documentation sequence number `09`.
10. Obtain written Student 2 approval of this blueprint and record it before Phase 3.

**No implementation code is authorized by this draft.**
