# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Livestock Management System · Created: 2026-05-24

## Philosophy

This model keeps the core identity and relationship structure relational (animals, holdings, movements, parentage) while using PostgreSQL JSONB columns for variable, species-specific, jurisdiction-specific, and user-defined data. The principle is: if a field is queried frequently, is part of a foreign key relationship, or is required by regulations across all jurisdictions, it gets a typed column. If it varies by species, breed, region, or farm, it goes in a JSONB `attributes` or `metadata` column.

This approach reflects a practical reality of livestock management: a dairy cattle operation in Ireland needs different fields (herd number, TB test results, BVD ear tissue tags) than a sheep station in New South Wales (NLIS PIC, breech strike score, mob identifiers) or a pig unit in Denmark (CHR number, movement zone, tail docking records). A fully normalized schema would either ignore these differences (losing data) or require dozens of nullable species-specific columns (wasting schema space) or hundreds of jurisdiction-specific tables (unmanageable complexity). JSONB columns provide a middle path: the core is enforced relationally, the variable periphery is stored flexibly and indexed selectively.

This pattern is used by platforms like Shopify (product metafields), Stripe (metadata on every object), and Farmbrite (custom fields on livestock records). It enables rapid MVP development because new field requirements can be met by extending the JSONB schema without database migrations, while retaining JOIN performance and referential integrity for the relationships that matter.

**Best for:** Multi-species, multi-jurisdiction platforms where field requirements vary widely between customers, and where rapid feature iteration is more important than exhaustive schema-level validation. Ideal for MVP and early-stage development.

**Trade-offs:**
- Pro: Fewer tables (~20) means simpler migrations and faster development
- Pro: Species-specific and jurisdiction-specific fields without schema changes
- Pro: Custom fields are a natural extension of the JSONB pattern
- Pro: Partial GIN indexes on JSONB give good query performance for known access patterns
- Pro: Easier to evolve schema over time as new standards or regulations emerge
- Con: JSONB fields lack foreign key constraints -- referential integrity must be enforced in application code
- Con: JSONB fields lack CHECK constraints -- data validation moves to the application layer
- Con: Complex JSONB queries can be slower than equivalent column-based queries without careful indexing
- Con: Schema documentation requires discipline -- the JSONB structure is implicit, not declarative
- Con: BI tools and reporting may struggle with nested JSONB structures

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ICAR ADE 1.5 | Core ICAR fields are typed columns; species-specific ICAR extensions stored in JSONB attributes |
| ISO 11784/11785 | RFID identifiers stored as typed columns in the identifiers array within animal JSONB |
| ISO 3166 | Country and subdivision as typed columns on holdings; jurisdiction-specific holding IDs in JSONB |
| EU Reg 1760/2000 | EU-specific bovine fields (passport number, BVD tag) in jurisdiction_data JSONB |
| EU Reg 21/2004 | Ovine/caprine EID fields in jurisdiction_data JSONB |
| NLIS Australia | NLIS-specific fields (PIC, NLIS device number, NVD reference) in jurisdiction_data JSONB |
| USDA ADT | US-specific fields (premises ID, AIN) in jurisdiction_data JSONB |
| GS1 EPCIS 2.0 | EPCIS export metadata stored in movement JSONB for direct serialization |
| ISO 22005 | Traceability chain maintained through relational movement records |

---

## Core Tables

```sql
-- ============================================================
-- MULTI-TENANCY
-- ============================================================

CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    subscription_tier TEXT NOT NULL DEFAULT 'free',
    country_code    CHAR(2) NOT NULL,        -- primary jurisdiction
    timezone        TEXT NOT NULL DEFAULT 'UTC',
    -- Tenant-level configuration stored in JSONB
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings example:
    -- {
    --   "weight_unit": "kg",
    --   "currency": "AUD",
    --   "species_enabled": ["cattle", "sheep"],
    --   "compliance_regimes": ["nlis", "lpa"],
    --   "custom_statuses": ["spelling", "backgrounding"],
    --   "default_breed_scoring_scale": "1-9"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    email           TEXT NOT NULL,
    display_name    TEXT NOT NULL,
    role            TEXT NOT NULL DEFAULT 'operator',
    preferences     JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE INDEX idx_users_tenant ON users(tenant_id);
```

