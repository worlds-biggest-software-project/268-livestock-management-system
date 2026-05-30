# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Livestock Management System · Created: 2026-05-24

## Philosophy

This model follows classical relational database design with a separate table for every domain concept, heavy use of foreign keys and junction tables, and strict normalization to third normal form (3NF). Every entity that the ICAR Animal Data Exchange standard recognizes gets its own first-class table with typed columns, constraints, and referential integrity enforced at the database level.

The design philosophy is that livestock management is fundamentally a regulated domain where data integrity, auditability, and standards compliance are non-negotiable. A normalized schema makes it straightforward to enforce business rules through constraints, generate regulatory reports with simple JOINs, and integrate with government traceability systems (UK LIS, Australian NLIS, EU bovine registers) that expect well-structured, unambiguous data.

This approach mirrors how national livestock databases and breed societies structure their own data -- separate registers for animals, holdings, movements, and health events, each with their own identifier schemes and validation rules.

**Best for:** Operations where regulatory compliance, breed society integration, and data integrity are the primary concerns, and where the team has strong SQL skills and values schema-enforced correctness.

**Trade-offs:**
- Pro: Maximum data integrity through foreign keys, check constraints, and unique indexes
- Pro: Clean mapping to ICAR ADE resource types for standards-compliant API exposure
- Pro: Straightforward regulatory reporting with standard SQL JOINs
- Pro: Well-understood by database administrators and BI tools
- Con: High table count (~45-55 tables) increases migration complexity
- Con: Adding species-specific or jurisdiction-specific fields requires schema migrations
- Con: Many JOINs required for composite views (animal + latest weight + breeding status + location)
- Con: Rigid schema slows down MVP iteration compared to document or hybrid approaches

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ICAR ADE 1.5 | Each ICAR resource type (icarAnimalCoreResource, icarWeightEventResource, etc.) maps to a dedicated table with matching field names |
| ISO 11784/11785 | The `animal_identifiers` table stores the 64-bit RFID code split into country_code (ISO 3166 numeric), national_id (12-digit), and animal_flag fields |
| ISO 3166 | Country and subdivision codes used in `jurisdictions` and `holdings` tables |
| EU Reg 1760/2000 | Bovine ear tag structure (2-letter country + 12-digit) modeled in `animal_identifiers` with scheme = 'EU_EAR_TAG' |
| EU Reg 21/2004 | Ovine/caprine EID requirements supported through the same identifier scheme |
| NLIS Australia | NLIS device numbers and PIC (Property Identification Code) stored in `animal_identifiers` and `holdings` |
| GS1 EPCIS 2.0 | Movement events structured to export as EPCIS ObjectEvents with bizStep and disposition fields |
| ISO 22005 | Traceability chain maintained through `movements` table linking animals to holdings with timestamps |
| OpenAPI 3.1 | Table structure designed to serialize directly to ICAR ADE JSON resources |

---

## Core Identity Tables

```sql
-- ============================================================
-- MULTI-TENANCY AND IDENTITY
-- ============================================================

CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    subscription_tier TEXT NOT NULL DEFAULT 'free'
        CHECK (subscription_tier IN ('free', 'standard', 'professional', 'enterprise')),
    timezone        TEXT NOT NULL DEFAULT 'UTC',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    email           TEXT NOT NULL,
    display_name    TEXT NOT NULL,
    role            TEXT NOT NULL DEFAULT 'operator'
        CHECK (role IN ('owner', 'manager', 'operator', 'viewer', 'veterinarian')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE INDEX idx_users_tenant ON users(tenant_id);
```

## Holdings and Locations

