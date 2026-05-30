# Data Model Suggestion 3: Hybrid Relational + Document (PostgreSQL JSONB)

> Project: Training Simulator Builder (Candidate #430)
> Generated: 2026-05-25

---

## Overview

This model uses PostgreSQL as a hybrid relational-document database. Stable, frequently-queried structural data (users, organizations, projects, learner attempts, LMS integrations) lives in normalized relational tables. Flexible, variable-structure data (scenario node graphs, AI generation results, template configurations, character style definitions, xAPI statement payloads) lives in JSONB columns within those tables or in dedicated document-oriented tables.

The key insight driving this design is that a training simulator builder has two fundamentally different categories of data:

1. **Structural data** that is stable, relational, and queried predictably: who created what, when was it published, which learner completed which scenario, what LMS connection is configured. This data changes shape rarely and benefits from foreign keys, indexes, and SQL joins.

2. **Content data** that is flexible, deeply nested, and changes shape frequently during product development: the scenario node graph itself (which is essentially a document), AI prompt/response payloads, character appearance configurations, template definitions, SCORM/cmi5 manifest data, and edge condition expressions. This data is tree-structured, varies by node type, and would require dozens of narrow subtype tables in a fully normalized model.

By storing content data as JSONB within relational tables, we get the schema flexibility of a document database with the transactional guarantees, referential integrity, and query power of PostgreSQL -- without introducing a second database system.

---

## Design Principles

1. **Normalize what is stable; document what is variable.** If a data shape has been stable for months and is used in WHERE clauses and JOINs, normalize it. If it is evolving, deeply nested, or varies by subtype, use JSONB.
2. **Promote hot query fields to columns.** Even on tables with JSONB content, extract frequently-filtered or frequently-sorted fields into dedicated columns alongside the JSONB blob. This gives the query planner something to work with.
3. **GIN indexes on JSONB for search.** Use GIN indexes on JSONB columns that need containment queries (@>, ?|, ?) or full-document search.
4. **JSON Schema validation in application layer.** PostgreSQL does not enforce JSONB structure natively; validate JSONB documents against JSON Schema definitions in the application before writes.
5. **Single database system.** All data in one PostgreSQL instance. No MongoDB, no Redis (except as an optional cache layer). Operational simplicity for both cloud and self-hosted deployment.

---

## Complete Schema

### Tenant and Identity (Relational)

```sql
-- ============================================================
-- ORGANIZATIONS (relational -- stable, frequently queried)
-- ============================================================
CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',
    -- JSONB: branding, feature flags, quota limits, notification prefs
    settings        JSONB NOT NULL DEFAULT '{
      "branding": { "primary_color": "#2563eb", "logo_url": null },
      "features": { "ai_generation": true, "voice_synthesis": true },
      "quotas": { "max_scenarios": 100, "max_ai_generations_per_month": 500 }
    }',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ
);

-- ============================================================
-- USERS (relational -- core identity, auth, access control)
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
        -- admin, author, reviewer, publisher, learner
    permissions     TEXT[] NOT NULL DEFAULT '{}',
    avatar_url      TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- JSONB: user preferences, notification settings, editor layout prefs
    preferences     JSONB NOT NULL DEFAULT '{
      "editor": { "grid_snap": true, "auto_save_interval_seconds": 30 },
      "notifications": { "email_on_review": true, "email_on_comment": false }
    }',
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ,
    UNIQUE (organization_id, email)
);

CREATE INDEX idx_users_org ON users(organization_id);
CREATE INDEX idx_users_email ON users(email);
```

### Scenario Authoring (Hybrid)

```sql
-- ============================================================
-- PROJECTS (relational -- stable grouping)
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

CREATE INDEX idx_projects_org ON projects(organization_id);
CREATE INDEX idx_projects_status ON projects(status);

-- ============================================================
-- SCENARIOS (hybrid -- relational metadata + JSONB graph)
-- This is the most important table in the schema.
-- ============================================================
CREATE TABLE scenarios (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    organization_id UUID NOT NULL REFERENCES organizations(id),

    -- Relational columns: stable, frequently filtered/sorted
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    status          VARCHAR(50) NOT NULL DEFAULT 'draft',
    version         INTEGER NOT NULL DEFAULT 1,
    locale          VARCHAR(10) NOT NULL DEFAULT 'en-US',
    difficulty_level VARCHAR(20),
    estimated_duration_minutes INTEGER,
    passing_score   DECIMAL(5,2),
    max_score       DECIMAL(8,2),
    is_template     BOOLEAN NOT NULL DEFAULT false,
    template_source_id UUID REFERENCES scenarios(id),
    node_count      INTEGER NOT NULL DEFAULT 0,  -- denormalized counter
    edge_count      INTEGER NOT NULL DEFAULT 0,  -- denormalized counter
    created_by      UUID NOT NULL REFERENCES users(id),
    published_at    TIMESTAMPTZ,
    published_by    UUID REFERENCES users(id),

    -- ============================================================
    -- JSONB: The complete scenario graph as a document
    -- This is the core content that changes shape frequently
    -- and varies dramatically between scenarios.
    -- ============================================================
    graph           JSONB NOT NULL DEFAULT '{
      "nodes": {},
      "edges": {},
      "variables": {},
      "entry_node_id": null
    }',

    -- JSONB: Scoring configuration (varies by scenario type)
    scoring_config  JSONB NOT NULL DEFAULT '{
      "mode": "cumulative",
      "show_score_during": false,
      "debrief_config": {
        "show_path_taken": true,
        "show_correct_path": true,
        "show_score_breakdown": true
      }
    }',

    -- JSONB: AI generation metadata
    ai_metadata     JSONB NOT NULL DEFAULT '{
      "generation_model": null,
      "source_documents": [],
      "generation_history": []
    }',

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ
);

CREATE INDEX idx_scenarios_project ON scenarios(project_id);
CREATE INDEX idx_scenarios_org ON scenarios(organization_id);
CREATE INDEX idx_scenarios_status ON scenarios(status);
CREATE INDEX idx_scenarios_template ON scenarios(is_template) WHERE is_template = true;

-- GIN index on graph for searching within scenario content
CREATE INDEX idx_scenarios_graph ON scenarios USING gin(graph jsonb_path_ops);
```

### Graph Document Structure (JSONB Schema)

The `scenarios.graph` JSONB column follows this structure:

```jsonc
// scenarios.graph JSON Schema
{
  "nodes": {
    "<node_uuid>": {
      "id": "uuid",
      "type": "dialogue|decision_point|consequence|free_response|media|information|delay|branch_merge|debrief|start|end",
      "title": "string",
      "position": { "x": 0.0, "y": 0.0 },

      // Common fields (present on all node types)
      "sort_order": 0,
      "is_entry_point": false,
      "is_terminal": false,

      // Type-specific fields -- this is where normalization breaks down
      // and JSONB shines. Each node type has different data.

      // dialogue node
      "character_id": "uuid|null",
      "dialogue_text": "string",
      "narrator_text": "string|null",
      "voice_audio_url": "string|null",

      // decision_point node
      "prompt_text": "string",
      "time_limit_seconds": null,

      // free_response node
      "prompt_text": "string",
      "evaluation_rubric": "string",  // AI evaluation instructions
      "expected_keywords": ["string"],
      "min_length": 0,
      "max_length": 500,

      // media node
      "media_url": "string",
      "media_type": "image|video|audio",
      "caption": "string|null",
      "auto_advance_seconds": null,

      // delay node
      "delay_seconds": 3,
      "message": "Thinking...",

      // debrief node
      "debrief_template": "summary|detailed|custom",
      "custom_debrief_html": "string|null",

      // Feedback and scoring (applies to several types)
      "feedback_text": "string|null",
      "score_value": 0.0,

      // Metadata
      "created_at": "iso8601",
      "updated_at": "iso8601"
    }
  },

  "edges": {
    "<edge_uuid>": {
      "id": "uuid",
      "source_node_id": "uuid",
      "target_node_id": "uuid",
      "type": "choice|fallthrough|conditional|timeout|default",
      "label": "string",        // displayed choice text
      "sort_order": 0,
      "score_delta": 0.0,
      "is_correct": null,       // null = no right/wrong
      "feedback_text": "string|null",

      // Conditions (for conditional edges)
      "conditions": [
        {
          "variable_id": "uuid",
          "operator": "eq|neq|gt|gte|lt|lte|contains",
          "value": "string",
          "logical_group": 0
        }
      ],

      // Variable mutations (applied when this edge is traversed)
      "mutations": [
        {
          "variable_id": "uuid",
          "operation": "set|increment|decrement|toggle",
          "value": "string"
        }
      ]
    }
  },

  "variables": {
    "<variable_uuid>": {
      "id": "uuid",
      "name": "string",
      "type": "string|number|boolean",
      "default_value": "string|null",
      "description": "string|null"
    }
  },

  "entry_node_id": "uuid|null"
}
```

### Scenario Versions and Publishing

```sql
-- ============================================================
-- SCENARIO VERSIONS (immutable published snapshots)
-- ============================================================
CREATE TABLE scenario_versions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scenario_id     UUID NOT NULL REFERENCES scenarios(id) ON DELETE CASCADE,
    version_number  INTEGER NOT NULL,
    -- Complete graph snapshot at publish time
    graph_snapshot  JSONB NOT NULL,
    scoring_config  JSONB NOT NULL,
    -- Denormalized metadata for version listing without loading full graph
    node_count      INTEGER NOT NULL,
    edge_count      INTEGER NOT NULL,
    changelog       TEXT,
    published_by    UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (scenario_id, version_number)
);

CREATE INDEX idx_versions_scenario ON scenario_versions(scenario_id);
```

### Characters and Voice (Hybrid)

```sql
-- ============================================================
-- CHARACTERS (hybrid -- relational identity + JSONB appearance)
-- ============================================================
CREATE TABLE characters (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    role_label      VARCHAR(100),
    is_library      BOOLEAN NOT NULL DEFAULT false,
    created_by      UUID NOT NULL REFERENCES users(id),

    -- JSONB: Visual appearance configuration
    -- This changes shape as we add avatar styles, expressions, poses
    appearance      JSONB NOT NULL DEFAULT '{
      "style": "realistic",
      "gender": null,
      "age_range": null,
      "ethnicity": null,
      "avatar_url": null,
      "expressions": {
        "neutral": null,
        "happy": null,
        "concerned": null,
        "angry": null,
        "confused": null
      },
      "poses": {
        "standing": null,
        "sitting": null,
        "gesture": null
      }
    }',

    -- JSONB: Voice configuration
    voice_config    JSONB NOT NULL DEFAULT '{
      "provider": null,
      "provider_voice_id": null,
      "language": "en-US",
      "style": "neutral",
      "speed_factor": 1.0,
      "pitch_factor": 1.0,
      "sample_audio_url": null
    }',

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ
);

CREATE INDEX idx_characters_org ON characters(organization_id);
CREATE INDEX idx_characters_library ON characters(is_library)
    WHERE is_library = true;

-- ============================================================
-- AUDIO CACHE (relational -- stable structure, lookup-heavy)
-- ============================================================
CREATE TABLE audio_cache (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    character_id    UUID NOT NULL REFERENCES characters(id),
    text_hash       VARCHAR(64) NOT NULL,
    text_content    TEXT NOT NULL,
    audio_url       TEXT NOT NULL,
    audio_format    VARCHAR(10) NOT NULL DEFAULT 'mp3',
    duration_ms     INTEGER,
    file_size_bytes BIGINT,
    -- JSONB: provider-specific metadata (varies by TTS provider)
    provider_metadata JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (character_id, text_hash)
);
```

### Policy Documents and AI (Hybrid)

```sql
-- ============================================================
-- POLICY DOCUMENTS (hybrid -- relational metadata + JSONB chunks)
-- ============================================================
CREATE TABLE policy_documents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    document_type   VARCHAR(50) NOT NULL,
    file_url        TEXT,
    file_type       VARCHAR(20),
    file_size_bytes BIGINT,
    uploaded_by     UUID NOT NULL REFERENCES users(id),
    processing_status VARCHAR(20) NOT NULL DEFAULT 'pending',

    -- JSONB: Extracted text and chunked content
    -- Stored as JSONB rather than in separate tables because:
    -- 1. Chunks are always accessed together (full document context for AI)
    -- 2. The chunk structure may change (different chunking strategies)
    -- 3. No need to query individual chunks via SQL
    extracted_content JSONB NOT NULL DEFAULT '{
      "full_text": null,
      "chunks": [],
      "metadata": {
        "page_count": null,
        "word_count": null,
        "language": null,
        "extraction_model": null
      }
    }',

    processed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ
);

CREATE INDEX idx_policy_docs_org ON policy_documents(organization_id);

-- ============================================================
-- DOCUMENT EMBEDDINGS (relational -- needs vector index)
-- Embeddings stay in a separate table because pgvector indexing
-- works on columns, not JSONB paths.
-- ============================================================
CREATE TABLE document_embeddings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id     UUID NOT NULL REFERENCES policy_documents(id) ON DELETE CASCADE,
    chunk_index     INTEGER NOT NULL,
    chunk_text      TEXT NOT NULL,
    token_count     INTEGER,
    embedding       VECTOR(1536),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_embeddings_document ON document_embeddings(document_id);
CREATE INDEX idx_embeddings_vector ON document_embeddings
    USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);

-- ============================================================
-- AI GENERATION JOBS (hybrid -- relational tracking + JSONB I/O)
-- ============================================================
CREATE TABLE ai_generation_jobs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scenario_id     UUID NOT NULL REFERENCES scenarios(id) ON DELETE CASCADE,
    job_type        VARCHAR(50) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    requested_by    UUID NOT NULL REFERENCES users(id),

    -- JSONB: Input and output payloads (highly variable by job type)
    input_payload   JSONB NOT NULL,
    output_payload  JSONB,

    -- Relational: trackable metrics
    model_used      VARCHAR(100),
    token_count_input  INTEGER,
    token_count_output INTEGER,
    cost_usd        DECIMAL(10,6),
    error_message   TEXT,

    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ai_jobs_scenario ON ai_generation_jobs(scenario_id);
CREATE INDEX idx_ai_jobs_status ON ai_generation_jobs(status);
```

### Templates (Document-Heavy)

```sql
-- ============================================================
-- SCENARIO TEMPLATES (JSONB-primary)
-- Templates are essentially documents with metadata headers.
-- ============================================================
CREATE TABLE scenario_templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    category        VARCHAR(100) NOT NULL,
    difficulty_level VARCHAR(20),
    estimated_duration_minutes INTEGER,
    thumbnail_url   TEXT,
    is_system       BOOLEAN NOT NULL DEFAULT true,
    organization_id UUID REFERENCES organizations(id),
    created_by      UUID REFERENCES users(id),
    usage_count     INTEGER NOT NULL DEFAULT 0,

    -- JSONB: The template's scenario graph (same structure as scenarios.graph)
    template_graph  JSONB NOT NULL,

    -- JSONB: Template configuration and customization hints
    template_config JSONB NOT NULL DEFAULT '{
      "customizable_fields": [],
      "placeholder_variables": {},
      "required_policy_types": [],
      "suggested_characters": [],
      "locales_available": ["en-US"]
    }',

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_templates_category ON scenario_templates(category);
CREATE INDEX idx_templates_system ON scenario_templates(is_system);
```

### Collaboration (Relational)

```sql
-- ============================================================
-- COMMENTS (relational -- threaded, frequently queried by filters)
-- ============================================================
CREATE TABLE comments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scenario_id     UUID NOT NULL REFERENCES scenarios(id) ON DELETE CASCADE,
    -- Target: node or edge within the JSONB graph
    target_node_id  UUID,   -- references a node.id within scenarios.graph
    target_edge_id  UUID,   -- references an edge.id within scenarios.graph
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
CREATE INDEX idx_comments_target_node ON comments(target_node_id);
CREATE INDEX idx_comments_unresolved ON comments(scenario_id)
    WHERE is_resolved = false AND deleted_at IS NULL;

-- ============================================================
-- REVIEW REQUESTS (relational -- workflow tracking)
-- ============================================================
CREATE TABLE review_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scenario_id     UUID NOT NULL REFERENCES scenarios(id) ON DELETE CASCADE,
    version_id      UUID REFERENCES scenario_versions(id),
    requested_by    UUID NOT NULL REFERENCES users(id),
    assigned_to     UUID NOT NULL REFERENCES users(id),
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
    review_notes    TEXT,
    -- JSONB: structured review checklist
    checklist       JSONB NOT NULL DEFAULT '{
      "items": [
        { "label": "Content accuracy", "checked": false },
        { "label": "Policy alignment", "checked": false },
        { "label": "Dialogue quality", "checked": false },
        { "label": "Scoring logic", "checked": false },
        { "label": "Accessibility", "checked": false }
      ]
    }',
    requested_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ
);

CREATE INDEX idx_reviews_scenario ON review_requests(scenario_id);
CREATE INDEX idx_reviews_assignee ON review_requests(assigned_to, status);
```

### LMS Integration (Relational + JSONB)

```sql
-- ============================================================
-- EXPORT PACKAGES (relational tracking + JSONB manifest)
-- ============================================================
CREATE TABLE export_packages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scenario_id     UUID NOT NULL REFERENCES scenarios(id) ON DELETE CASCADE,
    version_id      UUID NOT NULL REFERENCES scenario_versions(id),
    format          VARCHAR(30) NOT NULL,
    package_url     TEXT,
    file_size_bytes BIGINT,
    build_status    VARCHAR(20) NOT NULL DEFAULT 'pending',
    build_error     TEXT,
    exported_by     UUID NOT NULL REFERENCES users(id),
    -- JSONB: format-specific manifest data
    -- SCORM: parsed imsmanifest.xml fields
    -- cmi5: parsed cmi5.xml fields
    -- LTI: launch configuration
    manifest        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_exports_scenario ON export_packages(scenario_id);
CREATE INDEX idx_exports_format ON export_packages(format);

-- ============================================================
-- LTI REGISTRATIONS (relational -- security-critical lookup)
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
    -- JSONB: platform and tool keysets (JWK format -- naturally JSON)
    platform_keyset JSONB,
    tool_keyset     JSONB NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- JSONB: additional platform-specific configuration
    platform_config JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_lti_org ON lti_registrations(organization_id);
CREATE INDEX idx_lti_issuer ON lti_registrations(issuer);

-- ============================================================
-- LRS CONFIGURATIONS (relational -- credential storage)
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
    -- JSONB: xAPI profile configuration and statement mappings
    xapi_config     JSONB NOT NULL DEFAULT '{
      "profile_id": null,
      "verb_mappings": {},
      "activity_type_mappings": {},
      "context_template": {}
    }',
    last_sync_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Learner Experience (Relational + JSONB)

```sql
-- ============================================================
-- LEARNERS (relational -- identity and cross-referencing)
-- ============================================================
CREATE TABLE learners (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID REFERENCES organizations(id),
    user_id         UUID REFERENCES users(id),
    external_id     VARCHAR(255),
    email           VARCHAR(320),
    display_name    VARCHAR(255),
    lti_subject     VARCHAR(512),
    -- JSONB: learner profile data from LMS or xAPI agent
    profile_data    JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_learners_org ON learners(organization_id);
CREATE INDEX idx_learners_external ON learners(external_id);

-- ============================================================
-- LEARNER ATTEMPTS (hybrid -- relational tracking + JSONB state)
-- ============================================================
CREATE TABLE learner_attempts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    learner_id      UUID NOT NULL REFERENCES learners(id),
    scenario_id     UUID NOT NULL REFERENCES scenarios(id),
    version_id      UUID REFERENCES scenario_versions(id),
    attempt_number  INTEGER NOT NULL DEFAULT 1,

    -- Relational: queryable status and scores
    status          VARCHAR(30) NOT NULL DEFAULT 'in_progress',
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    duration_seconds INTEGER,
    total_score     DECIMAL(8,2),
    max_possible_score DECIMAL(8,2),
    score_percentage DECIMAL(5,2),
    passed          BOOLEAN,
    completion_status VARCHAR(20),
    success_status   VARCHAR(20),
    source          VARCHAR(30) NOT NULL DEFAULT 'hosted',

    -- JSONB: The complete attempt path and state
    -- This is written once per step and read for analytics/debrief
    path_data       JSONB NOT NULL DEFAULT '{
      "steps": [],
      "variable_state": {},
      "current_node_id": null,
      "current_step": 0
    }',

    -- JSONB: AI evaluation results for free-response nodes
    ai_evaluations  JSONB NOT NULL DEFAULT '[]',

    -- Text: SCORM suspend data for session resume
    suspend_data    TEXT,

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_attempts_learner ON learner_attempts(learner_id);
CREATE INDEX idx_attempts_scenario ON learner_attempts(scenario_id);
CREATE INDEX idx_attempts_status ON learner_attempts(status);
CREATE INDEX idx_attempts_started ON learner_attempts(started_at);
CREATE INDEX idx_attempts_completed ON learner_attempts(completed_at)
    WHERE completed_at IS NOT NULL;
```

### Attempt Path Data Structure (JSONB)

```jsonc
// learner_attempts.path_data JSON structure
{
  "steps": [
    {
      "step_number": 1,
      "node_id": "uuid",
      "node_type": "dialogue",
      "entered_at": "iso8601",
      "exited_at": "iso8601",
      "duration_ms": 4500,
      "edge_id": "uuid",          // the choice made (null if auto-advanced)
      "edge_label": "I'll report this to HR",
      "was_correct": true,
      "score_awarded": 10.0,
      "response_text": null,      // for free-response nodes
      "ai_evaluation": null       // filled in by AI evaluation
    },
    {
      "step_number": 2,
      "node_id": "uuid",
      "node_type": "free_response",
      "entered_at": "iso8601",
      "exited_at": "iso8601",
      "duration_ms": 23000,
      "edge_id": null,
      "response_text": "I would document the incident and contact HR...",
      "ai_evaluation": {
        "score": 8.5,
        "max_score": 10.0,
        "rubric_match": ["documented_incident", "contacted_hr"],
        "missed_points": ["witness_identification"],
        "feedback": "Good response, but consider also identifying witnesses."
      },
      "was_correct": true,
      "score_awarded": 8.5
    }
  ],
  "variable_state": {
    "reported_to_hr": true,
    "trust_score": 15
  },
  "current_node_id": "uuid",
  "current_step": 2
}
```

### xAPI Statements (JSONB-Primary)

```sql
-- ============================================================
-- XAPI STATEMENTS (JSONB-primary -- xAPI statements ARE JSON)
-- ============================================================
CREATE TABLE xapi_statements (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    statement_id    UUID NOT NULL UNIQUE,
    attempt_id      UUID REFERENCES learner_attempts(id),
    organization_id UUID NOT NULL REFERENCES organizations(id),

    -- Promoted fields for filtering without JSONB parsing
    verb_id         TEXT NOT NULL,
    actor_email     VARCHAR(320),
    timestamp       TIMESTAMPTZ NOT NULL,

    -- The complete xAPI statement as-is
    statement_json  JSONB NOT NULL,

    -- Sync tracking
    lrs_config_id   UUID REFERENCES lrs_configurations(id),
    sync_status     VARCHAR(20) NOT NULL DEFAULT 'pending',
    sync_error      TEXT,
    synced_at       TIMESTAMPTZ,

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_xapi_attempt ON xapi_statements(attempt_id);
CREATE INDEX idx_xapi_verb ON xapi_statements(verb_id);
CREATE INDEX idx_xapi_timestamp ON xapi_statements(timestamp);
CREATE INDEX idx_xapi_sync ON xapi_statements(sync_status)
    WHERE sync_status = 'pending';
-- GIN index for querying within xAPI statement content
CREATE INDEX idx_xapi_statement_gin ON xapi_statements
    USING gin(statement_json jsonb_path_ops);
```

### Analytics (Relational Materialized Views)

```sql
-- ============================================================
-- MATERIALIZED VIEWS for analytics dashboards
-- ============================================================

-- Scenario-level statistics (refreshed periodically)
CREATE MATERIALIZED VIEW mv_scenario_stats AS
SELECT
    s.id AS scenario_id,
    s.name,
    s.organization_id,
    COUNT(a.id) AS total_attempts,
    COUNT(a.id) FILTER (WHERE a.status = 'completed') AS completed_attempts,
    COUNT(a.id) FILTER (WHERE a.status = 'abandoned') AS abandoned_attempts,
    AVG(a.score_percentage) FILTER (WHERE a.status = 'completed') AS avg_score,
    AVG(a.duration_seconds) FILTER (WHERE a.status = 'completed') AS avg_duration,
    COUNT(a.id) FILTER (WHERE a.passed = true)::DECIMAL /
        NULLIF(COUNT(a.id) FILTER (WHERE a.status = 'completed'), 0) * 100
        AS pass_rate,
    MAX(a.started_at) AS last_attempt_at
FROM scenarios s
LEFT JOIN learner_attempts a ON a.scenario_id = s.id
WHERE s.deleted_at IS NULL
GROUP BY s.id, s.name, s.organization_id;

CREATE UNIQUE INDEX idx_mv_scenario_stats ON mv_scenario_stats(scenario_id);

-- Node-level statistics (computed from attempt path_data JSONB)
-- This query demonstrates the power of JSONB: extracting analytics
-- from the document-stored path data.
CREATE MATERIALIZED VIEW mv_node_stats AS
SELECT
    a.scenario_id,
    (step->>'node_id')::UUID AS node_id,
    COUNT(*) AS visit_count,
    AVG((step->>'duration_ms')::INTEGER) AS avg_duration_ms,
    AVG(CASE WHEN (step->>'was_correct')::BOOLEAN THEN 1.0 ELSE 0.0 END)
        FILTER (WHERE step->>'was_correct' IS NOT NULL) AS correct_rate,
    COUNT(*) FILTER (WHERE step->>'edge_id' IS NULL
        AND (step->>'node_type') != 'end') AS drop_off_count
FROM learner_attempts a,
     jsonb_array_elements(a.path_data->'steps') AS step
WHERE a.status IN ('completed', 'abandoned')
GROUP BY a.scenario_id, (step->>'node_id')::UUID;

CREATE UNIQUE INDEX idx_mv_node_stats ON mv_node_stats(scenario_id, node_id);

-- Refresh materialized views (run via cron or pg_cron)
-- REFRESH MATERIALIZED VIEW CONCURRENTLY mv_scenario_stats;
-- REFRESH MATERIALIZED VIEW CONCURRENTLY mv_node_stats;
```

### Audit Trail

```sql
-- ============================================================
-- AUDIT LOG (relational + JSONB for change details)
-- ============================================================
CREATE TABLE audit_log (
    id              BIGSERIAL PRIMARY KEY,
    organization_id UUID NOT NULL,
    user_id         UUID,
    action          VARCHAR(50) NOT NULL,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    -- JSONB: before/after diff (only store what changed)
    changes         JSONB,
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

-- Create monthly partitions
CREATE TABLE audit_log_2026_05 PARTITION OF audit_log
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
CREATE TABLE audit_log_2026_06 PARTITION OF audit_log
    FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');

CREATE INDEX idx_audit_org ON audit_log(organization_id, created_at);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
```

---

## Query Examples

### Load scenario for visual editor

```sql
-- Single query loads the entire scenario graph
SELECT
    s.id, s.name, s.status, s.version, s.graph, s.scoring_config,
    s.node_count, s.edge_count, s.passing_score, s.max_score
FROM scenarios s
WHERE s.id = $1 AND s.deleted_at IS NULL;
```

### Search within scenario dialogue text

```sql
-- Find scenarios containing specific dialogue using JSONB containment
SELECT s.id, s.name,
    jsonb_path_query_array(
        s.graph, '$.nodes.*.dialogue_text ? (@ like_regex "harassment" flag "i")'
    ) AS matching_dialogue
FROM scenarios s
WHERE s.organization_id = $1
  AND s.graph @@ '$.nodes.*.dialogue_text like_regex "harassment" flag "i"';
```

### Get all paths taken through a scenario with scores

```sql
-- Extract unique paths and their average scores from attempt data
SELECT
    jsonb_path_query_array(a.path_data, '$.steps[*].node_id') AS path,
    COUNT(*) AS frequency,
    AVG(a.score_percentage) AS avg_score
FROM learner_attempts a
WHERE a.scenario_id = $1
  AND a.status = 'completed'
GROUP BY jsonb_path_query_array(a.path_data, '$.steps[*].node_id')
ORDER BY frequency DESC;
```

### Update a single node within the scenario graph

```sql
-- Atomic update of a single node within the JSONB graph
UPDATE scenarios
SET graph = jsonb_set(
    graph,
    ARRAY['nodes', $2, 'dialogue_text'],
    to_jsonb($3::TEXT)
),
updated_at = now()
WHERE id = $1;
```

---

## Pros

1. **Best of both worlds for this product's data shape.** The scenario node graph is inherently a document -- a nested, variable-structure tree that is loaded as a unit and edited as a unit. Storing it as JSONB eliminates the 6+ tables (nodes, edges, variables, mutations, conditions, positions) required in the normalized model. Meanwhile, structural data (users, attempts, LMS integrations) retains full relational integrity. This matches how the product actually works: the visual editor loads and saves entire scenario graphs; the analytics dashboard queries across attempts by status and score.

2. **Schema evolution without migrations.** Adding a new node type, a new edge property, or a new condition operator requires zero database migrations. The JSONB graph structure evolves at the application layer. For an early-stage product iterating rapidly on its core authoring model, this eliminates the migration overhead that is the primary pain point of the fully normalized approach.

3. **Dramatically simpler queries for the editor.** Loading a scenario for the visual editor is a single `SELECT` returning one row with the complete graph. Saving is a single `UPDATE`. In the normalized model, loading requires joining 6+ tables and assembling the graph in application code; saving requires a multi-table transaction with deletes, inserts, and updates.

4. **JSONB path queries enable rich content search.** PostgreSQL's `jsonb_path_query` function supports regex matching within JSONB documents, enabling searches like "find all scenarios where any dialogue mentions data privacy" without extracting dialogue text into a separate table.

5. **Single database system.** No operational complexity of multiple database engines. Self-hosted customers deploy one PostgreSQL instance. Cloud deployment uses one managed database service. This is a significant advantage for a product that explicitly supports self-hosted deployment.

6. **Natural alignment with xAPI.** xAPI statements are JSON documents. Storing them as JSONB preserves their native format while allowing promoted columns (verb_id, timestamp, actor_email) for efficient filtering. The GIN index enables queries that search within statement content.

7. **SCORM/cmi5 manifest data is naturally JSON.** SCORM manifests (imsmanifest.xml) and cmi5 course structures (cmi5.xml) are tree-structured data that maps directly to JSONB. Storing parsed manifest data as JSONB avoids the awkward normalization of XML tree structures into relational tables.

---

## Cons

1. **No referential integrity within JSONB.** When a character is deleted, there is no foreign key constraint that flags or cascades to character_id references inside JSONB graph nodes. The application must handle orphan detection and cleanup. A normalized model with foreign keys prevents this class of bug entirely.

2. **JSONB updates are full-document rewrites internally.** When a single node's dialogue text is updated via `jsonb_set()`, PostgreSQL writes a new copy of the entire JSONB column value. For a large scenario with 200+ nodes, this means writing potentially hundreds of KB of JSONB for a single-field change. This creates write amplification and increased WAL volume. The normalized model updates a single row in a single table.

3. **JSONB indexing is less efficient than column indexes.** GIN indexes on JSONB support containment operators (`@>`, `?`, `?|`) but not range queries, sorting, or partial matching with the efficiency of B-tree indexes on dedicated columns. Complex analytical queries that filter and sort by fields within JSONB documents will be slower than equivalent queries on normalized columns.

4. **Schema drift risk.** Without database-level schema enforcement on JSONB columns, different scenarios may have inconsistent JSONB structures -- missing fields, wrong types, deprecated keys. Application-level JSON Schema validation mitigates this, but it is not as strong a guarantee as database constraints. Over time, "legacy" JSONB shapes accumulate.

5. **Materialized view refresh is blocking.** The `mv_node_stats` view that extracts analytics from JSONB path data requires `jsonb_array_elements` to unnest every step in every attempt. At scale (millions of attempts), this refresh becomes slow. `REFRESH MATERIALIZED VIEW CONCURRENTLY` requires a unique index and still holds significant memory during refresh.

6. **ORM support for JSONB is uneven.** While Prisma, TypeORM, and SQLAlchemy all support JSONB columns, their support for typed JSONB paths, JSONB operators, and JSONB indexing varies. Complex JSONB queries often require raw SQL, reducing the benefits of the ORM.

7. **Backup and restore of large JSONB columns is slower.** pg_dump with large JSONB columns produces large dump files. Logical replication of tables with frequently-updated large JSONB columns generates more WAL traffic than equivalent normalized tables.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Database | PostgreSQL 16+ with pgvector extension |
| JSONB Validation | JSON Schema validation in application middleware (Ajv for Node.js, jsonschema for Python) |
| ORM | Prisma with raw queries for complex JSONB operations; or Drizzle ORM (better JSONB support) |
| Materialized View Refresh | pg_cron extension for scheduled refresh; application-triggered for critical paths |
| Caching | Redis for hot scenario graphs and active attempt state |
| Full-text Search | PostgreSQL tsvector on promoted text columns; jsonb_to_tsvector for JSONB content search |
| Blob Storage | S3-compatible for SCORM packages, audio files, media assets |
| Monitoring | pg_stat_statements for query performance; custom metrics for JSONB column sizes |

---

## Migration and Scaling Considerations

### Early Stage (0-1K scenarios)
- Single PostgreSQL instance handles everything.
- JSONB graph documents are small (< 100 KB per scenario). Write amplification is negligible.
- No materialized views needed; run analytics queries directly.
- Focus on defining JSON Schema contracts for each JSONB column early to prevent schema drift.

### Growth Stage (1K-50K scenarios, 100K-1M attempts)
- Introduce materialized views for analytics dashboards.
- Add Redis caching for frequently-loaded scenario graphs (published versions are immutable and cache well).
- Monitor JSONB column sizes; consider extracting very large scenarios (500+ nodes) into a separate "large graph" storage pattern if write amplification becomes measurable.
- Partition audit_log by month.
- Consider promoting additional JSONB fields to columns as query patterns stabilize.

### Scale Stage (50K+ scenarios, 10M+ attempts)
- Move analytics to a dedicated analytical store (ClickHouse, TimescaleDB) fed by CDC.
- Archive old attempt path_data to cold storage, keeping only relational summary fields.
- Consider splitting the xapi_statements table into hot (pending sync) and cold (synced/archived) tables.
- For very large organizations, partition learner_attempts by organization_id using declarative partitioning.

### Migration from Normalized Model
If starting from Suggestion 1 (fully normalized):
1. Create the JSONB `graph` column on the scenarios table.
2. Write a migration that serializes data from nodes/edges/variables tables into the JSONB graph.
3. Run the application in dual-read mode (read from JSONB, fall back to tables).
4. Once validated, switch writes to JSONB-first.
5. Drop the separate nodes/edges/variables tables.

### Migration to Fully Normalized
If JSONB proves too limiting at scale:
1. Create the normalized tables (nodes, edges, variables, etc.).
2. Write a migration that extracts data from JSONB graphs into normalized tables.
3. Run in dual-write mode until validated.
4. Switch reads to normalized tables.
5. Remove the JSONB graph column.

This bidirectional migration path is a significant advantage of the hybrid approach: it does not lock you into either extreme.
