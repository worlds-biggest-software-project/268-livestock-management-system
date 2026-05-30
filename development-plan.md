# Livestock Management System — Phased Development Plan

> Project: 268-livestock-management-system · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four data-model suggestions. The database design adopts **Data Model Suggestion 3 (Hybrid Relational + JSONB)** as the backbone because it is explicitly optimised for multi-species, multi-jurisdiction MVP development — core identity and relationships are typed columns, while species- and jurisdiction-specific variation lives in indexed JSONB. Where regulatory audit trails demand immutability, the event-sourcing ideas from Suggestion 2 are folded in via a bi-temporal `animal_events` table plus an append-only `audit_log`.

---

## Product Summary

**What it is.** An open-source, AI-native platform for tracking individual animals and herds across their full lifecycle — identification, health, breeding, nutrition, performance, movement, and regulatory compliance — across mixed-species operations (cattle, sheep, goats, pigs, poultry, horses, deer).

**Primary users.** Commercial beef producers (200–5,000 head), dairy farm managers, sheep/goat producers under EU EID, stud/registry operators, integrated livestock businesses, and the veterinarians and nutritionists who serve them.

**Key differentiators.** (1) Open-source and self-hostable, avoiding per-head licence creep; (2) multi-species and multi-jurisdiction from day one; (3) standards-native — ICAR ADE as the integration backbone, ISO 11784/11785 RFID, EU/NLIS/USDA traceability, GS1 EPCIS export; (4) AI-native — predictive health alerts from sensor + history, breeding recommendations, feed ration optimisation, and a natural-language herd query interface exposed both in-app and via an MCP server (no incumbent ships one).

**Deployment model.** Self-hostable via Docker Compose (Postgres + API + worker + web). Also runnable as a managed SaaS. CLI provided for import/export and admin. The same API powers web, mobile (future), and the MCP server.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | Python 3.12 | The product is AI-heavy (predictive health, ration optimisation, NL query). Python has the strongest LLM/ML ecosystem (anthropic SDK, scikit-learn, pandas) and excellent Postgres tooling. A single language across API, workers, ML, and MCP server reduces surface area. |
| API framework | FastAPI | Async, first-class Pydantic v2 validation (critical for JSONB payload validation moved to the app layer per the hybrid model), and automatic OpenAPI 3.1 generation — `standards.md` mandates publishing an OAS 3.x spec for third-party integration. |
| Data validation | Pydantic v2 | The hybrid JSONB model pushes referential and shape validation into the app. Pydantic models define every JSONB payload (identifiers, event detail, jurisdiction data) so the flexibility of JSONB keeps a typed contract. |
| Database | PostgreSQL 16 | JSONB + GIN indexing is the core of the chosen data model; also provides row-level security for multi-tenancy, partitioning for sensor data, and `gen_random_uuid()`. |
| ORM / DB access | SQLAlchemy 2.0 (async) + Alembic | Async ORM matches FastAPI; Alembic handles migrations. Raw SQL used for JSONB containment (`@>`) queries where the ORM is awkward. |
| Task queue | Celery + Redis | Async workloads: sensor ingestion, projection refresh, withdrawal recalculation, AI inference, regulatory submission retries, EPCIS export. Redis doubles as cache and Celery broker. |
| Caching / broker | Redis 7 | Celery broker/result backend and hot-path cache (reference data, breed codes, per-tenant settings). |
| Frontend | Next.js 15 (React, TypeScript) + Tailwind + shadcn/ui | Mobile-first responsive dashboard required by `features.md`. Server components for fast list views; PWA-capable for offline field capture later. |
| LLM provider | Anthropic Claude (via `anthropic` SDK) with provider abstraction | AI-native features. A thin `LLMClient` interface allows swapping/self-hosting models for on-prem deployments. |
| ML (predictive health) | scikit-learn + pandas | Tabular sensor + history features for illness/heat prediction. Lightweight, self-hostable, no GPU required for MVP. |
| MCP server | Official Python MCP SDK | `standards.md` highlights the MCP gap; expose herd queries as MCP tools backed by the same service layer. |
| RFID parsing | Custom `iso11784` module | Decode/encode the 64-bit ISO 11784 code (country code, animal flag, 12-digit national ID); no mature PyPI lib covers this cleanly. |
| Containerisation | Docker + docker-compose | Self-hosted deployment is a core differentiator. One compose file: postgres, redis, api, worker, web. |
| Testing | pytest + pytest-asyncio + httpx + testcontainers | Unit + integration against a real ephemeral Postgres (JSONB/GIN behaviour cannot be faithfully mocked). Playwright for web E2E. |
| Code quality | ruff (lint+format), mypy (strict), pre-commit | Fast, single-tool lint+format for Python; mypy enforces typed contracts around JSONB. Frontend: eslint + prettier + tsc. |
| Package manager | uv (Python), pnpm (web) | uv for fast, reproducible Python installs; pnpm for the web workspace. |
| Auth | OAuth 2.0 + OIDC (Authlib), JWT sessions, PKCE for the developer API | `standards.md`: UK LIS/NLIS use OAuth 2.0; developer API must support delegated access with PKCE. |
| API spec / format | OpenAPI 3.1, REST/JSON, RFC 8288 Link headers for pagination | Matches every reviewed competitor API and the ICAR ADE conventions. |

