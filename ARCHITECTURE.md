# Cowork Booking architecture

Three services, three repos, one database with a schema per service (ADR-0022). Only Purchase calls the other two. This page says what runs where, who owns which data, how the services talk, and how they stay consistent without a shared transaction. Rules are in RULES.md, decisions D1-D28 in DECISIONS.md, and the reasons behind each structural choice in `adr/` (ADR-0001 to ADR-0020).

## Overview

![Context map](diagrams/context-map.svg)

Example: at 2026-10-05 10:00 Member A books Meeting Room A for 2026-10-07 09:00-10:30. Purchase prices it at THB 450.00 and holds the slot as BK-7KQ2M9. Payment collects THB 450.00 on its hosted page. Purchase reads the outcome, confirms the booking and asks Access for a grant. Access issues ticket code H7K3-9QXA, which Staff scan at the kiosk on the day.

- **Purchase** "Owns price and coverage" [extraction site]: members, spaces, availability, bookings, cancellation, the dashboard (PUR-T33).
- **Payment** "Owns payment outcomes" [extraction site]: payment sessions, the hosted mock checkout, attempts, refunds (PMT-T01).
- **Access** "Owns grants and issuance" [extraction site]: grants, ticket codes, the e-ticket, the check-in kiosk (AXS-T01).

"Shared references connect the models. They do not make them one shared object." [contexts site]. The split is the course exercise: "Three services are the course exercise, not a claim that every growing business should split its application; students compare the costs and benefits with alternative boundaries and modules within a monolith." [syllabus site]. ADR-0001 records the split and compares it with modules in one monolith. Account data lives on Member in Purchase; there is no Identity service (ADR-0002).

## Services and repos

| Service | Repo | Owns | Port | Image | Database |
|---|---|---|---|---|---|
| Purchase | cowork-booking-purchase | members, spaces, availability grid, bookings, price, coverage, cancellation, dashboard | 8001 | `cowork-booking-purchase:v1.0.0` | own postgres:16, database `purchase` |
| Payment | cowork-booking-payment | payment sessions, hosted checkout, attempts, refunds, operator money page | 8002 | `cowork-booking-payment:v1.0.0` | own postgres:16, database `payment` |
| Access | cowork-booking-access | grants, ticket codes, e-ticket, check-in kiosk, scan log | 8003 | `cowork-booking-access:v1.0.0` | own postgres:16, database `access` |
| Docs | cowork-booking-docs | rules, decisions, PRD, architecture, ADRs, diagrams, contracts index, integration stack | none | none | none |

- Every app container listens on 8000; compose maps it to 8001, 8002 or 8003.
- Each service repo is seeded by copy-and-prune from the seed at 5a1cf3d, straight to three repos (ADR-0006).
- Each contract has one authoritative copy beside its provider's code: Payment holds purchase-payment, Access holds purchase-access, Purchase holds its own public contract as `CONTRACT.md` and `openapi.yaml`. The docs repo keeps links only (ADR-0005).

## Containers and deployment

![Containers](diagrams/containers.svg)

Run each service alone from its own repo, or all three from the docs repo:

- **Per repo** `compose.yaml`: `app` (build `.`, port 8001, 8002 or 8003) and `db` (postgres:16, `pg_isready` healthcheck, named volume). Only this file publishes a DB port (5441, 5442 or 5443), so pytest on the host can reach it. Every port is published on 127.0.0.1 only (`"127.0.0.1:5441:5432"`, `"127.0.0.1:8001:8000"`), and POSTGRES_PASSWORD comes from the repo's `.env`, never user=password (ADR-0011; section "Environment variables"). Calls to a service that is not running follow the failure rules below.
- **Integration** `cowork-booking-docs/integration/compose.yaml`: builds the three services from the sibling folders, each with its own postgres:16. No DB port is published. `compose.e2e.yaml` adds `TEST_CLOCK_ENABLED=true` (D27) and the top-level `name: cowork-e2e`, so e2e data lives in its own volumes, and is used only by the e2e run (see "The e2e harness").