## Holdings

```sql
-- ============================================================
-- HOLDINGS — core fields relational, jurisdiction IDs in JSONB
-- ============================================================

CREATE TABLE holdings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            TEXT NOT NULL,
    holding_type    TEXT NOT NULL DEFAULT 'farm',
    country_code    CHAR(2) NOT NULL,
    subdivision_code TEXT,
    latitude        NUMERIC(10,7),
    longitude       NUMERIC(10,7),
    
    -- Jurisdiction-specific identifiers in JSONB
    jurisdiction_ids JSONB NOT NULL DEFAULT '{}',
    -- Examples:
    -- AU: {"pic": "3ABCD123", "lpa_accredited": true, "lpa_expiry": "2026-12-31"}
    -- UK: {"cph": "12/345/6789", "county_parish": "Devon"}
    -- IE: {"herd_number": "IE123456789", "tb_restricted": false}
    -- US: {"premises_id": "004ABCD", "premises_state": "TX"}
    
    -- Address and contact in JSONB (varies by country)
    address         JSONB NOT NULL DEFAULT '{}',
    -- {"line1": "...", "line2": "...", "city": "...", "state": "...", "postal_code": "..."}
    
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_holdings_tenant ON holdings(tenant_id);
CREATE INDEX idx_holdings_jurisdiction ON holdings USING GIN (jurisdiction_ids);

-- Sub-locations within a holding
CREATE TABLE paddocks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    holding_id      UUID NOT NULL REFERENCES holdings(id),
    name            TEXT NOT NULL,
    paddock_type    TEXT DEFAULT 'grazing',
    area_hectares   NUMERIC(10,2),
    capacity_head   INTEGER,
    attributes      JSONB NOT NULL DEFAULT '{}',
    -- {"grazing_type": "rotational", "water_source": "trough", "shelter": true}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_paddocks_holding ON paddocks(holding_id);
```

## Animals

