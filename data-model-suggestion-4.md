# Data Model Suggestion 4: Graph-Relational (Lineage, Movement Networks, and Relationship Analysis)

> Project: Livestock Management System · Created: 2026-05-24

## Philosophy

This model uses a relational core for operational CRUD (animal records, events, holdings) combined with a property graph layer for relationship-heavy queries: pedigree traversal, movement network analysis, disease contact tracing, breeding coefficient calculation, and sire/dam performance evaluation across generations. The graph layer is implemented using PostgreSQL's native recursive CTEs and a dedicated `graph_edges` table, avoiding the need for a separate graph database like Neo4j while enabling graph-style traversal queries.

The design recognizes that livestock management has several inherently graph-shaped problems. Pedigree analysis requires traversing parent-child relationships across many generations to calculate inbreeding coefficients, identify genetic contributors, and evaluate breeding merit. Movement tracing requires following the path of animals through a network of holdings to contain disease outbreaks -- if animal X was at holding A on date Y, which other animals were also there, and where have they since moved? Contact tracing requires finding all animals that shared a paddock or transport with a suspected infected animal within a time window.

These queries are notoriously difficult in pure relational models (requiring multiple self-joins or recursive CTEs with depth limits) but natural in graph models. PostgreSQL 16+ supports SQL/PGQ (Property Graph Queries) via the `CREATE PROPERTY GRAPH` and `GRAPH_TABLE` syntax, and even without PGQ, recursive CTEs with the `graph_edges` pattern provide efficient traversal. This model keeps operational tables relational for fast CRUD and reporting, while maintaining a parallel graph structure for relationship analysis.

**Best for:** Operations with pedigree-registered stud stock, breed societies needing inbreeding coefficient calculation, disease outbreak investigations requiring contact tracing, and AI systems that analyze genetic influence networks or movement patterns.

**Trade-offs:**
- Pro: Pedigree queries across unlimited generations without depth-limited self-joins
- Pro: Contact tracing and disease network analysis are natural graph traversals
- Pro: Inbreeding coefficient calculation from the graph without materialized views
- Pro: Movement network visualization is a direct graph rendering
- Pro: Sire/dam performance evaluation across descendant generations
- Pro: Can leverage PostgreSQL SQL/PGQ in PG 16+ for standards-based graph queries
- Con: Dual-write to both relational tables and graph edges adds application complexity
- Con: Graph edge maintenance requires triggers or application-layer synchronization
- Con: Developers must understand both SQL and graph query patterns
- Con: Graph traversal performance depends on edge index maintenance
- Con: Not all teams have graph modeling experience

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ICAR ADE 1.5 | Operational tables align with ICAR resource types; graph layer adds traversal capability on top |
| ISO 11784/11785 | RFID identifiers stored in relational `animal_identifiers` table |
| ISO 3166 | Country codes on holdings nodes in the graph |
| ICAR Breeding Values | Breeding value propagation analyzed via genetic influence graph |
| EU Reg 1760/2000 | Bovine traceability maintained through movement edges in the graph |
| NLIS Australia | Movement network between PICs modeled as holding-to-holding edges |
| ISO 22005 | Full traceability chain is a graph path from birth to current location |
| GS1 EPCIS 2.0 | Movement events export as EPCIS while the graph enables network analysis |

---

## Relational Core Tables