### Project Structure

```
livestock-management-system/
├── pyproject.toml                  # uv-managed; ruff, mypy, pytest config
├── README.md
├── LICENSE                         # Apache-2.0 (matches ICAR ADE licence)
├── Dockerfile                      # multi-stage: api + worker image
├── docker-compose.yml              # postgres, redis, api, worker, web
├── docker-compose.dev.yml
├── .env.example
├── alembic.ini
├── migrations/                     # Alembic migration scripts
│   └── versions/
├── src/
│   └── lms/
│       ├── __init__.py
│       ├── main.py                 # FastAPI app factory, router mounting
│       ├── config.py               # Pydantic Settings (env-driven)
│       ├── db/
│       │   ├── engine.py           # async engine/session, RLS session vars
│       │   ├── models/             # SQLAlchemy table models
│       │   │   ├── tenant.py
│       │   │   ├── holding.py
│       │   │   ├── animal.py
│       │   │   ├── event.py
│       │   │   ├── set.py
│       │   │   ├── feed.py
│       │   │   ├── medicine.py
│       │   │   ├── breeding.py
│       │   │   ├── device.py
│       │   │   └── audit.py
│       │   └── seed/               # species, breed (ICAR), reference seeds
│       ├── schemas/                # Pydantic v2: API DTOs + JSONB payloads
│       │   ├── animal.py
│       │   ├── identifiers.py      # identifier schemes incl. ISO 11784
│       │   ├── events/             # one module per event_category
│       │   ├── jurisdiction/       # AU/UK/IE/US jurisdiction_data schemas
│       │   └── common.py           # pagination, error envelope
│       ├── services/               # business logic (framework-agnostic)
│       │   ├── animals.py
│       │   ├── events.py           # append + denormalisation hooks
│       │   ├── health.py           # withdrawal calc, status
│       │   ├── reproduction.py     # gestation/breeding-status state machine
│       │   ├── feed.py             # ration nutrition calc
│       │   ├── movements.py        # traceability, current-location sync
│       │   ├── financial.py
│       │   └── reporting.py
│       ├── api/
│       │   ├── deps.py             # auth, tenant context, db session
│       │   ├── pagination.py       # RFC 8288 Link headers
│       │   └── routes/             # one router per resource
│       ├── standards/
│       │   ├── iso11784.py         # RFID code encode/decode
│       │   ├── icar.py             # ICAR ADE serialisation
│       │   └── epcis.py            # EPCIS 2.0 ObjectEvent export
│       ├── integrations/
│       │   ├── base.py             # TraceabilityProvider protocol
│       │   ├── uk_lis.py
│       │   ├── nlis_au.py
│       │   └── rfid_readers.py
│       ├── ai/
│       │   ├── llm.py              # LLMClient abstraction
│       │   ├── nl_query.py         # NL → structured query + answer
│       │   ├── health_predict.py   # sklearn model train/infer
│       │   ├── breeding_advisor.py
│       │   └── ration_optimiser.py
│       ├── mcp/
│       │   └── server.py           # MCP tools over the service layer
│       ├── workers/
│       │   ├── celery_app.py
│       │   └── tasks/              # ingestion, projections, AI, submissions
│       └── cli/
│           └── main.py             # import/export/admin (typer)
├── tests/
│   ├── conftest.py                 # testcontainers Postgres fixture
│   ├── unit/
│   ├── integration/
│   ├── e2e/
│   └── fixtures/                   # sample herds, ICAR sample payloads
└── web/                            # Next.js app (pnpm workspace)
    ├── package.json
    ├── app/
    ├── components/
    └── lib/api/                    # generated client from OpenAPI spec
```

---

## Phase 1: Foundation — Project Skeleton, Config, Database, Multi-Tenancy

### Purpose
Stand up the runnable skeleton: dependency management, configuration, an async Postgres connection with multi-tenant isolation, the migration toolchain, and the health-check endpoint. After this phase the app boots, connects to a real database, applies migrations, and enforces tenant scoping — every later phase builds on this spine.

### Tasks

#### 1.1 — Project scaffolding and tooling

**What**: Create the repository skeleton, dependency manifest, lint/format/type config, and Docker setup so the app builds and runs.

**Design**:
- `pyproject.toml` with dependencies: `fastapi`, `uvicorn[standard]`, `sqlalchemy[asyncio]>=2.0`, `asyncpg`, `alembic`, `pydantic>=2`, `pydantic-settings`, `celery`, `redis`, `authlib`, `anthropic`, `mcp`, `typer`; dev: `pytest`, `pytest-asyncio`, `httpx`, `testcontainers[postgresql]`, `ruff`, `mypy`.
- `config.py`:
  ```python
  class Settings(BaseSettings):
      database_url: str
      redis_url: str = "redis://localhost:6379/0"
      jwt_secret: str
      anthropic_api_key: str | None = None
      llm_model: str = "claude-opus-4-8"
      environment: Literal["dev", "test", "prod"] = "dev"
      model_config = SettingsConfigDict(env_prefix="LMS_", env_file=".env")
  ```
