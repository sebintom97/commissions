# Commission Management System

A Spring Boot REST API for managing salesperson commissions in a recruitment/staffing agency.
When a contractor placement is created, the system automatically calculates multi-step
financial margins, applies tiered commission rates, generates a 12-month revenue recognition
schedule, and tracks all payouts through an approval workflow — with a full double-entry ledger
for audit compliance.

![Java](https://img.shields.io/badge/Java-17-007396?logo=openjdk)
![Spring Boot](https://img.shields.io/badge/Spring_Boot-3.5.7-6DB33F?logo=springboot)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-12+-4169E1?logo=postgresql)
![Flyway](https://img.shields.io/badge/Flyway-migrations-CC0200?logo=flyway)
![JWT](https://img.shields.io/badge/Auth-JWT-black)
![Docker](https://img.shields.io/badge/Docker-ready-2496ED?logo=docker)

---

## Architecture overview

```
HTTP Request
    │
    ▼
Controller  (REST endpoint, input validation)
    │
    ▼
Service Interface → ServiceImpl
    │                    │
    │              ┌─────┴──────────────────────┐
    │              │                            │
    │         Domain Engines              Repository
    │    (CommissionCalculation,       (Spring Data JPA)
    │     RevenueRecognition,                  │
    │     DrawdownEngine)                      ▼
    │                                     PostgreSQL
    └──► LedgerService  (writes an immutable audit entry for every financial event)
```

Key patterns:
- **Service/interface split** — all business logic behind interfaces, enabling clean unit testing without Spring context
- **Domain engines** — `CommissionCalculationService`, `RevenueRecognitionEngine`, and `DrawdownEngine` are dedicated components for the complex domain logic, keeping `ServiceImpl` classes focused on orchestration
- **Flyway versioned migrations** — schema lives alongside code; Hibernate is set to `validate` only, never auto-modifies
- **DTO/Mapper pattern** — JPA entities never cross the HTTP boundary; request and response shapes are separate classes
- **Ledger-as-audit-trail** — every financial state change is written as an immutable ledger entry with a typed `entry_type`

---

## How to run locally

### Prerequisites
- Java 17+
- PostgreSQL 12+
- Maven 3.8+

### Option A — Docker (recommended)

```bash
docker-compose up -d
./mvnw spring-boot:run
```

### Option B — Manual setup

```bash
# 1. Create the database
psql -U postgres -c "CREATE DATABASE commissions_db;"
psql -U postgres -c "CREATE USER commissions_user WITH PASSWORD 'password';"
psql -U postgres -c "GRANT ALL ON DATABASE commissions_db TO commissions_user;"

# 2. Run
./mvnw spring-boot:run

# 3. Open interactive API docs
open http://localhost:8080/swagger-ui.html
```

> `application.properties` contains dev-only placeholder values (password, JWT secret).
> Override them via environment variables or an `application-local.properties` file (gitignored) for any real deployment.

---

## API overview

| Group | Endpoints | Purpose |
|---|---|---|
| Auth | `POST /api/auth/register`, `POST /api/auth/login` | JWT token issue |
| Placements | `CRUD /api/placements` | **Core feature** — triggers automatic commission calculation on create |
| Commission Plans | `CRUD /api/commission-plans` | Tracks per-placement commission lifecycle (PLANNED → CONFIRMED → RECOGNIZED → PAID) |
| Recognition Schedules | `GET /api/recognition-schedules` | Monthly revenue recognition entries |
| Drawdowns | `CRUD /api/drawdowns`, `POST .../approve`, `.../reject` | Payout request workflow |
| Ledger | `GET /api/ledger` | Immutable audit trail of all financial events |
| Reports | `GET /api/reports/salesperson/{id}/dashboard` | Financial summaries and top-performer rankings |

Full interactive documentation at `/swagger-ui.html` once the server is running.

### Example: create a placement

**POST** `/api/placements`

```json
{
  "salespersonId": 1,
  "clientId": 1,
  "contractorId": 1,
  "placementType": "CONTRACTOR",
  "status": "ACTIVE",
  "startDate": "2025-01-15",
  "endDate": "2025-12-31",
  "hoursPerWeek": 40,
  "weeksPerYear": 45,
  "payType": "SALARY",
  "annualSalary": 60000,
  "billRate": 45.50
}
```

The response includes auto-calculated fields:

```json
{
  "commissionTotal": 2034.40,
  "commissionPercentage": 15,
  "netAnnualMargin": 13562.64,
  "sequenceNumber": 1
}
```

Behind the scenes, a single `POST /api/placements` call:
1. Calculates all financial fields (margins, load costs, commission)
2. Creates a Commission Plan (status: `PLANNED`)
3. Generates a 12-month Recognition Schedule
4. Writes the opening Ledger entry

---

## Key technical decisions

**BigDecimal for all financial math**
All monetary values use `BigDecimal` with explicit `RoundingMode.HALF_UP` and a 2-decimal scale at output boundaries. This avoids the floating-point rounding errors that would compound across the multi-step commission calculation (salary → hourly cost → margin → annualised → net → commission).

**Flyway over Hibernate DDL auto**
`spring.jpa.hibernate.ddl-auto=validate` — Hibernate only validates the schema, never modifies it. All schema changes go through versioned Flyway scripts (`V1__` through `V11__`), giving a reproducible migration path that is safe to run against a production database.

**Revenue recognition as a first-class engine**
Rather than storing a single payout date, commissions are amortised over 12 months into a `recognition_schedule` table. This matches how recruitment agencies actually account for contractor revenue and makes the `DrawdownEngine` straightforward: available balance = `sum(recognized) − sum(paid)`.

**Service interface + impl split**
Every service has a `FooService` interface and a `FooServiceImpl`. This decouples controllers from implementation details, makes mocking trivial in unit tests, and is a convention expected in enterprise Java teams.

**CORS open for dev; restrict per environment**
`corsConfigurationSource()` allows all origins in the default profile. Restrict `allowedOrigins` to your frontend URL in `application-prod.properties`.

---

## Commission calculation walkthrough

```
Input:  Annual salary €60,000 | Bill rate €45.50/hr | 40 hrs/week | 45 weeks/year

Step 1 — Hourly pay cost with load factors
  Base:  €60,000 ÷ 52 ÷ 40      = €28.85/hr
  Leave (14.54%):  × 1.1454      = €33.05/hr
  PRSI  (11.25%):  × 1.1125      = €36.77/hr
  Pension (1.5%):  × 1.015       = €37.32/hr  ← hourlyPayCost

Step 2 — Margin per hour
  €45.50 − €37.32                = €8.18/hr

Step 3 — Weekly margin
  €8.18 × 40 hrs                 = €327.20/week

Step 4 — Gross annual margin
  €327.20 × 45 weeks             = €14,724.00

Step 5 — Net annual margin (after overheads)
  Admin (6%):  −€883.44
  Insurance (2%):  −€294.48
  Net margin                     = €13,546.08

Step 6 — Commission (first contract = 15%)
  €13,546.08 × 15%               = €2,031.91  ← commissionTotal
```

Second placement (same contractor, same client) → **10%**. Third and beyond → **8%**.

---

## Running tests

```bash
./mvnw test
```

- `CommissionCalculationServiceTest` — unit tests validating financial math against known values (no Spring context needed)
- `PlacementServiceIntegrationTest` — full `@SpringBootTest` integration test for the placement creation flow
- `AuthIntegrationTest` — register/login/token round-trip
- `ReportingServiceIntegrationTest` — dashboard and period summary assertions

---

## License

MIT
