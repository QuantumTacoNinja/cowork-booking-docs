# ADR-0003 Database per service

## Status

Superseded by ADR-0022 (accepted 2026-10-01)

## Context

Example: BK-7KQ2M9 is three rows in three databases.

- Purchase `bookings`: agreed_price_satang 45000, coverage pay, status confirmed.
- Payment `payment_sessions`: booking_reference BK-7KQ2M9, amount_satang 45000, payment_status paid.
- Access `grants`: booking_reference BK-7KQ2M9, ticket code H7K3-9QXA (stored as H7K39QXA), state issued.

They share only the text BK-7KQ2M9.

The seed has one database and one `bookings` row with price, paid flag and card_last4 (source inspected at 5a1cf3d). [extraction site]: "Booking record": "price · paid · card_last4", and "Moving the helpers into new files would leave this shared-data dependency." The target: "Store payments in their own table." and "Payments writes its record; Purchase owns the reservation." [extraction site]. [contexts site]: "Shared references connect the models. They do not make them one shared object."

Seed flaw F5 shows the cost of one row: cancel runs `DELETE FROM bookings` (app.py:917-930), and the paid amount and card_last4 go with it, because the payment record is the booking row (source inspected at 5a1cf3d).

## Decision

- Give each service its own postgres:16 database. Only that service creates, reads and writes it.
  - Purchase: members, spaces, bookings, test_clock.
  - Payment: payment_sessions, payment_attempts, refunds, test_clock.
  - Access: grants, scans, test_clock.
- Never connect to another service's database. Move data only through the HTTP APIs (ADR-0004).
- Carry references, never foreign keys: `booking_reference`, `payment_session_id`, `grant_id`, `member_ref`.
- Store what you need at call time. Payment stores amount_satang as sent (PMT-R04). Access stores space_id, space_name, valid_from and valid_until as sent (AXS-R03).
- Publish database ports 5441, 5442 and 5443 only in each repo's own compose file. The integration stack runs three databases with no published ports.

## Consequences

- Good: a cancel cannot erase revenue. Payment keeps the paid session and its refunds whatever Purchase does (PMT-R16, PMT-R17).
- Good: each service owns its schema, deploys it and rolls it back alone (ADR-0011).
- Good: Access builds the kiosk room list from its own grants, with no call to Purchase (AXS-R11).
- Bad: no joins across contexts. The dashboard shows bookings only; money totals live on Payment's operator page (PUR-R34, PMT-R17). The Operator reads two pages.
- Bad: no transaction spans two databases. Purchase reconciles (ADR-0007) and stores pending states (PUR-R32).
- Bad: copies go stale. A renamed space keeps its old name on grants already issued.
- Bad: three databases to run and back up.

## Alternatives considered

- **One database, one schema per service.** Cheaper to run. Rejected: nothing stops a cross-schema join, and the shared-data dependency the course warns about returns.
- **One database, prefixed tables** (the "Separate ownership, still one monolith" step). The course's intermediate step; we skip it (ADR-0006).
- **Read replicas or views across services.** Couples schemas and deploys.

## Rules and decisions

- Rules: PUR-R29, PUR-R32, PUR-R34, PUR-R35, PMT-R04, PMT-R16, PMT-R17, PMT-R18, AXS-R03, AXS-R04, AXS-R11, PUR-R38, PMT-R20, AXS-R19.
- Decisions: D13, D22, D24, D27, D28.
- Related: ADR-0001, ADR-0004, ADR-0007.

## Sources

- course site [extraction site] fetched 2026-09-30: E08, E09, E22, E38.
- course site [contexts site] fetched 2026-09-30: C12.
- Spacey main 5a1cf3d: one database, `bookings` row with paid and card_last4; seed flaw F5, app.py:917-930 (source inspected at 5a1cf3d).
- Project team (D24, D28).