```sql
-- ============================================================
-- HOLDINGS (farms, properties, paddocks)
-- ============================================================

CREATE TABLE holdings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            TEXT NOT NULL,
    holding_type    TEXT NOT NULL DEFAULT 'farm'
        CHECK (holding_type IN ('farm', 'feedlot', 'saleyard', 'abattoir', 'veterinary', 'quarantine')),
    -- ISO 3166-1 alpha-2 country code
    country_code    CHAR(2) NOT NULL,
    -- ISO 3166-2 subdivision code
    subdivision_code TEXT,
    address_line1   TEXT,
    address_line2   TEXT,
    city            TEXT,
    postal_code     TEXT,
    latitude        NUMERIC(10,7),
    longitude       NUMERIC(10,7),
    -- Government-issued property identifiers
    pic_code        TEXT,   -- AU: Property Identification Code
    cph_number      TEXT,   -- UK: County Parish Holding number
    herd_number     TEXT,   -- IE: Herd number
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_holdings_tenant ON holdings(tenant_id);

CREATE TABLE paddocks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    holding_id      UUID NOT NULL REFERENCES holdings(id),
    name            TEXT NOT NULL,
    area_hectares   NUMERIC(10,2),
    paddock_type    TEXT DEFAULT 'grazing'
        CHECK (paddock_type IN ('grazing', 'feedlot_pen', 'yard', 'shed', 'milking_parlour', 'quarantine')),
    capacity_head   INTEGER,
    latitude        NUMERIC(10,7),
    longitude       NUMERIC(10,7),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_paddocks_holding ON paddocks(holding_id);
```

## Animal Records

```sql
-- ============================================================
-- ANIMALS — core identity record per ICAR AnimalCoreResource
-- ============================================================

CREATE TABLE species (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            TEXT NOT NULL UNIQUE,  -- 'cattle', 'sheep', 'goat', 'pig', 'poultry', 'horse', 'deer'
    name            TEXT NOT NULL,
    icar_species_code TEXT  -- ICAR species enumeration
);

CREATE TABLE breeds (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    species_id      UUID NOT NULL REFERENCES species(id),
    code            TEXT NOT NULL,          -- e.g., 'AA' for Angus
    name            TEXT NOT NULL,          -- e.g., 'Aberdeen Angus'
    icar_breed_code TEXT,                   -- ICAR breed code
    breed_society   TEXT,
    UNIQUE (species_id, code)
);

CREATE INDEX idx_breeds_species ON breeds(species_id);

CREATE TABLE animals (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    species_id      UUID NOT NULL REFERENCES species(id),
    breed_id        UUID REFERENCES breeds(id),
    -- Cross-breed composition stored in animal_breed_fractions
    is_crossbred    BOOLEAN NOT NULL DEFAULT false,

    -- Visual / management identification
    management_tag  TEXT,       -- farm-level visual tag number
    name            TEXT,       -- optional animal name (stud stock)

    -- Birth details
    date_of_birth   DATE,
    birth_date_confidence CHAR(3) DEFAULT 'YMD',  -- ICAR: Y=year, M=month, D=day known
    birth_holding_id UUID REFERENCES holdings(id),
    birth_rank      SMALLINT,   -- number of offspring in same birth event
    rearing_rank    SMALLINT,   -- number reared by same dam

    -- Parentage
    dam_id          UUID REFERENCES animals(id),
    sire_id         UUID REFERENCES animals(id),

    -- Sex and reproductive status
    sex             TEXT NOT NULL CHECK (sex IN ('male', 'female', 'unknown')),
    is_castrated    BOOLEAN NOT NULL DEFAULT false,

    -- Current status
    status          TEXT NOT NULL DEFAULT 'active'
        CHECK (status IN ('active', 'sold', 'deceased', 'culled', 'butchered',
                          'lost', 'off_farm', 'quarantined', 'archived')),
    status_date     DATE,
    death_date      DATE,
    death_cause     TEXT,

    -- Current location (denormalized for query performance; authoritative source is movements)
    current_holding_id UUID REFERENCES holdings(id),
    current_paddock_id UUID REFERENCES paddocks(id),

    -- Breed society registration
    breed_soc_reg_number TEXT,
    breed_soc_reg_status TEXT CHECK (breed_soc_reg_status IN ('registered', 'not_registered', 'pending')),
    breed_soc_reg_date   DATE,

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_animals_tenant ON animals(tenant_id);
CREATE INDEX idx_animals_species ON animals(species_id);
CREATE INDEX idx_animals_breed ON animals(breed_id);
CREATE INDEX idx_animals_dam ON animals(dam_id);
CREATE INDEX idx_animals_sire ON animals(sire_id);
CREATE INDEX idx_animals_status ON animals(tenant_id, status);
CREATE INDEX idx_animals_holding ON animals(current_holding_id);
CREATE INDEX idx_animals_mgmt_tag ON animals(tenant_id, management_tag);

-- Cross-breed composition (for animals where is_crossbred = true)
CREATE TABLE animal_breed_fractions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    animal_id       UUID NOT NULL REFERENCES animals(id) ON DELETE CASCADE,
    breed_id        UUID NOT NULL REFERENCES breeds(id),
    fraction        NUMERIC(5,4) NOT NULL CHECK (fraction > 0 AND fraction <= 1),
    UNIQUE (animal_id, breed_id)
);

-- Multiple identification schemes per animal (RFID, ear tag, tattoo, brand, etc.)
CREATE TABLE animal_identifiers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    animal_id       UUID NOT NULL REFERENCES animals(id) ON DELETE CASCADE,
    scheme          TEXT NOT NULL,
        -- 'ISO_11784' (RFID), 'EU_EAR_TAG', 'NLIS_DEVICE', 'TATTOO', 'BRAND',
        -- 'MANAGEMENT_TAG', 'BREED_SOCIETY', 'USDA_AIN'
    identifier      TEXT NOT NULL,
    -- ISO 11784 decomposed fields (populated when scheme = 'ISO_11784')
    rfid_country_code  CHAR(3),    -- ISO 3166 numeric country code
    rfid_national_id   TEXT,        -- 12-digit national ID
    rfid_animal_flag   BOOLEAN,
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    applied_date    DATE,
    removed_date    DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (scheme, identifier)
);

CREATE INDEX idx_animal_identifiers_animal ON animal_identifiers(animal_id);
CREATE INDEX idx_animal_identifiers_scheme ON animal_identifiers(scheme, identifier);
```