```sql
-- ============================================================
-- MULTI-TENANCY (same as other models)
-- ============================================================

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

-- ============================================================
-- HOLDINGS
-- ============================================================

CREATE TABLE holdings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            TEXT NOT NULL,
    holding_type    TEXT NOT NULL DEFAULT 'farm',
    country_code    CHAR(2) NOT NULL,
    subdivision_code TEXT,
    pic_code        TEXT,
    cph_number      TEXT,
    herd_number     TEXT,
    latitude        NUMERIC(10,7),
    longitude       NUMERIC(10,7),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_holdings_tenant ON holdings(tenant_id);

CREATE TABLE paddocks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    holding_id      UUID NOT NULL REFERENCES holdings(id),
    name            TEXT NOT NULL,
    paddock_type    TEXT DEFAULT 'grazing',
    area_hectares   NUMERIC(10,2),
    capacity_head   INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- ANIMALS
-- ============================================================

CREATE TABLE animals (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    species         TEXT NOT NULL,
    primary_breed   TEXT,
    management_tag  TEXT,
    name            TEXT,
    sex             TEXT NOT NULL CHECK (sex IN ('male', 'female', 'unknown')),
    is_castrated    BOOLEAN NOT NULL DEFAULT false,
    date_of_birth   DATE,
    -- Direct parent references (also maintained in graph edges)
    dam_id          UUID REFERENCES animals(id),
    sire_id         UUID REFERENCES animals(id),
    -- Status
    status          TEXT NOT NULL DEFAULT 'active',
    status_date     DATE,
    -- Current location
    current_holding_id UUID REFERENCES holdings(id),
    current_paddock_id UUID REFERENCES paddocks(id),
    -- Cached metrics
    latest_weight_kg NUMERIC(8,2),
    latest_weight_date DATE,
    -- Generation depth (cached for query optimization)
    generation_number INTEGER,  -- 0 = founder, 1 = first known generation, etc.
    -- Inbreeding coefficient (cached, recalculated periodically)
    inbreeding_coefficient NUMERIC(8,6),
    inbreeding_calc_date DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_animals_tenant ON animals(tenant_id);
CREATE INDEX idx_animals_status ON animals(tenant_id, status);
CREATE INDEX idx_animals_dam ON animals(dam_id);
CREATE INDEX idx_animals_sire ON animals(sire_id);
CREATE INDEX idx_animals_holding ON animals(current_holding_id);
CREATE INDEX idx_animals_tag ON animals(tenant_id, management_tag);
CREATE INDEX idx_animals_breed ON animals(tenant_id, primary_breed);

CREATE TABLE animal_identifiers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    animal_id       UUID NOT NULL REFERENCES animals(id) ON DELETE CASCADE,
    scheme          TEXT NOT NULL,
    identifier      TEXT NOT NULL,
    rfid_country_code CHAR(3),
    rfid_national_id TEXT,
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    applied_date    DATE,
    removed_date    DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (scheme, identifier)
);

CREATE INDEX idx_identifiers_animal ON animal_identifiers(animal_id);
CREATE INDEX idx_identifiers_lookup ON animal_identifiers(scheme, identifier);
```

## Property Graph Layer

