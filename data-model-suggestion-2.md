# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Livestock Management System · Created: 2026-05-24

## Philosophy

This model treats every change to an animal's state as an immutable event appended to a central event store. The event store is the single source of truth. Read-optimized materialized views (projections) are rebuilt from events to serve queries. This is a Command Query Responsibility Segregation (CQRS) architecture where writes go to the event store and reads come from projections.

The design is rooted in the observation that livestock management is inherently event-driven: an animal is born (BirthEvent), moves to a new paddock (MovementEvent), receives a vaccination (TreatmentEvent), is weighed (WeightEvent), is diagnosed (DiagnosisEvent), is inseminated (InseminationEvent), calves (ParturitionEvent), and eventually is sold or dies (DisposalEvent). Each of these events is a fact that happened at a specific time and should never be deleted or modified -- only corrected by appending a new correction event. This aligns perfectly with regulatory traceability requirements (EU Regulation 1760/2000, NLIS, USDA ADT) where the complete history of what was recorded and when is legally mandated.

The approach is used by financial ledger systems, healthcare EHR systems, and supply chain platforms (GS1 EPCIS is itself an event-sourced standard). For livestock, it enables temporal queries ("what was the status of animal X on date Y?"), complete audit trails without a separate audit mechanism, and the ability to replay events to build new analytics projections without modifying the source data.

**Best for:** Operations where full regulatory audit trails are mandatory, where temporal queries are important (e.g., "show me all animals that were on this property during the disease outbreak window"), and where AI analytics benefit from the complete event history.

**Trade-offs:**
- Pro: Complete, immutable audit trail is inherent -- no separate audit mechanism needed
- Pro: Temporal queries are natural ("what was true at time T?")
- Pro: New read models can be added without changing the write schema
- Pro: Perfect alignment with GS1 EPCIS event model for supply chain traceability
- Pro: AI/ML pipelines can consume the raw event stream for pattern detection
- Con: Higher storage requirements (events are never deleted)
- Con: Read model consistency is eventually consistent (projection rebuild lag)
- Con: More complex application layer -- developers must think in events, not CRUD
- Con: Projection rebuild can be slow for large herds (100k+ lifetime events)
- Con: Simple queries ("list all active animals") require maintained projections rather than direct table scans
- Con: Correction/amendment workflow adds complexity (correction events, not UPDATEs)

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ICAR ADE 1.5 | Each ICAR event resource type maps directly to an event_type in the event store |
| GS1 EPCIS 2.0 | The event store IS an EPCIS-compatible event log; events export directly as EPCIS ObjectEvents |
| ISO 11784/11785 | RFID identifiers stored in IdentifierAssignedEvent and IdentifierRemovedEvent payloads |
| ISO 22005 | End-to-end traceability achieved by replaying the event stream for any animal |
| EU Reg 1760/2000 | Bovine identity events (tag application, passport issue) are first-class events |
| NLIS Australia | Movement events carry PIC codes and NLIS device numbers in the event payload |
| ISO 3166 | Country and subdivision codes embedded in event payloads for holdings |

---

## Event Store (Write Side)