- `main.py`: `create_app() -> FastAPI` factory mounting routers and a `GET /healthz` returning `{"status": "ok", "db": <bool>, "version": <str>}`.
- `Dockerfile` (multi-stage, uv install) and `docker-compose.yml` (postgres:16, redis:7, api, worker, web).
- `ruff` and `mypy --strict` configured in `pyproject.toml`.

**Testing**:
- `Unit: Settings loads from env vars with LMS_ prefix → correct typed values`.
- `Unit: Settings missing database_url → ValidationError naming the field`.
- `Integration: GET /healthz with DB up → 200, {"status":"ok","db":true}`.
- `Integration: GET /healthz with DB down → 503, {"db":false}`.
- `Smoke: docker compose build succeeds; api container responds on /healthz`.

#### 1.2 — Database engine, sessions, and multi-tenant RLS

**What**: Async SQLAlchemy engine and session dependency that sets the Postgres RLS tenant variable per request.

**Design**:
- `engine.py` exposes `get_session()` (FastAPI dependency) yielding an `AsyncSession`. On checkout it runs `SET LOCAL app.current_tenant = :tenant_id` so RLS policies (added in 1.3) isolate rows.
- A `TenantContext` dataclass (`tenant_id: UUID`, `user_id: UUID`, `role: str`) resolved by an auth dependency (stubbed here, real in Phase 7).
- Helper `jsonb_contains(column, value)` building `column @> :value` for GIN-indexed lookups.

**Testing**:
- `Integration (testcontainers): two tenants insert rows; session scoped to tenant A cannot read tenant B's rows`.
- `Unit: jsonb_contains produces the expected SQL fragment and bound param`.

#### 1.3 — Core schema migrations: tenants, users, holdings, paddocks

**What**: First Alembic migration creating multi-tenancy, identity, and holdings tables with RLS policies and GIN indexes.

**Design**: Implement the `tenants`, `users`, `holdings`, `paddocks` tables from Data Model Suggestion 3 (typed core columns + `settings`/`jurisdiction_ids`/`address`/`attributes` JSONB, GIN index on `holdings.jurisdiction_ids`). Add RLS:
```sql
ALTER TABLE holdings ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON holdings
  USING (tenant_id = current_setting('app.current_tenant')::uuid);
```
`tenants.settings` JSONB validated by a Pydantic `TenantSettings` model (`weight_unit`, `currency`, `species_enabled: list[str]`, `compliance_regimes: list[str]`, `custom_statuses: list[str]`).

**Testing**:
- `Integration: alembic upgrade head then downgrade base runs clean`.
- `Unit: TenantSettings defaults (weight_unit="kg", currency="USD", species_enabled=[]) applied`.
- `Integration: insert holding with AU jurisdiction_ids {"pic":"3ABCD123"} → GIN containment query @> '{"pic":"3ABCD123"}' returns it`.
- `Integration: RLS blocks cross-tenant holding reads`.

---

## Phase 2: Animal Records & Identification (Core Value)

### Purpose
Deliver the heart of the product: individual animal records with multi-scheme identification (including ISO 11784 RFID), multi-species attributes, parentage, and CRUD over a tenant-scoped REST API. This is the minimum the product is for — everything else hangs off the animal record.

### Tasks

#### 2.1 — Animals schema and Pydantic contracts

**What**: Migration for the `animals` table plus typed Pydantic models for every JSONB column.

**Design**: Implement the `animals` table from Suggestion 3 (typed: `species`, `management_tag`, `sex`, `date_of_birth`, `dam_id`, `sire_id`, `primary_breed`, `status`, `current_holding_id`, denormalised `latest_weight_*`; JSONB: `identifiers`, `breed_detail`, `species_attributes`, `jurisdiction_data`, `breed_society`, `custom_fields`) with all listed B-tree and GIN indexes. Pydantic payload models:
```python
class Identifier(BaseModel):
    scheme: Literal["ISO_11784","EU_EAR_TAG","NLIS_DEVICE","TATTOO","BRAND","MANAGEMENT_TAG","BREED_SOCIETY","USDA_AIN"]
    value: str
    is_primary: bool = False
    applied_date: date | None = None
    # ISO 11784 decomposed (validated when scheme == ISO_11784)
    country_code: str | None = None   # ISO 3166 numeric, 3 digits
    national_id: str | None = None    # 12 digits

class AnimalCreate(BaseModel):
    species: SpeciesCode
    sex: Literal["male","female","unknown"]
    management_tag: str | None = None
    date_of_birth: date | None = None
    dam_id: UUID | None = None
    sire_id: UUID | None = None
    primary_breed: str | None = None
    identifiers: list[Identifier] = []
    species_attributes: dict[str, Any] = {}
    jurisdiction_data: dict[str, Any] = {}
```
A model validator rejects more than one `is_primary` identifier and validates ISO 11784 fields when present.