```yaml
# integration/compose.yaml (one of three pairs); values come from integration/.env
services:
  purchase-db:
    image: postgres:16
    environment: {POSTGRES_USER: purchase, POSTGRES_PASSWORD: "${PURCHASE_DB_PASSWORD}", POSTGRES_DB: purchase}
    healthcheck: {test: ["CMD-SHELL", "pg_isready -U purchase"], interval: 2s, retries: 30}
  purchase:
    build: ../../cowork-booking-purchase
    ports: ["127.0.0.1:8001:8000"]
    environment:   # only Purchase's own variables; no shared env_file
      DATABASE_URL: "postgresql://purchase:${PURCHASE_DB_PASSWORD}@purchase-db:5432/purchase"
      SECRET_KEY: "${PURCHASE_SECRET_KEY}"
      APP_REVISION: "${APP_REVISION}"
      PUBLIC_URL: http://localhost:8001
      PAYMENT_INTERNAL_URL: http://payment:8000
      PAYMENT_PUBLIC_URL: http://localhost:8002
      PAYMENT_API_TOKEN: "${PAYMENT_API_TOKEN}"
      ACCESS_INTERNAL_URL: http://access:8000
      ACCESS_PUBLIC_URL: http://localhost:8003
      ACCESS_API_TOKEN: "${ACCESS_API_TOKEN}"
      OPERATOR_EMAIL: "${OPERATOR_EMAIL}"
    depends_on: {purchase-db: {condition: service_healthy}}
```

From integration/.env, Payment gets only `PAYMENT_SECRET_KEY` (as SECRET_KEY), `PAYMENT_API_TOKEN` and `OPERATOR_PASSWORD`, and Access only `ACCESS_SECRET_KEY`, `ACCESS_API_TOKEN` and `STAFF_PASSWORD`; each also gets `APP_REVISION`, its own `DATABASE_URL` and its `PUBLIC_URL` (`http://localhost:8002` or `http://localhost:8003`), from which Payment builds the session url and Access the ticket_url. So a compromised Access container holds no Payment token, and a cookie signed by one service never verifies at another (ADR-0019). `integration/.env.example` lists every variable with an empty value.

| integration/.env | Becomes, in the containers | Purpose |
|---|---|---|
| PURCHASE_DB_PASSWORD, PAYMENT_DB_PASSWORD, ACCESS_DB_PASSWORD | POSTGRES_PASSWORD of that database container, and the password inside that service's DATABASE_URL | one password per database |
| PURCHASE_SECRET_KEY, PAYMENT_SECRET_KEY, ACCESS_SECRET_KEY | SECRET_KEY of that service only | a different key per service |
| PAYMENT_API_TOKEN, ACCESS_API_TOKEN | the same name in Purchase and in the provider | bearer tokens |
| OPERATOR_EMAIL (Purchase), OPERATOR_PASSWORD (Payment), STAFF_PASSWORD (Access) | the same name, one container each | operator and kiosk sign-in |
| APP_REVISION | APP_REVISION in all three | shown by GET /health |
| E2E_OPERATOR_PASSWORD | no container: read only by the e2e suite | the Purchase password of the OPERATOR_EMAIL account on the e2e stack; empty in .env.example |

Run all three (no test clock):

```bash
# from the mother folder; frees ports 8001-8003
for s in purchase payment access; do (cd cowork-booking-$s && docker compose down); done
cd cowork-booking-docs/integration && docker compose up -d --build --wait
```

The e2e command, which adds `compose.e2e.yaml` and so opens POST /_test/clock on all three services, is under "The e2e harness"; never use it for a normal run. The integration README (M6) keeps the same split.

- Server-to-server calls use the internal URL (`http://payment:8000`); the browser uses the public one (`http://localhost:8002/pay/ps_...`).
- Browsers share cookies across ports on one host, so each service has its own cookie name (D15). On localhost every service, and any other page served from localhost on any port, receives all three cookies and is same-site for SameSite: run nothing else on localhost while using the stack. A real deployment gives each service its own host name and leaves the cookie domain unset, so each cookie is host-only.
- The apps do not wait for each other: Purchase calls Payment and Access only when a request needs them.

## Data ownership

![Data model](diagrams/data-model.svg)

A service reads and writes only its own database (ADR-0003; PUR-R35, PMT-R18, AXS-R04). Foreign keys exist only inside one database. A value from another service is a plain column, never a foreign key. Column names below are a guide for M5; the contracts pin only the JSON field names. Business times (bookings.created_at, hold_expires_at, refund_requested_at, bookings.cancelled_at, payment_sessions.created_at, paid_at, attempted_at, refunds.created_at, revoked_at, scanned_at) are written from `clock.now()` in the transaction that records the fact, never by a column default, because rules and the weekly sums read them under the test clock (D27).