```sql
-- ============================================================
-- EVENT STORE — the single source of truth
-- ============================================================

-- Aggregate roots: the minimal identity records that events reference
CREATE TABLE aggregate_roots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    aggregate_type  TEXT NOT NULL
        CHECK (aggregate_type IN ('animal', 'holding', 'animal_set', 'device', 'medicine')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_aggregates_tenant ON aggregate_roots(tenant_id);
CREATE INDEX idx_aggregates_type ON aggregate_roots(tenant_id, aggregate_type);

-- The immutable event store
CREATE TABLE events (
    -- Event identity
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sequence_number BIGINT NOT NULL,   -- global ordering (use a SEQUENCE)
    
    -- Aggregate reference
    aggregate_id    UUID NOT NULL REFERENCES aggregate_roots(id),
    aggregate_type  TEXT NOT NULL,
    aggregate_version BIGINT NOT NULL,  -- per-aggregate version for optimistic concurrency
    
    -- Tenant isolation
    tenant_id       UUID NOT NULL,
    
    -- Event classification (aligned with ICAR ADE resource types)
    event_type      TEXT NOT NULL,
    /*
        Supported event_types:
        -- Identity & Registration
        'animal.registered', 'animal.identifier_assigned', 'animal.identifier_removed',
        'animal.breed_society_registered', 'animal.status_changed',
        -- Movement
        'animal.birth', 'animal.arrival', 'animal.departure', 'animal.death',
        'animal.internal_transfer', 'animal.import', 'animal.export',
        -- Health
        'animal.diagnosed', 'animal.treated', 'animal.vaccinated',
        'animal.health_status_observed', 'animal.withdrawal_started',
        'animal.withdrawal_cleared',
        -- Reproduction
        'animal.heat_detected', 'animal.inseminated', 'animal.pregnancy_checked',
        'animal.parturition', 'animal.abortion', 'animal.do_not_breed',
        'animal.embryo_flushed', 'animal.lactation_started', 'animal.dried_off',
        -- Weight & Measurement
        'animal.weighed', 'animal.condition_scored', 'animal.conformation_scored',
        'animal.carcass_observed',
        -- Feed
        'animal.fed', 'group.fed', 'animal.ration_assigned',
        -- Milking
        'animal.milked', 'animal.test_day_recorded',
        -- Genetics
        'animal.breeding_value_published',
        -- Sets / Groups
        'animal.joined_set', 'animal.left_set',
        -- Sensor
        'animal.sensor_reading', 'device.reading_received',
        -- Financial
        'animal.purchased', 'animal.sold', 'animal.cost_recorded',
        -- Corrections
        'event.corrected', 'event.voided',
        -- Holdings
        'holding.registered', 'holding.updated',
    */
    
    -- When the event occurred in the real world
    occurred_at     TIMESTAMPTZ NOT NULL,
    -- When the event was recorded in the system (bi-temporal)
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    
    -- The event payload — all domain-specific data
    payload         JSONB NOT NULL,
    
    -- Metadata
    recorded_by     UUID,           -- user who recorded the event
    source_system   TEXT,           -- 'web_ui', 'mobile_app', 'rfid_reader', 'sensor', 'api', 'import'
    causation_id    UUID,           -- the command/request that caused this event
    correlation_id  UUID,           -- links related events across aggregates
    
    -- For correction events: which event is being corrected
    corrects_event_id UUID REFERENCES events(event_id),
    
    -- Constraints
    UNIQUE (aggregate_id, aggregate_version)
);

-- Critical indexes for event replay and querying
CREATE INDEX idx_events_aggregate ON events(aggregate_id, aggregate_version);
CREATE INDEX idx_events_tenant_type ON events(tenant_id, event_type);
CREATE INDEX idx_events_occurred ON events(occurred_at);
CREATE INDEX idx_events_recorded ON events(recorded_at);
CREATE INDEX idx_events_sequence ON events(sequence_number);
CREATE INDEX idx_events_correlation ON events(correlation_id);
CREATE INDEX idx_events_type ON events(event_type);

-- GIN index on payload for JSONB containment queries
CREATE INDEX idx_events_payload ON events USING GIN (payload);

-- Sequence for global ordering
CREATE SEQUENCE event_sequence_seq;
```

### Example Event Payloads