```sql
-- ============================================================
-- ANIMALS — universal fields as columns, variable data as JSONB
-- ============================================================

CREATE TABLE animals (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    
    -- Universal identification (relational)
    species         TEXT NOT NULL,  -- 'cattle', 'sheep', 'goat', 'pig', 'poultry', 'horse', 'deer'
    management_tag  TEXT,
    name            TEXT,
    
    -- Identifiers array in JSONB (multiple schemes per animal)
    identifiers     JSONB NOT NULL DEFAULT '[]',
    -- [
    --   {"scheme": "ISO_11784", "value": "826000012345678", "country_code": "826",
    --    "national_id": "000012345678", "is_primary": true, "applied_date": "2024-03-16"},
    --   {"scheme": "EU_EAR_TAG", "value": "UK123456789012", "is_primary": false},
    --   {"scheme": "MANAGEMENT_TAG", "value": "A-142", "is_primary": false},
    --   {"scheme": "NLIS_DEVICE", "value": "982000012345678", "is_primary": false}
    -- ]
    
    -- Universal biological fields (relational)
    sex             TEXT NOT NULL CHECK (sex IN ('male', 'female', 'unknown')),
    date_of_birth   DATE,
    dam_id          UUID REFERENCES animals(id),
    sire_id         UUID REFERENCES animals(id),
    
    -- Breed: primary breed as column, cross-breed detail in JSONB
    primary_breed   TEXT,
    breed_detail    JSONB NOT NULL DEFAULT '{}',
    -- Pure: {"breed_code": "AA", "breed_name": "Aberdeen Angus", "icar_code": "AN"}
    -- Cross: {"fractions": [{"breed_code": "AA", "fraction": 0.5}, {"breed_code": "HE", "fraction": 0.5}]}
    
    -- Status (relational for querying)
    status          TEXT NOT NULL DEFAULT 'active',
    status_date     DATE,
    
    -- Current location (relational for querying)
    current_holding_id UUID REFERENCES holdings(id),
    current_paddock_id UUID REFERENCES paddocks(id),
    
    -- Species-specific attributes in JSONB
    species_attributes JSONB NOT NULL DEFAULT '{}',
    -- Cattle: {
    --   "is_castrated": false,
    --   "horn_status": "polled",
    --   "colour": "black",
    --   "brand": "Bar-Z left hip",
    --   "condition_score_scale": "1-9",
    --   "frame_score": 6,
    --   "muscle_score": "A"
    -- }
    -- Sheep: {
    --   "is_castrated": false,
    --   "wool_type": "medium",
    --   "breech_wrinkle_score": 2,
    --   "dag_score": 1,
    --   "birth_type": "twin"
    -- }
    -- Dairy cattle: {
    --   "is_castrated": false,
    --   "current_lactation": 3,
    --   "breeding_status": "pregnant",
    --   "expected_due_date": "2026-08-15",
    --   "milking_speed": "fast"
    -- }
    
    -- Jurisdiction-specific data in JSONB
    jurisdiction_data JSONB NOT NULL DEFAULT '{}',
    -- AU: {"nlis_status": "registered", "lpa_status": "assured", "mob": "M-12"}
    -- UK: {"passport_number": "UK123456789012", "bvd_tag_number": "...", "tb_test_date": "2025-11-01"}
    -- IE: {"ai_code": "...", "carbon_class": "A"}
    
    -- Breed society registration in JSONB
    breed_society   JSONB NOT NULL DEFAULT '{}',
    -- {"society_name": "Angus Australia", "reg_number": "DVGA123",
    --  "reg_status": "registered", "reg_date": "2024-06-01", "single_entry_code": "..."}
    
    -- User-defined custom fields
    custom_fields   JSONB NOT NULL DEFAULT '{}',
    -- Fully user-defined per tenant: {"paddock_preference": "creek", "temperament": "quiet"}
    
    -- Latest measurements (denormalized for list views)
    latest_weight_kg NUMERIC(8,2),
    latest_weight_date DATE,
    latest_condition_score NUMERIC(3,1),
    
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_animals_tenant ON animals(tenant_id);
CREATE INDEX idx_animals_species ON animals(tenant_id, species);
CREATE INDEX idx_animals_status ON animals(tenant_id, status);
CREATE INDEX idx_animals_holding ON animals(current_holding_id);
CREATE INDEX idx_animals_dam ON animals(dam_id);
CREATE INDEX idx_animals_sire ON animals(sire_id);
CREATE INDEX idx_animals_tag ON animals(tenant_id, management_tag);
CREATE INDEX idx_animals_breed ON animals(tenant_id, primary_breed);

-- GIN indexes for JSONB queries
CREATE INDEX idx_animals_identifiers ON animals USING GIN (identifiers);
CREATE INDEX idx_animals_species_attrs ON animals USING GIN (species_attributes);
CREATE INDEX idx_animals_jurisdiction ON animals USING GIN (jurisdiction_data);
CREATE INDEX idx_animals_custom ON animals USING GIN (custom_fields);
```

## Events (Unified Event Table with JSONB Detail)

```sql
-- ============================================================
-- EVENTS — single table for all animal events, detail in JSONB
-- ============================================================

CREATE TABLE animal_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    animal_id       UUID REFERENCES animals(id),
    
    -- Event classification
    event_category  TEXT NOT NULL
        CHECK (event_category IN ('movement', 'health', 'reproduction',
                                   'weight', 'feed', 'milking', 'financial',
                                   'status_change', 'group', 'sensor')),
    event_type      TEXT NOT NULL,
    -- movement: 'birth', 'arrival', 'departure', 'death', 'internal_transfer'
    -- health: 'diagnosis', 'treatment', 'vaccination', 'health_status', 'withdrawal'
    -- reproduction: 'heat', 'insemination', 'pregnancy_check', 'parturition', 'abortion', 'dry_off'
    -- weight: 'weigh', 'condition_score', 'conformation_score', 'carcass'
    -- feed: 'individual_feed', 'group_feed', 'ration_change'
    -- milking: 'milking_visit', 'test_day'
    -- financial: 'purchase', 'sale', 'cost'
    -- status_change: 'status_update'
    -- group: 'join_set', 'leave_set'
    -- sensor: 'sensor_reading', 'alert'
    
    event_date      TIMESTAMPTZ NOT NULL,
    
    -- Common fields used in filtering (relational for performance)
    holding_id      UUID REFERENCES holdings(id),   -- where the event occurred
    recorded_by     UUID REFERENCES users(id),
    
    -- All event-specific detail in JSONB
    detail          JSONB NOT NULL DEFAULT '{}',
    
    -- Metadata
    source          TEXT DEFAULT 'manual',  -- 'manual', 'rfid', 'sensor', 'api', 'import'
    notes           TEXT,
    
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_events_tenant ON animal_events(tenant_id);
CREATE INDEX idx_events_animal ON animal_events(animal_id);
CREATE INDEX idx_events_category ON animal_events(tenant_id, event_category);
CREATE INDEX idx_events_type ON animal_events(event_type);
CREATE INDEX idx_events_date ON animal_events(event_date);
CREATE INDEX idx_events_holding ON animal_events(holding_id);
CREATE INDEX idx_events_detail ON animal_events USING GIN (detail);
```