## Animal Sets (Groups, Mobs, Herds)

```sql
-- ============================================================
-- ANIMAL SETS — groups, mobs, herds, flocks
-- ============================================================

CREATE TABLE animal_sets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            TEXT NOT NULL,
    set_type        TEXT NOT NULL DEFAULT 'mob'
        CHECK (set_type IN ('herd', 'flock', 'mob', 'pen', 'breeding_group', 'treatment_group')),
    purpose         TEXT,
    holding_id      UUID REFERENCES holdings(id),
    paddock_id      UUID REFERENCES paddocks(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_animal_sets_tenant ON animal_sets(tenant_id);

CREATE TABLE animal_set_memberships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    animal_id       UUID NOT NULL REFERENCES animals(id),
    animal_set_id   UUID NOT NULL REFERENCES animal_sets(id),
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    left_at         TIMESTAMPTZ,
    UNIQUE (animal_id, animal_set_id, joined_at)
);

CREATE INDEX idx_set_memberships_animal ON animal_set_memberships(animal_id);
CREATE INDEX idx_set_memberships_set ON animal_set_memberships(animal_set_id);
```

## Movement and Traceability

```sql
-- ============================================================
-- MOVEMENTS — per ICAR MovementArrival/Departure/Birth/Death
-- ============================================================

CREATE TABLE movements (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    animal_id       UUID NOT NULL REFERENCES animals(id),
    movement_type   TEXT NOT NULL
        CHECK (movement_type IN ('birth', 'arrival', 'departure', 'death',
                                  'internal_transfer', 'import', 'export')),
    event_date      TIMESTAMPTZ NOT NULL,
    -- Origin and destination
    from_holding_id UUID REFERENCES holdings(id),
    from_paddock_id UUID REFERENCES paddocks(id),
    to_holding_id   UUID REFERENCES holdings(id),
    to_paddock_id   UUID REFERENCES paddocks(id),
    -- Transport details
    transport_ref   TEXT,        -- consignment or waybill reference
    vehicle_reg     TEXT,
    -- Regulatory references
    movement_permit_number TEXT,
    envd_number     TEXT,        -- AU: electronic National Vendor Declaration
    -- EPCIS export fields
    epcis_biz_step  TEXT,        -- e.g., 'urn:epcglobal:cbv:bizstep:shipping'
    epcis_disposition TEXT,      -- e.g., 'urn:epcglobal:cbv:disp:in_transit'
    notes           TEXT,
    recorded_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_movements_tenant ON movements(tenant_id);
CREATE INDEX idx_movements_animal ON movements(animal_id);
CREATE INDEX idx_movements_date ON movements(event_date);
CREATE INDEX idx_movements_type ON movements(movement_type);
CREATE INDEX idx_movements_to_holding ON movements(to_holding_id);
```

