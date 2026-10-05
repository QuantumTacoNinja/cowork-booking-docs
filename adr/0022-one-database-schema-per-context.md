# ADR-0022 One database, one schema and role per context

## Status

Accepted, 2026-10-05. Supersedes ADR-0003.

## Context

Example: BK-7KQ2M9 is three rows in **one** PostgreSQL database, each in the schema of the context that owns it.

- `purchase.bookings`: id 42, reference BK-7KQ2M9, agreed_price_satang 45000, status confirmed.
- `payment.payment_sessions`: booking_id 42, amount_satang 45000, payment_status paid.
- `access.grants`: booking_id 42, ticket code H7K3-9QXA, state issued.

ADR-0003 gave each service its own postgres:16 server. All three run the same engine and version, so the split buys no technical capability. What it costs is listed in ADR-0003 itself: no joins across contexts, no transaction across two databases, three databases to run and back up, and references that the database cannot check.

ADR-0003 rejected "one database, one schema per service" because "nothing stops a cross-schema join". PostgreSQL roles do stop it: a role without `USAGE` or `SELECT` on another schema cannot read it, and the database itself refuses the query.

The course's intermediate step is "Separate ownership, still one monolith" [extraction site]. The Spacey application runs one PostgreSQL today (Spacey: source inspected at b4fec38, `deploy/startup-app.nomad.hcl`, one `DATABASE_URL`).

## Decision

- Run **one** PostgreSQL 16 database.
- Give each context its own **schema** and **login role**. Each role owns, and may read and write, only its own schema:
  - `purchase`: members, spaces, bookings, test_clock.
  - `payment`: payment_sessions, payment_attempts, refunds, test_clock.
  - `access`: grants, scans, test_clock.
- No role is granted `SELECT` on another context's schema. Data still moves only through the HTTP APIs (ADR-0004, unchanged).
- Link contexts by **`booking_id`**, the integer `purchase.bookings.id`, with a real foreign key, **`ON DELETE RESTRICT`**, never `CASCADE`:

  ```sql
  GRANT USAGE ON SCHEMA purchase TO access_role, payment_role;
  GRANT REFERENCES (id) ON purchase.bookings TO access_role, payment_role;  -- FK only, no SELECT
  -- in access:   booking_id INTEGER NOT NULL UNIQUE REFERENCES purchase.bookings (id) ON DELETE RESTRICT
  -- in payment:  booking_id INTEGER NOT NULL REFERENCES purchase.bookings (id) ON DELETE RESTRICT
  ```

- `booking_reference` (BK-, D22) stays as Purchase's public, unguessable reference for URLs and people. It is no longer the join key.
- Each service keeps its own `DATABASE_URL`, now naming its own role on the shared database.
- **Scaling path, not now:** one primary with streaming **read replicas** (master/slave). Writes and read-your-own-write paths (booking, payment, grant, cancel) stay on the primary. Lag-tolerant reads (the Grafana report, the operator dashboard) may move to a replica. A replica copies every schema, so replicas add read capacity and availability. They do not separate ownership; roles do.

## Consequences

- Good: the database enforces ownership (roles) **and** integrity (foreign keys). A `booking_id` that does not exist is refused, and so is an orphaned grant or payment session.
- Good: `ON DELETE RESTRICT` makes a hard delete of a booking with a payment or grant fail, so cancel must be a state change. This is what PUR-R28 and AXS-R17 already require, and it fixes seed flaw F5 at the database level, not only by convention.
- Good: one database to run, back up and restore. A cross-context read-only view for reporting (ADR-0001 style, its own read-only role) becomes possible without copying data.
- Bad: the contexts' schemas are coupled through the foreign keys. A Purchase migration that touches `bookings.id` needs Access and Payment to agree, and Purchase must exist before the other schemas are created.
- Bad: services still cannot share a transaction, because each uses its own connection and role. Reconciliation (ADR-0007) and pending states (PUR-R32) stay as they are.
- Bad: one database is one failure domain: if it is down, all three services are down. This was already true for the member journey, which needs Purchase.
- Bad: independent rollback (ADR-0011) still holds for code, but schema changes must stay additive in every schema, not just in one service's database.
- Follow-up, not done by this ADR: `POST /grants` and `POST /payment-sessions` must carry `booking_id` (a contract-v2 change, ADR-0005), and the compose files must run one postgres:16 with three roles. Until then, the code still follows ADR-0003.

## Alternatives considered

- **Database per service (ADR-0003).** Superseded: same engine three times, no integrity across contexts, three databases to operate.
- **One database, one schema per context, no foreign keys.** The ownership is the same, but nothing checks that a `booking_id` exists. Rejected: the foreign key is the main gain of sharing a database.
- **A hub on `bookings`** (`bookings.grant_id`, `bookings.payment_session_id`). Rejected: Purchase's table would hold other contexts' ids, written by them or written twice, and it cannot hold 1:N records (attempts, refunds).
- **Read replicas per context.** Rejected as an ownership tool: every replica holds every schema. Kept as the scaling path above.

## Rules and decisions

- Rules: PUR-R28, PUR-R29, PUR-R32, PUR-R35, PMT-R04, PMT-R16, PMT-R18, AXS-R01, AXS-R03, AXS-R04, AXS-R17.
- Decisions: D13, D22, D28.
- Supersedes: ADR-0003. Related: ADR-0001, ADR-0004, ADR-0005, ADR-0007, ADR-0011.

## Sources

- course site [extraction site] fetched 2026-09-30: E04, E10, E13.
- Spacey main b4fec38: one PostgreSQL, one `DATABASE_URL` (source inspected at b4fec38).
- Access team (spacey Access context), 2026-10-05.