**Testing**:
- `Unit: AnimalCreate with two is_primary identifiers → ValidationError`.
- `Unit: ISO_11784 identifier with 11-digit national_id → ValidationError`.
- `Integration: insert animal with species_attributes {"horn_status":"polled"} → round-trips intact`.

#### 2.2 — ISO 11784/11785 RFID module

**What**: Encode/decode the 64-bit ISO 11784 animal identification code.

**Design**: `standards/iso11784.py`:
```python
@dataclass(frozen=True)
class RfidCode:
    country_code: int      # ISO 3166 numeric (1..999) or manufacturer code
    national_id: int       # up to 12 digits
    animal_flag: bool      # bit 1: 1 = animal application
def decode(raw: str | int) -> RfidCode: ...   # from 15-digit decimal or 64-bit
def encode(code: RfidCode) -> str: ...          # canonical 15-digit string
def is_valid(raw: str) -> bool: ...
```
Decoding splits the standard 15-digit decimal form into 3-digit country + 12-digit national ID and sets the animal flag; reject out-of-range values.

**Testing**:
- `Unit: decode("826000012345678") → country_code=826, national_id=12345678, animal_flag=True`.
- `Unit: encode(decode(x)) == x for a table of fixture codes (round-trip)`.
- `Unit: decode of 14-digit string → ValueError`.

#### 2.3 — Animal CRUD API

**What**: REST endpoints for create/read/update/list/archive animals, OpenAPI-documented.

**Design**:
| Method | Path | Body / Query | Response |
|--------|------|--------------|----------|
| POST | `/v1/animals` | `AnimalCreate` | `201 AnimalRead` |
| GET | `/v1/animals/{id}` | — | `200 AnimalRead` / `404` |
| PATCH | `/v1/animals/{id}` | `AnimalUpdate` | `200 AnimalRead` |
| GET | `/v1/animals` | `species,status,holding_id,tag,rfid,page,page_size` | `200 list` + `Link` header |
| POST | `/v1/animals/{id}/archive` | — | `200` (status→archived) |

`rfid` filter uses `identifiers @> [{"scheme":"ISO_11784","value":<v>}]`. Errors use a shared envelope `{"error":{"code","message","field"}}`. `dam_id`/`sire_id` validated to exist in-tenant and not be self-referential.

**Testing**:
- `Integration: POST valid animal → 201, body has generated id and echoes identifiers`.
- `Integration: GET by id, wrong tenant → 404 (RLS)`.
- `Integration: list filtered by rfid → returns only the matching animal`.
- `Integration: list returns RFC 8288 Link: rel="next" when more pages exist`.
- `Integration: PATCH dam_id = self → 422`.

#### 2.4 — Species & breed reference data (ICAR)

**What**: Seed species and ICAR breed codes; expose read-only reference endpoints.

**Design**: `ref_species` and `ref_breeds` seed loaded from a committed JSON derived from ICAR breed coding (`db/seed/breeds.json`). Endpoints `GET /v1/ref/species`, `GET /v1/ref/breeds?species=cattle`. Cached in Redis.

**Testing**:
- `Integration: seed loads ≥ the 7 README species; GET /v1/ref/species returns them`.
- `Integration: GET /v1/ref/breeds?species=cattle includes "AA" Aberdeen Angus`.
- `Unit: breeds.json validates against the BreedRef Pydantic model`.

---

## Phase 3: Event Engine — Health, Weight, Reproduction, Movement

### Purpose
Add the unified `animal_events` table and the service layer that appends events and updates denormalised animal state. This phase delivers health logging, vaccination/treatment with withdrawal tracking, weighing, breeding records, and movement/traceability — the bulk of the MVP feature set — through one consistent event mechanism.

### Tasks

#### 3.1 — Unified event table and append service

**What**: Migration for `animal_events` plus an `append_event()` service with per-category validation and denormalisation hooks.

**Design**: Implement `animal_events` from Suggestion 3 (`event_category`, `event_type`, `event_date`, `holding_id`, `recorded_by`, `detail` JSONB, `source`, GIN index on `detail`; bi-temporal: keep `event_date` = occurred time and `created_at` = recorded time). Service:
```python
async def append_event(s, ctx, animal_id, category, type, event_date, detail: BaseModel, source="manual") -> EventRead
```
A registry maps `(category, type) → Pydantic detail model`; the raw `detail` is validated before insert. After insert, dispatch to a denormalisation hook (weight→`latest_weight_*`, status_change→`status`, movement→`current_holding_id`).

**Testing**:
- `Unit: append_event with detail not matching the registered model → ValidationError`.
- `Integration: append weigh event → animal.latest_weight_kg/date updated`.
- `Integration: unknown (category,type) pair → 422`.

#### 3.2 — Health events: diagnosis, treatment, vaccination, withdrawal

**What**: Health event types with automatic meat/milk withdrawal end-date calculation.