```sql
-- ============================================================
-- PROPERTY GRAPH — nodes and edges for relationship traversal
-- ============================================================

-- Graph nodes represent entities that participate in relationships
CREATE TABLE graph_nodes (
    id              UUID PRIMARY KEY,   -- same UUID as the source entity
    tenant_id       UUID NOT NULL,
    node_type       TEXT NOT NULL
        CHECK (node_type IN ('animal', 'holding', 'paddock', 'animal_set',
                              'semen_straw', 'breed', 'veterinarian')),
    -- Cached label for graph visualization
    label           TEXT NOT NULL,      -- display name in graph views
    -- Node properties (denormalized for graph queries without JOINs back to source)
    properties      JSONB NOT NULL DEFAULT '{}',
    -- animal: {"species": "cattle", "breed": "Angus", "sex": "female", "dob": "2022-03-15", "status": "active"}
    -- holding: {"type": "farm", "country": "AU", "pic": "3ABCD123"}
    -- semen_straw: {"bull_name": "...", "company": "...", "breed": "Angus"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_graph_nodes_tenant ON graph_nodes(tenant_id);
CREATE INDEX idx_graph_nodes_type ON graph_nodes(tenant_id, node_type);
CREATE INDEX idx_graph_nodes_props ON graph_nodes USING GIN (properties);

-- Graph edges represent typed, directed, optionally temporal relationships
CREATE TABLE graph_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    -- Relationship endpoints
    from_node_id    UUID NOT NULL REFERENCES graph_nodes(id),
    to_node_id      UUID NOT NULL REFERENCES graph_nodes(id),
    -- Relationship type
    edge_type       TEXT NOT NULL,
    /*
        Pedigree edges:
          'dam_of'        animal -> animal (dam -> offspring)
          'sire_of'       animal -> animal (sire -> offspring)
          'granddam_of'   (materialized for 2-generation queries)
          'grandsire_of'  (materialized for 2-generation queries)

        Movement edges:
          'moved_to'      animal -> holding (with temporal validity)
          'located_at'    animal -> paddock (current location)
          'transported_with' animal -> animal (shared transport)

        Contact edges:
          'co_located'    animal -> animal (shared paddock in time window)
          'co_transported' animal -> animal (shared vehicle)

        Breeding edges:
          'mated_with'    animal -> animal (insemination event)
          'semen_from'    semen_straw -> animal (AI straw origin)

        Group edges:
          'member_of'     animal -> animal_set

        Health edges:
          'treated_by'    animal -> veterinarian
          'exposed_to'    animal -> animal (disease exposure)
    */
    -- Temporal validity (when this relationship was/is active)
    valid_from      TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_to        TIMESTAMPTZ,   -- NULL = currently active
    -- Edge properties
    properties      JSONB NOT NULL DEFAULT '{}',
    -- moved_to: {"movement_id": "...", "transport_ref": "CON-2024-0045", "permit": "MP-2024-1234"}
    -- dam_of: {"birth_rank": 1, "birth_date": "2024-03-15"}
    -- co_located: {"paddock_id": "...", "overlap_days": 14}
    -- mated_with: {"method": "artificial", "straw_id": "STR-AA-2024-0033"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Critical indexes for graph traversal
CREATE INDEX idx_edges_from ON graph_edges(from_node_id);
CREATE INDEX idx_edges_to ON graph_edges(to_node_id);
CREATE INDEX idx_edges_type ON graph_edges(edge_type);
CREATE INDEX idx_edges_tenant_type ON graph_edges(tenant_id, edge_type);
CREATE INDEX idx_edges_temporal ON graph_edges(valid_from, valid_to);
CREATE INDEX idx_edges_from_type ON graph_edges(from_node_id, edge_type);
CREATE INDEX idx_edges_to_type ON graph_edges(to_node_id, edge_type);
CREATE INDEX idx_edges_props ON graph_edges USING GIN (properties);

-- Composite index for common pedigree traversal
CREATE INDEX idx_edges_pedigree ON graph_edges(to_node_id, edge_type)
    WHERE edge_type IN ('dam_of', 'sire_of');
```

## PostgreSQL SQL/PGQ Property Graph Definition (PG 16+)

```sql
-- ============================================================
-- SQL/PGQ PROPERTY GRAPH DEFINITION
-- Available in PostgreSQL 16+ with SQL/PGQ support
-- ============================================================

CREATE PROPERTY GRAPH livestock_graph
    VERTEX TABLES (
        graph_nodes
            KEY (id)
            LABEL animal     PROPERTIES (label, properties) WHERE (node_type = 'animal'),
            LABEL holding    PROPERTIES (label, properties) WHERE (node_type = 'holding'),
            LABEL paddock    PROPERTIES (label, properties) WHERE (node_type = 'paddock'),
            LABEL animal_set PROPERTIES (label, properties) WHERE (node_type = 'animal_set')
    )
    EDGE TABLES (
        graph_edges
            KEY (id)
            SOURCE KEY (from_node_id) REFERENCES graph_nodes
            DESTINATION KEY (to_node_id) REFERENCES graph_nodes
            LABEL dam_of       PROPERTIES (valid_from, properties) WHERE (edge_type = 'dam_of'),
            LABEL sire_of      PROPERTIES (valid_from, properties) WHERE (edge_type = 'sire_of'),
            LABEL moved_to     PROPERTIES (valid_from, valid_to, properties) WHERE (edge_type = 'moved_to'),
            LABEL located_at   PROPERTIES (valid_from, valid_to, properties) WHERE (edge_type = 'located_at'),
            LABEL co_located   PROPERTIES (valid_from, valid_to, properties) WHERE (edge_type = 'co_located'),
            LABEL mated_with   PROPERTIES (valid_from, properties) WHERE (edge_type = 'mated_with'),
            LABEL member_of    PROPERTIES (valid_from, valid_to, properties) WHERE (edge_type = 'member_of'),
            LABEL exposed_to   PROPERTIES (valid_from, properties) WHERE (edge_type = 'exposed_to')
    );
```

