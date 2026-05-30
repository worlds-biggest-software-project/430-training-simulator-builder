# Data Model Suggestion 4: Graph Database (Neo4j) + PostgreSQL Polyglot

> Project: Training Simulator Builder (Candidate #430)
> Generated: 2026-05-25

---

## Overview

This model uses a polyglot persistence architecture with **Neo4j** as the primary store for the scenario graph (the branching narrative structure that is the product's core domain) and **PostgreSQL** for everything else (users, organizations, learner attempts, LMS integrations, audit logs, analytics aggregations). The key thesis is that a branching training scenario **is a graph** -- and a graph database models, queries, and traverses graphs natively rather than simulating them with join tables and recursive CTEs.

A training simulator builder's core data structure -- nodes connected by edges representing decision paths, with conditional branching, variable-driven routing, and cycle detection -- maps directly to the labeled property graph model that Neo4j implements. Every operation the visual editor performs (add a node, connect two nodes, find all paths from start to end, detect unreachable nodes, validate that every branch converges, check for infinite loops) is a native graph operation expressed concisely in Cypher rather than an expensive SQL construction.

The trade-off is operational complexity: two database systems instead of one. This model is justified only if the graph traversal requirements are central enough to the product's quality and performance that the overhead of a second database is warranted. For a branching scenario authoring tool, they are.

---

## Design Principles

1. **Graph data in Neo4j; everything else in PostgreSQL.** The division is clean: if the data is nodes-and-edges (scenarios, their structure, character relationships), it lives in Neo4j. If the data is tabular/transactional (users, attempts, billing, LMS configs), it lives in PostgreSQL.
2. **Neo4j is the source of truth for scenario structure.** The visual editor reads from and writes to Neo4j. Published scenario snapshots are serialized from Neo4j to PostgreSQL (as JSONB) for the learner player and SCORM export pipeline.
3. **No cross-database joins.** Services query one database at a time. Application-level composition joins data from both sources where needed.
4. **Eventual consistency between databases is acceptable for non-critical paths.** Published scenario snapshots in PostgreSQL may lag Neo4j by seconds. This is acceptable because publishing is an explicit, user-initiated action.
5. **Self-hosted deployment must remain feasible.** Both Neo4j Community Edition and PostgreSQL are open source. The architecture must not require Neo4j Enterprise features.

---

## Neo4j Graph Schema

### Node Labels and Properties

```cypher
// ============================================================
// SCENARIO (top-level container)
// ============================================================
// Label: Scenario
// Properties:
//   id:                     String (UUID)
//   organization_id:        String (UUID, links to PostgreSQL)
//   project_id:             String (UUID, links to PostgreSQL)
//   name:                   String
//   description:            String
//   status:                 String (draft|in_review|approved|published|archived)
//   version:                Integer
//   locale:                 String
//   training_objective:     String
//   difficulty_level:       String
//   estimated_duration_min: Integer
//   passing_score:          Float
//   max_score:              Float
//   is_template:            Boolean
//   created_by:             String (UUID, links to PostgreSQL)
//   created_at:             DateTime
//   updated_at:             DateTime

CREATE CONSTRAINT scenario_id IF NOT EXISTS
  FOR (s:Scenario) REQUIRE s.id IS UNIQUE;

CREATE INDEX scenario_org IF NOT EXISTS
  FOR (s:Scenario) ON (s.organization_id);

CREATE INDEX scenario_status IF NOT EXISTS
  FOR (s:Scenario) ON (s.status);

// ============================================================
// SCENE NODE (a single point in the scenario graph)
// ============================================================
// Label: SceneNode
// Additional labels by type: DialogueNode, DecisionPoint,
//   FreeResponseNode, MediaNode, ConsequenceNode, BranchMerge,
//   DebriefNode, DelayNode, InformationNode
//
// Common properties:
//   id:                String (UUID)
//   title:             String
//   position_x:        Float
//   position_y:        Float
//   sort_order:        Integer
//   is_entry_point:    Boolean
//   is_terminal:       Boolean
//   score_value:       Float
//   feedback_text:     String
//   created_at:        DateTime
//   updated_at:        DateTime
//
// DialogueNode additional properties:
//   character_id:      String (UUID)
//   dialogue_text:     String
//   narrator_text:     String
//   voice_audio_url:   String
//
// DecisionPoint additional properties:
//   prompt_text:       String
//   time_limit_sec:    Integer
//
// FreeResponseNode additional properties:
//   prompt_text:       String
//   eval_rubric:       String
//   expected_keywords: List<String>
//   min_length:        Integer
//   max_length:        Integer
//
// MediaNode additional properties:
//   media_url:         String
//   media_type:        String (image|video|audio)
//   caption:           String
//   auto_advance_sec:  Integer
//
// DebriefNode additional properties:
//   debrief_template:  String (summary|detailed|custom)
//   custom_html:       String
//
// DelayNode additional properties:
//   delay_seconds:     Integer
//   message:           String

CREATE CONSTRAINT scene_node_id IF NOT EXISTS
  FOR (n:SceneNode) REQUIRE n.id IS UNIQUE;

// ============================================================
// CHARACTER
// ============================================================
// Label: Character
// Properties:
//   id:                String (UUID)
//   organization_id:   String (UUID)
//   name:              String
//   role_label:        String
//   avatar_style:      String
//   avatar_url:        String
//   gender:            String
//   age_range:         String
//   voice_provider:    String
//   voice_id:          String
//   voice_language:    String
//   voice_style:       String
//   is_library:        Boolean
//   created_by:        String (UUID)
//   created_at:        DateTime

CREATE CONSTRAINT character_id IF NOT EXISTS
  FOR (c:Character) REQUIRE c.id IS UNIQUE;

// ============================================================
// VARIABLE (scenario state variable)
// ============================================================
// Label: Variable
// Properties:
//   id:                String (UUID)
//   name:              String
//   var_type:          String (string|number|boolean)
//   default_value:     String
//   description:       String

CREATE CONSTRAINT variable_id IF NOT EXISTS
  FOR (v:Variable) REQUIRE v.id IS UNIQUE;

// ============================================================
// POLICY DOCUMENT (for AI grounding)
// ============================================================
// Label: PolicyDocument
// Properties:
//   id:                String (UUID)
//   organization_id:   String (UUID)
//   name:              String
//   document_type:     String
//   file_url:          String

CREATE CONSTRAINT policy_doc_id IF NOT EXISTS
  FOR (p:PolicyDocument) REQUIRE p.id IS UNIQUE;
```

### Relationship Types

```cypher
// ============================================================
// RELATIONSHIPS
// ============================================================

// Scenario contains nodes
// (Scenario)-[:CONTAINS_NODE]->(SceneNode)
//   No properties needed; membership relationship

// Node-to-node transitions (the branching structure)
// (SceneNode)-[:LEADS_TO {properties}]->(SceneNode)
//   Properties:
//     id:              String (UUID)
//     edge_type:       String (choice|fallthrough|conditional|timeout|default)
//     label:           String (displayed choice text)
//     sort_order:      Integer
//     score_delta:     Float
//     is_correct:      Boolean
//     feedback_text:   String

// Conditional edge with conditions
// (SceneNode)-[:LEADS_TO_IF {properties}]->(SceneNode)
//   Additional properties:
//     conditions:      List<String> (serialized condition expressions)
//     conditions_json: String (JSON-encoded conditions array for complex cases)

// Variable mutations on transitions
// (SceneNode)-[:LEADS_TO]->(SceneNode)
//     mutations_json:  String (JSON-encoded mutations array)
//       [{ "variable_id": "uuid", "operation": "set", "value": "true" }]

// Scenario uses character
// (Scenario)-[:USES_CHARACTER]->(Character)

// Node features character (speaking in this node)
// (SceneNode)-[:SPOKEN_BY]->(Character)

// Scenario tracks variable
// (Scenario)-[:TRACKS_VARIABLE]->(Variable)

// Scenario grounded in policy
// (Scenario)-[:GROUNDED_IN]->(PolicyDocument)
//   Properties:
//     relevance_note:  String
//     linked_by:       String (UUID)
//     linked_at:       DateTime

// Template relationship
// (Scenario)-[:DERIVED_FROM]->(Scenario)
//   Properties:
//     derived_at:      DateTime
//     derived_by:      String (UUID)
```

### Example Graph Creation

```cypher
// Create a simple harassment training scenario

// Create the scenario
CREATE (s:Scenario {
  id: 'scenario-001',
  organization_id: 'org-001',
  project_id: 'proj-001',
  name: 'Workplace Harassment Response',
  status: 'draft',
  version: 1,
  locale: 'en-US',
  training_objective: 'Learn to respond appropriately to witnessed workplace harassment',
  passing_score: 70.0,
  max_score: 100.0,
  created_by: 'user-001',
  created_at: datetime()
})

// Create characters
CREATE (mgr:Character {
  id: 'char-001',
  name: 'Alex Chen',
  role_label: 'Department Manager',
  avatar_style: 'realistic'
})

CREATE (wit:Character {
  id: 'char-002',
  name: 'Jordan Rivera',
  role_label: 'Colleague (Witness)',
  avatar_style: 'realistic'
})

// Create nodes
CREATE (start:SceneNode:DialogueNode {
  id: 'node-001',
  title: 'Scene: Break Room Incident',
  is_entry_point: true,
  is_terminal: false,
  dialogue_text: 'You overhear Alex making inappropriate comments to a junior team member in the break room.',
  narrator_text: 'You are getting coffee when you witness this exchange.',
  position_x: 100, position_y: 200,
  created_at: datetime()
})

CREATE (decide:SceneNode:DecisionPoint {
  id: 'node-002',
  title: 'Your Response',
  prompt_text: 'What do you do?',
  time_limit_sec: 30,
  position_x: 400, position_y: 200,
  created_at: datetime()
})

CREATE (intervene:SceneNode:DialogueNode {
  id: 'node-003',
  title: 'Direct Intervention',
  dialogue_text: 'You step in and say, "Hey Alex, that is not appropriate. Let us talk about this."',
  score_value: 30.0,
  position_x: 300, position_y: 100,
  created_at: datetime()
})

CREATE (report:SceneNode:DialogueNode {
  id: 'node-004',
  title: 'Report to HR',
  dialogue_text: 'You document what you witnessed and file a report with HR.',
  score_value: 40.0,
  position_x: 300, position_y: 300,
  created_at: datetime()
})

CREATE (ignore:SceneNode:DialogueNode {
  id: 'node-005',
  title: 'Walk Away',
  dialogue_text: 'You decide it is not your business and leave the break room.',
  score_value: 0.0,
  feedback_text: 'Bystander inaction can perpetuate harassment. Company policy requires witnesses to report.',
  position_x: 300, position_y: 500,
  created_at: datetime()
})

CREATE (debrief:SceneNode:DebriefNode {
  id: 'node-006',
  title: 'Scenario Debrief',
  is_terminal: true,
  debrief_template: 'detailed',
  position_x: 700, position_y: 200,
  created_at: datetime()
})

// Create variable
CREATE (v_reported:Variable {
  id: 'var-001',
  name: 'reported_to_hr',
  var_type: 'boolean',
  default_value: 'false',
  description: 'Whether the learner reported the incident'
})

// Create relationships: scenario structure
CREATE (s)-[:CONTAINS_NODE]->(start)
CREATE (s)-[:CONTAINS_NODE]->(decide)
CREATE (s)-[:CONTAINS_NODE]->(intervene)
CREATE (s)-[:CONTAINS_NODE]->(report)
CREATE (s)-[:CONTAINS_NODE]->(ignore)
CREATE (s)-[:CONTAINS_NODE]->(debrief)
CREATE (s)-[:USES_CHARACTER]->(mgr)
CREATE (s)-[:USES_CHARACTER]->(wit)
CREATE (s)-[:TRACKS_VARIABLE]->(v_reported)
CREATE (start)-[:SPOKEN_BY]->(mgr)

// Create edges (the branching structure)
CREATE (start)-[:LEADS_TO {
  id: 'edge-001', edge_type: 'fallthrough', sort_order: 0
}]->(decide)

CREATE (decide)-[:LEADS_TO {
  id: 'edge-002', edge_type: 'choice',
  label: 'Intervene directly',
  sort_order: 1, score_delta: 30.0, is_correct: true
}]->(intervene)

CREATE (decide)-[:LEADS_TO {
  id: 'edge-003', edge_type: 'choice',
  label: 'Report to HR',
  sort_order: 2, score_delta: 40.0, is_correct: true,
  mutations_json: '[{"variable_id":"var-001","operation":"set","value":"true"}]'
}]->(report)

CREATE (decide)-[:LEADS_TO {
  id: 'edge-004', edge_type: 'choice',
  label: 'Walk away',
  sort_order: 3, score_delta: 0.0, is_correct: false
}]->(ignore)

CREATE (intervene)-[:LEADS_TO {
  id: 'edge-005', edge_type: 'fallthrough', sort_order: 0
}]->(debrief)

CREATE (report)-[:LEADS_TO {
  id: 'edge-006', edge_type: 'fallthrough', sort_order: 0
}]->(debrief)

CREATE (ignore)-[:LEADS_TO {
  id: 'edge-007', edge_type: 'fallthrough', sort_order: 0
}]->(debrief);
```

---

## Critical Graph Queries

These queries demonstrate operations that are natural in Cypher but complex in SQL.

### Find all paths from start to end

```cypher
// Find every possible path through the scenario
MATCH (s:Scenario {id: $scenarioId})-[:CONTAINS_NODE]->(entry:SceneNode {is_entry_point: true})
MATCH (s)-[:CONTAINS_NODE]->(terminal:SceneNode {is_terminal: true})
MATCH path = (entry)-[:LEADS_TO*]->(terminal)
RETURN [n IN nodes(path) | n.id] AS node_ids,
       [r IN relationships(path) | r.label] AS choices,
       reduce(score = 0.0, r IN relationships(path) | score + coalesce(r.score_delta, 0.0))
         AS total_score,
       length(path) AS path_length
ORDER BY total_score DESC;
```

### Detect unreachable nodes

```cypher
// Find nodes that cannot be reached from the entry point
MATCH (s:Scenario {id: $scenarioId})-[:CONTAINS_NODE]->(entry:SceneNode {is_entry_point: true})
MATCH (s)-[:CONTAINS_NODE]->(all_nodes:SceneNode)
WHERE NOT exists(
  (entry)-[:LEADS_TO*0..]->(all_nodes)
)
RETURN all_nodes.id AS unreachable_node_id,
       all_nodes.title AS unreachable_node_title;
```

### Detect dead-end nodes (non-terminal nodes with no outgoing edges)

```cypher
MATCH (s:Scenario {id: $scenarioId})-[:CONTAINS_NODE]->(n:SceneNode)
WHERE NOT n.is_terminal
  AND NOT exists((n)-[:LEADS_TO]->())
RETURN n.id AS dead_end_node_id, n.title AS dead_end_title;
```

### Detect cycles (infinite loops)

```cypher
// Find cycles in the scenario graph
MATCH (s:Scenario {id: $scenarioId})-[:CONTAINS_NODE]->(n:SceneNode)
MATCH path = (n)-[:LEADS_TO*2..]->(n)
RETURN DISTINCT [node IN nodes(path) | node.id] AS cycle_nodes,
       length(path) AS cycle_length
LIMIT 10;
```

### Compute graph statistics

```cypher
// Scenario complexity metrics for the editor dashboard
MATCH (s:Scenario {id: $scenarioId})-[:CONTAINS_NODE]->(n:SceneNode)
OPTIONAL MATCH (n)-[e:LEADS_TO]->()
WITH s,
     count(DISTINCT n) AS node_count,
     count(DISTINCT e) AS edge_count,
     collect(DISTINCT labels(n)) AS node_types
MATCH (s)-[:CONTAINS_NODE]->(entry:SceneNode {is_entry_point: true})
MATCH (s)-[:CONTAINS_NODE]->(terminal:SceneNode {is_terminal: true})
OPTIONAL MATCH path = (entry)-[:LEADS_TO*]->(terminal)
WITH s, node_count, edge_count, node_types,
     count(path) AS total_paths,
     CASE WHEN count(path) > 0 THEN
       avg(length(path))
     ELSE 0 END AS avg_path_length,
     CASE WHEN count(path) > 0 THEN
       max(length(path))
     ELSE 0 END AS max_path_length
RETURN node_count, edge_count, total_paths,
       avg_path_length, max_path_length,
       node_types;
```

### Find all nodes connected to a specific character

```cypher
MATCH (c:Character {id: $characterId})<-[:SPOKEN_BY]-(n:SceneNode)
      <-[:CONTAINS_NODE]-(s:Scenario)
RETURN s.id AS scenario_id, s.name AS scenario_name,
       n.id AS node_id, n.title AS node_title,
       n.dialogue_text AS dialogue
ORDER BY s.name, n.sort_order;
```

### Validate scenario completeness before publishing

```cypher
// Comprehensive validation query
MATCH (s:Scenario {id: $scenarioId})

// Check: has entry point
OPTIONAL MATCH (s)-[:CONTAINS_NODE]->(entry:SceneNode {is_entry_point: true})

// Check: has at least one terminal node
OPTIONAL MATCH (s)-[:CONTAINS_NODE]->(terminal:SceneNode {is_terminal: true})

// Check: all non-terminal nodes have outgoing edges
OPTIONAL MATCH (s)-[:CONTAINS_NODE]->(dead:SceneNode)
WHERE NOT dead.is_terminal AND NOT exists((dead)-[:LEADS_TO]->())

// Check: all paths reach a terminal
OPTIONAL MATCH (s)-[:CONTAINS_NODE]->(unreachable:SceneNode)
WHERE entry IS NOT NULL AND NOT exists((entry)-[:LEADS_TO*0..]->(unreachable))

RETURN
  entry IS NOT NULL AS has_entry_point,
  terminal IS NOT NULL AS has_terminal,
  collect(DISTINCT dead.id) AS dead_end_nodes,
  collect(DISTINCT unreachable.id) AS unreachable_nodes,
  size(collect(DISTINCT dead.id)) = 0 AS no_dead_ends,
  size(collect(DISTINCT unreachable.id)) = 0 AS all_reachable;
```

### Serialize scenario graph for publishing

```cypher
// Export the full graph as a JSON-compatible structure
MATCH (s:Scenario {id: $scenarioId})-[:CONTAINS_NODE]->(n:SceneNode)
OPTIONAL MATCH (n)-[e:LEADS_TO]->(target:SceneNode)
OPTIONAL MATCH (n)-[:SPOKEN_BY]->(c:Character)
WITH s, n, collect({
  id: e.id,
  target_node_id: target.id,
  edge_type: e.edge_type,
  label: e.label,
  sort_order: e.sort_order,
  score_delta: e.score_delta,
  is_correct: e.is_correct,
  feedback_text: e.feedback_text,
  conditions_json: e.conditions_json,
  mutations_json: e.mutations_json
}) AS edges, c
RETURN {
  scenario: properties(s),
  nodes: collect({
    properties: properties(n),
    labels: labels(n),
    edges: edges,
    character: CASE WHEN c IS NOT NULL THEN properties(c) ELSE null END
  })
} AS scenario_export;
```

---

## PostgreSQL Schema (Non-Graph Data)

```sql
-- ============================================================
-- ORGANIZATIONS
-- ============================================================
CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ
);

-- ============================================================
-- USERS
-- ============================================================
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    email           VARCHAR(320) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    password_hash   TEXT,
    auth_provider   VARCHAR(50) NOT NULL DEFAULT 'local',
    auth_subject    VARCHAR(512),
    role            VARCHAR(50) NOT NULL DEFAULT 'author',
    permissions     TEXT[] NOT NULL DEFAULT '{}',
    avatar_url      TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    preferences     JSONB NOT NULL DEFAULT '{}',
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ,
    UNIQUE (organization_id, email)
);

CREATE INDEX idx_users_org ON users(organization_id);
CREATE INDEX idx_users_email ON users(email);

-- ============================================================
-- PROJECTS (lightweight grouping -- references Neo4j scenarios)
-- ============================================================
CREATE TABLE projects (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    status          VARCHAR(50) NOT NULL DEFAULT 'active',
    category        VARCHAR(100),
    tags            TEXT[] NOT NULL DEFAULT '{}',
    default_locale  VARCHAR(10) NOT NULL DEFAULT 'en-US',
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ
);

-- ============================================================
-- PUBLISHED SCENARIO SNAPSHOTS
-- Serialized from Neo4j at publish time for the learner player,
-- SCORM export, and offline use. This is a read-optimized copy.
-- ============================================================
CREATE TABLE published_scenarios (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scenario_id     VARCHAR(255) NOT NULL,  -- matches Neo4j Scenario.id
    version_number  INTEGER NOT NULL,
    name            VARCHAR(255) NOT NULL,
    graph_snapshot  JSONB NOT NULL,     -- full serialized graph from Neo4j
    scoring_config  JSONB NOT NULL,
    characters_data JSONB NOT NULL,
    node_count      INTEGER NOT NULL,
    edge_count      INTEGER NOT NULL,
    changelog       TEXT,
    published_by    UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (scenario_id, version_number)
);

CREATE INDEX idx_published_scenario ON published_scenarios(scenario_id);

-- ============================================================
-- LEARNERS
-- ============================================================
CREATE TABLE learners (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID REFERENCES organizations(id),
    user_id         UUID REFERENCES users(id),
    external_id     VARCHAR(255),
    email           VARCHAR(320),
    display_name    VARCHAR(255),
    lti_subject     VARCHAR(512),
    profile_data    JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_learners_org ON learners(organization_id);
CREATE INDEX idx_learners_external ON learners(external_id);

-- ============================================================
-- LEARNER ATTEMPTS
-- ============================================================
CREATE TABLE learner_attempts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    learner_id      UUID NOT NULL REFERENCES learners(id),
    scenario_id     VARCHAR(255) NOT NULL,  -- matches Neo4j Scenario.id
    version_number  INTEGER NOT NULL,
    attempt_number  INTEGER NOT NULL DEFAULT 1,
    status          VARCHAR(30) NOT NULL DEFAULT 'in_progress',
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    duration_seconds INTEGER,
    total_score     DECIMAL(8,2),
    max_possible_score DECIMAL(8,2),
    score_percentage DECIMAL(5,2),
    passed          BOOLEAN,
    completion_status VARCHAR(20),
    success_status  VARCHAR(20),
    source          VARCHAR(30) NOT NULL DEFAULT 'hosted',
    path_data       JSONB NOT NULL DEFAULT '{"steps":[],"variable_state":{},"current_node_id":null}',
    ai_evaluations  JSONB NOT NULL DEFAULT '[]',
    suspend_data    TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_attempts_learner ON learner_attempts(learner_id);
CREATE INDEX idx_attempts_scenario ON learner_attempts(scenario_id);
CREATE INDEX idx_attempts_status ON learner_attempts(status);
CREATE INDEX idx_attempts_started ON learner_attempts(started_at);

-- ============================================================
-- POLICY DOCUMENTS (content and embeddings)
-- ============================================================
CREATE TABLE policy_documents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    neo4j_id        VARCHAR(255),  -- links to Neo4j PolicyDocument node
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    document_type   VARCHAR(50) NOT NULL,
    file_url        TEXT,
    file_type       VARCHAR(20),
    file_size_bytes BIGINT,
    uploaded_by     UUID NOT NULL REFERENCES users(id),
    processing_status VARCHAR(20) NOT NULL DEFAULT 'pending',
    extracted_content JSONB NOT NULL DEFAULT '{}',
    processed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ
);

CREATE TABLE document_embeddings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id     UUID NOT NULL REFERENCES policy_documents(id) ON DELETE CASCADE,
    chunk_index     INTEGER NOT NULL,
    chunk_text      TEXT NOT NULL,
    token_count     INTEGER,
    embedding       VECTOR(1536),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_embeddings_vector ON document_embeddings
    USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);

-- ============================================================
-- AI GENERATION JOBS
-- ============================================================
CREATE TABLE ai_generation_jobs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scenario_id     VARCHAR(255) NOT NULL,
    job_type        VARCHAR(50) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    requested_by    UUID NOT NULL REFERENCES users(id),
    input_payload   JSONB NOT NULL,
    output_payload  JSONB,
    model_used      VARCHAR(100),
    token_count_input  INTEGER,
    token_count_output INTEGER,
    cost_usd        DECIMAL(10,6),
    error_message   TEXT,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- EXPORT PACKAGES
-- ============================================================
CREATE TABLE export_packages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scenario_id     VARCHAR(255) NOT NULL,
    version_number  INTEGER NOT NULL,
    format          VARCHAR(30) NOT NULL,
    package_url     TEXT,
    file_size_bytes BIGINT,
    build_status    VARCHAR(20) NOT NULL DEFAULT 'pending',
    build_error     TEXT,
    manifest        JSONB NOT NULL DEFAULT '{}',
    exported_by     UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- LTI REGISTRATIONS
-- ============================================================
CREATE TABLE lti_registrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    platform_name   VARCHAR(255) NOT NULL,
    issuer          TEXT NOT NULL,
    client_id       VARCHAR(255) NOT NULL,
    deployment_id   VARCHAR(255),
    auth_endpoint   TEXT NOT NULL,
    token_endpoint  TEXT NOT NULL,
    keyset_url      TEXT NOT NULL,
    platform_keyset JSONB,
    tool_keyset     JSONB NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    platform_config JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- LRS CONFIGURATIONS
-- ============================================================
CREATE TABLE lrs_configurations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    endpoint_url    TEXT NOT NULL,
    auth_type       VARCHAR(20) NOT NULL,
    auth_credentials_encrypted TEXT NOT NULL,
    xapi_version    VARCHAR(10) NOT NULL DEFAULT '1.0.3',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    xapi_config     JSONB NOT NULL DEFAULT '{}',
    last_sync_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- XAPI STATEMENTS
-- ============================================================
CREATE TABLE xapi_statements (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    statement_id    UUID NOT NULL UNIQUE,
    attempt_id      UUID REFERENCES learner_attempts(id),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    verb_id         TEXT NOT NULL,
    actor_email     VARCHAR(320),
    timestamp       TIMESTAMPTZ NOT NULL,
    statement_json  JSONB NOT NULL,
    lrs_config_id   UUID REFERENCES lrs_configurations(id),
    sync_status     VARCHAR(20) NOT NULL DEFAULT 'pending',
    sync_error      TEXT,
    synced_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_xapi_sync ON xapi_statements(sync_status)
    WHERE sync_status = 'pending';
CREATE INDEX idx_xapi_timestamp ON xapi_statements(timestamp);

-- ============================================================
-- COMMENTS (reference Neo4j node/edge IDs by string)
-- ============================================================
CREATE TABLE comments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scenario_id     VARCHAR(255) NOT NULL,  -- Neo4j Scenario.id
    target_node_id  VARCHAR(255),           -- Neo4j SceneNode.id
    target_edge_id  VARCHAR(255),           -- Neo4j LEADS_TO.id
    parent_id       UUID REFERENCES comments(id) ON DELETE CASCADE,
    author_id       UUID NOT NULL REFERENCES users(id),
    body            TEXT NOT NULL,
    is_resolved     BOOLEAN NOT NULL DEFAULT false,
    resolved_by     UUID REFERENCES users(id),
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ
);

CREATE INDEX idx_comments_scenario ON comments(scenario_id);
CREATE INDEX idx_comments_node ON comments(target_node_id);

-- ============================================================
-- REVIEW REQUESTS
-- ============================================================
CREATE TABLE review_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scenario_id     VARCHAR(255) NOT NULL,
    requested_by    UUID NOT NULL REFERENCES users(id),
    assigned_to     UUID NOT NULL REFERENCES users(id),
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
    review_notes    TEXT,
    checklist       JSONB NOT NULL DEFAULT '{"items":[]}',
    requested_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ
);

-- ============================================================
-- AUDIT LOG
-- ============================================================
CREATE TABLE audit_log (
    id              BIGSERIAL PRIMARY KEY,
    organization_id UUID NOT NULL,
    user_id         UUID,
    action          VARCHAR(50) NOT NULL,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       VARCHAR(255) NOT NULL,  -- UUID or Neo4j ID
    changes         JSONB,
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_org ON audit_log(organization_id, created_at);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);

-- ============================================================
-- ANALYTICS (materialized from learner_attempts)
-- ============================================================
CREATE MATERIALIZED VIEW mv_scenario_stats AS
SELECT
    a.scenario_id,
    COUNT(*) AS total_attempts,
    COUNT(*) FILTER (WHERE a.status = 'completed') AS completed_attempts,
    AVG(a.score_percentage) FILTER (WHERE a.status = 'completed') AS avg_score,
    AVG(a.duration_seconds) FILTER (WHERE a.status = 'completed') AS avg_duration,
    COUNT(*) FILTER (WHERE a.passed = true)::DECIMAL /
        NULLIF(COUNT(*) FILTER (WHERE a.status = 'completed'), 0) * 100
        AS pass_rate,
    MAX(a.started_at) AS last_attempt_at
FROM learner_attempts a
GROUP BY a.scenario_id;

CREATE UNIQUE INDEX idx_mv_scenario_stats ON mv_scenario_stats(scenario_id);
```

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    Application Layer                         │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │ Scenario     │  │ Learner      │  │ Analytics /      │  │
│  │ Authoring    │  │ Player       │  │ Compliance       │  │
│  │ Service      │  │ Service      │  │ Service          │  │
│  └──────┬───────┘  └──────┬───────┘  └────────┬─────────┘  │
│         │                 │                    │             │
│    ┌────┴────┐       ┌────┴────┐          ┌────┴────┐       │
│    │ Neo4j   │       │ PG Read │          │ PG Read │       │
│    │ Driver  │       │ (snaps) │          │ (stats) │       │
│    └────┬────┘       └────┬────┘          └────┬────┘       │
└─────────┼────────────────┼─────────────────────┼────────────┘
          │                │                     │
          ▼                ▼                     ▼
   ┌──────────────┐  ┌──────────────────────────────────┐
   │   Neo4j      │  │         PostgreSQL                │
   │              │  │                                   │
   │  Scenarios   │  │  users, organizations, learners   │
   │  Nodes       │  │  learner_attempts, xapi_stmts     │
   │  Edges       │  │  published_scenarios (snapshots)   │
   │  Characters  │  │  lti, lrs, exports, audit_log     │
   │  Variables   │  │  policy_docs, embeddings (pgvec)  │
   │              │  │  comments, reviews, ai_jobs        │
   └──────┬───────┘  └──────────────────────────────────┘
          │
          │ Publish action:
          │ serialize graph → insert published_scenarios
          ▼
   ┌──────────────┐
   │ Sync Worker  │ (one-directional: Neo4j → PostgreSQL)
   └──────────────┘
```

---

## Data Flow: Publish Scenario

1. Author clicks "Publish" in the visual editor.
2. Authoring Service runs the validation Cypher query against Neo4j.
3. If valid, Authoring Service serializes the full graph using the export query.
4. Serialized graph is written to `published_scenarios` in PostgreSQL.
5. SCORM/cmi5 export worker picks up the new version and generates the package from the PostgreSQL snapshot.
6. Published version is now available to the Learner Player Service (reads PostgreSQL only).

This means the Learner Player Service never touches Neo4j -- it serves published snapshots from PostgreSQL. This isolates the authoring workload (Neo4j) from the delivery workload (PostgreSQL), providing natural load separation.

---

## Pros

1. **Graph queries are the product's core differentiator.** The visual editor needs to: validate graph connectivity (are all nodes reachable?), detect cycles (infinite loops in branching), find dead-end nodes, compute all possible paths, calculate path-based scores, and display graph complexity metrics. In Neo4j, each of these is a concise Cypher query. In PostgreSQL, each requires recursive CTEs with careful termination conditions, cycle detection logic, and performance tuning. The Cypher queries shown above are 5-15 lines each; the equivalent SQL would be 30-80 lines each and would perform significantly worse on deep graphs.

2. **Natural modeling of the domain.** A branching scenario is a directed graph. Nodes are scenes. Edges are choices. Characters are connected to scenes. Variables are tracked across the graph. Modeling this as a labeled property graph is a direct representation of the domain, not an abstraction forced into a tabular structure. This reduces the impedance mismatch between the domain model and the data model, making the code more readable and the data model self-documenting.

3. **Efficient graph mutations.** Adding a node, connecting two nodes, removing an edge, and repositioning a node are all local operations in a graph database -- they affect only the nodes and relationships involved, not the entire scenario document. Compare this to the JSONB approach (Suggestion 3), where updating a single node requires rewriting the entire graph JSONB column.

4. **Workload separation.** Authoring (Neo4j) and delivery (PostgreSQL) are separated by design. The learner player reads published snapshots from PostgreSQL and never hits Neo4j. This means high learner traffic does not affect authoring performance, and complex authoring queries do not affect learner delivery latency.

5. **Subgraph queries for collaboration.** "Show me all nodes this character appears in" or "find all decision points where the learner can fail" are subgraph pattern matching queries that Neo4j handles natively. These are useful for the collaboration and review workflow: a reviewer can query for specific patterns in the scenario graph to focus their review.

6. **Graph algorithms for analytics enrichment.** Neo4j's Graph Data Science library provides algorithms for community detection, centrality analysis, and pathfinding that can identify structural patterns in scenarios: bottleneck nodes (high betweenness centrality), redundant paths, and cluster patterns. These enable "scenario health" dashboards that no competitor offers.

---

## Cons

1. **Operational complexity doubles.** Two database systems means two backup strategies, two monitoring configurations, two failure modes, two sets of driver libraries, two connection management patterns, and two skill sets required from the operations team. For a startup-stage product, this is the most significant drawback.

2. **Self-hosted deployment is harder.** The product's README explicitly supports self-hosted deployment. Requiring both Neo4j and PostgreSQL significantly raises the bar for self-hosted installation compared to a single PostgreSQL instance. Docker Compose mitigates this for development, but production deployments in enterprise environments with IT approval processes are harder to justify.

3. **No cross-database transactions.** When a scenario is published, the graph is serialized from Neo4j and inserted into PostgreSQL. If the PostgreSQL insert fails after the Neo4j status update, the databases are inconsistent. This requires implementing the Saga pattern or compensating transactions, adding application complexity.

4. **Neo4j Community Edition limitations.** Neo4j Community Edition (AGPL v3) supports only a single database per server, has no online backup, no role-based access control, no clustering, and no subquery execution in transactions. These limitations may force an upgrade to Neo4j Enterprise ($$$) or AuraDB (managed SaaS) as the product scales. The AGPL license also has copyleft implications that may conflict with the product's commercial licensing.

5. **JSONB serialization gap.** The learner player and SCORM export pipeline consume graph data from PostgreSQL JSONB snapshots, not from Neo4j directly. This means maintaining a serialization/deserialization layer that must stay in sync with the Neo4j schema. Schema changes in Neo4j require corresponding changes to the serialization logic and potentially to the PostgreSQL snapshot structure.

6. **Limited ACID guarantees on complex mutations.** While Neo4j supports transactions, complex graph mutations involving many nodes and relationships can encounter lock contention in concurrent multi-author scenarios. Neo4j's write concurrency model is less mature than PostgreSQL's MVCC.

7. **Smaller ecosystem.** Neo4j has fewer ORM options, fewer migration tools, fewer monitoring integrations, and a smaller hiring pool than PostgreSQL. The Cypher query language, while expressive for graph operations, is less widely known than SQL.

8. **Cost at scale.** Neo4j AuraDB (managed cloud) pricing is significantly higher per-GB than managed PostgreSQL. For a product that stores potentially millions of scenario nodes and edges, the cost difference is material. Self-managed Neo4j on bare metal is cheaper but adds operational burden.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Graph Database | Neo4j Community Edition 5.x (development/self-hosted); Neo4j AuraDB (cloud production) |
| Relational Database | PostgreSQL 16+ with pgvector |
| Neo4j Driver | neo4j-driver (JavaScript/TypeScript) or neo4j (Python) |
| ORM (PostgreSQL) | Prisma or Drizzle ORM |
| Graph Visualization | React Flow or D3.js with data sourced from Neo4j queries |
| Sync Worker | Node.js worker process or BullMQ job queue |
| Caching | Redis for published scenario snapshots |
| Blob Storage | S3-compatible for SCORM packages, audio, media |
| Monitoring | Neo4j metrics exporter + Prometheus; PostgreSQL pg_stat_statements |

---

## Migration and Scaling Considerations

### Early Stage (0-1K scenarios)
- Run Neo4j Community Edition and PostgreSQL in Docker Compose.
- Single Neo4j instance is sufficient; graph queries are fast on small datasets.
- Published scenario snapshots in PostgreSQL eliminate any Neo4j scaling pressure from learner traffic.
- Focus on getting the Cypher query patterns right; Neo4j schema changes are schema-free (no migrations needed for property additions).

### Growth Stage (1K-50K scenarios)
- Move to Neo4j AuraDB or a dedicated Neo4j server with more memory (graph databases are memory-intensive).
- Implement read caching for published scenarios in Redis.
- Add PostgreSQL read replicas for analytics workload.
- Monitor Neo4j heap usage and tune page cache size.

### Scale Stage (50K+ scenarios)
- Consider Neo4j Enterprise Edition for clustering, online backup, and RBAC.
- Evaluate whether the graph query requirements still justify the operational overhead. If graph validation queries have stabilized and are called infrequently (only at publish time), the application-level graph validation using PostgreSQL recursive CTEs may become adequate, allowing Neo4j to be retired.
- Alternatively, consider replacing Neo4j with Apache AGE (a PostgreSQL extension that adds Cypher query support to PostgreSQL), consolidating to a single database engine while retaining graph query capabilities.

### Alternative: Apache AGE (PostgreSQL Graph Extension)
Apache AGE adds openCypher query support to PostgreSQL, enabling graph queries within the same database engine. This eliminates the operational overhead of two databases while retaining most graph query capabilities. The trade-off is that AGE's performance on deep graph traversals is slower than native Neo4j, and its feature set is smaller (no Graph Data Science library). For this product, AGE may be the right compromise if the graph validation queries shown above are the primary use case (versus continuous graph analytics).

### Migration Path: Start with PostgreSQL, Add Neo4j Later
If the product starts with Suggestion 1 (normalized PostgreSQL) or Suggestion 3 (hybrid JSONB):
1. Introduce Neo4j alongside PostgreSQL for the authoring service only.
2. Migrate existing scenario data from PostgreSQL to Neo4j via a one-time ETL.
3. Point the visual editor at Neo4j; keep the learner player on PostgreSQL.
4. Decommission the scenario graph tables in PostgreSQL (keep the published_scenarios table).

This incremental approach validates the graph database hypothesis before committing to the operational overhead.