### Event Detail JSONB Examples

```sql
-- Movement: departure
-- detail: {
--   "from_holding_id": "...", "from_holding_name": "Home Farm",
--   "to_holding_id": "...", "to_holding_name": "Feedlot Alpha",
--   "transport_ref": "CON-2024-0045",
--   "vehicle_reg": "AB12CDE",
--   "movement_permit": "MP-2024-1234",
--   "envd_number": "eNVD-2024-5678",
--   "epcis": {
--     "biz_step": "urn:epcglobal:cbv:bizstep:shipping",
--     "disposition": "urn:epcglobal:cbv:disp:in_transit"
--   }
-- }

-- Health: treatment
-- detail: {
--   "treatment_type": "antibiotic",
--   "diagnosis_event_id": "...",
--   "medicine": {
--     "id": "...", "name": "Excenel RTU", "active_ingredient": "Ceftiofur",
--     "batch_number": "EX-2024-8891"
--   },
--   "dose": {"amount": 6.0, "unit": "ml", "route": "injection_sc"},
--   "withdrawal": {
--     "meat_days": 8, "meat_end_date": "2025-07-20",
--     "milk_days": 0, "milk_end_date": null
--   },
--   "administered_by": {"id": "...", "name": "Dr. Sarah Jones"}
-- }

-- Reproduction: insemination
-- detail: {
--   "insemination_type": "artificial",
--   "sire": {"id": "...", "name": "Dunlouise New Design", "breed": "Angus"},
--   "straw": {
--     "id": "STR-AA-2024-0033",
--     "company": "Cogent Breeding",
--     "batch": "COG-2024-B15"
--   },
--   "technician": {"id": "...", "name": "Jim Murray"},
--   "heat_event_id": "..."
-- }

-- Reproduction: parturition
-- detail: {
--   "calving_ease": "unassisted",
--   "offspring_count": 1,
--   "offspring": [
--     {
--       "animal_id": "...",
--       "sex": "female",
--       "birth_weight_kg": 38.2,
--       "condition": "alive",
--       "presentation": "normal"
--     }
--   ]
-- }

-- Weight: weigh
-- detail: {
--   "weight_kg": 342.5,
--   "method": "scale",
--   "device_id": "SCALE-001",
--   "condition_score": 3.5,
--   "previous_weight_kg": 310.0,
--   "previous_weight_date": "2025-05-01",
--   "daily_gain_kg": 1.08
-- }

-- Milking: milking_visit
-- detail: {
--   "session": "morning",
--   "milk_weight_kg": 14.2,
--   "fat_pct": 4.1,
--   "protein_pct": 3.4,
--   "somatic_cell_count": 125000,
--   "conductivity": 5.2,
--   "duration_seconds": 420,
--   "device_id": "ROBOT-A3"
-- }

-- Financial: sale
-- detail: {
--   "sale_price": 1850.00,
--   "currency": "AUD",
--   "buyer": "ABC Meats Pty Ltd",
--   "sale_method": "saleyard",
--   "lot_number": "42",
--   "price_per_kg": 5.40,
--   "dressed_weight_kg": 342.6,
--   "invoice_ref": "INV-2025-0891"
-- }
```