**Design**: Detail models `DiagnosisDetail`, `TreatmentDetail` (medicine ref, dose, route, batch, `withdrawal`), `HealthStatusDetail`. On a treatment, compute `withdrawal.meat_end_date = event_date + medicine.default_withdrawal_meat_days` (override allowed) and likewise for milk; store in `detail.withdrawal`. Endpoint `GET /v1/animals/{id}/health` (timeline) and `GET /v1/animals/under-withdrawal` (uses the documented withdrawal query).

**Testing**:
- `Unit: treatment with medicine meat_days=8, event 2025-07-12 → meat_end_date 2025-07-20`.
- `Integration: under-withdrawal lists animals with meat_end_date >= today only`.
- `Integration: vaccination event appears in /health timeline ordered by date desc`.

#### 3.3 — Weight & measurement events

**What**: Weigh, condition-score, and carcass events with daily-gain computation.

**Design**: `WeightDetail` (`weight_kg`, `method`, `device_id`, `condition_score`). On append, look up the previous weigh event and compute `daily_gain_kg = (new-prev)/days` into `detail`. `GET /v1/animals/{id}/weights` returns the growth series.

**Testing**:
- `Unit: 310kg on 2025-05-01 → 342.5kg on 2025-06-01 → daily_gain ≈ 1.05`.
- `Integration: first-ever weigh has daily_gain null and updates latest_weight_*`.

#### 3.4 — Reproduction events and breeding-status state machine

**What**: Heat, insemination, pregnancy-check, parturition, abortion, dry-off events driving a derived breeding status.

**Design**: State machine over `species_attributes.breeding_status`:
`open → inseminated → (pregnancy_check pregnant) pregnant → (parturition) lactating/open → (dry_off) dry`; `pregnancy_check not_pregnant` → `open`; `abortion` → `open`; `do_not_breed` is terminal until cleared. Parturition with `offspring[]` optionally creates offspring animal records linking `dam_id`/`sire_id`. `expected_due_date` set from species gestation table on a confirmed pregnancy.

**Testing**:
- `Unit: open + insemination → inseminated; + pregnancy_check pregnant → pregnant with due date`.
- `Unit: pregnant + parturition → lactating, offspring created with dam link`.
- `Unit: inseminated + pregnancy_check not_pregnant → open`.

#### 3.5 — Movement & traceability events

**What**: Birth, arrival, departure, death, internal-transfer events that maintain authoritative location history and current location.

**Design**: `MovementDetail` (`from_holding_id`, `to_holding_id`, `transport_ref`, `movement_permit`, `envd_number`, `epcis{biz_step,disposition}`). Appending updates `animals.current_holding_id/current_paddock_id`. `GET /v1/animals/{id}/movements`; `GET /v1/holdings/{id}/residents?at=<date>` returns animals resident on a date for disease-trace windows.

**Testing**:
- `Integration: departure H1→H2 then arrival at H2 → current_holding_id == H2`.
- `Integration: residents?at=<window date> returns animals present on that date but not those moved out earlier`.
- `Integration: death event sets status=deceased and status_date`.

---

## Phase 4: Feed, Financial & Reporting

### Purpose
Round out the MVP table-stakes features: feed/nutrition management with ration nutrition calculation, financial transactions per animal/group, and operational + compliance reports. After this phase the platform covers every "Must-have (MVP)" item in `features.md`.

### Tasks

#### 4.1 — Feeds, rations, and nutrition calculation

**What**: Feed reference data, ration composition, and computed ration nutrition.

**Design**: `feeds` (typed core + `nutrition` JSONB) and `rations` (`ingredients` JSONB array + `calculated_nutrition`). Service `compute_ration_nutrition(ration)` aggregates DM-weighted protein/ME and cost-per-head-day from ingredient feeds. `feed`/`group_feed` events recorded via the event engine.

**Testing**:
- `Unit: ration of barley/hay/mineral → calculated crude_protein_pct and cost_per_head_day match hand-computed fixture`.
- `Integration: group_feed event for an animal_set records and totals cost`.

#### 4.2 — Financial transactions

**What**: Purchase/sale/cost records linked to animals or sets, with per-animal P&L rollup.

**Design**: `financial_transactions` table (typed `transaction_type`, `amount`, `currency_code`, `event_date`, optional `animal_id`/`animal_set_id`). Service `animal_pl(animal_id)` sums purchase, sale, treatment, feed, and other costs. Endpoint `GET /v1/animals/{id}/financials`.

**Testing**:
- `Unit: purchase 800 + feed 120 + sale 1850 → net_profit 930`.
- `Integration: sale transaction created when a sale event is appended (cross-link)`.

#### 4.3 — Reporting & compliance exports

**What**: Operational and regulatory reports: herd inventory, movement register, medicine/withdrawal register, and a holding register report.

**Design**: `services/reporting.py` produces CSV/JSON. `GET /v1/reports/inventory`, `/reports/movements?from&to`, `/reports/medicines`. Reports assemble from events without manual re-entry (the compliance-automation differentiator). Output column sets aligned to EU Reg 1760/2000 holding-register fields where applicable.

**Testing**:
- `Integration: inventory report counts active animals per holding correctly`.
- `Integration: movements report for a date range returns only in-range movements as CSV with expected headers`.