| Service | Table | Main columns |
|---|---|---|
| Purchase | members | id, email (unique, lower-cased), display_name, password_hash, is_operator, plan_active |
| Purchase | spaces | id, name, capacity, hourly_rate_satang (BIGINT), archived_at |
| Purchase | bookings | reference (unique, BK-), member_id, space_id, start_time, end_time, blocks, party_size, note, agreed_price_satang (BIGINT), coverage, status, created_at (timestamptz, written in the insert), cancelled_at (timestamptz, written in the cancel transaction; shown to an operator as "Cancelled 2026-10-05 11:00"), hold_expires_at, payment_session_id, payment_outcome, cancel_reason, refund_amount_satang, refund_reason, refund_status, refund_attempt, refund_requested_at (timestamptz, set when a refund above 0 is recorded: cancel step 1, amount_mismatch, slot_unavailable; shown in the all-bookings list as "Refund requested"), grant_id, grant_status, ticket_url (PUR-T17, PUR-T34). The booking JSON field `payment_status` is this `payment_outcome` (not_required, unpaid, paid); it is not Payment's own payment_status (PMT-T04), which has no not_required |
| Payment | payment_sessions | id (ps_), booking_reference (unique), amount_satang (BIGINT), currency, description, success_url, cancel_url, expires_at, status, payment_status, created_at (timestamptz, written in the create transaction; the order of unpaid sessions on /operator, PMT-R17), paid_at (timestamptz, set in the pay transaction, PMT-R12) |
| Payment | payment_attempts | id (pa_), session_id, card_brand, card_last4, outcome, decline_code, attempted_at (timestamptz) (PMT-R13) |
| Payment | refunds | id (re_), payment_session_id, booking_reference, amount_satang, reason, attempt, status, created_at (timestamptz, a business time); unique (payment_session_id, attempt) (PMT-R14) |
| Access | grants | id (gr_), booking_reference (unique), member_ref, space_id, space_name, valid_from, valid_until, ticket_code (unique), ticket_token (unique), status, revoked_at (timestamptz, null until the first revoke; a repeat revoke keeps it); a tombstone has no code, token or space (AXS-T07) |
| Access | scans | id, scanned_at, space_id (selected room), input (normalised), result, grant_id (AXS-R15) |
| All three | test_clock | one row: the override instant or null (D27) |

Purchase also keeps the seed's race guard, now partial, so expired and cancelled rows never block (PUR-R22):

```sql
ALTER TABLE bookings ADD CONSTRAINT bookings_no_overlap
  EXCLUDE USING gist (space_id WITH =, tstzrange(start_time, end_time) WITH &&)
  WHERE (status IN ('held', 'confirmed'));
```

Cross-service references (values only):

| Value | Minted by | Stored by | Used for |
|---|---|---|---|
| booking_reference BK-7KQ2M9 | Purchase (PUR-R29) | Payment sessions and refunds, Access grants | natural key for session create, grant and revoke |
| payment_session_id ps_... | Payment | Purchase bookings | reconcile, expire, refund |
| amount_satang 45000 | Purchase (agreed price, PUR-R17) | Payment sessions | collected as given, never re-priced (PMT-R04) |
| grant_id gr_..., ticket_url | Access | Purchase bookings | "View e-ticket" link |
| member_ref, space_id, space_name, valid_from, valid_until | Purchase | Access grants, copies as sent | e-ticket, kiosk room list, window check (AXS-R03) |

Payment never receives an email or a name: the description holds the space and the Bangkok time only (PUR-R23). Access stores member_ref as an opaque value and never shows it (AXS-T18).

## Service-to-service calls

Only Purchase calls. Payment and Access call no one: no webhooks, no callbacks (ADR-0004; PUR-R35, PMT-R18, AXS-R04). The Member's browser moves between services by redirect.

```python
# payment_client.py in Purchase; access_client.py is the same shape
r = requests.get(f"{PAYMENT_INTERNAL_URL}/payment-sessions/{session_id}",
                 headers={"Authorization": f"Bearer {PAYMENT_API_TOKEN}"}, timeout=5)
```

Every call follows the same three steps:

1. Commit what Purchase has decided, if anything (for example: held, or cancelled with its refund amount).
2. Make the HTTP call with no transaction open, `timeout=5`. A timeout, a connection error or a 5xx answer means "unreachable" (PUR-R35), the same definition as in the three contracts.
3. In a new short transaction, lock the booking row (`SELECT ... FOR UPDATE`), check it is still in the state the call assumed, and store the answer. If another worker got there first, drop the answer.