## Health and Treatment

```sql
-- ============================================================
-- HEALTH EVENTS — diagnosis, treatment, vaccination, withdrawal
-- ============================================================

CREATE TABLE medicines (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            TEXT NOT NULL,
    active_ingredient TEXT,
    manufacturer    TEXT,
    registration_number TEXT,     -- government drug registration
    default_withdrawal_meat_days INTEGER,
    default_withdrawal_milk_days INTEGER,
    default_dose_ml NUMERIC(8,2),
    unit            TEXT DEFAULT 'ml',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_medicines_tenant ON medicines(tenant_id);

CREATE TABLE diagnosis_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    animal_id       UUID NOT NULL REFERENCES animals(id),
    event_date      TIMESTAMPTZ NOT NULL,
    diagnosis_code  TEXT,        -- no universal standard; free text or farm-defined codes
    diagnosis_description TEXT NOT NULL,
    severity        TEXT CHECK (severity IN ('mild', 'moderate', 'severe', 'critical')),
    body_part       TEXT,
    diagnosed_by    UUID REFERENCES users(id),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_diagnosis_events_animal ON diagnosis_events(animal_id);
CREATE INDEX idx_diagnosis_events_date ON diagnosis_events(event_date);

CREATE TABLE treatment_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    animal_id       UUID NOT NULL REFERENCES animals(id),
    event_date      TIMESTAMPTZ NOT NULL,
    diagnosis_event_id UUID REFERENCES diagnosis_events(id),
    medicine_id     UUID REFERENCES medicines(id),
    treatment_type  TEXT NOT NULL
        CHECK (treatment_type IN ('vaccination', 'antibiotic', 'anthelmintic',
                                   'anti_inflammatory', 'supplement', 'topical',
                                   'surgical', 'other')),
    dose_amount     NUMERIC(10,3),
    dose_unit       TEXT DEFAULT 'ml',
    route           TEXT CHECK (route IN ('oral', 'injection_im', 'injection_sc',
                                          'injection_iv', 'topical', 'intramammary', 'other')),
    batch_number    TEXT,
    -- Withdrawal tracking
    withdrawal_meat_end_date DATE,
    withdrawal_milk_end_date DATE,
    administered_by UUID REFERENCES users(id),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_treatment_events_animal ON treatment_events(animal_id);
CREATE INDEX idx_treatment_events_date ON treatment_events(event_date);
CREATE INDEX idx_treatment_events_withdrawal ON treatment_events(withdrawal_meat_end_date);

CREATE TABLE health_status_observations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    animal_id       UUID NOT NULL REFERENCES animals(id),
    event_date      TIMESTAMPTZ NOT NULL,
    status_code     TEXT NOT NULL,   -- e.g., 'tb_free', 'brucellosis_free', 'johnes_suspect'
    status_value    TEXT NOT NULL CHECK (status_value IN ('positive', 'negative', 'suspect', 'unknown')),
    test_type       TEXT,            -- e.g., 'skin_test', 'blood_test', 'milk_elisa'
    lab_reference   TEXT,
    valid_from      DATE,
    valid_until     DATE,
    recorded_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_health_status_animal ON health_status_observations(animal_id);
```