```sql
-- animal.registered
-- {
--   "species": "cattle",
--   "breed_code": "AA",
--   "breed_name": "Aberdeen Angus",
--   "sex": "female",
--   "date_of_birth": "2024-03-15",
--   "dam_id": "550e8400-e29b-41d4-a716-446655440001",
--   "sire_id": "550e8400-e29b-41d4-a716-446655440002",
--   "management_tag": "A-142",
--   "name": null,
--   "birth_holding_id": "550e8400-e29b-41d4-a716-446655440010"
-- }

-- animal.identifier_assigned
-- {
--   "scheme": "ISO_11784",
--   "identifier": "826000012345678",
--   "rfid_country_code": "826",
--   "rfid_national_id": "000012345678",
--   "applied_date": "2024-03-16"
-- }

-- animal.weighed
-- {
--   "weight_kg": 342.5,
--   "method": "scale",
--   "device_id": "SCALE-001",
--   "condition_score": 3.5
-- }

-- animal.treated
-- {
--   "treatment_type": "vaccination",
--   "medicine_name": "Bovilis BVD",
--   "medicine_id": "550e8400-e29b-41d4-a716-446655440099",
--   "dose_amount": 2.0,
--   "dose_unit": "ml",
--   "route": "injection_sc",
--   "batch_number": "BVD-2024-0891",
--   "withdrawal_meat_days": 0,
--   "withdrawal_milk_days": 0,
--   "administered_by_name": "Dr. Sarah Jones"
-- }

-- animal.departure
-- {
--   "from_holding_id": "550e8400-e29b-41d4-a716-446655440010",
--   "from_holding_pic": "3ABCD123",
--   "to_holding_id": "550e8400-e29b-41d4-a716-446655440020",
--   "to_holding_pic": "3EFGH456",
--   "transport_ref": "CON-2024-0045",
--   "movement_permit_number": "MP-2024-1234",
--   "epcis_biz_step": "urn:epcglobal:cbv:bizstep:shipping",
--   "epcis_disposition": "urn:epcglobal:cbv:disp:in_transit"
-- }

-- animal.inseminated
-- {
--   "insemination_type": "artificial",
--   "sire_id": "550e8400-e29b-41d4-a716-446655440002",
--   "straw_id": "STR-AA-2024-0033",
--   "straw_bull_name": "Dunlouise New Design",
--   "straw_company": "Cogent Breeding",
--   "technician_name": "Jim Murray"
-- }

-- animal.parturition
-- {
--   "calving_ease": "unassisted",
--   "offspring": [
--     {
--       "offspring_id": "550e8400-e29b-41d4-a716-446655440200",
--       "sex": "female",
--       "birth_weight_kg": 38.2,
--       "condition": "alive"
--     }
--   ]
-- }

-- event.corrected
-- {
--   "reason": "Weight was recorded in lbs, should have been kg",
--   "original_event_type": "animal.weighed",
--   "corrected_payload": {
--     "weight_kg": 155.4,
--     "method": "scale"
--   }
-- }
```

## Projection Tables (Read Side)