## Animal Sets

```sql
-- ============================================================
-- ANIMAL SETS — groups, mobs, herds
-- ============================================================

CREATE TABLE animal_sets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            TEXT NOT NULL,
    set_type        TEXT NOT NULL DEFAULT 'mob',
    holding_id      UUID REFERENCES holdings(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    attributes      JSONB NOT NULL DEFAULT '{}',
    -- {"purpose": "backgrounding", "target_weight_kg": 450, "target_date": "2026-03-01"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sets_tenant ON animal_sets(tenant_id);

CREATE TABLE animal_set_memberships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    animal_id       UUID NOT NULL REFERENCES animals(id),
    animal_set_id   UUID NOT NULL REFERENCES animal_sets(id),
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    left_at         TIMESTAMPTZ,
    UNIQUE (animal_id, animal_set_id, joined_at)
);

CREATE INDEX idx_memberships_animal ON animal_set_memberships(animal_id);
CREATE INDEX idx_memberships_set ON animal_set_memberships(animal_set_id);
```

## Feed and Nutrition

```sql
-- ============================================================
-- FEED — reference data with JSONB nutritional profiles
-- ============================================================

CREATE TABLE feeds (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            TEXT NOT NULL,
    feed_type       TEXT NOT NULL,
    -- Nutritional profile in JSONB (varies by feed analysis)
    nutrition       JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "dry_matter_pct": 88.5,
    --   "crude_protein_pct": 12.3,
    --   "metabolisable_energy_mj": 11.5,
    --   "ndf_pct": 45.2,
    --   "adf_pct": 28.1,
    --   "calcium_pct": 0.45,
    --   "phosphorus_pct": 0.35,
    --   "starch_pct": 22.0
    -- }
    cost_per_tonne  NUMERIC(10,2),
    currency_code   CHAR(3) DEFAULT 'USD',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_feeds_tenant ON feeds(tenant_id);

CREATE TABLE rations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            TEXT NOT NULL,
    target_species  TEXT,
    target_class    TEXT,
    ingredients     JSONB NOT NULL DEFAULT '[]',
    -- [
    --   {"feed_id": "...", "feed_name": "Barley", "inclusion_pct": 40, "amount_kg": 4.0},
    --   {"feed_id": "...", "feed_name": "Hay", "inclusion_pct": 50, "amount_kg": 5.0},
    --   {"feed_id": "...", "feed_name": "Mineral Mix", "inclusion_pct": 10, "amount_kg": 1.0}
    -- ]
    total_dm_kg     NUMERIC(8,2),
    calculated_nutrition JSONB NOT NULL DEFAULT '{}',
    -- {"crude_protein_pct": 13.5, "metabolisable_energy_mj": 10.8, "cost_per_head_day": 2.45}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rations_tenant ON rations(tenant_id);
```

## Medicines

```sql
-- ============================================================
-- MEDICINES — reference data
-- ============================================================

CREATE TABLE medicines (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            TEXT NOT NULL,
    active_ingredient TEXT,
    manufacturer    TEXT,
    registration_number TEXT,
    default_withdrawal_meat_days INTEGER,
    default_withdrawal_milk_days INTEGER,
    -- Species-specific dosage guidelines in JSONB
    dosage_guidelines JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "cattle": {"dose_ml_per_kg": 0.01, "route": "injection_sc", "max_dose_ml": 10},
    --   "sheep": {"dose_ml_per_kg": 0.02, "route": "oral", "max_dose_ml": 5}
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_medicines_tenant ON medicines(tenant_id);
```

## Breeding Values