| Call | Purchase sends it when | On an answer | On no answer |
|---|---|---|---|
| POST /payment-sessions | right after the held insert commits (PUR-R23) | 201, or 200 for a repeat: store payment_session_id, 303 the browser to PAYMENT_PUBLIC_URL/pay/<id>, never to the answer's `url` (PUR-R23) | the booking stays held without a session; flash "Payment is not reachable. Please try again." or JSON 503; "Continue to payment" repeats the call until the payment deadline (PUR-Q09, PUR-R40) |
| GET /payment-sessions/{id} | at the reconcile points of PUR-R24: the booking page and its return URL, GET /api/bookings/<ref>, My bookings (the page and GET /api/bookings/mine), the cancel confirm screen, the pre-insert sweep, the Member's own lapsed holds (PUR-R39), and the Operator's Reconcile of one booking or of all held; the POST cancel uses expire in place of the GET (PUR-R31). "Reconcile all held" and the sweep stop after the first read with no answer | paid: fulfil (PUR-R25); unpaid after the hold: expired; unpaid inside the hold: no change | no change; the page says "Payment status unknown, refresh later"; a conflicting insert gets 503 or a flashed "try again" (D13) |
| POST /payment-sessions/{id}/expire | the Member or Operator cancels a held booking (PUR-R31) | unpaid: cancelled, or expired if the hold had lapsed; paid: confirm without a grant, then cancel as confirmed | the cancel is refused with 503 or a flashed "try again"; nothing changes |
| POST /refunds | cancel step 3, after the revoke succeeded (PUR-R32); the full refunds amount_mismatch and slot_unavailable (PUR-R25) | store succeeded or failed with refund_attempt; after failed, only the Operator's Retry sends attempt+1 (PUR-R33) | refund_status stays pending (committed with refund_attempt before the call: cancel step 1, the amount_mismatch or slot_unavailable request, or the Operator's attempt n+1), "refund pending"; the retry sends the same attempt, so a lost answer never pays twice (PMT-R14) |
| POST /grants | the booking becomes confirmed, and only while it stays confirmed with no cancel in progress (PUR-R26) | store grant_id, ticket_url, grant_status issued | grant_status pending (set when the booking was confirmed), ticket "being prepared"; retried at the retry points: the booking page (the page or GET /api/bookings/<ref>), the owner's Retry and the Operator's Retry (D20, PUR-R26) |
| POST /grants/{booking_reference}/revoke | cancel step 2 (PUR-R32) | grant_status revoked; an unknown reference becomes a revoked tombstone (AXS-R02) | grant_status stays revoke_pending (set in cancel step 1, PUR-R32), "revocation pending"; the refund waits (PUR-Q10); the retry revokes first |

- Any other 4xx answer (400, 401, 404, 409) is a Purchase or config defect, the same rule for every call: Purchase logs the operation, the booking reference and the status (never the token), stores nothing, and handles the step like "unreachable": pending, or "try again" where the request needs the answer (PUR-R35).
- Purchase never calls GET /grants in v1 (PUR-R26).
- Every JSON error in all three services is `{"error": {"code": "...", "message": "..."}}`. RULES.md rows quote only the message: `{"error": "Slot just taken"}` means `{"error": {"code": "slot_taken", "message": "Slot just taken"}}`. Only `GET /health` keeps the seed shape.
- Retries are safe because every write has a natural key: booking_reference for a session, a grant and a revoke; (payment_session_id, attempt) for a refund (ADR-0014; PMT-R03, PMT-R14, AXS-R01, AXS-R17).
- Consumer tests stub the provider at one module, `payment_client.py` or `access_client.py`, using the contract examples.

## Consistency and reconciliation

There is no distributed transaction. Each service commits its own facts. Purchase pulls the answers it needs and moves its booking toward the outcome they decide: "Purchase interprets the result" [extraction site].

- **Hold (D11).** A booking is slot-blocking when it is confirmed, or held with `hold_expires_at > :now`, where `:now` is `clock.now()` passed as a parameter (PUR-R12). A lapsed hold stays `held` in the table until something touches it, so the pre-insert sweep reconciles the space's stale holds before the insert, and the partial EXCLUDE catches the rest (PUR-R22). One held booking per Member (PUR-R39). From 10:13 to 10:15 the hold can no longer be paid, but still blocks (PUR-R40).
- **Session expiry (D12).** `expires_at = hold_expires_at - 2 min` (10:15 hold, 10:13 session). Payment refuses attempts from expires_at on (PMT-R11), so its answer is final before the hold ends. That is why a lost redirect needs no webhook (ADR-0004).
- **Reconciliation (D13).** Lazy, on touch, with no background job (ADR-0007; PUR-R24). A GET may run this sync because it only moves a booking toward an outcome already decided (PUR-R37).
- **Fulfilment (D14).** Confirm only when the session's booking_reference, amount_satang and currency equal the booking's (PUR-R25).
- **Cancel (D18, D19).** A held cancel expires the session first, so pay and cancel cannot both win (PMT-R06). A confirmed cancel runs three stored steps: commit, revoke, refund (PUR-R32).
- **Issuance (D20).** Grant on confirmation; a failure leaves the booking confirmed with a ticket "being prepared".

Invariant: every paid session ends as a confirmed booking or a refund attempt (D13).