## Reproduction and Breeding

```sql
-- ============================================================
-- REPRODUCTION — per ICAR Repro* event resources
-- ============================================================

CREATE TABLE heat_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    animal_id       UUID NOT NULL REFERENCES animals(id),
    event_date      TIMESTAMPTZ NOT NULL,
    detection_method TEXT CHECK (detection_method IN ('visual', 'sensor', 'tail_paint',
                                                       'activity_monitor', 'progesterone', 'other')),
    intensity       TEXT CHECK (intensity IN ('weak', 'normal', 'strong')),
    recorded_by     UUID REFERENCES users(id),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_heat_events_animal ON heat_events(animal_id);

CREATE TABLE insemination_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    animal_id       UUID NOT NULL REFERENCES animals(id),  -- the dam
    event_date      TIMESTAMPTZ NOT NULL,
    sire_id         UUID REFERENCES animals(id),
    insemination_type TEXT NOT NULL
        CHECK (insemination_type IN ('natural', 'artificial', 'embryo_transfer')),
    -- Semen straw details (for AI)
    straw_id        TEXT,
    straw_breed     TEXT,
    straw_bull_name TEXT,
    straw_company   TEXT,
    technician      UUID REFERENCES users(id),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_insemination_events_animal ON insemination_events(animal_id);
CREATE INDEX idx_insemination_events_sire ON insemination_events(sire_id);

CREATE TABLE pregnancy_check_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    animal_id       UUID NOT NULL REFERENCES animals(id),
    event_date      TIMESTAMPTZ NOT NULL,
    result          TEXT NOT NULL CHECK (result IN ('pregnant', 'not_pregnant', 'inconclusive')),
    method          TEXT CHECK (method IN ('rectal_palpation', 'ultrasound', 'blood_test', 'other')),
    estimated_due_date DATE,
    fetus_count     SMALLINT,
    checked_by      UUID REFERENCES users(id),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pregnancy_checks_animal ON pregnancy_check_events(animal_id);

CREATE TABLE parturition_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    dam_id          UUID NOT NULL REFERENCES animals(id),
    event_date      TIMESTAMPTZ NOT NULL,
    calving_ease    TEXT CHECK (calving_ease IN ('unassisted', 'easy_pull', 'hard_pull',
                                                  'surgical', 'malpresentation')),
    offspring_count SMALLINT NOT NULL DEFAULT 1,
    notes           TEXT,
    recorded_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_parturition_events_dam ON parturition_events(dam_id);

-- Links parturition to offspring born
CREATE TABLE parturition_offspring (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    parturition_event_id UUID NOT NULL REFERENCES parturition_events(id) ON DELETE CASCADE,
    offspring_id    UUID NOT NULL REFERENCES animals(id),
    birth_weight_kg NUMERIC(6,2),
    birth_condition TEXT CHECK (birth_condition IN ('alive', 'stillborn', 'died_within_24h')),
    presentation    TEXT,
    UNIQUE (parturition_event_id, offspring_id)
);

CREATE TABLE gestations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    animal_id       UUID NOT NULL REFERENCES animals(id),
    insemination_event_id UUID REFERENCES insemination_events(id),
    parturition_event_id  UUID REFERENCES parturition_events(id),
    status          TEXT NOT NULL DEFAULT 'confirmed'
        CHECK (status IN ('suspected', 'confirmed', 'completed', 'aborted', 'false_positive')),
    expected_due_date DATE,
    actual_date     DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_gestations_animal ON gestations(animal_id);
```

## Weight and Measurement