```sql
-- ============================================================
-- PROJECTIONS — materialized read models rebuilt from events
-- ============================================================

-- Tracks which events each projection has processed
CREATE TABLE projection_checkpoints (
    projection_name TEXT PRIMARY KEY,
    last_sequence   BIGINT NOT NULL DEFAULT 0,
    last_updated    TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- PROJECTION: Current Animal State
-- ============================================================

CREATE TABLE proj_animals (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    species         TEXT NOT NULL,
    breed_code      TEXT,
    breed_name      TEXT,
    sex             TEXT NOT NULL,
    is_castrated    BOOLEAN NOT NULL DEFAULT false,
    date_of_birth   DATE,
    management_tag  TEXT,
    name            TEXT,
    dam_id          UUID,
    sire_id         UUID,
    status          TEXT NOT NULL DEFAULT 'active',
    status_date     DATE,
    death_date      DATE,
    death_cause     TEXT,
    -- Current location (updated by movement events)
    current_holding_id UUID,
    current_holding_name TEXT,
    current_paddock_id UUID,
    current_paddock_name TEXT,
    -- Latest measurements (updated by weight/condition events)
    latest_weight_kg NUMERIC(8,2),
    latest_weight_date DATE,
    latest_condition_score NUMERIC(3,1),
    -- Breeding status (updated by repro events)
    breeding_status TEXT,  -- 'open', 'inseminated', 'pregnant', 'lactating', 'dry', 'do_not_breed'
    expected_due_date DATE,
    current_lactation_number SMALLINT,
    -- Active identifiers (denormalized for quick lookup)
    primary_rfid    TEXT,
    primary_ear_tag TEXT,
    -- Withdrawal status
    under_meat_withdrawal BOOLEAN DEFAULT false,
    meat_withdrawal_end DATE,
    under_milk_withdrawal BOOLEAN DEFAULT false,
    milk_withdrawal_end DATE,
    -- Version tracking
    last_event_id   UUID,
    last_event_at   TIMESTAMPTZ,
    event_count     INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_animals_tenant ON proj_animals(tenant_id);
CREATE INDEX idx_proj_animals_status ON proj_animals(tenant_id, status);
CREATE INDEX idx_proj_animals_holding ON proj_animals(current_holding_id);
CREATE INDEX idx_proj_animals_rfid ON proj_animals(primary_rfid);
CREATE INDEX idx_proj_animals_tag ON proj_animals(tenant_id, management_tag);

-- ============================================================
-- PROJECTION: Animal Identifiers
-- ============================================================

CREATE TABLE proj_animal_identifiers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    animal_id       UUID NOT NULL,
    tenant_id       UUID NOT NULL,
    scheme          TEXT NOT NULL,
    identifier      TEXT NOT NULL,
    is_primary      BOOLEAN DEFAULT false,
    is_active       BOOLEAN DEFAULT true,
    applied_date    DATE,
    removed_date    DATE,
    UNIQUE (scheme, identifier)
);

CREATE INDEX idx_proj_identifiers_animal ON proj_animal_identifiers(animal_id);
CREATE INDEX idx_proj_identifiers_lookup ON proj_animal_identifiers(scheme, identifier);

-- ============================================================
-- PROJECTION: Movement History
-- ============================================================

CREATE TABLE proj_movement_history (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id        UUID NOT NULL,
    tenant_id       UUID NOT NULL,
    animal_id       UUID NOT NULL,
    movement_type   TEXT NOT NULL,
    occurred_at     TIMESTAMPTZ NOT NULL,
    from_holding_id UUID,
    from_holding_name TEXT,
    to_holding_id   UUID,
    to_holding_name TEXT,
    transport_ref   TEXT,
    movement_permit TEXT
);

CREATE INDEX idx_proj_movements_animal ON proj_movement_history(animal_id);
CREATE INDEX idx_proj_movements_date ON proj_movement_history(occurred_at);
CREATE INDEX idx_proj_movements_holding ON proj_movement_history(to_holding_id);

-- ============================================================
-- PROJECTION: Weight and Growth Tracking
-- ============================================================

CREATE TABLE proj_weight_history (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id        UUID NOT NULL,
    tenant_id       UUID NOT NULL,
    animal_id       UUID NOT NULL,
    weighed_at      TIMESTAMPTZ NOT NULL,
    weight_kg       NUMERIC(8,2) NOT NULL,
    condition_score NUMERIC(3,1),
    method          TEXT,
    daily_gain_kg   NUMERIC(6,3)  -- calculated from previous weight
);

CREATE INDEX idx_proj_weights_animal ON proj_weight_history(animal_id);
CREATE INDEX idx_proj_weights_date ON proj_weight_history(weighed_at);

-- ============================================================
-- PROJECTION: Health and Treatment Timeline
-- ============================================================

CREATE TABLE proj_health_timeline (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id        UUID NOT NULL,
    tenant_id       UUID NOT NULL,
    animal_id       UUID NOT NULL,
    event_type      TEXT NOT NULL,  -- 'diagnosis', 'treatment', 'vaccination', 'health_status'
    occurred_at     TIMESTAMPTZ NOT NULL,
    description     TEXT NOT NULL,
    medicine_name   TEXT,
    dose_amount     NUMERIC(10,3),
    dose_unit       TEXT,
    withdrawal_meat_end DATE,
    withdrawal_milk_end DATE,
    administered_by TEXT
);

CREATE INDEX idx_proj_health_animal ON proj_health_timeline(animal_id);
CREATE INDEX idx_proj_health_date ON proj_health_timeline(occurred_at);
CREATE INDEX idx_proj_health_withdrawal ON proj_health_timeline(withdrawal_meat_end);

-- ============================================================
-- PROJECTION: Reproduction Timeline
-- ============================================================

CREATE TABLE proj_repro_timeline (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id        UUID NOT NULL,
    tenant_id       UUID NOT NULL,
    animal_id       UUID NOT NULL,
    event_type      TEXT NOT NULL,  -- 'heat', 'insemination', 'pregnancy_check', 'parturition', 'abortion'
    occurred_at     TIMESTAMPTZ NOT NULL,
    sire_id         UUID,
    sire_name       TEXT,
    result          TEXT,          -- 'pregnant', 'not_pregnant', etc.
    calving_ease    TEXT,
    offspring_count SMALLINT,
    notes           TEXT
);

CREATE INDEX idx_proj_repro_animal ON proj_repro_timeline(animal_id);
CREATE INDEX idx_proj_repro_date ON proj_repro_timeline(occurred_at);

-- ============================================================
-- PROJECTION: Current Holdings
-- ============================================================

CREATE TABLE proj_holdings (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    name            TEXT NOT NULL,
    holding_type    TEXT NOT NULL,
    country_code    CHAR(2) NOT NULL,
    subdivision_code TEXT,
    pic_code        TEXT,
    cph_number      TEXT,
    herd_number     TEXT,
    address         TEXT,
    latitude        NUMERIC(10,7),
    longitude       NUMERIC(10,7),
    animal_count    INTEGER DEFAULT 0,
    last_event_at   TIMESTAMPTZ
);

CREATE INDEX idx_proj_holdings_tenant ON proj_holdings(tenant_id);

-- ============================================================
-- PROJECTION: Daily Milking Summary (Dairy)
-- ============================================================

CREATE TABLE proj_daily_milking (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    animal_id       UUID NOT NULL,
    milk_date       DATE NOT NULL,
    total_milk_kg   NUMERIC(8,2) NOT NULL DEFAULT 0,
    milking_count   SMALLINT NOT NULL DEFAULT 0,
    avg_fat_pct     NUMERIC(5,2),
    avg_protein_pct NUMERIC(5,2),
    max_scc         INTEGER,
    UNIQUE (animal_id, milk_date)
);

CREATE INDEX idx_proj_milking_animal ON proj_daily_milking(animal_id);
CREATE INDEX idx_proj_milking_date ON proj_daily_milking(milk_date);

-- ============================================================
-- PROJECTION: Financial Summary per Animal
-- ============================================================

CREATE TABLE proj_animal_financials (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    animal_id       UUID NOT NULL UNIQUE,
    purchase_price  NUMERIC(12,2),
    sale_price      NUMERIC(12,2),
    total_treatment_cost NUMERIC(12,2) DEFAULT 0,
    total_feed_cost NUMERIC(12,2) DEFAULT 0,
    total_other_cost NUMERIC(12,2) DEFAULT 0,
    total_income    NUMERIC(12,2) DEFAULT 0,
    net_profit_loss NUMERIC(12,2) DEFAULT 0,
    last_updated    TIMESTAMPTZ
);

CREATE INDEX idx_proj_financials_tenant ON proj_animal_financials(tenant_id);
```