## Operational Event Tables

```sql
-- ============================================================
-- EVENTS — relational tables for operational CRUD
-- (Simplified; same structure as Suggestion 1 but fewer tables)
-- ============================================================

CREATE TABLE movement_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    animal_id       UUID NOT NULL REFERENCES animals(id),
    movement_type   TEXT NOT NULL,
    event_date      TIMESTAMPTZ NOT NULL,
    from_holding_id UUID REFERENCES holdings(id),
    to_holding_id   UUID REFERENCES holdings(id),
    from_paddock_id UUID REFERENCES paddocks(id),
    to_paddock_id   UUID REFERENCES paddocks(id),
    transport_ref   TEXT,
    movement_permit TEXT,
    recorded_by     UUID REFERENCES users(id),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_move_events_animal ON movement_events(animal_id);
CREATE INDEX idx_move_events_date ON movement_events(event_date);

CREATE TABLE health_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    animal_id       UUID NOT NULL REFERENCES animals(id),
    event_type      TEXT NOT NULL,  -- 'diagnosis', 'treatment', 'vaccination', 'health_status'
    event_date      TIMESTAMPTZ NOT NULL,
    description     TEXT,
    medicine_name   TEXT,
    dose_amount     NUMERIC(10,3),
    dose_unit       TEXT,
    route           TEXT,
    batch_number    TEXT,
    withdrawal_meat_end DATE,
    withdrawal_milk_end DATE,
    severity        TEXT,
    recorded_by     UUID REFERENCES users(id),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_health_events_animal ON health_events(animal_id);
CREATE INDEX idx_health_events_date ON health_events(event_date);

CREATE TABLE repro_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    animal_id       UUID NOT NULL REFERENCES animals(id),
    event_type      TEXT NOT NULL,  -- 'heat', 'insemination', 'pregnancy_check', 'parturition', 'abortion'
    event_date      TIMESTAMPTZ NOT NULL,
    sire_id         UUID REFERENCES animals(id),
    result          TEXT,
    calving_ease    TEXT,
    offspring_count SMALLINT,
    method          TEXT,
    straw_id        TEXT,
    recorded_by     UUID REFERENCES users(id),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_repro_events_animal ON repro_events(animal_id);
CREATE INDEX idx_repro_events_date ON repro_events(event_date);

CREATE TABLE weight_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    animal_id       UUID NOT NULL REFERENCES animals(id),
    event_date      TIMESTAMPTZ NOT NULL,
    weight_kg       NUMERIC(8,2) NOT NULL,
    method          TEXT,
    condition_score NUMERIC(3,1),
    device_id       TEXT,
    recorded_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_weight_events_animal ON weight_events(animal_id);
CREATE INDEX idx_weight_events_date ON weight_events(event_date);

CREATE TABLE milking_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    animal_id       UUID NOT NULL REFERENCES animals(id),
    event_date      TIMESTAMPTZ NOT NULL,
    session         TEXT,
    milk_weight_kg  NUMERIC(8,2) NOT NULL,
    fat_pct         NUMERIC(5,2),
    protein_pct     NUMERIC(5,2),
    somatic_cell_count INTEGER,
    device_id       TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_milking_events_animal ON milking_events(animal_id);
CREATE INDEX idx_milking_events_date ON milking_events(event_date);

CREATE TABLE feed_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    animal_id       UUID REFERENCES animals(id),
    animal_set_id   UUID,
    event_date      TIMESTAMPTZ NOT NULL,
    feed_name       TEXT NOT NULL,
    feed_type       TEXT,
    amount_kg       NUMERIC(8,2) NOT NULL,
    cost            NUMERIC(10,2),
    recorded_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_feed_events_animal ON feed_events(animal_id);
CREATE INDEX idx_feed_events_date ON feed_events(event_date);

CREATE TABLE financial_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    animal_id       UUID REFERENCES animals(id),
    event_type      TEXT NOT NULL,
    event_date      DATE NOT NULL,
    amount          NUMERIC(12,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    counterparty    TEXT,
    reference       TEXT,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_financial_events_animal ON financial_events(animal_id);
```

