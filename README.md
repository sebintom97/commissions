# Commission Management System

> **An AI-assisted architecture and design exercise.**
> This code was generated largely with AI assistance. I use it to study how a
> system like this is structured. It is not code I engineered or can walk
> through line by line, and it has never run in production.

A Spring Boot REST API that models commission management for a recruitment
agency: clients, contractors, placements, salespeople, commission plans,
payout (drawdown) requests, revenue recognition schedules, a ledger of
financial events, and reporting.

## Stack

Java 17 · Spring Boot 3.5 · PostgreSQL · Flyway (11 migrations) ·
Spring Security with JWT · Springdoc OpenAPI · Docker Compose

## Known issues

Found by reviewing the code and running the test suite. None of these are
fixed yet.

- **A wrong password returns `500`, not `401`.** The catch-all
  `@ExceptionHandler(Exception.class)` in `GlobalExceptionHandler` also catches
  `BadCredentialsException`, which nothing handles explicitly.
- **Not all tests pass.** In the last recorded run, 3 of 26 did not:
  `AuthIntegrationTest.testInvalidLogin` (the 500 above), and two errors in
  `PlacementServiceIntegrationTest` caused by a `DataIntegrityViolationException`.
- **CORS is wide open.** `SecurityConfig` allows every origin, method and
  header on every path. That's fine for local use and unsafe anywhere else.
- **Swagger points at the wrong host off the default port.** `OpenApiConfig`
  hardcodes `http://localhost:8080` as the server, so "Try it out" misfires
  when the app runs on any other port.
- **The ledger is not double-entry.** It writes one row per financial event,
  which makes it a typed transaction log. An earlier version of this README
  described it as a double-entry ledger, which was wrong.

## Running it locally

Requires Java 17+ and Docker.

```bash
docker compose up -d postgres
./mvnw spring-boot:run
```

Database credentials are in `docker-compose.yml`. Flyway builds the schema on
first start.

- Swagger UI: http://localhost:8080/swagger-ui.html
- OpenAPI spec: http://localhost:8080/api-docs

Most endpoints need a JWT. Register with `POST /api/auth/register`, log in
with `POST /api/auth/login`, then paste the token into Swagger's
**Authorize** dialog.

To run on another port, override the server and data source rather than
editing config:

```bash
SERVER_PORT=8085 \
SPRING_DATASOURCE_URL=jdbc:postgresql://localhost:5432/commissions_db \
./mvnw spring-boot:run
```