## Reference Data Tables

```sql
-- ============================================================
-- REFERENCE DATA — shared, not event-sourced
-- ============================================================

CREATE TABLE ref_species (
    code            TEXT PRIMARY KEY,
    name            TEXT NOT NULL,
    icar_species_code TEXT
);

CREATE TABLE ref_breeds (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    species_code    TEXT NOT NULL REFERENCES ref_species(code),
    code            TEXT NOT NULL,
    name            TEXT NOT NULL,
    icar_breed_code TEXT,
    UNIQUE (species_code, code)
);

CREATE TABLE ref_medicines (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    name            TEXT NOT NULL,
    active_ingredient TEXT,
    manufacturer    TEXT,
    registration_number TEXT,
    default_withdrawal_meat_days INTEGER,
    default_withdrawal_milk_days INTEGER,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE ref_feeds (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    name            TEXT NOT NULL,
    feed_type       TEXT NOT NULL,
    dry_matter_pct  NUMERIC(5,2),
    crude_protein_pct NUMERIC(5,2),
    metabolisable_energy_mj NUMERIC(6,2),
    cost_per_tonne  NUMERIC(10,2),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Multi-tenancy and user tables (same as Suggestion 1)
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    subscription_tier TEXT NOT NULL DEFAULT 'free',
    timezone        TEXT NOT NULL DEFAULT 'UTC',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    email           TEXT NOT NULL,
    display_name    TEXT NOT NULL,
    role            TEXT NOT NULL DEFAULT 'operator',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);
```