| A paid session meets | Purchase does | Ends as |
|---|---|---|
| a held booking with the same reference, amount and currency | confirm, then POST /grants | confirmed booking |
| a held booking with a different amount or currency | cancelled with amount_mismatch and a full refund, sent after commit | refund attempt |
| a booking already expired | full refund with slot_unavailable | refund attempt |
| a held cancel whose expire answers paid | confirm without a grant, then cancel as confirmed (PUR-R30) | confirmed booking, then cancelled with the policy refund |

A held booking that nobody touches stays held until the Operator's Reconcile (one booking, or "Reconcile all held") or the next sweep of its space closes it (the D13 trade-off). Until then the dashboard counts it as held (PUR-R34).

Flows:

- Book and pay: held, session, hosted page, return, confirm, grant. ![Book and pay](diagrams/seq-book-pay.svg)
- Decline and retry: the session stays open; the Member retries on the same page (PMT-R10). ![Decline and retry](diagrams/seq-decline-retry.svg)
- Lost redirect: the next touch finds paid and confirms, even after the hold lapsed. ![Lost redirect](diagrams/seq-lost-redirect.svg)
- Hold expiry: unpaid after 10:15 becomes expired; the slot is free. ![Hold expiry](diagrams/seq-hold-expiry.svg)
- Coverage skips collection: plan and free bookings confirm at once, with no Payment call (PUR-R20). ![Coverage](diagrams/seq-coverage.svg)
- Cancel a held booking: expire first (PUR-R31). ![Cancel held](diagrams/seq-cancel-held.svg)
- Cancel a confirmed booking: commit, revoke, refund (PUR-R32). ![Cancel confirmed](diagrams/seq-cancel-confirmed.svg)
- Refund failure and the Operator's attempt 2 (PUR-R33, PMT-R16). ![Refund retry](diagrams/seq-refund-retry.svg)
- Check-in: code, room, window, revocation (AXS-R13, AXS-R14). ok shows "Door unlocked (mock)": the lock is mocked, and "Payment success ≠ access provisioned ≠ a successful door opening." [contexts site] (ADR-0017). ![Check-in](diagrams/seq-checkin.svg)

## States

![States](diagrams/states.svg)

| Context | Record | Stored states | Derived, never stored |
|---|---|---|---|
| Purchase | Booking | held to confirmed, expired or cancelled; confirmed to cancelled (PUR-R28) | Completed (confirmed, end passed) |
| Purchase | Follow-up fields | grant_status not_requested, pending, issued, revoke_pending or revoked; payment_outcome not_required, unpaid or paid; refund_status none, pending, succeeded or failed (PUR-T34) | none |
| Payment | Session | open to complete or expired; payment_status unpaid or paid (PMT-R05) | none |
| Payment | Attempt, Refund | attempt succeeded or declined(code); refund succeeded or failed (PMT-R16) | "needs manual follow-up" (PMT-T15) |
| Access | Grant | issued to checked_in; revoked from either (AXS-R16) | not_yet_valid, expired, no_show |

## Security

