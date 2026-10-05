# Architecture decision records

One file per decision, `adr/NNNN-<slug>.md`, cited as ADR-NNNN. Each has Status, Context, Decision, Consequences, Alternatives considered, Rules and decisions, and Sources. Project decisions D1-D28 are in [../DECISIONS.md](../DECISIONS.md); an ADR records why a design holds them together.

To change a decision, add a new ADR that supersedes the old one, and set the old one's status to "Superseded by ADR-NNNN". Never rewrite an accepted ADR's decision in place.

| ADR | Title | Status | Rules |
|---|---|---|---|
| [ADR-0001](0001-three-contexts.md) | Three contexts, three services | Accepted, 2026-10-01 | PUR-R17, PUR-R19, PUR-R26, PUR-R29, PUR-R32, PUR-R35, PMT-R04, PMT-R18, AXS-R04, AXS-R17 |
| [ADR-0002](0002-member-in-purchase.md) | Account data on Member in Purchase, no Identity context | Accepted, 2026-10-01 | PUR-R01, PUR-R02, PUR-R03, PUR-R04, PUR-R05, PUR-R06, PUR-R19, AXS-R03, AXS-R09 |
| [ADR-0003](0003-database-per-service.md) | Database per service | Superseded by ADR-0022 | PUR-R29, PUR-R32, PUR-R34, PUR-R35, PMT-R04, PMT-R16, PMT-R17, PMT-R18, AXS-R03, AXS-R04, AXS-R11, PUR-R38, PMT-R20, AXS-R19 |
| [ADR-0004](0004-purchase-only-caller-no-webhooks.md) | Only Purchase calls other services; no webhooks | Accepted, 2026-10-01 | PUR-R22, PUR-R24, PUR-R25, PUR-R26, PUR-R31, PUR-R32, PUR-R35, PUR-R40, PMT-R01, PMT-R06, PMT-R10, PMT-R11, PMT-R18, AXS-R04, AXS-R17 |
| [ADR-0005](0005-contracts-beside-provider.md) | Contracts live beside the provider's code | Accepted, 2026-10-01 | PUR-R23, PUR-R26, PUR-R35, PMT-R01, PMT-R02, PMT-R03, PMT-R06, PMT-R14, AXS-R01, AXS-R02, AXS-R03, AXS-R04, AXS-R17 |
| [ADR-0006](0006-copy-and-prune-seeding.md) | Seed three repos by copy-and-prune | Accepted, 2026-10-01 | PUR-R17, PUR-R22, PMT-R08, PMT-R13, AXS-R05 |
| [ADR-0007](0007-lazy-hold-expiry-reconciliation.md) | Lazy hold expiry and reconciliation | Accepted, 2026-10-01 | PUR-R12, PUR-R21, PUR-R22, PUR-R24, PUR-R25, PUR-R31, PUR-R34, PUR-R39, PUR-R40, PMT-R05, PMT-R11 |
| [ADR-0008](0008-bearer-ticket-link.md) | The e-ticket link is a bearer link | Accepted, 2026-10-01 | AXS-R05, AXS-R07, AXS-R08, AXS-R09, AXS-R10, AXS-R14, AXS-R17, PUR-R05, PUR-R26 |
| [ADR-0009](0009-samesite-csrf.md) | SameSite=Lax cookies are the CSRF mitigation | Accepted, 2026-10-01 | PUR-R03, PUR-R37, PMT-R10, PMT-R19, AXS-R11, AXS-R18 |
| [ADR-0010](0010-segno-qr.md) | segno draws the ticket QR | Accepted, 2026-10-01 | AXS-R06, AXS-R07, AXS-R10, AXS-R12 |
| [ADR-0011](0011-delivery-without-nomad.md) | Delivery without Nomad; rollback redeploys the previous tag | Accepted, 2026-10-01 | PUR-R03, PMT-R19, AXS-R18, PUR-R38, PMT-R20, AXS-R19, PMT-R01, AXS-R04 |
| [ADR-0012](0012-fixed-utc7-offset.md) | One fixed UTC+7 offset | Accepted, 2026-10-01 | PUR-R07, PUR-R08, PUR-R10, PUR-R13, PUR-R34, PMT-R02, PMT-R08, AXS-R03, AXS-R13 |
| [ADR-0013](0013-test-clock.md) | One test clock, off by default | Accepted, 2026-10-01 | PUR-R38, PMT-R20, AXS-R19, PUR-R10, PUR-R12, PUR-R30, PMT-R05, PMT-R08, PMT-R11, AXS-R13, AXS-R16 |
| [ADR-0014](0014-natural-key-idempotency.md) | Natural keys make repeats safe; no Idempotency-Key header | Accepted, 2026-10-01 | PUR-R21, PUR-R22, PUR-R29, PUR-R33, PUR-R39, PMT-R03, PMT-R06, PMT-R12, PMT-R14, PMT-R15, AXS-R01, AXS-R02, AXS-R05, AXS-R17 |
| [ADR-0015](0015-two-workers-one-connection.md) | Two gunicorn workers, one database connection each | Accepted, 2026-10-01 | PUR-R22, PUR-R25, PUR-R32, PUR-R35, PMT-R06, PMT-R12, PMT-R15 |
| [ADR-0016](0016-accepted-security-trade-offs.md) | Accepted security trade-offs | Accepted, 2026-10-01 | PUR-R01, PUR-R02, PUR-R03, PUR-R04, PUR-R05, PUR-R39, PMT-R08, PMT-R09, PMT-R13, PMT-R17, AXS-R09, AXS-R11, AXS-R15 |
| [ADR-0017](0017-mock-lock.md) | The lock is mocked | Accepted, 2026-10-01 | AXS-R04, AXS-R13, AXS-R14, AXS-R15, AXS-R16, AXS-R17 |
| [ADR-0018](0018-mock-checkout-test-cards.md) | Hosted mock checkout with test cards | Accepted, 2026-10-01 | PMT-R02, PMT-R04, PMT-R07, PMT-R08, PMT-R09, PMT-R10, PMT-R11, PMT-R12, PMT-R16, PUR-R20, PUR-R23 |
| [ADR-0019](0019-service-authentication.md) | Service authentication | Accepted, 2026-10-01 | PMT-R01, AXS-R04, PMT-R17, AXS-R11, PUR-R02, PUR-R03, PUR-R04, PUR-R05, PUR-R06, PUR-R35, PMT-R07, AXS-R09 |
| [ADR-0020](0020-card-data-handling.md) | Card data handling | Accepted, 2026-10-01 | PMT-R08, PMT-R13, PMT-R16, PMT-R17, PMT-R19, PUR-R36 |
| [ADR-0021](0021-course-target-alignment.md) | Alignment with the course's "ready for splitting" target | Accepted | PUR-R17, PUR-R20, PUR-R23, PUR-R24, PUR-R25, PUR-R26, PUR-R41, AXS-R05, AXS-R09 |
| [ADR-0022](0022-one-database-schema-per-context.md) | One database, one schema and role per context | Accepted, 2026-10-05 | PUR-R28, PUR-R29, PUR-R32, PUR-R35, PMT-R04, PMT-R16, PMT-R18, AXS-R01, AXS-R03, AXS-R04, AXS-R17 |