```sql
-- ============================================================
-- WEIGHTS AND MEASUREMENTS — per ICAR WeightEvent
-- ============================================================

CREATE TABLE weight_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    animal_id       UUID NOT NULL REFERENCES animals(id),
    event_date      TIMESTAMPTZ NOT NULL,
    weight_kg       NUMERIC(8,2) NOT NULL,
    weight_method   TEXT CHECK (weight_method IN ('scale', 'estimated', 'carcass', 'birth')),
    device_id       TEXT,          -- weighing equipment identifier
    condition_score NUMERIC(3,1), -- body condition score (typically 1-9 or 1-5 scale)
    notes           TEXT,
    recorded_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_weight_events_animal ON weight_events(animal_id);
CREATE INDEX idx_weight_events_date ON weight_events(event_date);

CREATE TABLE carcass_observations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    animal_id       UUID NOT NULL REFERENCES animals(id),
    event_date      TIMESTAMPTZ NOT NULL,
    hot_weight_kg   NUMERIC(8,2),
    cold_weight_kg  NUMERIC(8,2),
    dressing_pct    NUMERIC(5,2),
    fat_score       TEXT,
    muscle_score    TEXT,
    marbling_score  TEXT,
    meat_colour     TEXT,
    fat_colour      TEXT,
    ph_value        NUMERIC(4,2),
    abattoir_ref    TEXT,
    lot_number      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_carcass_animal ON carcass_observations(animal_id);
```

## Feed and Nutrition

```sql
-- ============================================================
-- FEED AND NUTRITION — per ICAR Feed* resources
-- ============================================================

CREATE TABLE feeds (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            TEXT NOT NULL,
    feed_type       TEXT NOT NULL
        CHECK (feed_type IN ('pasture', 'hay', 'silage', 'grain', 'concentrate',
                              'mineral', 'supplement', 'complete_ration', 'other')),
    dry_matter_pct  NUMERIC(5,2),
    crude_protein_pct NUMERIC(5,2),
    metabolisable_energy_mj NUMERIC(6,2),
    cost_per_tonne  NUMERIC(10,2),
    currency_code   CHAR(3) DEFAULT 'USD',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_feeds_tenant ON feeds(tenant_id);

CREATE TABLE rations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            TEXT NOT NULL,
    target_species_id UUID REFERENCES species(id),
    target_class    TEXT,    -- e.g., 'growing', 'lactating', 'dry', 'finishing'
    total_dm_kg     NUMERIC(8,2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE ration_ingredients (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ration_id       UUID NOT NULL REFERENCES rations(id) ON DELETE CASCADE,
    feed_id         UUID NOT NULL REFERENCES feeds(id),
    inclusion_pct   NUMERIC(5,2) NOT NULL,
    amount_kg       NUMERIC(8,2),
    UNIQUE (ration_id, feed_id)
);

CREATE TABLE feed_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    animal_id       UUID REFERENCES animals(id),       -- NULL if group feeding
    animal_set_id   UUID REFERENCES animal_sets(id),   -- NULL if individual feeding
    event_date      TIMESTAMPTZ NOT NULL,
    ration_id       UUID REFERENCES rations(id),
    feed_id         UUID REFERENCES feeds(id),         -- single-feed events
    amount_kg       NUMERIC(8,2) NOT NULL,
    cost            NUMERIC(10,2),
    currency_code   CHAR(3) DEFAULT 'USD',
    recorded_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_feed_events_animal ON feed_events(animal_id);
CREATE INDEX idx_feed_events_set ON feed_events(animal_set_id);
CREATE INDEX idx_feed_events_date ON feed_events(event_date);

CREATE TABLE feed_storage (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    holding_id      UUID NOT NULL REFERENCES holdings(id),
    feed_id         UUID NOT NULL REFERENCES feeds(id),
    quantity_tonnes NUMERIC(10,2) NOT NULL DEFAULT 0,
    last_updated    TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Milking (Dairy)

```sql
-- ============================================================
-- MILKING — per ICAR MilkingVisitEvent and DailyMilkingAverages
-- ============================================================