## Graph Synchronization Triggers

```sql
-- ============================================================
-- TRIGGERS — keep graph edges in sync with relational tables
-- ============================================================

-- When an animal is inserted, create a graph node and pedigree edges
CREATE OR REPLACE FUNCTION sync_animal_to_graph()
RETURNS TRIGGER AS $$
BEGIN
    -- Create or update graph node
    INSERT INTO graph_nodes (id, tenant_id, node_type, label, properties)
    VALUES (
        NEW.id, NEW.tenant_id, 'animal',
        COALESCE(NEW.name, NEW.management_tag, NEW.id::text),
        jsonb_build_object(
            'species', NEW.species,
            'breed', NEW.primary_breed,
            'sex', NEW.sex,
            'dob', NEW.date_of_birth,
            'status', NEW.status
        )
    )
    ON CONFLICT (id) DO UPDATE SET
        label = EXCLUDED.label,
        properties = EXCLUDED.properties,
        updated_at = now();

    -- Create dam edge
    IF NEW.dam_id IS NOT NULL THEN
        INSERT INTO graph_edges (tenant_id, from_node_id, to_node_id, edge_type,
                                  valid_from, properties)
        VALUES (NEW.tenant_id, NEW.dam_id, NEW.id, 'dam_of',
                COALESCE(NEW.date_of_birth::timestamptz, now()),
                jsonb_build_object('birth_date', NEW.date_of_birth))
        ON CONFLICT DO NOTHING;
    END IF;

    -- Create sire edge
    IF NEW.sire_id IS NOT NULL THEN
        INSERT INTO graph_edges (tenant_id, from_node_id, to_node_id, edge_type,
                                  valid_from, properties)
        VALUES (NEW.tenant_id, NEW.sire_id, NEW.id, 'sire_of',
                COALESCE(NEW.date_of_birth::timestamptz, now()),
                jsonb_build_object('birth_date', NEW.date_of_birth))
        ON CONFLICT DO NOTHING;
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_animal_graph_sync
    AFTER INSERT OR UPDATE ON animals
    FOR EACH ROW EXECUTE FUNCTION sync_animal_to_graph();

-- When a movement is recorded, create a movement edge
CREATE OR REPLACE FUNCTION sync_movement_to_graph()
RETURNS TRIGGER AS $$
BEGIN
    -- Close previous location edge
    UPDATE graph_edges
    SET valid_to = NEW.event_date
    WHERE from_node_id = NEW.animal_id
      AND edge_type = 'moved_to'
      AND valid_to IS NULL;

    -- Create new movement edge
    IF NEW.to_holding_id IS NOT NULL THEN
        INSERT INTO graph_edges (tenant_id, from_node_id, to_node_id, edge_type,
                                  valid_from, properties)
        VALUES (NEW.tenant_id, NEW.animal_id, NEW.to_holding_id, 'moved_to',
                NEW.event_date,
                jsonb_build_object(
                    'movement_id', NEW.id,
                    'movement_type', NEW.movement_type,
                    'transport_ref', NEW.transport_ref
                ));
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_movement_graph_sync
    AFTER INSERT ON movement_events
    FOR EACH ROW EXECUTE FUNCTION sync_movement_to_graph();

-- When a breeding event is recorded, create a mating edge
CREATE OR REPLACE FUNCTION sync_repro_to_graph()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.event_type = 'insemination' AND NEW.sire_id IS NOT NULL THEN
        INSERT INTO graph_edges (tenant_id, from_node_id, to_node_id, edge_type,
                                  valid_from, properties)
        VALUES (NEW.tenant_id, NEW.sire_id, NEW.animal_id, 'mated_with',
                NEW.event_date,
                jsonb_build_object(
                    'method', NEW.method,
                    'straw_id', NEW.straw_id
                ));
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_repro_graph_sync
    AFTER INSERT ON repro_events
    FOR EACH ROW EXECUTE FUNCTION sync_repro_to_graph();
```