## Key Query Patterns

### Temporal Query: Animal State at a Point in Time

```sql
-- Replay events to determine animal state on a specific date
-- "What was this animal's status and location on 2025-06-15?"
SELECT e.event_type, e.occurred_at, e.payload
FROM events e
WHERE e.aggregate_id = '550e8400-e29b-41d4-a716-446655440001'
  AND e.aggregate_type = 'animal'
  AND e.occurred_at <= '2025-06-15T23:59:59Z'
  AND e.event_type IN ('animal.registered', 'animal.arrival', 'animal.departure',
                         'animal.internal_transfer', 'animal.status_changed')
ORDER BY e.aggregate_version ASC;
```

### Disease Outbreak Window Query

```sql
-- "Which animals were at holding X between date A and date B?"
-- Uses the movement projection for fast lookup
SELECT DISTINCT m.animal_id, a.management_tag, a.primary_rfid
FROM proj_movement_history m
JOIN proj_animals a ON a.id = m.animal_id
WHERE m.to_holding_id = '550e8400-e29b-41d4-a716-446655440010'
  AND m.occurred_at <= '2025-07-01'
  AND (
    NOT EXISTS (
      SELECT 1 FROM proj_movement_history m2
      WHERE m2.animal_id = m.animal_id
        AND m2.from_holding_id = m.to_holding_id
        AND m2.occurred_at <= '2025-07-01'
        AND m2.occurred_at > m.occurred_at
    )
    OR EXISTS (
      SELECT 1 FROM proj_movement_history m2
      WHERE m2.animal_id = m.animal_id
        AND m2.from_holding_id = m.to_holding_id
        AND m2.occurred_at BETWEEN '2025-06-15' AND '2025-07-01'
    )
  );
```

### EPCIS Export

```sql
-- Export movement events as EPCIS 2.0 ObjectEvents
SELECT 
    e.event_id AS "eventID",
    'ObjectEvent' AS "type",
    e.occurred_at AS "eventTime",
    e.recorded_at AS "recordTime",
    e.payload->>'epcis_biz_step' AS "bizStep",
    e.payload->>'epcis_disposition' AS "disposition",
    jsonb_build_object(
        'type', 'urn:epc:id:sgtin',
        'value', ai.identifier
    ) AS "epcList"
FROM events e
JOIN proj_animal_identifiers ai ON ai.animal_id = e.aggregate_id AND ai.is_primary = true
WHERE e.event_type IN ('animal.departure', 'animal.arrival', 'animal.death')
  AND e.tenant_id = $1
  AND e.occurred_at BETWEEN $2 AND $3
ORDER BY e.occurred_at;
```