---

## Phase 5: Standards & Interoperability — ICAR ADE, EPCIS, OpenAPI

### Purpose
Make the platform interoperable: serialise core records to the ICAR ADE format, export movement events as GS1 EPCIS 2.0 ObjectEvents, and publish a clean OpenAPI 3.1 spec. This unlocks integration with existing farm software, national databases, and supply-chain partners — the stated integration backbone.

### Tasks

#### 5.1 — ICAR ADE serialisation

**What**: Map internal animal and event records to ICAR ADE JSON resources and import them back.

**Design**: `standards/icar.py` with `to_icar_animal(animal) -> dict` and `to_icar_event(event) -> dict` matching ICAR ADE resource shapes (`icarAnimalCoreResource`, `icarWeightEventResource`, etc.), plus `from_icar_animal(dict) -> AnimalCreate`. Endpoints `GET /v1/animals/{id}/icar` and `POST /v1/import/icar`.

**Testing**:
- `Fixture: committed ICAR sample resource imports → animal created with matching fields`.
- `Unit: to_icar_animal output validates against the committed ICAR JSON schema`.
- `Round-trip: from_icar(to_icar(animal)) preserves identity fields`.

#### 5.2 — GS1 EPCIS 2.0 export

**What**: Export movement events as EPCIS 2.0 ObjectEvents in JSON-LD.

**Design**: `standards/epcis.py` `movement_to_object_event(event, primary_id) -> dict` producing `{type:"ObjectEvent", eventTime, recordTime, bizStep, disposition, epcList:[urn...]}` using the stored `detail.epcis` fields and the animal's primary identifier. `GET /v1/reports/epcis?from&to` returns an EPCIS document.

**Testing**:
- `Unit: a departure event → ObjectEvent with bizStep "shipping" and the animal's RFID urn in epcList`.
- `Integration: EPCIS export over a range validates as a well-formed EPCIS 2.0 JSON document`.

#### 5.3 — OpenAPI 3.1 spec & generated web client

**What**: Curate the auto-generated OpenAPI spec and generate a typed TS client for the web app.

**Design**: Tag/group routes, add examples, ensure the error envelope and pagination are documented. Commit `openapi.json`; generate `web/lib/api/` via `openapi-typescript`.

**Testing**:
- `Test: generated openapi.json is valid OpenAPI 3.1 (schema validation)`.
- `Test: every route has a summary and at least one response example (lint script)`.

---

## Phase 6: Web Application (Mobile-First Dashboard)

### Purpose
Provide the producer-facing UI demanded by `features.md` (mobile-first in-field capture): authentication, herd inventory, animal detail with event timelines, and quick-entry flows for weighing, treatments, and movements.

### Tasks

#### 6.1 — App shell, auth, and herd inventory

**What**: Next.js shell with login, tenant context, and a filterable/paginated animal list.

**Design**: Server components for the inventory list calling the typed API client; filter bar (species/status/holding/tag/rfid); responsive card/table layout. Auth via the OIDC flow (Phase 7) with a dev stub initially.

**Testing**:
- `E2E (Playwright): log in → inventory lists seeded animals; filter by species narrows results`.
- `E2E: pagination Next loads page 2`.

#### 6.2 — Animal detail & event timelines

**What**: Animal profile with tabs for health, weight (chart), reproduction, movements, financials.

**Design**: Each tab fetches the corresponding timeline endpoint; weight tab renders the growth series; withdrawal status banner when under withdrawal.

**Testing**:
- `E2E: open an animal with a treatment under withdrawal → withdrawal banner shows the end date`.
- `E2E: weight tab renders ≥2 points and a daily-gain figure`.

#### 6.3 — Quick-entry capture flows

**What**: Mobile-optimised forms to record a weigh, treatment, and movement in the field, including RFID-tag lookup.

**Design**: A search-by-RFID/tag box resolves the animal then opens the relevant quick-entry sheet; optimistic UI; forms post to the event endpoints.

**Testing**:
- `E2E: scan/enter RFID → animal resolves → record weigh → appears in timeline`.
- `E2E: record treatment → withdrawal banner appears immediately`.

---

## Phase 7: Authentication, Authorisation & Developer API

### Purpose
Replace the auth stub with production OAuth 2.0 / OIDC, role-based authorisation, and a PKCE-protected developer API — required for the multi-tenant SaaS mode and for the government integrations (UK LIS, NLIS) that mandate OAuth.

### Tasks

#### 7.1 — OIDC login & JWT sessions

**What**: Real authentication via an OIDC provider with JWT-backed sessions and tenant resolution.

**Design**: Authlib OIDC client; on callback, resolve/provision the user and set `TenantContext`. Short-lived access JWT + refresh. RLS tenant variable set from the verified token.

**Testing**:
- `Integration (mocked OIDC): valid code exchange → session issued, TenantContext populated`.
- `Integration: expired/invalid JWT → 401`.

#### 7.2 — Role-based authorisation

