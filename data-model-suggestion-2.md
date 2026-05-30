# Data Model Suggestion 2: Event-Sourced / CQRS Model

> Project: Training Simulator Builder (Candidate #430)
> Generated: 2026-05-25

---

## Overview

This model uses Event Sourcing with Command Query Responsibility Segregation (CQRS) as its foundational architecture. Every change to every entity -- scenario graph mutations, learner interactions, collaboration actions, publishing decisions -- is captured as an immutable event. The current state of any entity is derived by replaying its event stream. Read models (projections) are materialized from these events and optimized independently for each query use case: the visual editor, the learner player, the analytics dashboard, the compliance report generator.

This architecture is a natural fit for a training simulator builder because the product has several characteristics that align perfectly with event sourcing:

1. **Collaborative authoring** requires tracking who changed what, when, and why -- event sourcing provides this by definition.
2. **Version history** is a core feature requirement; event streams are version histories.
3. **Compliance audit trails** are mandatory; an immutable append-only event log is the strongest possible audit mechanism.
4. **Branching scenario replay** requires the ability to reconstruct past states of scenarios for version comparison.
5. **Learner path tracking** is inherently event-oriented: a learner's journey through a scenario is a sequence of discrete events.

---

## Design Principles

1. **Events are the source of truth.** No mutable state tables. All read models are derived and can be reconstructed from events at any time.
2. **Commands are validated before events are emitted.** Business rules are enforced in command handlers, not in the read models.
3. **Projections are disposable.** Any read model can be dropped and rebuilt from the event stream without data loss.
4. **Event streams are partitioned by aggregate.** Each scenario, each learner attempt, and each organization is its own aggregate with its own event stream.
5. **Eventual consistency between write and read models.** Projections may lag behind the event store by milliseconds to seconds.
6. **Snapshots for performance.** Frequently-accessed aggregates (active scenarios, ongoing attempts) maintain periodic snapshots to avoid replaying thousands of events.

---

## Event Store Schema

```sql
-- ============================================================
-- EVENT STORE (the single source of truth)
-- ============================================================
CREATE TABLE events (
    global_position    BIGSERIAL NOT NULL,           -- global ordering
    stream_id          VARCHAR(255) NOT NULL,         -- e.g., 'scenario-{uuid}'
    stream_position    BIGINT NOT NULL,               -- position within stream
    event_type         VARCHAR(200) NOT NULL,          -- e.g., 'ScenarioNodeAdded'
    event_data         JSONB NOT NULL,                 -- event payload
    metadata           JSONB NOT NULL DEFAULT '{}',    -- causation_id, correlation_id, user_id, etc.
    occurred_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, stream_position)
);

-- Global ordering index for projections that process all events
CREATE UNIQUE INDEX idx_events_global ON events(global_position);

-- Fast lookup by event type (for specific projectors)
CREATE INDEX idx_events_type ON events(event_type);

-- Time-based queries for audit and analytics
CREATE INDEX idx_events_occurred ON events(occurred_at);

-- Category-based queries (e.g., all scenario streams)
CREATE INDEX idx_events_stream_prefix ON events(stream_id text_pattern_ops);

-- ============================================================
-- SNAPSHOTS (periodic state snapshots for fast aggregate loading)
-- ============================================================
CREATE TABLE snapshots (
    stream_id          VARCHAR(255) NOT NULL,
    stream_position    BIGINT NOT NULL,
    snapshot_type      VARCHAR(200) NOT NULL,
    snapshot_data      JSONB NOT NULL,
    created_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, stream_position)
);

-- Keep only the latest snapshot per stream
CREATE INDEX idx_snapshots_latest ON snapshots(stream_id, stream_position DESC);

-- ============================================================
-- SUBSCRIPTIONS (projection checkpoint tracking)
-- ============================================================
CREATE TABLE projection_checkpoints (
    projection_name    VARCHAR(200) PRIMARY KEY,
    last_global_position BIGINT NOT NULL DEFAULT 0,
    last_updated_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Event Types Catalogue

### Scenario Authoring Events

```
Stream: scenario-{scenario_id}

ScenarioCreated
  { project_id, name, description, training_objective, locale, created_by }

ScenarioRenamed
  { name }

ScenarioDescriptionUpdated
  { description }

ScenarioTrainingObjectiveSet
  { training_objective }

ScenarioNodeAdded
  { node_id, node_type, title, position_x, position_y, character_id,
    dialogue_text, narrator_text, media_url, media_type, feedback_text,
    is_entry_point, is_terminal, score_value, time_limit_seconds }

ScenarioNodeUpdated
  { node_id, changed_fields: { field_name: { old, new } } }

ScenarioNodeRemoved
  { node_id }

ScenarioNodeRepositioned
  { node_id, position_x, position_y }

ScenarioEdgeAdded
  { edge_id, source_node_id, target_node_id, edge_type, label,
    sort_order, score_delta, is_correct, feedback_text }

ScenarioEdgeUpdated
  { edge_id, changed_fields: { field_name: { old, new } } }

ScenarioEdgeRemoved
  { edge_id }

ScenarioVariableCreated
  { variable_id, name, variable_type, default_value, description }

ScenarioVariableUpdated
  { variable_id, changed_fields }

ScenarioVariableRemoved
  { variable_id }

VariableMutationAdded
  { mutation_id, edge_id?, node_id?, variable_id, operation, value }

VariableMutationRemoved
  { mutation_id }

EdgeConditionAdded
  { condition_id, edge_id, variable_id, operator, value, logical_group }

EdgeConditionRemoved
  { condition_id }

ScenarioScoringConfigured
  { passing_score, max_score }

ScenarioDifficultySet
  { difficulty_level, estimated_duration_minutes }

ScenarioSubmittedForReview
  { requested_by, assigned_to, notes }

ScenarioReviewCompleted
  { reviewer_id, status, review_notes }
    -- status: approved, rejected, revision_requested

ScenarioPublished
  { version_number, published_by, snapshot_data }

ScenarioUnpublished
  { reason }

ScenarioArchived
  { archived_by, reason }

ScenarioRestoredFromArchive
  { restored_by }

ScenarioDeletedSoft
  { deleted_by }
```

### Character and Voice Events

```
Stream: character-{character_id}

CharacterCreated
  { organization_id, name, description, role_label, avatar_url,
    avatar_style, gender, age_range, ethnicity, voice_profile_id, created_by }

CharacterAppearanceUpdated
  { changed_fields }

CharacterVoiceAssigned
  { voice_profile_id }

CharacterDeleted
  { deleted_by }

Stream: voice-profile-{profile_id}

VoiceProfileCreated
  { organization_id, name, provider, provider_voice_id, language,
    gender, style, speed_factor, pitch_factor }

VoiceProfileUpdated
  { changed_fields }

AudioGenerated
  { text_hash, text_content, audio_url, audio_format, duration_ms,
    file_size_bytes }
```

### AI Generation Events

```
Stream: ai-job-{job_id}

AIGenerationRequested
  { scenario_id, job_type, input_prompt, input_context, requested_by }

AIGenerationStarted
  { model_used }

AIGenerationCompleted
  { output_content, token_count_input, token_count_output, cost_usd }

AIGenerationFailed
  { error_message }

AIContentAccepted
  { accepted_by, applied_nodes: [node_id], applied_edges: [edge_id] }

AIContentRejected
  { rejected_by, reason }
```

### Policy Document Events

```
Stream: policy-doc-{document_id}

PolicyDocumentUploaded
  { organization_id, name, document_type, file_url, file_type,
    file_size_bytes, uploaded_by }

PolicyDocumentProcessed
  { extracted_text_length, chunk_count, embedding_model }

PolicyDocumentChunked
  { chunks: [{ chunk_index, content, token_count }] }

PolicyDocumentLinkedToScenario
  { scenario_id, relevance_note, linked_by }

PolicyDocumentUnlinkedFromScenario
  { scenario_id, unlinked_by }

PolicyDocumentDeleted
  { deleted_by }
```

### Collaboration Events

```
Stream: scenario-{scenario_id} (same stream as authoring events)

CommentAdded
  { comment_id, node_id?, edge_id?, parent_comment_id?,
    author_id, body }

CommentEdited
  { comment_id, old_body, new_body }

CommentResolved
  { comment_id, resolved_by }

CommentReopened
  { comment_id, reopened_by }

CommentDeleted
  { comment_id, deleted_by }
```

### Learner Experience Events

```
Stream: attempt-{attempt_id}

AttemptStarted
  { learner_id, scenario_id, version_id, source, lti_launch_id }

NodeEntered
  { node_id, step_number, entered_at }

NodeExited
  { node_id, exited_at, duration_ms }

ChoiceMade
  { node_id, edge_id, step_number, was_correct, score_awarded,
    response_text }

FreeResponseSubmitted
  { node_id, step_number, response_text }

FreeResponseEvaluated
  { node_id, step_number, ai_evaluation, score_awarded, was_correct }

VariableChanged
  { variable_id, old_value, new_value, triggered_by_edge_id }

AttemptCompleted
  { total_score, max_possible_score, score_percentage, passed,
    completion_status, success_status, duration_seconds }

AttemptAbandoned
  { last_node_id, last_step_number, reason }

AttemptTimedOut
  { last_node_id, last_step_number }

AttemptResumed
  { resume_from_node_id, resume_step_number }

AttemptSuspended
  { suspend_data }
```

### xAPI and LMS Events

```
Stream: xapi-{statement_id}

XAPIStatementGenerated
  { statement_id, attempt_id, actor_json, verb_id, verb_display,
    object_json, result_json, context_json, timestamp }

XAPIStatementSentToLRS
  { lrs_config_id, response_status }

XAPIStatementSendFailed
  { lrs_config_id, error_message, retry_count }

Stream: export-{export_id}

ExportRequested
  { scenario_id, version_id, format, requested_by }

ExportBuildStarted
  { }

ExportBuildCompleted
  { package_url, file_size_bytes, manifest_data }

ExportBuildFailed
  { error_message }
```

### Organization and User Events

```
Stream: org-{organization_id}

OrganizationCreated
  { name, slug, plan_tier }

OrganizationSettingsUpdated
  { changed_fields }

UserInvited
  { user_id, email, role_ids, invited_by }

UserJoined
  { user_id, email, display_name, auth_provider }

UserRoleGranted
  { user_id, role_id, granted_by }

UserRoleRevoked
  { user_id, role_id, revoked_by }

UserDeactivated
  { user_id, deactivated_by, reason }

UserReactivated
  { user_id, reactivated_by }

LTIRegistrationCreated
  { registration_id, platform_name, issuer, client_id }

LTIRegistrationUpdated
  { registration_id, changed_fields }

LRSConfigurationCreated
  { config_id, name, endpoint_url, auth_type }

LRSConfigurationUpdated
  { config_id, changed_fields }
```

---

## Read Model Projections

### Projection 1: Scenario Editor View

This projection builds the current state of a scenario's node graph for the visual editor.

```sql
-- ============================================================
-- READ MODEL: scenario_editor_view
-- ============================================================
CREATE TABLE rm_scenarios (
    id                  UUID PRIMARY KEY,
    project_id          UUID NOT NULL,
    organization_id     UUID NOT NULL,
    name                VARCHAR(255) NOT NULL,
    description         TEXT,
    training_objective  TEXT,
    status              VARCHAR(50) NOT NULL,
    version             INTEGER NOT NULL,
    locale              VARCHAR(10) NOT NULL,
    passing_score       DECIMAL(5,2),
    max_score           DECIMAL(8,2),
    difficulty_level    VARCHAR(20),
    estimated_duration_minutes INTEGER,
    is_template         BOOLEAN NOT NULL DEFAULT false,
    created_by          UUID NOT NULL,
    published_at        TIMESTAMPTZ,
    node_count          INTEGER NOT NULL DEFAULT 0,
    edge_count          INTEGER NOT NULL DEFAULT 0,
    last_event_position BIGINT NOT NULL,
    updated_at          TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_scenarios_org ON rm_scenarios(organization_id);
CREATE INDEX idx_rm_scenarios_project ON rm_scenarios(project_id);
CREATE INDEX idx_rm_scenarios_status ON rm_scenarios(status);

CREATE TABLE rm_scenario_nodes (
    id              UUID PRIMARY KEY,
    scenario_id     UUID NOT NULL REFERENCES rm_scenarios(id) ON DELETE CASCADE,
    node_type       VARCHAR(50) NOT NULL,
    title           VARCHAR(255),
    position_x      FLOAT NOT NULL,
    position_y      FLOAT NOT NULL,
    character_id    UUID,
    dialogue_text   TEXT,
    narrator_text   TEXT,
    media_url       TEXT,
    media_type      VARCHAR(50),
    feedback_text   TEXT,
    is_entry_point  BOOLEAN NOT NULL DEFAULT false,
    is_terminal     BOOLEAN NOT NULL DEFAULT false,
    score_value     DECIMAL(8,2) DEFAULT 0,
    time_limit_seconds INTEGER,
    sort_order      INTEGER NOT NULL DEFAULT 0
);

CREATE INDEX idx_rm_nodes_scenario ON rm_scenario_nodes(scenario_id);

CREATE TABLE rm_scenario_edges (
    id              UUID PRIMARY KEY,
    scenario_id     UUID NOT NULL REFERENCES rm_scenarios(id) ON DELETE CASCADE,
    source_node_id  UUID NOT NULL,
    target_node_id  UUID NOT NULL,
    edge_type       VARCHAR(50) NOT NULL,
    label           VARCHAR(255),
    sort_order      INTEGER NOT NULL DEFAULT 0,
    score_delta     DECIMAL(8,2) DEFAULT 0,
    is_correct      BOOLEAN,
    feedback_text   TEXT
);

CREATE INDEX idx_rm_edges_scenario ON rm_scenario_edges(scenario_id);
CREATE INDEX idx_rm_edges_source ON rm_scenario_edges(source_node_id);

CREATE TABLE rm_scenario_variables (
    id              UUID PRIMARY KEY,
    scenario_id     UUID NOT NULL REFERENCES rm_scenarios(id) ON DELETE CASCADE,
    name            VARCHAR(100) NOT NULL,
    variable_type   VARCHAR(20) NOT NULL,
    default_value   TEXT,
    description     TEXT
);

CREATE TABLE rm_variable_mutations (
    id              UUID PRIMARY KEY,
    edge_id         UUID,
    node_id         UUID,
    variable_id     UUID NOT NULL,
    operation       VARCHAR(20) NOT NULL,
    value           TEXT NOT NULL
);

CREATE TABLE rm_edge_conditions (
    id              UUID PRIMARY KEY,
    edge_id         UUID NOT NULL,
    variable_id     UUID NOT NULL,
    operator        VARCHAR(20) NOT NULL,
    value           TEXT NOT NULL,
    logical_group   INTEGER NOT NULL DEFAULT 0
);
```

### Projection 2: Learner Player View

Optimized for the runtime delivery of scenarios to learners.

```sql
-- ============================================================
-- READ MODEL: learner_player_view
-- ============================================================
CREATE TABLE rm_published_scenarios (
    scenario_id     UUID NOT NULL,
    version_number  INTEGER NOT NULL,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    locale          VARCHAR(10) NOT NULL,
    passing_score   DECIMAL(5,2),
    max_score       DECIMAL(8,2),
    difficulty_level VARCHAR(20),
    estimated_duration_minutes INTEGER,
    -- Full graph serialized for single-fetch loading
    graph_data      JSONB NOT NULL,
    -- Character data embedded for offline/SCORM use
    characters_data JSONB NOT NULL,
    published_at    TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (scenario_id, version_number)
);

CREATE TABLE rm_active_attempts (
    id              UUID PRIMARY KEY,
    learner_id      UUID NOT NULL,
    scenario_id     UUID NOT NULL,
    version_number  INTEGER NOT NULL,
    current_node_id UUID,
    current_step    INTEGER NOT NULL DEFAULT 0,
    variable_state  JSONB NOT NULL DEFAULT '{}',
    running_score   DECIMAL(8,2) NOT NULL DEFAULT 0,
    status          VARCHAR(30) NOT NULL,
    started_at      TIMESTAMPTZ NOT NULL,
    last_activity_at TIMESTAMPTZ NOT NULL,
    path_taken      UUID[] NOT NULL DEFAULT '{}',
    suspend_data    TEXT
);

CREATE INDEX idx_rm_attempts_learner ON rm_active_attempts(learner_id);
CREATE INDEX idx_rm_attempts_scenario ON rm_active_attempts(scenario_id);
CREATE INDEX idx_rm_attempts_status ON rm_active_attempts(status);
```

### Projection 3: Analytics Dashboard

Pre-aggregated analytics optimized for dashboard rendering.

```sql
-- ============================================================
-- READ MODEL: analytics_dashboard
-- ============================================================
CREATE TABLE rm_scenario_stats (
    scenario_id         UUID PRIMARY KEY,
    total_attempts      INTEGER NOT NULL DEFAULT 0,
    completed_attempts  INTEGER NOT NULL DEFAULT 0,
    abandoned_attempts  INTEGER NOT NULL DEFAULT 0,
    average_score       DECIMAL(5,2),
    average_duration_seconds INTEGER,
    pass_rate           DECIMAL(5,2),
    median_score        DECIMAL(5,2),
    score_std_dev       DECIMAL(5,2),
    last_attempt_at     TIMESTAMPTZ,
    updated_at          TIMESTAMPTZ NOT NULL
);

CREATE TABLE rm_node_stats (
    scenario_id         UUID NOT NULL,
    node_id             UUID NOT NULL,
    visit_count         INTEGER NOT NULL DEFAULT 0,
    avg_time_on_node_ms INTEGER,
    correct_choice_rate DECIMAL(5,2),
    drop_off_count      INTEGER NOT NULL DEFAULT 0,
    -- Per-edge selection counts
    edge_selections     JSONB NOT NULL DEFAULT '{}',
    updated_at          TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (scenario_id, node_id)
);

CREATE TABLE rm_path_frequencies (
    scenario_id         UUID NOT NULL,
    path_hash           VARCHAR(64) NOT NULL,  -- SHA-256 of node sequence
    path_nodes          UUID[] NOT NULL,
    frequency           INTEGER NOT NULL DEFAULT 1,
    avg_score           DECIMAL(5,2),
    avg_duration_seconds INTEGER,
    last_traversed_at   TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (scenario_id, path_hash)
);

CREATE TABLE rm_learner_summaries (
    learner_id          UUID NOT NULL,
    scenario_id         UUID NOT NULL,
    total_attempts      INTEGER NOT NULL DEFAULT 0,
    best_score          DECIMAL(5,2),
    latest_score        DECIMAL(5,2),
    best_passed         BOOLEAN,
    total_time_seconds  INTEGER NOT NULL DEFAULT 0,
    first_attempt_at    TIMESTAMPTZ NOT NULL,
    last_attempt_at     TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (learner_id, scenario_id)
);

CREATE INDEX idx_rm_learner_summaries_scenario ON rm_learner_summaries(scenario_id);
```

### Projection 4: Compliance Evidence

Purpose-built read model for regulatory audit reporting.

```sql
-- ============================================================
-- READ MODEL: compliance_evidence
-- ============================================================
CREATE TABLE rm_compliance_completions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL,
    learner_id          UUID NOT NULL,
    learner_email       VARCHAR(320),
    learner_name        VARCHAR(255),
    scenario_id         UUID NOT NULL,
    scenario_name       VARCHAR(255) NOT NULL,
    scenario_category   VARCHAR(100),
    version_number      INTEGER NOT NULL,
    attempt_id          UUID NOT NULL,
    completed_at        TIMESTAMPTZ NOT NULL,
    total_score         DECIMAL(8,2),
    score_percentage    DECIMAL(5,2),
    passed              BOOLEAN NOT NULL,
    duration_seconds    INTEGER,
    path_taken          UUID[],
    -- Denormalized for report generation without joins
    regulation_refs     TEXT[],
    xapi_statement_ids  UUID[]
);

CREATE INDEX idx_rm_compliance_org ON rm_compliance_completions(organization_id);
CREATE INDEX idx_rm_compliance_learner ON rm_compliance_completions(learner_id);
CREATE INDEX idx_rm_compliance_scenario ON rm_compliance_completions(scenario_id);
CREATE INDEX idx_rm_compliance_completed ON rm_compliance_completions(completed_at);
CREATE INDEX idx_rm_compliance_regulation ON rm_compliance_completions USING gin(regulation_refs);

CREATE TABLE rm_compliance_gaps (
    organization_id     UUID NOT NULL,
    learner_id          UUID NOT NULL,
    scenario_id         UUID NOT NULL,
    required_by         DATE,
    status              VARCHAR(30) NOT NULL,
        -- not_started, in_progress, failed, overdue
    last_attempt_at     TIMESTAMPTZ,
    last_score          DECIMAL(5,2),
    PRIMARY KEY (organization_id, learner_id, scenario_id)
);
```

### Projection 5: Collaboration View

```sql
-- ============================================================
-- READ MODEL: collaboration
-- ============================================================
CREATE TABLE rm_comments (
    id              UUID PRIMARY KEY,
    scenario_id     UUID NOT NULL,
    node_id         UUID,
    edge_id         UUID,
    parent_id       UUID,
    author_id       UUID NOT NULL,
    author_name     VARCHAR(255),
    body            TEXT NOT NULL,
    is_resolved     BOOLEAN NOT NULL DEFAULT false,
    resolved_by     UUID,
    reply_count     INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ
);

CREATE INDEX idx_rm_comments_scenario ON rm_comments(scenario_id);
CREATE INDEX idx_rm_comments_node ON rm_comments(node_id);
CREATE INDEX idx_rm_comments_unresolved ON rm_comments(scenario_id)
    WHERE is_resolved = false;

CREATE TABLE rm_review_requests (
    id              UUID PRIMARY KEY,
    scenario_id     UUID NOT NULL,
    scenario_name   VARCHAR(255),
    requested_by    UUID NOT NULL,
    requester_name  VARCHAR(255),
    assigned_to     UUID NOT NULL,
    assignee_name   VARCHAR(255),
    status          VARCHAR(30) NOT NULL,
    review_notes    TEXT,
    requested_at    TIMESTAMPTZ NOT NULL,
    completed_at    TIMESTAMPTZ
);

CREATE INDEX idx_rm_reviews_assignee ON rm_review_requests(assigned_to, status);
```

### Projection 6: xAPI Statement Outbox

```sql
-- ============================================================
-- READ MODEL: xAPI outbox (transactional outbox pattern)
-- ============================================================
CREATE TABLE rm_xapi_outbox (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    statement_json  JSONB NOT NULL,
    lrs_config_id   UUID NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    retry_count     INTEGER NOT NULL DEFAULT 0,
    last_error      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    sent_at         TIMESTAMPTZ
);

CREATE INDEX idx_rm_xapi_outbox_pending ON rm_xapi_outbox(status, created_at)
    WHERE status = 'pending';
```

---

## Command Handlers (Pseudocode)

```typescript
// ============================================================
// Example: Adding a node to a scenario
// ============================================================
class AddNodeToScenarioHandler {
  async handle(command: AddNodeToScenario): Promise<void> {
    // 1. Load aggregate from event stream
    const scenario = await this.repository.load(command.scenarioId);

    // 2. Validate business rules
    if (scenario.status === 'published') {
      throw new Error('Cannot modify a published scenario');
    }
    if (command.isEntryPoint && scenario.hasEntryPoint()) {
      throw new Error('Scenario already has an entry point');
    }
    if (command.characterId && !await this.characterExists(command.characterId)) {
      throw new Error('Character not found');
    }

    // 3. Apply the command (produces events)
    scenario.addNode({
      nodeId: generateUUID(),
      nodeType: command.nodeType,
      title: command.title,
      positionX: command.positionX,
      positionY: command.positionY,
      characterId: command.characterId,
      dialogueText: command.dialogueText,
      isEntryPoint: command.isEntryPoint,
      isTerminal: command.isTerminal,
      scoreValue: command.scoreValue,
    });

    // 4. Save (appends events to stream with optimistic concurrency)
    await this.repository.save(scenario);
    // Events are now in the store; projections will pick them up
  }
}

// ============================================================
// Example: Processing a learner choice
// ============================================================
class MakeChoiceHandler {
  async handle(command: MakeChoice): Promise<void> {
    const attempt = await this.repository.load(command.attemptId);

    if (attempt.status !== 'in_progress') {
      throw new Error('Attempt is not in progress');
    }

    const scenario = await this.scenarioReader.getPublished(
      attempt.scenarioId, attempt.versionNumber
    );

    const currentNode = scenario.getNode(attempt.currentNodeId);
    const edge = currentNode.getEdge(command.edgeId);

    if (!edge) {
      throw new Error('Invalid edge for current node');
    }

    // Check edge conditions against current variable state
    if (edge.hasConditions() && !edge.conditionsMet(attempt.variableState)) {
      throw new Error('Edge conditions not met');
    }

    attempt.makeChoice({
      nodeId: currentNode.id,
      edgeId: edge.id,
      stepNumber: attempt.currentStep + 1,
      wasCorrect: edge.isCorrect,
      scoreAwarded: edge.scoreDelta,
    });

    // Apply variable mutations
    for (const mutation of edge.mutations) {
      attempt.applyVariableMutation(mutation);
    }

    // Check if target node is terminal
    const targetNode = scenario.getNode(edge.targetNodeId);
    if (targetNode.isTerminal) {
      attempt.complete({
        totalScore: attempt.runningScore,
        maxPossibleScore: scenario.maxScore,
        passed: attempt.runningScore >= scenario.passingScore,
      });
    }

    await this.repository.save(attempt);
  }
}
```

---

## Event Processing Pipeline

```
┌──────────────┐
│  Commands    │──> Command Handler ──> Validate ──> Emit Events
└──────────────┘                                        │
                                                        ▼
                                              ┌─────────────────┐
                                              │  Event Store     │
                                              │  (PostgreSQL)    │
                                              └────────┬────────┘
                                                       │
                            ┌──────────────────────────┼──────────────────────┐
                            │                          │                      │
                            ▼                          ▼                      ▼
                   ┌────────────────┐      ┌───────────────────┐   ┌─────────────────┐
                   │ Editor         │      │ Analytics          │   │ xAPI Outbox      │
                   │ Projection     │      │ Projection         │   │ Projection       │
                   └───────┬────────┘      └───────┬───────────┘   └────────┬────────┘
                           │                       │                        │
                           ▼                       ▼                        ▼
                   ┌────────────────┐      ┌───────────────────┐   ┌─────────────────┐
                   │ rm_scenarios   │      │ rm_scenario_stats  │   │ rm_xapi_outbox   │
                   │ rm_nodes       │      │ rm_node_stats      │   │    → LRS         │
                   │ rm_edges       │      │ rm_path_freqs      │   └─────────────────┘
                   └────────────────┘      └───────────────────┘

                   ┌────────────────┐      ┌───────────────────┐
                   │ Compliance     │      │ Collaboration      │
                   │ Projection     │      │ Projection         │
                   └───────┬────────┘      └───────┬───────────┘
                           │                       │
                           ▼                       ▼
                   ┌────────────────┐      ┌───────────────────┐
                   │ rm_compliance  │      │ rm_comments        │
                   │ _completions   │      │ rm_reviews         │
                   └────────────────┘      └───────────────────┘
```

---

## Projection Handlers (Pseudocode)

```typescript
// ============================================================
// Analytics Projection: processes attempt events to build stats
// ============================================================
class AnalyticsProjection {
  async handle(event: DomainEvent): Promise<void> {
    switch (event.type) {
      case 'AttemptCompleted': {
        // Update scenario stats
        await this.db.query(`
          INSERT INTO rm_scenario_stats (scenario_id, total_attempts,
            completed_attempts, average_score, pass_rate, updated_at)
          VALUES ($1, 1, 1, $2, CASE WHEN $3 THEN 100 ELSE 0 END, now())
          ON CONFLICT (scenario_id) DO UPDATE SET
            total_attempts = rm_scenario_stats.total_attempts + 1,
            completed_attempts = rm_scenario_stats.completed_attempts + 1,
            average_score = (
              rm_scenario_stats.average_score * rm_scenario_stats.completed_attempts
              + $2
            ) / (rm_scenario_stats.completed_attempts + 1),
            pass_rate = (
              rm_scenario_stats.pass_rate * rm_scenario_stats.completed_attempts
              + CASE WHEN $3 THEN 100 ELSE 0 END
            ) / (rm_scenario_stats.completed_attempts + 1),
            updated_at = now()
        `, [event.data.scenarioId, event.data.scorePercentage, event.data.passed]);

        // Update path frequency
        const pathHash = sha256(event.data.pathTaken.join(','));
        await this.db.query(`
          INSERT INTO rm_path_frequencies
            (scenario_id, path_hash, path_nodes, frequency, avg_score, last_traversed_at)
          VALUES ($1, $2, $3, 1, $4, now())
          ON CONFLICT (scenario_id, path_hash) DO UPDATE SET
            frequency = rm_path_frequencies.frequency + 1,
            avg_score = (rm_path_frequencies.avg_score * rm_path_frequencies.frequency + $4)
              / (rm_path_frequencies.frequency + 1),
            last_traversed_at = now()
        `, [event.data.scenarioId, pathHash, event.data.pathTaken, event.data.scorePercentage]);
        break;
      }

      case 'ChoiceMade': {
        // Update node-level stats
        await this.db.query(`
          INSERT INTO rm_node_stats (scenario_id, node_id, visit_count,
            edge_selections, updated_at)
          VALUES ($1, $2, 1, jsonb_build_object($3::text, 1), now())
          ON CONFLICT (scenario_id, node_id) DO UPDATE SET
            visit_count = rm_node_stats.visit_count + 1,
            edge_selections = rm_node_stats.edge_selections ||
              jsonb_build_object($3::text,
                COALESCE((rm_node_stats.edge_selections->>$3::text)::int, 0) + 1),
            updated_at = now()
        `, [event.data.scenarioId, event.data.nodeId, event.data.edgeId]);
        break;
      }

      case 'AttemptAbandoned': {
        // Track drop-off point
        await this.db.query(`
          UPDATE rm_node_stats
          SET drop_off_count = drop_off_count + 1, updated_at = now()
          WHERE scenario_id = $1 AND node_id = $2
        `, [event.data.scenarioId, event.data.lastNodeId]);
        break;
      }
    }
  }
}
```

---

## Pros

1. **Complete, immutable audit trail by default.** Every change to every scenario, every learner interaction, every review decision is permanently recorded as an event. Compliance officers can reconstruct the exact state of any scenario at any point in time. This is not a bolted-on audit log -- it is the system's source of truth. For a product targeting regulated industries (healthcare, financial services, government), this is a fundamental architectural advantage.

2. **Version history is free.** The README specifies "version history for scenario content" as a feature requirement. In an event-sourced system, version history is inherent -- replaying events up to any point in time reconstructs that version. Diffing between versions is comparing event sequences. No separate version table or snapshot mechanism is needed (snapshots exist only as a performance optimization, not as a data requirement).

3. **Independent read model optimization.** The visual editor, the learner player, the analytics dashboard, and the compliance report generator all have radically different query patterns. CQRS allows each to have its own read model, optimized for its specific access patterns, without compromising the others. The analytics projection can use pre-aggregated counters; the editor projection can use a fully normalized graph; the player projection can use a denormalized JSONB blob for single-fetch loading.

4. **Natural fit for learner path tracking.** A learner's journey through a branching scenario is literally a sequence of events: NodeEntered, ChoiceMade, FreeResponseSubmitted, AttemptCompleted. Event sourcing models this directly rather than reverse-engineering it from mutable state snapshots.

5. **Collaborative editing conflict resolution.** When multiple authors edit the same scenario, event sourcing provides a clear conflict resolution model: events are appended to the stream with optimistic concurrency control. If two authors modify the same scenario concurrently, one will receive a concurrency error and can retry with the latest state. The event stream also enables real-time collaboration features (broadcasting events to connected WebSocket clients).

6. **Temporal queries for analytics.** "What was the pass rate for this scenario last quarter?" is answered by replaying events in a time window rather than querying snapshot data that may have been overwritten. This is valuable for compliance trend analysis and training effectiveness reporting.

---

## Cons

1. **Significant implementation complexity.** Event sourcing requires building or adopting: an event store, command handlers, aggregate root classes, projection handlers, snapshot management, subscription management, and idempotent event processing. This is substantially more infrastructure than a CRUD application with a relational database. For an early-stage product, this complexity delays time-to-market.

2. **Eventual consistency confuses users.** After an author saves a scenario change, the visual editor read model may not immediately reflect the update. The delay is typically milliseconds, but under load it can be seconds. Users expect immediate feedback when they add a node or move an edge. Mitigating this requires either synchronous projection updates (which negates CQRS scaling benefits) or client-side optimistic updates (which adds front-end complexity).

3. **Event schema evolution is painful.** When the event schema changes (adding a field to ScenarioNodeAdded, renaming a property, changing the structure of a condition), all existing events in the store retain their original schema. The system must handle upcasting (transforming old event formats to new formats during replay) indefinitely. In a rapidly evolving product where the node graph model changes frequently during early development, this version management overhead is substantial.

4. **Debugging is harder.** When a bug causes incorrect state in a read model, diagnosing the issue requires tracing through the event stream to find which event or projection handler produced the wrong state. This is more complex than inspecting a single row in a relational table. The tooling ecosystem for event-sourced debugging is less mature than for relational databases.

5. **Event store growth requires active management.** Every user action produces events. A scenario with 100 nodes and active authoring may accumulate thousands of events. Multiplied across thousands of scenarios and millions of learner attempts, the event store grows rapidly. Snapshot intervals, event archiving policies, and stream compaction strategies must be designed and maintained.

6. **Read model rebuilds are expensive at scale.** If a projection has a bug and needs to be rebuilt, replaying millions of events can take hours. During the rebuild, the affected read model serves stale or incomplete data. For the compliance evidence projection -- which may be required for an active regulatory audit -- this downtime is unacceptable.

7. **Team learning curve.** Most backend engineers are not experienced with event sourcing. Hiring, onboarding, and code review all become harder. The pattern is well-documented but not mainstream in the eLearning industry.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Event Store | PostgreSQL (custom table) for simplicity; EventStoreDB for dedicated event store features |
| Command/Query Bus | MediatR (C#), ts-bus or custom (TypeScript), or Axon Framework (Java) |
| Projection Engine | Custom subscription-based projectors reading from the events table |
| Read Model Database | PostgreSQL for all projections (simplicity); Redis for hot projections (active attempts) |
| Message Broker | PostgreSQL LISTEN/NOTIFY for low-volume; NATS or RabbitMQ for high-volume event distribution |
| Snapshot Storage | Same PostgreSQL event store (snapshots table) |
| Serialization | JSON with schema registry for event versioning |
| Monitoring | OpenTelemetry for tracing event processing latency; Prometheus for projection lag metrics |

---

## Migration and Scaling Considerations

### Bootstrap Phase
- Use PostgreSQL as both event store and read model database to minimize operational overhead.
- Start with synchronous projections (project in the same transaction as event write) for the editor view to avoid eventual consistency UX issues. Other projections can be asynchronous.
- Keep event schemas simple and stable during initial development. Resist premature event granularity -- it is easier to split coarse events later than to merge fine-grained events.

### Growth Phase
- Move to asynchronous projections for all read models as write volume increases.
- Implement snapshotting for scenarios with more than 500 events (aggressive authoring over months).
- Partition the events table by stream category (scenario events, attempt events, organization events) to manage table size.
- Add Redis-backed projections for the active-attempts read model to handle concurrent learner sessions with low latency.

### Scale Phase
- Consider migrating the event store to EventStoreDB for built-in subscriptions, projections, and stream management.
- Archive completed attempt event streams to cold storage (S3-backed) after the compliance retention period, keeping only snapshots in the hot event store.
- Run separate projection instances per organization or per projection type for horizontal scaling.
- Implement event store sharding by organization for large multi-tenant deployments.

### Migration from CRUD
If the project starts with a relational model (Suggestion 1) and later needs event sourcing capabilities, the migration path is:
1. Introduce event emission alongside existing CRUD operations (dual-write).
2. Build projections that produce the same state as the existing tables.
3. Validate projection output against existing tables.
4. Switch reads to projections; switch writes to command handlers.
5. Decommission the old CRUD tables.

This strangler-fig approach allows incremental adoption without a big-bang migration.
