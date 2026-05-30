# Data Model Suggestion 1: Normalized Relational Database (PostgreSQL)

> Project: Training Simulator Builder (Candidate #430)
> Generated: 2026-05-25

---

## Overview

This model uses a fully normalized relational schema in PostgreSQL. Every entity -- scenarios, nodes, characters, learner attempts, analytics events -- lives in its own table with explicit foreign key relationships. The design follows third normal form (3NF) throughout, with referential integrity enforced at the database level. This approach maximizes data consistency, makes complex analytical queries straightforward, and aligns well with the compliance audit requirements central to this product.

---

## Design Principles

1. **Strict normalization (3NF):** No data duplication; every fact stored once.
2. **Referential integrity everywhere:** Foreign keys with cascading rules prevent orphaned records.
3. **Temporal tracking:** `created_at` and `updated_at` on every mutable table; soft deletes via `deleted_at` where appropriate.
4. **Multi-tenancy via `organization_id`:** Row-level security for SaaS deployment; removable for self-hosted.
5. **Audit-ready:** Separate audit log table captures all mutations for compliance evidence.
6. **Standards alignment:** Schema structures mirror xAPI statement patterns and SCORM data model elements where applicable.

---

## Complete Schema

### Tenant and Identity

```sql
-- ============================================================
-- ORGANIZATIONS (multi-tenant root)
-- ============================================================
CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    logo_url        TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ
);

CREATE INDEX idx_organizations_slug ON organizations(slug);

-- ============================================================
-- USERS
-- ============================================================
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    email           VARCHAR(320) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    password_hash   TEXT,
    auth_provider   VARCHAR(50) NOT NULL DEFAULT 'local',  -- local, oidc, saml
    auth_subject    VARCHAR(512),                           -- external IdP subject
    avatar_url      TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ,
    UNIQUE (organization_id, email)
);

CREATE INDEX idx_users_org ON users(organization_id);
CREATE INDEX idx_users_email ON users(email);

-- ============================================================
-- ROLES AND PERMISSIONS
-- ============================================================
CREATE TABLE roles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(100) NOT NULL,  -- admin, author, reviewer, publisher, learner
    permissions     TEXT[] NOT NULL DEFAULT '{}',
    is_system       BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE user_roles (
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    granted_by      UUID REFERENCES users(id),
    PRIMARY KEY (user_id, role_id)
);
```

### Scenario Authoring Core

```sql
-- ============================================================
-- PROJECTS (top-level grouping for scenarios)
-- ============================================================
CREATE TABLE projects (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    status          VARCHAR(50) NOT NULL DEFAULT 'draft',
        -- draft, in_review, approved, published, archived
    category        VARCHAR(100),
        -- compliance, sales, clinical, leadership, custom
    tags            TEXT[] NOT NULL DEFAULT '{}',
    default_locale  VARCHAR(10) NOT NULL DEFAULT 'en-US',
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ
);

CREATE INDEX idx_projects_org ON projects(organization_id);
CREATE INDEX idx_projects_status ON projects(status);
CREATE INDEX idx_projects_category ON projects(category);

-- ============================================================
-- SCENARIOS (a single simulation within a project)
-- ============================================================
CREATE TABLE scenarios (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    version         INTEGER NOT NULL DEFAULT 1,
    status          VARCHAR(50) NOT NULL DEFAULT 'draft',
    training_objective TEXT,
    estimated_duration_minutes INTEGER,
    difficulty_level VARCHAR(20),  -- beginner, intermediate, advanced
    passing_score   DECIMAL(5,2),  -- minimum score to pass (percentage)
    max_score       DECIMAL(8,2),  -- maximum achievable score
    locale          VARCHAR(10) NOT NULL DEFAULT 'en-US',
    is_template     BOOLEAN NOT NULL DEFAULT false,
    template_source_id UUID REFERENCES scenarios(id),
    created_by      UUID NOT NULL REFERENCES users(id),
    published_at    TIMESTAMPTZ,
    published_by    UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ
);

CREATE INDEX idx_scenarios_project ON scenarios(project_id);
CREATE INDEX idx_scenarios_status ON scenarios(status);
CREATE INDEX idx_scenarios_template ON scenarios(is_template) WHERE is_template = true;

-- ============================================================
-- SCENARIO VERSIONS (immutable snapshots for publishing)
-- ============================================================
CREATE TABLE scenario_versions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scenario_id     UUID NOT NULL REFERENCES scenarios(id) ON DELETE CASCADE,
    version_number  INTEGER NOT NULL,
    snapshot_data   JSONB NOT NULL,   -- full serialized scenario graph
    changelog       TEXT,
    published_by    UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (scenario_id, version_number)
);

CREATE INDEX idx_scenario_versions_scenario ON scenario_versions(scenario_id);
```

### Node Graph (Branching Logic)

```sql
-- ============================================================
-- SCENARIO NODES (decision points, dialogue, debrief, etc.)
-- ============================================================
CREATE TABLE scenario_nodes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scenario_id     UUID NOT NULL REFERENCES scenarios(id) ON DELETE CASCADE,
    node_type       VARCHAR(50) NOT NULL,
        -- start, dialogue, decision_point, consequence,
        -- branch_merge, debrief, end, free_response,
        -- media, information, delay
    title           VARCHAR(255),
    position_x      FLOAT NOT NULL DEFAULT 0,  -- canvas position for visual editor
    position_y      FLOAT NOT NULL DEFAULT 0,
    character_id    UUID REFERENCES characters(id),
    dialogue_text   TEXT,
    narrator_text   TEXT,
    media_url       TEXT,
    media_type      VARCHAR(50),  -- image, video, audio
    feedback_text   TEXT,         -- shown after learner makes a choice
    is_entry_point  BOOLEAN NOT NULL DEFAULT false,
    is_terminal     BOOLEAN NOT NULL DEFAULT false,
    score_value     DECIMAL(8,2) DEFAULT 0,
    time_limit_seconds INTEGER,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_nodes_scenario ON scenario_nodes(scenario_id);
CREATE INDEX idx_nodes_type ON scenario_nodes(node_type);
CREATE INDEX idx_nodes_character ON scenario_nodes(character_id);
CREATE INDEX idx_nodes_entry ON scenario_nodes(is_entry_point) WHERE is_entry_point = true;

-- ============================================================
-- NODE EDGES (connections between nodes)
-- ============================================================
CREATE TABLE node_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scenario_id     UUID NOT NULL REFERENCES scenarios(id) ON DELETE CASCADE,
    source_node_id  UUID NOT NULL REFERENCES scenario_nodes(id) ON DELETE CASCADE,
    target_node_id  UUID NOT NULL REFERENCES scenario_nodes(id) ON DELETE CASCADE,
    edge_type       VARCHAR(50) NOT NULL DEFAULT 'choice',
        -- choice, fallthrough, conditional, timeout, default
    label           VARCHAR(255),         -- displayed choice text
    sort_order      INTEGER NOT NULL DEFAULT 0,
    score_delta     DECIMAL(8,2) DEFAULT 0,
    is_correct      BOOLEAN,              -- null = no right/wrong
    feedback_text   TEXT,                  -- immediate feedback on selection
    condition_expression TEXT,             -- for conditional edges
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT no_self_loop CHECK (source_node_id != target_node_id)
);

CREATE INDEX idx_edges_scenario ON node_edges(scenario_id);
CREATE INDEX idx_edges_source ON node_edges(source_node_id);
CREATE INDEX idx_edges_target ON node_edges(target_node_id);

-- ============================================================
-- NODE VARIABLES (state tracked across the scenario)
-- ============================================================
CREATE TABLE scenario_variables (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scenario_id     UUID NOT NULL REFERENCES scenarios(id) ON DELETE CASCADE,
    name            VARCHAR(100) NOT NULL,
    variable_type   VARCHAR(20) NOT NULL,  -- string, number, boolean
    default_value   TEXT,
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (scenario_id, name)
);

-- ============================================================
-- VARIABLE MUTATIONS (how edges/nodes change variables)
-- ============================================================
CREATE TABLE variable_mutations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    edge_id         UUID REFERENCES node_edges(id) ON DELETE CASCADE,
    node_id         UUID REFERENCES scenario_nodes(id) ON DELETE CASCADE,
    variable_id     UUID NOT NULL REFERENCES scenario_variables(id) ON DELETE CASCADE,
    operation       VARCHAR(20) NOT NULL,  -- set, increment, decrement, toggle
    value           TEXT NOT NULL,
    CONSTRAINT mutation_has_source CHECK (
        (edge_id IS NOT NULL AND node_id IS NULL) OR
        (edge_id IS NULL AND node_id IS NOT NULL)
    )
);

-- ============================================================
-- EDGE CONDITIONS (conditional branching based on variables)
-- ============================================================
CREATE TABLE edge_conditions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    edge_id         UUID NOT NULL REFERENCES node_edges(id) ON DELETE CASCADE,
    variable_id     UUID NOT NULL REFERENCES scenario_variables(id) ON DELETE CASCADE,
    operator        VARCHAR(20) NOT NULL,  -- eq, neq, gt, gte, lt, lte, contains
    value           TEXT NOT NULL,
    logical_group   INTEGER NOT NULL DEFAULT 0  -- for AND/OR grouping
);

CREATE INDEX idx_edge_conditions_edge ON edge_conditions(edge_id);
```

### Characters and Voice

```sql
-- ============================================================
-- CHARACTERS
-- ============================================================
CREATE TABLE characters (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    role_label      VARCHAR(100),  -- e.g., "Manager", "HR Representative"
    avatar_url      TEXT,
    avatar_style    VARCHAR(50),   -- realistic, illustrated, silhouette
    gender          VARCHAR(20),
    age_range       VARCHAR(20),
    ethnicity       VARCHAR(50),
    voice_profile_id UUID REFERENCES voice_profiles(id),
    is_library      BOOLEAN NOT NULL DEFAULT false,  -- shared across org
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ
);

CREATE INDEX idx_characters_org ON characters(organization_id);

-- ============================================================
-- VOICE PROFILES
-- ============================================================
CREATE TABLE voice_profiles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    provider        VARCHAR(50) NOT NULL,  -- elevenlabs, azure, google, custom
    provider_voice_id VARCHAR(255) NOT NULL,
    language        VARCHAR(10) NOT NULL DEFAULT 'en-US',
    gender          VARCHAR(20),
    style           VARCHAR(50),   -- calm, authoritative, friendly, neutral
    speed_factor    DECIMAL(3,2) NOT NULL DEFAULT 1.00,
    pitch_factor    DECIMAL(3,2) NOT NULL DEFAULT 1.00,
    sample_audio_url TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- GENERATED AUDIO CACHE
-- ============================================================
CREATE TABLE audio_cache (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    voice_profile_id UUID NOT NULL REFERENCES voice_profiles(id),
    text_hash       VARCHAR(64) NOT NULL,  -- SHA-256 of input text
    text_content    TEXT NOT NULL,
    audio_url       TEXT NOT NULL,
    audio_format    VARCHAR(10) NOT NULL DEFAULT 'mp3',
    duration_ms     INTEGER,
    file_size_bytes BIGINT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (voice_profile_id, text_hash)
);
```

### Policy Documents and AI Content

```sql
-- ============================================================
-- POLICY DOCUMENTS (uploaded source material for AI generation)
-- ============================================================
CREATE TABLE policy_documents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    document_type   VARCHAR(50) NOT NULL,
        -- policy, handbook, regulation, sop, guideline
    file_url        TEXT,
    file_type       VARCHAR(20),   -- pdf, docx, txt, html
    file_size_bytes BIGINT,
    extracted_text  TEXT,           -- full text extraction
    chunk_count     INTEGER,
    embedding_model VARCHAR(100),
    uploaded_by     UUID NOT NULL REFERENCES users(id),
    processed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ
);

CREATE INDEX idx_policy_docs_org ON policy_documents(organization_id);

-- ============================================================
-- DOCUMENT CHUNKS (for RAG-based AI generation)
-- ============================================================
CREATE TABLE document_chunks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id     UUID NOT NULL REFERENCES policy_documents(id) ON DELETE CASCADE,
    chunk_index     INTEGER NOT NULL,
    content         TEXT NOT NULL,
    token_count     INTEGER,
    embedding       VECTOR(1536),  -- pgvector for similarity search
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_chunks_document ON document_chunks(document_id);
CREATE INDEX idx_chunks_embedding ON document_chunks
    USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);

-- ============================================================
-- AI GENERATION JOBS
-- ============================================================
CREATE TABLE ai_generation_jobs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scenario_id     UUID NOT NULL REFERENCES scenarios(id) ON DELETE CASCADE,
    job_type        VARCHAR(50) NOT NULL,
        -- dialogue_generation, branch_suggestion, quality_review,
        -- scenario_variation, response_evaluation, translation
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
        -- pending, processing, completed, failed
    input_prompt    TEXT NOT NULL,
    input_context   JSONB,           -- policy doc refs, existing dialogue, etc.
    output_content  JSONB,           -- generated content
    model_used      VARCHAR(100),
    token_count_input  INTEGER,
    token_count_output INTEGER,
    cost_usd        DECIMAL(10,6),
    error_message   TEXT,
    requested_by    UUID NOT NULL REFERENCES users(id),
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ai_jobs_scenario ON ai_generation_jobs(scenario_id);
CREATE INDEX idx_ai_jobs_status ON ai_generation_jobs(status);

-- ============================================================
-- SCENARIO-DOCUMENT ASSOCIATIONS
-- ============================================================
CREATE TABLE scenario_policy_documents (
    scenario_id     UUID NOT NULL REFERENCES scenarios(id) ON DELETE CASCADE,
    document_id     UUID NOT NULL REFERENCES policy_documents(id) ON DELETE CASCADE,
    relevance_note  TEXT,
    added_by        UUID NOT NULL REFERENCES users(id),
    added_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (scenario_id, document_id)
);
```

### Templates

```sql
-- ============================================================
-- SCENARIO TEMPLATES
-- ============================================================
CREATE TABLE scenario_templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    category        VARCHAR(100) NOT NULL,
        -- workplace_harassment, workplace_safety, data_privacy,
        -- code_of_conduct, sales_conversation, clinical_communication,
        -- leadership_coaching, custom
    difficulty_level VARCHAR(20),
    estimated_duration_minutes INTEGER,
    thumbnail_url   TEXT,
    template_data   JSONB NOT NULL,   -- serialized scenario graph
    is_system       BOOLEAN NOT NULL DEFAULT true,  -- built-in vs user-created
    organization_id UUID REFERENCES organizations(id),  -- null = global
    created_by      UUID REFERENCES users(id),
    usage_count     INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_templates_category ON scenario_templates(category);
CREATE INDEX idx_templates_system ON scenario_templates(is_system);
```

### Collaboration

```sql
-- ============================================================
-- COMMENTS (threaded discussion on scenario elements)
-- ============================================================
CREATE TABLE comments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scenario_id     UUID NOT NULL REFERENCES scenarios(id) ON DELETE CASCADE,
    node_id         UUID REFERENCES scenario_nodes(id) ON DELETE CASCADE,
    edge_id         UUID REFERENCES node_edges(id) ON DELETE CASCADE,
    parent_comment_id UUID REFERENCES comments(id) ON DELETE CASCADE,
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
CREATE INDEX idx_comments_node ON comments(node_id);
CREATE INDEX idx_comments_parent ON comments(parent_comment_id);

-- ============================================================
-- REVIEW WORKFLOWS
-- ============================================================
CREATE TABLE review_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scenario_id     UUID NOT NULL REFERENCES scenarios(id) ON DELETE CASCADE,
    version_id      UUID REFERENCES scenario_versions(id),
    requested_by    UUID NOT NULL REFERENCES users(id),
    assigned_to     UUID NOT NULL REFERENCES users(id),
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
        -- pending, in_review, approved, rejected, revision_requested
    review_notes    TEXT,
    requested_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ
);

CREATE INDEX idx_reviews_scenario ON review_requests(scenario_id);
CREATE INDEX idx_reviews_assignee ON review_requests(assigned_to);
CREATE INDEX idx_reviews_status ON review_requests(status);
```

### LMS Integration and Export

```sql
-- ============================================================
-- EXPORT PACKAGES (SCORM, xAPI, cmi5 exports)
-- ============================================================
CREATE TABLE export_packages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scenario_id     UUID NOT NULL REFERENCES scenarios(id) ON DELETE CASCADE,
    version_id      UUID NOT NULL REFERENCES scenario_versions(id),
    format          VARCHAR(30) NOT NULL,
        -- scorm_12, scorm_2004, xapi, cmi5, lti_13, html5_standalone
    package_url     TEXT,
    file_size_bytes BIGINT,
    manifest_data   JSONB,           -- parsed imsmanifest.xml or cmi5.xml
    build_status    VARCHAR(20) NOT NULL DEFAULT 'pending',
    build_error     TEXT,
    exported_by     UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_exports_scenario ON export_packages(scenario_id);
CREATE INDEX idx_exports_format ON export_packages(format);

-- ============================================================
-- LTI REGISTRATIONS (LTI 1.3 platform registrations)
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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_lti_org ON lti_registrations(organization_id);
CREATE INDEX idx_lti_issuer ON lti_registrations(issuer);

-- ============================================================
-- LTI LAUNCHES (deep link launch records)
-- ============================================================
CREATE TABLE lti_launches (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    registration_id UUID NOT NULL REFERENCES lti_registrations(id),
    scenario_id     UUID NOT NULL REFERENCES scenarios(id),
    learner_id      UUID REFERENCES learners(id),
    launch_type     VARCHAR(30) NOT NULL,  -- resource_link, deep_link
    nonce           VARCHAR(255) NOT NULL,
    state           VARCHAR(255),
    claims          JSONB NOT NULL,        -- full LTI JWT claims
    grade_posted    BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- LRS CONFIGURATION (xAPI Learning Record Store connections)
-- ============================================================
CREATE TABLE lrs_configurations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    endpoint_url    TEXT NOT NULL,
    auth_type       VARCHAR(20) NOT NULL,  -- basic, oauth2
    auth_credentials_encrypted TEXT NOT NULL,
    xapi_version    VARCHAR(10) NOT NULL DEFAULT '1.0.3',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_sync_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Learner Experience and Analytics

```sql
-- ============================================================
-- LEARNERS (may or may not be registered users)
-- ============================================================
CREATE TABLE learners (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID REFERENCES organizations(id),
    user_id         UUID REFERENCES users(id),
    external_id     VARCHAR(255),   -- LMS user ID, SCORM learner_id
    email           VARCHAR(320),
    display_name    VARCHAR(255),
    lti_subject     VARCHAR(512),   -- LTI 1.3 subject claim
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_learners_org ON learners(organization_id);
CREATE INDEX idx_learners_user ON learners(user_id);
CREATE INDEX idx_learners_external ON learners(external_id);

-- ============================================================
-- LEARNER ATTEMPTS (a single run through a scenario)
-- ============================================================
CREATE TABLE learner_attempts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    learner_id      UUID NOT NULL REFERENCES learners(id),
    scenario_id     UUID NOT NULL REFERENCES scenarios(id),
    version_id      UUID REFERENCES scenario_versions(id),
    attempt_number  INTEGER NOT NULL DEFAULT 1,
    status          VARCHAR(30) NOT NULL DEFAULT 'in_progress',
        -- in_progress, completed, abandoned, timed_out
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    duration_seconds INTEGER,
    total_score     DECIMAL(8,2),
    max_possible_score DECIMAL(8,2),
    score_percentage DECIMAL(5,2),
    passed          BOOLEAN,
    completion_status VARCHAR(20),   -- SCORM: completed, incomplete, not attempted
    success_status   VARCHAR(20),    -- SCORM: passed, failed, unknown
    suspend_data    TEXT,            -- SCORM suspend_data for resume
    source          VARCHAR(30) NOT NULL DEFAULT 'hosted',
        -- hosted, scorm, lti, cmi5
    lti_launch_id   UUID REFERENCES lti_launches(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_attempts_learner ON learner_attempts(learner_id);
CREATE INDEX idx_attempts_scenario ON learner_attempts(scenario_id);
CREATE INDEX idx_attempts_status ON learner_attempts(status);
CREATE INDEX idx_attempts_started ON learner_attempts(started_at);

-- ============================================================
-- ATTEMPT STEPS (each node visited during an attempt)
-- ============================================================
CREATE TABLE attempt_steps (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    attempt_id      UUID NOT NULL REFERENCES learner_attempts(id) ON DELETE CASCADE,
    node_id         UUID NOT NULL REFERENCES scenario_nodes(id),
    edge_id         UUID REFERENCES node_edges(id),  -- the choice made
    step_number     INTEGER NOT NULL,
    entered_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    exited_at       TIMESTAMPTZ,
    duration_ms     INTEGER,
    response_text   TEXT,           -- for free-response nodes
    score_awarded   DECIMAL(8,2),
    was_correct     BOOLEAN,
    ai_evaluation   JSONB,          -- AI assessment of free-text response
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_steps_attempt ON attempt_steps(attempt_id);
CREATE INDEX idx_steps_node ON attempt_steps(node_id);

-- ============================================================
-- XAPI STATEMENTS (outbound xAPI statement log)
-- ============================================================
CREATE TABLE xapi_statements (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    statement_id    UUID NOT NULL UNIQUE,  -- xAPI statement UUID
    attempt_id      UUID REFERENCES learner_attempts(id),
    actor_json      JSONB NOT NULL,
    verb_id         TEXT NOT NULL,
    verb_display    JSONB NOT NULL,
    object_json     JSONB NOT NULL,
    result_json     JSONB,
    context_json    JSONB,
    timestamp       TIMESTAMPTZ NOT NULL,
    stored          TIMESTAMPTZ NOT NULL DEFAULT now(),
    lrs_config_id   UUID REFERENCES lrs_configurations(id),
    sync_status     VARCHAR(20) NOT NULL DEFAULT 'pending',
        -- pending, sent, failed, not_applicable
    sync_error      TEXT,
    synced_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_xapi_attempt ON xapi_statements(attempt_id);
CREATE INDEX idx_xapi_verb ON xapi_statements(verb_id);
CREATE INDEX idx_xapi_sync ON xapi_statements(sync_status);
CREATE INDEX idx_xapi_timestamp ON xapi_statements(timestamp);

-- ============================================================
-- COMPLIANCE EVIDENCE REPORTS
-- ============================================================
CREATE TABLE compliance_reports (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    report_type     VARCHAR(50) NOT NULL,
        -- completion_summary, competency_evidence, regulatory_audit
    regulation_ref  VARCHAR(100),   -- OSHA, FINRA, HIPAA, GDPR, etc.
    scenario_ids    UUID[] NOT NULL,
    date_range_start DATE NOT NULL,
    date_range_end  DATE NOT NULL,
    generated_data  JSONB NOT NULL,
    file_url        TEXT,
    generated_by    UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_compliance_org ON compliance_reports(organization_id);
CREATE INDEX idx_compliance_regulation ON compliance_reports(regulation_ref);
```

### Audit Trail

```sql
-- ============================================================
-- AUDIT LOG (all mutations for compliance trail)
-- ============================================================
CREATE TABLE audit_log (
    id              BIGSERIAL PRIMARY KEY,
    organization_id UUID NOT NULL,
    user_id         UUID,
    action          VARCHAR(50) NOT NULL,
        -- create, update, delete, publish, export, login, review
    entity_type     VARCHAR(50) NOT NULL,
        -- scenario, node, edge, character, user, export, etc.
    entity_id       UUID NOT NULL,
    old_values      JSONB,
    new_values      JSONB,
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_org ON audit_log(organization_id);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_user ON audit_log(user_id);
CREATE INDEX idx_audit_created ON audit_log(created_at);

-- Partition by month for performance at scale
-- CREATE TABLE audit_log_2026_05 PARTITION OF audit_log
--     FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
```

### Analytics Aggregations

```sql
-- ============================================================
-- SCENARIO ANALYTICS (pre-computed analytics per scenario)
-- ============================================================
CREATE TABLE scenario_analytics (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scenario_id     UUID NOT NULL REFERENCES scenarios(id) ON DELETE CASCADE,
    computed_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    total_attempts  INTEGER NOT NULL DEFAULT 0,
    completed_attempts INTEGER NOT NULL DEFAULT 0,
    average_score   DECIMAL(5,2),
    average_duration_seconds INTEGER,
    pass_rate       DECIMAL(5,2),
    most_common_path UUID[],          -- ordered array of node IDs
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_scenario_analytics ON scenario_analytics(scenario_id);

-- ============================================================
-- NODE ANALYTICS (per-node aggregate metrics)
-- ============================================================
CREATE TABLE node_analytics (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scenario_id     UUID NOT NULL REFERENCES scenarios(id) ON DELETE CASCADE,
    node_id         UUID NOT NULL REFERENCES scenario_nodes(id) ON DELETE CASCADE,
    computed_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    visit_count     INTEGER NOT NULL DEFAULT 0,
    avg_time_on_node_ms INTEGER,
    correct_choice_rate DECIMAL(5,2),
    edge_distribution JSONB,  -- { edge_id: count, ... }
    drop_off_count  INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_node_analytics_scenario ON node_analytics(scenario_id);
CREATE INDEX idx_node_analytics_node ON node_analytics(node_id);
```

---

## Key Relationships Diagram

```
organizations ──< users ──< user_roles >── roles
     │
     ├──< projects ──< scenarios ──< scenario_nodes ──< node_edges
     │                    │               │                   │
     │                    │               ├── variable_mutations
     │                    │               │
     │                    ├──< scenario_variables
     │                    ├──< scenario_versions
     │                    ├──< export_packages
     │                    ├──< comments
     │                    ├──< review_requests
     │                    └──< learner_attempts ──< attempt_steps
     │                                                    │
     ├──< characters ──── voice_profiles                  └── xapi_statements
     │
     ├──< policy_documents ──< document_chunks
     │
     ├──< lti_registrations ──< lti_launches
     │
     └──< lrs_configurations
```

---

## Pros

1. **Data integrity is guaranteed.** Foreign keys, check constraints, and unique constraints prevent invalid states -- critical for a product that generates compliance evidence reports. An orphaned node or a broken edge in a scenario graph would be a product-breaking bug, and the relational model makes such bugs structurally impossible at the database level.

2. **Complex analytical queries are natural.** Computing path frequency analysis, decision-point error rates, pass/fail rates by department, and time-on-node distributions requires joins across attempts, steps, nodes, and edges. SQL is purpose-built for these queries, and PostgreSQL's query planner handles them efficiently with proper indexing.

3. **Compliance audit trail is straightforward.** The audit_log table with old_values/new_values JSONB columns provides a complete, immutable record of every change. Combined with xapi_statements, this creates the regulatory evidence chain that compliance officers in healthcare, financial services, and government require.

4. **Mature tooling and ecosystem.** PostgreSQL has battle-tested support for row-level security (multi-tenancy), partitioning (audit logs), full-text search (scenario search), and pgvector (policy document embeddings). Every major cloud provider offers managed PostgreSQL. ORM support (Prisma, Drizzle, TypeORM, SQLAlchemy) is excellent.

5. **Team familiarity.** Most backend engineers know relational schemas. Onboarding new contributors is fast. Schema migrations are well-understood (Flyway, Alembic, Prisma Migrate).

6. **Transaction safety for collaborative editing.** Multiple authors editing the same scenario can use row-level locking or optimistic concurrency control (version columns) to prevent lost updates -- a solved problem in relational databases.

---

## Cons

1. **Schema rigidity penalizes rapid iteration.** The node graph model is the product's core differentiator, and its shape will evolve significantly during early development. Every structural change (adding a new node type, a new edge property, a new condition operator) requires a migration. In the first year of development, this friction adds up.

2. **Graph traversal is expensive in SQL.** Computing "all paths from start to end" or "find all nodes reachable from this decision point" requires recursive CTEs, which are functional but significantly slower and harder to write than native graph queries. For a product where the visual editor constantly needs to validate graph connectivity, detect cycles, and compute reachability, this is a real operational cost.

3. **Flexible/variable node content is awkward to normalize.** Different node types (dialogue, free-response, media, delay) have different data requirements. A fully normalized design would split each into a subtype table, creating a proliferation of tables and complex polymorphic joins. The pragmatic alternative (nullable columns on a single `scenario_nodes` table) violates normalization principles.

4. **Scaling the analytics workload competes with the authoring workload.** Heavy analytical queries (path frequency across 100K attempts) running on the same database as the real-time authoring and delivery workload can cause contention. Read replicas mitigate this but add operational complexity.

5. **JSONB is a crutch.** Several columns in this schema already use JSONB (snapshot_data, template_data, ai evaluation results). Every JSONB column is an admission that the relational model does not fit the data shape well. If too many columns become JSONB, the benefits of normalization are diluted.

6. **No native full-text search for scenario content.** While PostgreSQL has `tsvector`/`tsquery`, searching across dialogue text in thousands of nodes requires careful index management and is less capable than a dedicated search engine.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Database | PostgreSQL 16+ with pgvector extension |
| Connection pooler | PgBouncer or Supabase Supavisor |
| Migrations | Prisma Migrate (if Node.js) or Alembic (if Python) |
| ORM | Prisma (TypeScript) or SQLAlchemy 2.0 (Python) |
| Read replicas | 1+ read replica for analytics queries from day one |
| Full-text search | PostgreSQL tsvector initially; Meilisearch or Typesense when search complexity grows |
| Caching | Redis for session data, computed analytics, and audio cache metadata |
| Blob storage | S3-compatible (AWS S3, Cloudflare R2, MinIO for self-hosted) for SCORM packages, audio, media |

---

## Migration and Scaling Considerations

### Early Stage (0-1K scenarios, <10K learner attempts)
- Single PostgreSQL instance is sufficient for both read and write workloads.
- All tables in a single schema. No partitioning needed.
- Focus on getting the schema right; migrations are cheap at low data volume.

### Growth Stage (1K-50K scenarios, 100K-1M attempts)
- Add a read replica dedicated to analytics queries and dashboard rendering.
- Partition `audit_log` and `xapi_statements` by month to keep query performance stable.
- Implement connection pooling (PgBouncer) to handle concurrent authoring sessions.
- Consider materialized views for `scenario_analytics` and `node_analytics` to avoid recomputing aggregates on every dashboard load.

### Scale Stage (50K+ scenarios, 10M+ attempts)
- Partition `learner_attempts` and `attempt_steps` by organization_id or date range.
- Move analytics aggregation to a dedicated analytical database (e.g., ClickHouse, TimescaleDB) via CDC (Change Data Capture) or batch ETL.
- Consider extracting the xAPI statement pipeline into a separate service with its own datastore if xAPI volume becomes a bottleneck.
- Row-level security for multi-tenancy works well up to ~1000 tenants; beyond that, consider schema-per-tenant or separate databases per large enterprise customer.

### Self-Hosted Deployment
- The entire schema runs on a single PostgreSQL instance, making self-hosted deployment straightforward.
- Provide a Docker Compose configuration with PostgreSQL, the application, and MinIO for blob storage.
- Migration scripts must be idempotent and backward-compatible to support customer-managed upgrades.