**What**: Enforce roles (`owner`, `manager`, `operator`, `viewer`, `veterinarian`) across endpoints.

**Design**: `require_role(*roles)` dependency. Viewer is read-only; veterinarian may write health events but not financials; operator may record events but not manage users.

**Testing**:
- `Integration: viewer POST /v1/animals → 403`.
- `Integration: veterinarian POST health event → 201; POST financial → 403`.

#### 7.3 — Developer API with PKCE & API keys

**What**: Third-party delegated access via OAuth 2.0 Authorization Code + PKCE, plus server-to-server API keys.

**Design**: PKCE authorize/token endpoints; scoped API keys (`animals:read`, `events:write`, ...). Rate limiting via Redis. Mirrors the UK LIS integrator pattern in `standards.md`.

**Testing**:
- `Integration: PKCE flow with valid verifier → token; wrong verifier → 400`.
- `Integration: API key lacking events:write → 403 on event create`.
- `Integration: requests over the rate limit → 429`.

---

## Phase 8: Sensors, RFID Readers & Async Ingestion

### Purpose
Enable hardware integration and high-volume telemetry: device registration, a sensor-reading ingestion pipeline, and RFID-reader bulk capture — the data foundation for the AI health features.

### Tasks

#### 8.1 — Device registry & sensor ingestion

**What**: Register devices and ingest sensor readings into the time-series table via Celery.

**Design**: `devices` and `sensor_readings` (PK `(device_id, time)`, `metrics` JSONB) per Suggestion 3, with monthly partitioning. `POST /v1/devices/{id}/readings` enqueues a Celery task that bulk-inserts and updates `devices.last_reading_at`. Optional alert events on threshold breaches from `devices.config`.

**Testing**:
- `Integration: POST a batch of readings → task inserts rows; query by animal+time range returns them`.
- `Unit: reading exceeding a configured temperature threshold → alert event enqueued`.

#### 8.2 — RFID reader bulk capture

**What**: Accept a batch of RFID reads from a yard/race reader and resolve them to animals, recording presence/weigh sessions.

**Design**: `POST /v1/readers/{id}/session` with a list of RFID codes; resolve via the identifiers GIN index; unknown codes returned for review; optionally pair with a connected scale to create weigh events.

**Testing**:
- `Integration: session with 3 known + 1 unknown RFID → 3 resolved, 1 in unresolved list`.
- `Integration (mocked scale): paired read+weight → weigh events created for resolved animals`.

---

## Phase 9: AI-Native Features

### Purpose
Deliver the differentiating AI layer: predictive health alerts, breeding recommendations, feed-ration optimisation, and a natural-language herd query interface. These build on the event history (Phase 3), sensor data (Phase 8), and breeding values.

### Tasks

#### 9.1 — LLM client abstraction & NL herd query

**What**: A natural-language interface that answers questions across herd records.

**Design**: `ai/llm.py` `LLMClient` protocol (`complete`, `tool_call`) with an Anthropic implementation. `ai/nl_query.py` exposes herd data to the model as safe, tenant-scoped tool functions (`find_animals`, `get_timeline`, `aggregate`) — the model plans tool calls; raw SQL is never generated by the model. System prompt template:
```
You are a livestock herd assistant. Answer using ONLY the provided tools,
which are scoped to the current tenant. Cite animal tags/ids in answers.
Never invent records. If data is insufficient, say so.
```
Endpoint `POST /v1/ai/query {question}` → `{answer, citations[], tool_trace[]}`.

**Testing**:
- `Integration (mocked LLM): "how many pregnant cows?" → model calls aggregate tool → answer reflects fixture count`.
- `Unit: tools refuse cross-tenant ids → error surfaced, no data leak`.
- `Integration: ambiguous question with no data → answer states insufficient data`.

#### 9.2 — Predictive health alerts

**What**: Flag animals likely to be ill or in heat before clinical signs, from sensor + history features.

**Design**: `ai/health_predict.py` builds a feature frame (rolling activity, rumination, temperature deltas, days-in-milk, recent treatments) and trains a scikit-learn classifier; a daily Celery task scores active animals and raises `sensor`/`alert` events with a probability and contributing factors. Cold-start uses rule thresholds until enough labelled data exists.

**Testing**:
- `Unit: feature builder produces expected columns from a fixture sensor series`.
- `Unit: model train+predict on synthetic separable data → ROC-AUC > 0.9`.
- `Integration: scoring task raises alert events for animals above the threshold`.

#### 9.3 — Breeding advisor & ration optimiser

**What**: Recommend mating pairs from genetic merit/health/performance, and optimise rations to nutrient targets at least cost.

**Design**: `breeding_advisor.recommend(dam_id) -> ranked sires` combining breeding values, inbreeding avoidance (shared-ancestor check via parentage), and trait targets. `ration_optimiser.optimise(targets, available_feeds) -> mix` as a linear program (scipy) minimising cost subject to protein/ME/DM constraints. Endpoints under `/v1/ai/breeding` and `/v1/ai/ration`.