CREATE TABLE milking_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    animal_id       UUID NOT NULL REFERENCES animals(id),
    event_date      TIMESTAMPTZ NOT NULL,
    session         TEXT CHECK (session IN ('morning', 'midday', 'evening', 'robot')),
    milk_weight_kg  NUMERIC(8,2) NOT NULL,
    fat_pct         NUMERIC(5,2),
    protein_pct     NUMERIC(5,2),
    somatic_cell_count INTEGER,   -- cells per ml
    conductivity    NUMERIC(6,2),
    duration_seconds INTEGER,
    device_id       TEXT,         -- milking parlour/robot identifier
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_milking_events_animal ON milking_events(animal_id);
CREATE INDEX idx_milking_events_date ON milking_events(event_date);

CREATE TABLE lactations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    animal_id       UUID NOT NULL REFERENCES animals(id),
    lactation_number SMALLINT NOT NULL,
    start_date      DATE NOT NULL,
    end_date        DATE,          -- NULL if still lactating
    dry_off_date    DATE,
    total_milk_kg   NUMERIC(10,2),
    total_fat_kg    NUMERIC(8,2),
    total_protein_kg NUMERIC(8,2),
    peak_milk_kg    NUMERIC(8,2),
    peak_date       DATE,
    days_in_milk    INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (animal_id, lactation_number)
);

CREATE INDEX idx_lactations_animal ON lactations(animal_id);
```

## Financial

```sql
-- ============================================================
-- FINANCIAL — purchases, sales, costs
-- ============================================================