## Key Graph Query Patterns

### Pedigree: Full Ancestry (Recursive CTE)

```sql
-- Trace full ancestry of an animal (all ancestors, unlimited depth)
WITH RECURSIVE ancestry AS (
    -- Base case: the animal itself
    SELECT
        n.id, n.label, n.properties,
        0 AS depth,
        ARRAY[n.id] AS path,
        NULL::text AS relationship
    FROM graph_nodes n
    WHERE n.id = $animal_id

    UNION ALL

    -- Recursive: follow dam_of and sire_of edges BACKWARDS (parent -> offspring)
    SELECT
        parent.id, parent.label, parent.properties,
        a.depth + 1,
        a.path || parent.id,
        e.edge_type
    FROM ancestry a
    JOIN graph_edges e ON e.to_node_id = a.id
        AND e.edge_type IN ('dam_of', 'sire_of')
    JOIN graph_nodes parent ON parent.id = e.from_node_id
    WHERE parent.id != ALL(a.path)  -- prevent cycles
      AND a.depth < 10              -- depth limit
)
SELECT id, label, properties->>'breed' AS breed,
       depth, relationship,
       properties->>'sex' AS sex,
       properties->>'dob' AS date_of_birth
FROM ancestry
WHERE depth > 0
ORDER BY depth, relationship;
```

### Pedigree: All Descendants of a Sire

```sql
-- Find all descendants of a sire across generations
WITH RECURSIVE descendants AS (
    SELECT
        n.id, n.label, n.properties,
        0 AS generation,
        ARRAY[n.id] AS path
    FROM graph_nodes n
    WHERE n.id = $sire_id

    UNION ALL

    SELECT
        child.id, child.label, child.properties,
        d.generation + 1,
        d.path || child.id
    FROM descendants d
    JOIN graph_edges e ON e.from_node_id = d.id
        AND e.edge_type IN ('sire_of', 'dam_of')
    JOIN graph_nodes child ON child.id = e.to_node_id
    WHERE child.id != ALL(d.path)
      AND d.generation < 5
)
SELECT id, label,
       generation,
       properties->>'species' AS species,
       properties->>'breed' AS breed,
       properties->>'sex' AS sex,
       properties->>'status' AS status
FROM descendants
WHERE generation > 0
ORDER BY generation, label;
```

### Inbreeding Coefficient Calculation

```sql
-- Simplified Wright's coefficient: find common ancestors and calculate F
-- This finds shared ancestors between dam and sire of a target animal
WITH RECURSIVE
dam_ancestors AS (
    SELECT n.id, 0 AS depth, ARRAY[n.id] AS path
    FROM animals a
    JOIN graph_nodes n ON n.id = a.dam_id
    WHERE a.id = $animal_id
    UNION ALL
    SELECT parent.id, da.depth + 1, da.path || parent.id
    FROM dam_ancestors da
    JOIN graph_edges e ON e.to_node_id = da.id AND e.edge_type IN ('dam_of', 'sire_of')
    JOIN graph_nodes parent ON parent.id = e.from_node_id
    WHERE parent.id != ALL(da.path) AND da.depth < 8
),
sire_ancestors AS (
    SELECT n.id, 0 AS depth, ARRAY[n.id] AS path
    FROM animals a
    JOIN graph_nodes n ON n.id = a.sire_id
    WHERE a.id = $animal_id
    UNION ALL
    SELECT parent.id, sa.depth + 1, sa.path || parent.id
    FROM sire_ancestors sa
    JOIN graph_edges e ON e.to_node_id = sa.id AND e.edge_type IN ('dam_of', 'sire_of')
    JOIN graph_nodes parent ON parent.id = e.from_node_id
    WHERE parent.id != ALL(sa.path) AND sa.depth < 8
),
common_ancestors AS (
    SELECT da.id,
           da.depth AS dam_path_length,
           sa.depth AS sire_path_length
    FROM dam_ancestors da
    JOIN sire_ancestors sa ON da.id = sa.id
)
-- Wright's formula: F = SUM over common ancestors of (0.5)^(n1 + n2 + 1)
SELECT COALESCE(SUM(POWER(0.5, dam_path_length + sire_path_length + 1)), 0)
       AS inbreeding_coefficient
FROM common_ancestors;
```