```sql
-- ============================================================
-- BREEDING VALUES — stored as JSONB array per animal evaluation
-- ============================================================

CREATE TABLE breeding_evaluations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    animal_id       UUID NOT NULL REFERENCES animals(id),
    evaluation_date DATE NOT NULL,
    source          TEXT NOT NULL,         -- 'BREEDPLAN', 'AHDB', 'CDCB', 'CRV'
    evaluation_group TEXT,
    -- All trait values in JSONB (varies by species, breed, evaluation system)
    values          JSONB NOT NULL DEFAULT '{}',
    -- Beef example (BREEDPLAN):
    -- {
    --   "200DWT": {"value": 32.5, "accuracy": 0.78, "percentile": 85},
    --   "400DWT": {"value": 58.2, "accuracy": 0.72, "percentile": 80},
    --   "EMA": {"value": 4.1, "accuracy": 0.55, "percentile": 72},
    --   "RIB_FAT": {"value": -0.3, "accuracy": 0.50, "percentile": 45},
    --   "MILK": {"value": 12.0, "accuracy": 0.60, "percentile": 65}
    -- }
    -- Dairy example (CDCB):
    -- {
    --   "TPI": {"value": 2850, "reliability": 0.75},
    --   "MILK": {"value": 1200, "reliability": 0.72},
    --   "FAT": {"value": 65, "reliability": 0.71},
    --   "PROTEIN": {"value": 42, "reliability": 0.70},
    --   "SCS": {"value": 2.85, "reliability": 0.68},
    --   "PL": {"value": 5.2, "reliability": 0.55}
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_breeding_evals_animal ON breeding_evaluations(animal_id);
CREATE INDEX idx_breeding_evals_date ON breeding_evaluations(evaluation_date);
CREATE INDEX idx_breeding_evals_source ON breeding_evaluations(tenant_id, source);
```

## Devices and Sensors

```sql
-- ============================================================
-- DEVICES
-- ============================================================

CREATE TABLE devices (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    device_type     TEXT NOT NULL,
    manufacturer    TEXT,
    model           TEXT,
    serial_number   TEXT,
    holding_id      UUID REFERENCES holdings(id),
    animal_id       UUID REFERENCES animals(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    config          JSONB NOT NULL DEFAULT '{}',
    -- {"reading_interval_minutes": 15, "alert_thresholds": {"temperature_high": 40.0, "activity_low": 10}}
    last_reading_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_devices_tenant ON devices(tenant_id);
CREATE INDEX idx_devices_animal ON devices(animal_id);

-- Sensor readings use the animal_events table with event_category = 'sensor'
-- For high-volume sensor data, consider a separate time-series table:

CREATE TABLE sensor_readings (
    time            TIMESTAMPTZ NOT NULL,
    device_id       UUID NOT NULL REFERENCES devices(id),
    animal_id       UUID,
    metrics         JSONB NOT NULL,
    -- {"activity_index": 85, "rumination_minutes": 42, "temperature_c": 38.6, "steps": 1200}
    PRIMARY KEY (device_id, time)
);

CREATE INDEX idx_sensor_animal ON sensor_readings(animal_id);
CREATE INDEX idx_sensor_time ON sensor_readings(time);
-- Consider partitioning by time range for large installations
```

## Audit Trail

```sql
-- ============================================================
-- AUDIT LOG
-- ============================================================

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    user_id         UUID REFERENCES users(id),
    table_name      TEXT NOT NULL,
    record_id       UUID NOT NULL,
    action          TEXT NOT NULL CHECK (action IN ('insert', 'update', 'delete')),
    changes         JSONB,  -- {"field": {"old": "...", "new": "..."}, ...}
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_tenant ON audit_log(tenant_id);
CREATE INDEX idx_audit_record ON audit_log(table_name, record_id);
CREATE INDEX idx_audit_time ON audit_log(created_at);
```

## Key Query Patterns

### Find Animal by RFID Tag

```sql
-- JSONB containment query using GIN index
SELECT id, management_tag, species, primary_breed, status
FROM animals
WHERE tenant_id = $1
  AND identifiers @> '[{"scheme": "ISO_11784", "value": "826000012345678"}]';
```

### List Animals Under Withdrawal

```sql
-- Query treatment events for active withdrawals
SELECT DISTINCT a.id, a.management_tag, a.species,
       e.detail->'withdrawal'->>'meat_end_date' AS meat_withdrawal_end,
       e.detail->'medicine'->>'name' AS medicine_name
FROM animal_events e
JOIN animals a ON a.id = e.animal_id
WHERE e.tenant_id = $1
  AND e.event_type = 'treatment'
  AND (e.detail->'withdrawal'->>'meat_end_date')::date >= CURRENT_DATE;
```