### Event Correction

```sql
-- To correct a weight that was recorded incorrectly:
-- 1. Void the original event
INSERT INTO events (event_id, sequence_number, aggregate_id, aggregate_type,
                    aggregate_version, tenant_id, event_type, occurred_at,
                    payload, recorded_by, corrects_event_id)
VALUES (
    gen_random_uuid(),
    nextval('event_sequence_seq'),
    '550e8400-e29b-41d4-a716-446655440001',
    'animal',
    43,  -- next version
    $tenant_id,
    'event.corrected',
    now(),
    '{"reason": "Weight was in lbs, should be kg", "original_event_type": "animal.weighed", "corrected_payload": {"weight_kg": 155.4, "method": "scale"}}'::jsonb,
    $user_id,
    '550e8400-e29b-41d4-a716-44665544ORIG'  -- the original event
);
-- 2. Projection rebuilder picks this up and updates proj_weight_history and proj_animals
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 2 | aggregate_roots, events |
| Projection Infrastructure | 1 | projection_checkpoints |
| Projection: Animals | 2 | proj_animals, proj_animal_identifiers |
| Projection: Movements | 1 | proj_movement_history |
| Projection: Weight | 1 | proj_weight_history |
| Projection: Health | 1 | proj_health_timeline |
| Projection: Reproduction | 1 | proj_repro_timeline |
| Projection: Holdings | 1 | proj_holdings |
| Projection: Milking | 1 | proj_daily_milking |
| Projection: Financial | 1 | proj_animal_financials |
| Reference Data | 4 | ref_species, ref_breeds, ref_medicines, ref_feeds |
| Multi-tenancy | 2 | tenants, users |
| **Total** | **18** | Plus the events table which stores all domain data |

---

## Key Design Decisions

1. **Single events table for all event types.** Rather than separate tables per event type, all events go into one append-only table. The `event_type` column discriminates, and the `payload` JSONB column carries the domain-specific data. This simplifies the write path and enables cross-event-type queries.

2. **Bi-temporal timestamps.** Every event has both `occurred_at` (when it happened in the real world) and `recorded_at` (when the system learned about it). This supports retroactive corrections and "what did we know when?" queries -- critical for disease outbreak investigations.

3. **Aggregate versioning for optimistic concurrency.** The `UNIQUE (aggregate_id, aggregate_version)` constraint prevents conflicting concurrent writes to the same animal. If two users try to record events for the same animal simultaneously, one will get a version conflict and must retry.

4. **Correction events instead of mutation.** Events are never updated or deleted. Corrections are modeled as new `event.corrected` events that reference the original via `corrects_event_id`. Projection rebuilders understand this and apply the correction.

5. **Projections are disposable and rebuildable.** Every projection table can be dropped and rebuilt from the event store. This means new read models (e.g., a carbon footprint projection) can be added at any time by writing a new projection handler and replaying events.

6. **Reference data is mutable.** Species, breeds, medicines, and feeds are stored in traditional mutable tables because they are configuration data, not domain events. Changing a medicine's withdrawal period does not require an event.

7. **GIN index on payload.** The JSONB GIN index enables queries like "find all events where the sire was X" without needing to pre-define which fields to index, at the cost of larger index size.

8. **Global sequence number for ordering.** The `sequence_number` provides a total ordering across all aggregates, enabling projection handlers to process events in guaranteed order without clock skew issues.

9. **Projection denormalization for query performance.** The `proj_animals` table includes denormalized fields like `latest_weight_kg`, `breeding_status`, and `primary_rfid` so that the most common list-and-filter queries hit a single table rather than joining across projections.

10. **EPCIS-native event structure.** Movement events include EPCIS bizStep and disposition fields in their payload, making export to GS1 EPCIS 2.0 a simple SELECT-and-transform rather than a complex mapping exercise.