### Disease Contact Tracing

```sql
-- Find all animals that were co-located with a suspect animal
-- within a time window (e.g., 30 days before and after detection date)
WITH suspect_locations AS (
    -- Where was the suspect animal in the window?
    SELECT e.to_node_id AS holding_id, e.valid_from, 
           COALESCE(e.valid_to, now()) AS valid_to
    FROM graph_edges e
    WHERE e.from_node_id = $suspect_animal_id
      AND e.edge_type = 'moved_to'
      AND e.valid_from <= $detection_date + INTERVAL '30 days'
      AND COALESCE(e.valid_to, now()) >= $detection_date - INTERVAL '30 days'
),
contact_animals AS (
    -- Which other animals were at those holdings during those times?
    SELECT DISTINCT e.from_node_id AS animal_id,
           sl.holding_id,
           GREATEST(e.valid_from, sl.valid_from) AS overlap_start,
           LEAST(COALESCE(e.valid_to, now()), sl.valid_to) AS overlap_end
    FROM graph_edges e
    JOIN suspect_locations sl ON e.to_node_id = sl.holding_id
    WHERE e.edge_type = 'moved_to'
      AND e.from_node_id != $suspect_animal_id
      AND e.valid_from <= sl.valid_to
      AND COALESCE(e.valid_to, now()) >= sl.valid_from
)
SELECT ca.animal_id, a.management_tag, a.species, a.primary_breed,
       a.status, a.current_holding_id,
       h.name AS contact_holding,
       ca.overlap_start, ca.overlap_end,
       ca.overlap_end - ca.overlap_start AS overlap_duration
FROM contact_animals ca
JOIN animals a ON a.id = ca.animal_id
JOIN holdings h ON h.id = ca.holding_id
ORDER BY ca.overlap_start;
```

### Movement Network Analysis

```sql
-- Find all holdings connected to a suspect holding within N hops
-- (for movement restriction zone determination)
WITH RECURSIVE holding_network AS (
    SELECT
        $suspect_holding_id AS holding_id,
        0 AS hops,
        ARRAY[$suspect_holding_id] AS path
    
    UNION ALL
    
    SELECT
        CASE 
            WHEN e.to_node_id = hn.holding_id THEN n_from.id
            ELSE e.to_node_id
        END AS holding_id,
        hn.hops + 1,
        hn.path || CASE 
            WHEN e.to_node_id = hn.holding_id THEN n_from.id
            ELSE e.to_node_id
        END
    FROM holding_network hn
    JOIN graph_edges e ON (e.to_node_id = hn.holding_id OR e.from_node_id = hn.holding_id)
        AND e.edge_type = 'moved_to'
        AND e.valid_from >= $lookback_date
    JOIN graph_nodes n_from ON n_from.id = e.from_node_id AND n_from.node_type = 'animal'
    WHERE hn.hops < $max_hops
      AND CASE 
            WHEN e.to_node_id = hn.holding_id THEN e.from_node_id
            ELSE e.to_node_id
          END != ALL(hn.path)
)
SELECT DISTINCT h.id, h.name, h.pic_code, h.country_code,
       MIN(hn.hops) AS min_hops
FROM holding_network hn
JOIN holdings h ON h.id = hn.holding_id
GROUP BY h.id, h.name, h.pic_code, h.country_code
ORDER BY min_hops;
```