**Testing**:
- `Unit: optimiser meets min protein/ME at minimal cost vs a brute-force fixture`.
- `Unit: advisor down-ranks a sire sharing a recent ancestor with the dam`.

#### 9.4 — MCP server

**What**: Expose herd queries to external AI agents over the Model Context Protocol.

**Design**: `mcp/server.py` registers tools (`find_animals`, `animal_timeline`, `herd_summary`, `under_withdrawal`) backed by the same service layer and auth/tenant scoping as the API. Ships as a separate entrypoint in the Docker image.

**Testing**:
- `Integration (MCP test client): list tools → expected tool set; call find_animals → tenant-scoped results`.
- `Integration: call without a valid tenant token → rejected`.

---

## Phase 10: Regulatory Integrations & Hardening

### Purpose
Connect to mandatory government traceability systems and prepare for production: UK LIS and Australian NLIS submission with retries, GDPR data-subject tooling, and operational hardening.

### Tasks

#### 10.1 — Traceability provider framework + UK LIS & NLIS

**What**: Submit movements/births/deaths to UK LIS and Australian NLIS with a pluggable provider interface.

**Design**: `integrations/base.py`:
```python
class TraceabilityProvider(Protocol):
    async def submit_movement(self, movement: MovementDetail, animal: Animal) -> SubmissionResult: ...
    async def fetch_status(self, ref: str) -> SubmissionStatus: ...
```
UK LIS via OAuth 2.0/Azure B2C + subscription key; NLIS via registered developer access with a sandbox toggle. Submissions run as Celery tasks with idempotency keys and exponential-backoff retry; results stored against the source event.

**Testing**:
- `Integration (mocked LIS sandbox): submit movement → submission recorded with provider ref`.
- `Integration: transient 503 → task retries then succeeds; status updated`.
- `Unit: idempotency key prevents duplicate submission of the same movement`.

#### 10.2 — GDPR & audit tooling

**What**: Data-subject export/erasure and a complete audit trail.

**Design**: `audit_log` (JSONB before/after) written via a SQLAlchemy event hook on mutating operations. CLI/endpoints for tenant data export (JSON bundle) and operator-PII erasure (animal records de-identified, not deleted, to preserve traceability lawful basis).

**Testing**:
- `Integration: updating an animal writes an audit_log row with old/new diff`.
- `Integration: erasure request removes operator PII but retains animal traceability records`.

#### 10.3 — Production hardening

**What**: Observability, backups, and load validation.

**Design**: Structured JSON logging with correlation ids; `/metrics` (Prometheus); DB backup job; rate-limit and pagination defaults reviewed; sensor table partition rollover task.

**Testing**:
- `Load: list endpoint stays <300ms p95 at 100k animals/tenant (seeded)`.
- `Integration: correlation id propagates request → log → Celery task`.
- `Smoke: backup + restore reproduces a tenant's data`.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation                ─── required by everything
    │
Phase 2: Animal Records & ID       ─── requires Phase 1
    │
Phase 3: Event Engine              ─── requires Phase 2
    ├── Phase 4: Feed/Financial/Reporting   ─── requires Phase 3
    ├── Phase 5: Standards (ICAR/EPCIS/OAS)  ─── requires Phase 3 (parallel with 4)
    └── Phase 8: Sensors & RFID Ingestion    ─── requires Phase 3 (parallel with 4,5)
         │
Phase 6: Web App                   ─── requires Phase 2,3 (best after 5 for typed client)
Phase 7: Auth & Developer API      ─── requires Phase 1,2 (parallel with 4,5,8)
    │
Phase 9: AI-Native Features        ─── requires Phase 3 (9.2 also needs Phase 8)
    │
Phase 10: Regulatory & Hardening   ─── requires Phase 3,5,7
```

**Parallelism opportunities:**
- After **Phase 3**, Phases **4, 5, and 8** can be developed concurrently (independent feature areas over the same event engine).
- **Phase 7** (auth) can proceed in parallel with 4/5/8 once Phases 1–2 are done; the web app (Phase 6) consumes both the typed client from Phase 5 and auth from Phase 7.
- Within **Phase 9**, 9.1 (NL query/MCP) and 9.3 (breeding/ration) need only Phase 3; 9.2 (predictive health) additionally needs Phase 8.

---

## Definition of Done (per phase)

1. All tasks in the phase implemented.
2. All unit and integration tests pass (`pytest`), including testcontainers-backed Postgres tests.
3. `ruff check` and `ruff format --check` pass; web: `eslint` + `tsc` pass.
4. `mypy --strict` passes for `src/lms`.
5. `docker compose build` succeeds and the affected services start cleanly.
6. The phase's primary capability works end-to-end (API and, where relevant, web E2E via Playwright).
7. New config options documented in `.env.example` and README.
8. New/changed API endpoints appear in the generated `openapi.json` with summaries and examples.
9. Alembic migration(s) created, and `upgrade head` / `downgrade` run clean.
10. New JSONB payload shapes have a corresponding Pydantic model and round-trip test (the hybrid model's validation contract).
11. Multi-tenant isolation verified (no cross-tenant read/write) for any new tenant-scoped table.
```