| Topic | What we do | Source |
|---|---|---|
| Login session | Signed `purchase_session` cookie, 12 h from login whatever the activity. Logout clears it in that browser; a copied cookie still works until the 12 h end (accepted) | PUR-R03, D15, ADR-0016 |
| Cookies | `purchase_session`, `payment_session`, `access_session`: HttpOnly and SameSite=Lax; Secure when PUBLIC_URL starts with https | PUR-R03, PMT-R19, AXS-R18 |
| CSRF | SameSite=Lax instead of tokens. Every state change is a POST; a GET runs only the idempotent sync. Except, accepted: POST /login and /logout, and the kiosk room choice (ADR-0016) | PUR-R37, ADR-0009, ADR-0016 |
| SECRET_KEY | Required in all three. Unset, empty, shorter than 32 characters or the seed's public default "dev-secret-key-not-for-production": the process exits at start. API tokens need 32 characters too, OPERATOR_PASSWORD and STAFF_PASSWORD 12 (ADR-0019) | PUR-R03, PMT-R19, AXS-R18, ADR-0019 |
| Service tokens | `Authorization: Bearer` with PAYMENT_API_TOKEN or ACCESS_API_TOKEN. Missing, wrong or another scheme: 401 before any validation or lookup. Payment and Access refuse to start without their token. Compare UTF-8 bytes: `hmac.compare_digest(given.encode("utf-8"), expected.encode("utf-8"))`, so a non-ASCII header gets 401, never a 500 | PMT-R01, AXS-R04, ADR-0019 |
| HTTP Basic | Payment GET /operator checks OPERATOR_PASSWORD; the Access kiosk checks STAFF_PASSWORD; constant-time compare of UTF-8 bytes, as for tokens; only the password is checked (the e2e suite sends user operator or staff) | PMT-R17, AXS-R11, ADR-0019 |
| Ownership | Booking pages and actions for the owner or an operator, else 404; anonymous callers get the login step or 401 before any lookup. Operator pages 404 to non-operators | PUR-R05, PUR-R06, D17 |
| Card data | Store only brand and last4. Never store, log, flash or put in a URL the number, expiry or CVC; the card form posts in the body | PMT-R13, ADR-0020, ADR-0018 |
| Hosted page | GET /pay/{id} needs no login; the 128-bit random session id is the link | PMT-R07, PMT-Q05 |
| Ticket link | `/t/<ticket_token>`, 128-bit, view-only, no email; `Referrer-Policy: no-referrer`. Whoever holds the link holds entry for the window | AXS-R09, ADR-0008 |
| Messages | Flask `flash()` only; never render text from `?error=`, `?message=` or `?code=` | PUR-R36, PMT-R10, AXS-R14, D28 |
| Access log | gunicorn logs the path without the query string and without the referer (kept from the seed, snippet under Runtime), so `?session_id=` never reaches the log; the `/t/` path is logged (accepted), and Payment's log records the `/pay/<id>` and `/payment-sessions/<id>` paths, so the session id is logged at the same trust level as Payment's database (accepted, PMT-Q05). Purchase never logs a ticket_url. No service logs request bodies or the Cookie and Authorization headers | PUR-R36, PMT-R13, AXS-R09, PUR-R26, ADR-0020 |
| Caching | `Cache-Control: no-store` on every Purchase page served to a logged-in caller, on Payment's `/pay/<id>` and `/operator`, and on Access `/t/<ticket_token>` and `/checkin`, so after logout on a shared computer Back shows no booking, ticket link or email. One Purchase test checks the header on `/bookings/<ref>` | ADR-0016, PUR-R05 |
| Request size | Flask `MAX_CONTENT_LENGTH = 64 * 1024` in all three apps, so a larger body gets 413 before it is parsed; the booking note is at most 500 characters (PUR-R14) | PUR-R14, D28 |
| Output | Jinja autoescape on in all templates; the only output marked safe (`Markup` or the `safe` filter) is segno's SVG, built from the stored ticket code | ADR-0010 |
| Accepted trade-offs | The five rows of ADR-0016 (logout replay inside 12 h, registration enumeration, bearer ticket link, HTTP Basic, mock card data) plus its smaller ones: login and logout CSRF, forged kiosk room change, one site on localhost, the first registrant of OPERATOR_EMAIL, no failed-login or kiosk limits, the bearer payment link, a pending revoke, the Operator's own bookings under the operator policy, no password change or reset (PUR-Q16), and hold cycling (PUR-Q17); all listed in the PRD section 3 table | D15, D16, D17, ADR-0009, ADR-0016 |

## Time and the test clock

- **One fixed offset (D2).** Bangkok is UTC+7 with no daylight saving, so a fixed `timezone(timedelta(hours=7))` is exact and needs no tzdata (ADR-0012). Store timestamptz. JSON needs an offset: `"2026-10-07T09:00:00+07:00"` is accepted, `"2026-10-07T09:00:00"` gets 400. The form sends a date and a block; the server builds the instant (PUR-R07, PMT-R02, AXS-R03).
- **One clock (D27).** Every service reads time only through `clock.now()`. SQL gets it as a parameter, never `now()` or `CURRENT_TIMESTAMP` in business logic; audit-only `created_at` defaults are fine, but the business times listed under "Data ownership" come from `clock.now()` (ADR-0013; PUR-R38, PMT-R20, AXS-R19).
- **Test override.** With `TEST_CLOCK_ENABLED=true` (only in `compose.e2e.yaml`), `POST /_test/clock` stores an instant in the one-row `test_clock` table; `null` clears it. The override is a fixed instant: `clock.now()` returns exactly the stored value until the next POST, and never advances (ADR-0013). It reads the `test_clock` row on every call (no per-process cache), so both workers see a new instant from the next request. Otherwise the route answers 404, and `clock.now()` returns real time and never reads `test_clock`, even if an e2e run left a row in the volume. The e2e suite sets the same instant on all three services:

```bash
for p in 8001 8002 8003; do
  curl -s -X POST localhost:$p/_test/clock -H 'Content-Type: application/json' -d '{"now": "2026-10-05T10:15:00+07:00"}'
done   # Member A's unpaid hold on BK-7KQ2M9 has now lapsed on every service
```

A `now` without an offset, or not a time at all, gets 400 and changes nothing (D2).

### The e2e harness

The e2e suite runs serially against one shared stack that is never reset (no reset hook exists). It starts the stack with the test clock on, the only place this command is used:

```bash
cd cowork-booking-docs/integration && docker compose -f compose.yaml -f compose.e2e.yaml up -d --build --wait   # e2e only: enables POST /_test/clock
```

So:

- Each test registers its own Members with unique emails, and the Operator creates a uniquely named space per test and finds its space_id with GET /api/spaces.
- The helper registers OPERATOR_EMAIL with E2E_OPERATOR_PASSWORD, or on "Email already registered" logs in with it; if that login fails, the run stops at start with "OPERATOR_EMAIL is registered with another password: remove the e2e volumes". Manual checks on the e2e stack never register OPERATOR_EMAIL (the integration README says so).
- `compose.e2e.yaml` sets the top-level `name: cowork-e2e`, so e2e accounts, bookings and the `test_clock` row live in their own volumes, never in those of a normal run. Both projects use ports 8001-8003: stop one before starting the other.
- One helper sets the same instant on all three services before any login in that test.
- A Purchase login is valid only while login_at <= clock.now() < login_at + 12 h (PUR-R03). After moving the clock 12 h or more past a login, or back before it, the helper logs every actor in again.
- Shared figures (Payment totals, kiosk room lists and scan lists) are asserted as before-and-after deltas, or by the marker rows of the booking under test (purchase-payment.md section 8).
- The wrong_room case first issues a grant in a second space, so Staff can select that room (AXS-R11).
- The suite reads OPERATOR_EMAIL, E2E_OPERATOR_PASSWORD, PAYMENT_API_TOKEN, ACCESS_API_TOKEN, OPERATOR_PASSWORD and STAFF_PASSWORD from `integration/.env`, the file compose uses, and fails at start if any is empty.
- Rows are scoped to the booking under test: Payment rows by `data-booking-reference`, Purchase all-bookings rows by `data-booking-reference`, kiosk scans by before-and-after deltas of `data-scan-result` rows at a room the test created.
- e2e uses the per-booking Reconcile. The exact counts of "Reconcile all held" are asserted in Purchase integration tests with a fresh database, because earlier runs leave held bookings on the shared stack.
- "Held cancel racing with payment" pays with `allow_redirects=False`, never follows success_url and reads nothing in Purchase before the cancel, because every read reconciles (contract purchase-public.md section 11; PUR-R31 row 12).

## Runtime

| Part | Version | Where | Note |
|---|---|---|---|
| Python | 3.12 | all | `python:3.12-slim` image |
| Flask with Jinja | 3.1.2 | all | server-rendered pages and JSON |
| gunicorn | 23.0.0 | all | `workers = 2` in `gunicorn.conf.py`; no `preload_app`, so each worker opens its own connection (ADR-0015) |
| psycopg[binary] | 3.2.13 | all | one connection per worker; explicit transactions for booking, fulfilment, pay and refund (D28) |
| PostgreSQL | 16 | one per service | btree_gist for the Purchase EXCLUDE |
| requests | 2.32.3 | Purchase calls | `timeout=5` on every call |
| segno | pinned in M5 | Access only | QR as inline SVG; pure Python; the only new runtime dependency (ADR-0010) |
| pytest | 8.4.2 | tooling | per-service CI and the e2e suite |

```python
# gunicorn.conf.py (all three)
workers = 2  # ponytail: 2 workers x 1 DB connection is the ceiling; psycopg_pool is the upgrade path
timeout = 30  # gunicorn default, named: a request that waits on Payment stops after one 5 s timeout (PUR-R24)
accesslog = "-"
# kept from the seed: %(U)s is the path without the query string; no referer
access_log_format = '%(h)s %(t)s "%(m)s %(U)s %(H)s" %(s)s %(b)s %(M)sms "%(a)s"'
```

- Each service keeps from the seed: `/health` with `revision` (503 when the DB is down), the fail-fast DB connect, the query-string scrubbing, `base.html` with `style.css`, and the `money` and `local_time` filters in THB.
- Tables are created at start with `CREATE ... IF NOT EXISTS`; there is no migration tool in v1.
- Not used: an ORM, an SPA framework, a message broker, Kubernetes, a service mesh or an API gateway. The browser reaches each service on its own port.

## Delivery and rollback

Each service is delivered on its own (D26, ADR-0011). The course grades "Service delivered, independently deployable" and asks teams to "use version control, pull-request review, continuous delivery, and rollback as coordination and safety systems;" [syllabus site].

- **CI per service** (`.github/workflows/ci.yml`): pytest against a postgres:16 service container, then `docker build`. No registry push, no deploy.
- **Docs CI** (`.github/workflows/docs.yml`): `python3 scripts/check_docs.py`, then renders every `diagrams/*.mmd` with mermaid-cli.
- **Deploy**: `git checkout v1.0.0 && docker compose up -d --build` in that service's repo. The DB volume stays.
- **Rollback**: redeploy the previous tag the same way. Keep schema changes additive (new tables, new nullable columns), so the previous tag still starts on the newer schema.
- **Contract changes**: tag the provider `contract-v1`, `contract-v2`; keep a change backward compatible and deploy the provider before the consumer.
- **Removed**: the seed's pipeline pushed to a registry and deployed to Nomad on every push to main (Spacey: source inspected at 5a1cf3d, `.github/workflows/delivery.yml`, `deploy/startup-app.nomad.hcl`). Both are deleted before the seed commit. There is no auto-revert; rollback is a manual redeploy.