### Sire Performance Across Progeny

```sql
-- Evaluate a sire's performance by aggregating descendant weight data
WITH sire_progeny AS (
    SELECT e.to_node_id AS progeny_id
    FROM graph_edges e
    WHERE e.from_node_id = $sire_id
      AND e.edge_type = 'sire_of'
)
SELECT 
    COUNT(DISTINCT sp.progeny_id) AS total_progeny,
    COUNT(DISTINCT w.animal_id) AS progeny_with_weights,
    AVG(w.weight_kg) AS avg_weight_kg,
    MIN(w.weight_kg) AS min_weight_kg,
    MAX(w.weight_kg) AS max_weight_kg,
    STDDEV(w.weight_kg) AS stddev_weight_kg
FROM sire_progeny sp
LEFT JOIN weight_events w ON w.animal_id = sp.progeny_id
LEFT JOIN animals a ON a.id = sp.progeny_id
WHERE a.status = 'active';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Multi-tenancy | 2 | tenants, users |
| Holdings | 2 | holdings, paddocks |
| Animals | 2 | animals, animal_identifiers |
| Graph Layer | 2 | graph_nodes, graph_edges |
| Movement Events | 1 | movement_events |
| Health Events | 1 | health_events |
| Reproduction Events | 1 | repro_events |
| Weight Events | 1 | weight_events |
| Milking Events | 1 | milking_events |
| Feed Events | 1 | feed_events |
| Financial Events | 1 | financial_events |
| **Total** | **15** | Plus 3 trigger functions for graph sync |

---

## Key Design Decisions

1. **Dual representation: relational + graph.** Operational data lives in relational tables for fast CRUD, filtering, and reporting. The graph layer (`graph_nodes` + `graph_edges`) provides traversal capability for relationship-heavy queries. Triggers keep them synchronized.

2. **Temporal edges for movement tracing.** Every `moved_to` edge has `valid_from` and `valid_to` timestamps, enabling temporal overlap queries for disease contact tracing. When an animal moves, the previous edge is closed and a new one is opened.

3. **Cached inbreeding coefficient.** Wright's inbreeding coefficient is computationally expensive (recursive CTE across multiple generations). The result is cached on the `animals` table and recalculated periodically or on demand, with the calculation date recorded.

4. **Graph nodes duplicate key properties.** The `properties` JSONB on `graph_nodes` contains denormalized copies of species, breed, sex, and status from the animals table. This avoids JOINs back to the source table during graph traversals, which would defeat the purpose of the graph layer.

5. **Edge types cover multiple relationship domains.** The `graph_edges` table handles pedigree (dam_of, sire_of), movement (moved_to), breeding (mated_with), grouping (member_of), and disease (exposed_to) relationships in a single table. The `edge_type` column discriminates, and partial indexes optimize common traversal patterns.

6. **SQL/PGQ-ready design.** The `graph_nodes` and `graph_edges` tables are structured to work with PostgreSQL 16+ SQL/PGQ `CREATE PROPERTY GRAPH` syntax, enabling future migration to standards-based graph queries without schema changes.

7. **Trigger-based synchronization.** Graph edges are created automatically when animals, movements, and breeding events are inserted. This ensures the graph is always consistent with the relational data without requiring application-layer dual-writes.

8. **Contact tracing as a first-class capability.** The `co_located` edge type can be materialized periodically by a background job that detects temporal overlaps in `moved_to` edges. This pre-computes the most expensive contact tracing query for rapid disease response.

9. **Generation numbering for query optimization.** The `generation_number` cached on animals provides a fast filter for pedigree depth without requiring recursive traversal, enabling queries like "show me all generation-3 descendants."

10. **Separate event tables for type safety.** Unlike Suggestion 3 (unified event table with JSONB), this model keeps separate relational event tables for movements, health, reproduction, and weights. This preserves column-level constraints and foreign keys while the graph layer handles cross-entity relationship queries.