### Species-Specific Query (Wool Sheep)

```sql
-- Find sheep with high breech wrinkle scores
SELECT id, management_tag, 
       species_attributes->>'wool_type' AS wool_type,
       (species_attributes->>'breech_wrinkle_score')::int AS wrinkle_score
FROM animals
WHERE tenant_id = $1
  AND species = 'sheep'
  AND (species_attributes->>'breech_wrinkle_score')::int >= 4;
```

### Jurisdiction-Specific Query (UK TB Testing)

```sql
-- Find cattle overdue for TB testing (UK)
SELECT id, management_tag,
       jurisdiction_data->>'passport_number' AS passport,
       (jurisdiction_data->>'tb_test_date')::date AS last_test
FROM animals
WHERE tenant_id = $1
  AND species = 'cattle'
  AND jurisdiction_data ? 'tb_test_date'
  AND (jurisdiction_data->>'tb_test_date')::date < CURRENT_DATE - INTERVAL '365 days';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Multi-tenancy | 2 | tenants, users |
| Holdings | 2 | holdings, paddocks |
| Animals | 1 | animals (with 5 JSONB columns for variable data) |
| Events | 1 | animal_events (unified, with JSONB detail) |
| Animal Sets | 2 | animal_sets, animal_set_memberships |
| Feed & Nutrition | 2 | feeds, rations |
| Medicines | 1 | medicines |
| Breeding Values | 1 | breeding_evaluations (JSONB values per evaluation system) |
| Devices & Sensors | 2 | devices, sensor_readings |
| Audit | 1 | audit_log |
| **Total** | **15** | Significantly fewer tables than normalized model |

---

## Key Design Decisions

1. **Single animals table with multiple JSONB columns.** Rather than junction tables for identifiers, breed fractions, and species-specific attributes, each concern gets its own JSONB column. This trades referential integrity for development speed and query simplicity -- a single SELECT returns all animal data without JOINs.

2. **Unified event table.** All animal events (movements, treatments, weights, reproduction) go into one `animal_events` table, differentiated by `event_category` and `event_type`. The `detail` JSONB column carries the event-specific payload. This reduces the table count from ~15 event tables to 1.

3. **JSONB for jurisdiction-specific identifiers.** Rather than nullable columns for every possible government ID (PIC, CPH, herd number, premises ID), all jurisdiction-specific identifiers go into a single JSONB column. A GIN index enables efficient lookups.

4. **Separate JSONB columns per concern.** The animals table has `identifiers`, `species_attributes`, `jurisdiction_data`, `breed_society`, and `custom_fields` as separate JSONB columns rather than one giant `metadata` blob. This enables targeted GIN indexes and clearer application-layer validation.

5. **Ration ingredients as JSONB array.** Rather than a separate `ration_ingredients` junction table, the ingredient list is stored as a JSONB array on the ration. This is appropriate because rations are always loaded as a whole document and individual ingredient rows are never queried independently.

6. **Denormalized latest measurements.** The `latest_weight_kg` and `latest_condition_score` on the animals table are denormalized from weight events. This avoids a subquery or JOIN for the most common list view (animal inventory with latest weight).

7. **Separate sensor_readings table.** Despite the JSONB philosophy, sensor readings get their own table because their volume (millions of rows per month for large operations) and access pattern (time-range queries) differ fundamentally from business events.

8. **Tenant settings in JSONB.** Weight units, currency, enabled species, compliance regimes, and custom statuses are tenant-level configuration stored in JSONB. This avoids a separate settings table and makes tenant configuration a single read.

9. **GIN indexes on all queryable JSONB columns.** Every JSONB column that users might filter by gets a GIN index. This provides acceptable performance for containment queries (`@>`) while keeping the schema flexible.

10. **Custom fields as a first-class JSONB column.** The `custom_fields` column on animals is explicitly designed for user-defined fields, mirroring Farmbrite's custom fields feature. Application-level schema validation can be tenant-configured without database schema changes.