## Environment variables

Commit only `.env.example` files, with every secret left empty (`SECRET_KEY=`), so a copied example fails fast; generate values with `python -c 'import secrets; print(secrets.token_hex(32))'`. Never set `TEST_CLOCK_ENABLED` in a Dockerfile, a service `compose.yaml` or `.env.example`. Each container gets only the variables in its own table below.

| All services | Example (integration stack) | Purpose |
|---|---|---|
| DATABASE_URL | `postgresql://purchase:<password>@purchase-db:5432/purchase` | own database only; fail fast when unreachable |
| SECRET_KEY | a different random value per service | required; signs the cookie and flashes |
| APP_REVISION | the git SHA | shown by GET /health |
| PUBLIC_URL | `http://localhost:8001` | the service's browser-facing base URL; Secure cookie when https; each service builds its own links from it (Purchase success_url and cancel_url, the Payment hosted-page url, the Access ticket_url) |
| TEST_CLOCK_ENABLED | `true` in `compose.e2e.yaml` only | enables POST /_test/clock |
| POSTGRES_PASSWORD | per-repo `.env` only, empty in `.env.example` | read by the repo's own `compose.yaml`, not by the app: the `db` container's password and the password inside that repo's DATABASE_URL. The integration stack uses PURCHASE_DB_PASSWORD, PAYMENT_DB_PASSWORD and ACCESS_DB_PASSWORD from integration/.env in its place |

| Purchase | Example | Purpose |
|---|---|---|
| PAYMENT_INTERNAL_URL | `http://payment:8000` | server-to-server calls to Payment |
| PAYMENT_PUBLIC_URL | `http://localhost:8002` | the hosted-page redirect PAYMENT_PUBLIC_URL/pay/<id> (PUR-R23) and the "Payment totals" link |
| PAYMENT_API_TOKEN | shared secret | bearer token on every Payment call |
| ACCESS_INTERNAL_URL | `http://access:8000` | server-to-server calls to Access |
| ACCESS_PUBLIC_URL | `http://localhost:8003` | a ticket_url is stored only when it starts with ACCESS_PUBLIC_URL/t/ (PUR-R26) |
| ACCESS_API_TOKEN | shared secret | bearer token on every Access call |
| OPERATOR_EMAIL | `operator@example.com` | the Member with this email becomes the Operator (PUR-R04) |

| Payment | Example | Purpose |
|---|---|---|
| PAYMENT_API_TOKEN | same value as in Purchase | required; checks the bearer token |
| OPERATOR_PASSWORD | shared secret | required; HTTP Basic for GET /operator |

| Access | Example | Purpose |
|---|---|---|
| ACCESS_API_TOKEN | same value as in Purchase | required; checks the bearer token |
| STAFF_PASSWORD | shared secret | required; HTTP Basic for the kiosk |

## Repository layout

```
cowork-booking-docs/  README.md SOURCES.md GLOSSARY.md RULES.md ID_MAP.md DECISIONS.md OPEN_QUESTIONS.md PRD.md
                      BUSINESS_MODEL.md ARCHITECTURE.md adr/ diagrams/ contracts/README.md (links to provider copies)
                      REVIEW_LOG.md TRACEABILITY.md inventory/ scripts/check_docs.py
                      integration/{compose.yaml,compose.e2e.yaml,.env.example,e2e/,README.md} .github/workflows/docs.yml
cowork-booking-<svc>/ app.py (or a small package past ~600 lines) <domain>.py clock.py templates/ static/ tests/
                      openapi.yaml CONTRACT.md (all three; ADR-0005) PROVENANCE.md README.md Dockerfile compose.yaml
                      gunicorn.conf.py pytest.ini requirements.txt .github/workflows/ci.yml
```

`<domain>.py` is `purchase.py`, `payment.py` or `access.py`. Purchase adds `payment_client.py` and `access_client.py`, the one boundary its tests stub.

## Against the course target

The course's "ready for splitting" target keeps one application with Purchase as coordinator, Payments and Access owning their records, and no module reading another's tables. Cowork Booking keeps that ownership and call direction and takes the next step, separate services. The step-by-step mapping and the deliberate differences (hosted checkout, issuance at confirmation, the code shown on the booking page read live from Access) are in ADR-0021.