CREATE TABLE financial_transactions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    transaction_type TEXT NOT NULL
        CHECK (transaction_type IN ('purchase', 'sale', 'treatment_cost', 'feed_cost',
                                     'veterinary_fee', 'transport_cost', 'insurance',
                                     'registration_fee', 'other_income', 'other_expense')),
    animal_id       UUID REFERENCES animals(id),
    animal_set_id   UUID REFERENCES animal_sets(id),
    event_date      DATE NOT NULL,
    amount          NUMERIC(12,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    counterparty    TEXT,           -- buyer/seller name
    reference       TEXT,           -- invoice or receipt number
    notes           TEXT,
    recorded_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_financial_tenant ON financial_transactions(tenant_id);
CREATE INDEX idx_financial_animal ON financial_transactions(animal_id);
CREATE INDEX idx_financial_date ON financial_transactions(event_date);
```

## Breeding Values and Genetics

```sql
-- ============================================================
-- BREEDING VALUES — per ICAR BreedingValueResource
-- ============================================================

CREATE TABLE breeding_values (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    animal_id       UUID NOT NULL REFERENCES animals(id),
    evaluation_date DATE NOT NULL,
    trait_code      TEXT NOT NULL,      -- e.g., '200DWT', 'MILK', 'EMA', 'RIB_FAT'
    trait_name      TEXT NOT NULL,
    value           NUMERIC(10,3) NOT NULL,
    accuracy        NUMERIC(5,3),       -- 0.00 to 1.00
    percentile      NUMERIC(5,2),
    source          TEXT,               -- breed society or genetic evaluation provider
    evaluation_group TEXT,              -- e.g., 'BREEDPLAN', 'AHDB', 'CDCB'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_breeding_values_animal ON breeding_values(animal_id);
CREATE INDEX idx_breeding_values_trait ON breeding_values(trait_code);
```

## Sensor and Device Data

```sql
-- ============================================================
-- DEVICES AND SENSORS
-- ============================================================

CREATE TABLE devices (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    device_type     TEXT NOT NULL
        CHECK (device_type IN ('rfid_reader', 'scale', 'activity_monitor',
                                'rumination_sensor', 'temperature_sensor',
                                'gps_collar', 'milking_robot', 'weather_station')),
    manufacturer    TEXT,
    model           TEXT,
    serial_number   TEXT,
    holding_id      UUID REFERENCES holdings(id),
    animal_id       UUID REFERENCES animals(id),   -- for animal-mounted devices
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_reading_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_devices_tenant ON devices(tenant_id);
CREATE INDEX idx_devices_animal ON devices(animal_id);

CREATE TABLE sensor_readings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    device_id       UUID NOT NULL REFERENCES devices(id),
    animal_id       UUID REFERENCES animals(id),
    reading_time    TIMESTAMPTZ NOT NULL,
    metric_type     TEXT NOT NULL,   -- 'activity_index', 'rumination_minutes', 'temperature_c', 'steps'
    value           NUMERIC(12,4) NOT NULL,
    unit            TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sensor_readings_device ON sensor_readings(device_id);
CREATE INDEX idx_sensor_readings_animal ON sensor_readings(animal_id);
CREATE INDEX idx_sensor_readings_time ON sensor_readings(reading_time);
-- Partition this table by reading_time for large-scale sensor data
```

## Audit Log

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
    old_values      JSONB,
    new_values      JSONB,
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_log_tenant ON audit_log(tenant_id);
CREATE INDEX idx_audit_log_record ON audit_log(table_name, record_id);
CREATE INDEX idx_audit_log_time ON audit_log(created_at);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Multi-tenancy & Identity | 2 | tenants, users |
| Holdings & Locations | 2 | holdings, paddocks |
| Animal Records | 4 | animals, animal_breed_fractions, animal_identifiers, species/breeds reference |
| Animal Sets | 2 | animal_sets, animal_set_memberships |
| Movements | 1 | movements |
| Health & Treatment | 4 | medicines, diagnosis_events, treatment_events, health_status_observations |
| Reproduction | 6 | heat_events, insemination_events, pregnancy_check_events, parturition_events, parturition_offspring, gestations |
| Weight & Carcass | 2 | weight_events, carcass_observations |
| Feed & Nutrition | 5 | feeds, rations, ration_ingredients, feed_events, feed_storage |
| Milking (Dairy) | 2 | milking_events, lactations |
| Financial | 1 | financial_transactions |
| Genetics | 1 | breeding_values |
| Devices & Sensors | 2 | devices, sensor_readings |
| Audit | 1 | audit_log |
| Reference Data | 2 | species, breeds |
| **Total** | **37** | |

---

## Key Design Decisions

1. **One table per ICAR event type.** Each ICAR ADE resource type (WeightEvent, TreatmentEvent, MovementArrival, etc.) maps to a dedicated table rather than a generic events table. This preserves type safety and allows per-event-type constraints, indexes, and foreign keys.

2. **Multiple identifier schemes via junction table.** Animals can carry many identifiers simultaneously (RFID tag, ear tag, management number, breed society registration). The `animal_identifiers` table supports any scheme, with ISO 11784 fields decomposed for RFID tags.

3. **Denormalized current location on animals table.** While the authoritative movement history is in the `movements` table, the current holding and paddock are denormalized onto `animals` for fast filtering. A trigger or application logic keeps these in sync.

4. **Separate species and breeds reference tables.** These are tenant-independent reference data seeded from ICAR breed codes, enabling multi-species support without per-tenant breed duplication.

5. **Gestation links insemination to parturition.** The `gestations` table creates an explicit lifecycle record connecting mating to birth, enabling reproduction performance analysis without complex date-range queries.

6. **Withdrawal date tracking on treatments.** Meat and milk withdrawal end dates are calculated at insert time and stored, enabling simple queries for animals currently under withdrawal -- critical for food safety compliance.

7. **Sensor readings separated from business events.** High-volume sensor telemetry (activity monitors, temperature sensors) is stored in a separate table designed for time-series partitioning, keeping the transactional tables lean.

8. **EPCIS-ready movement records.** The movements table includes `epcis_biz_step` and `epcis_disposition` fields, allowing direct export to GS1 EPCIS 2.0 format for supply chain visibility without data transformation.

9. **Audit log with JSONB snapshots.** A single audit table captures before/after state for any record change, providing regulatory audit trail without event sourcing complexity.

10. **Row-level security ready.** Every operational table includes `tenant_id` as the first column in composite indexes, enabling PostgreSQL RLS policies for multi-tenant data isolation.
