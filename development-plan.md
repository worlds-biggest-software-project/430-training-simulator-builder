# Training Simulator Builder — Phased Development Plan

> Project: 430-training-simulator-builder · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesizes `research.md`, `features.md`, `standards.md`, `README.md`, and `data-model-suggestion-1..4.md`. All core research documents are present.

The product is an AI-native, scenario-based simulation authoring platform: subject-matter experts build branching training simulations (compliance, skills) via a visual node-graph editor with AI-assisted dialogue generation, deploy them via SCORM/xAPI/cmi5/LTI 1.3, deliver them to learners with completion tracking, and analyze learner paths. It supports both self-hosted (data-privacy-constrained orgs) and cloud-hosted deployment.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language (backend + frontend) | **TypeScript** (Node.js 22 LTS) | The product is API-, integration-, and frontend-heavy (visual editor, learner player, LMS export). A single language across the node-graph editor, the runtime engine that must execute identically on server and in exported HTML5 packages, and the API removes a serialization/duplication boundary. The scenario runtime engine compiles to a browser bundle that ships inside SCORM/cmi5 packages — only achievable cleanly if the engine is written in JS/TS. |
| Backend framework | **Fastify 5** + **TypeBox** | Fastify's schema-first design produces an OpenAPI 3.1 document automatically (standards.md requires OpenAPI 3.1 + JSON Schema Draft 2020-12). TypeBox gives JSON Schema and TS types from one definition, so request/response validation and the published spec stay in sync. Faster than Express; first-class plugin system for the LTI/SCORM/xAPI sub-systems. |
| Frontend framework | **React 18 + Vite + TypeScript** | The visual node-graph editor is the core UX. React Flow (xyflow) is the de-facto library for production node-graph editors and drives the authoring canvas. Vite gives fast HMR for editor iteration and produces the standalone learner-player bundle for export packages. |
| Node-graph canvas | **React Flow (@xyflow/react)** | Purpose-built for draggable node/edge editors with custom node types, minimap, and panning — directly maps to `scenario_nodes`/`node_edges`. Avoids hand-rolling SVG canvas logic. |
| Database | **PostgreSQL 16 + pgvector** (hybrid relational/JSONB — data-model-suggestion-3) | The node graph is the product's differentiator and its shape changes constantly during early dev; storing the graph as validated JSONB on `scenarios` avoids a migration per node-type change. Stable/transactional/audit data (users, attempts, LMS configs, xAPI log) stays relational for integrity and analytical SQL. pgvector enables RAG over uploaded policy documents. This avoids the migration churn of pure 3NF (suggestion-1), the read-model complexity of event sourcing (suggestion-2), and the operational cost of a second graph database (suggestion-4). |
| ORM / query | **Drizzle ORM** | Type-safe, SQL-first, lightweight migrations (`drizzle-kit`). Models JSONB columns as typed objects, which the hybrid model relies on. |
| Migrations | **drizzle-kit** | Generated, reviewable SQL migrations; idempotent and backward-compatible for customer-managed self-hosted upgrades. |
| Task queue | **BullMQ on Redis** | AI generation, voice synthesis, export-package builds, and xAPI/LRS delivery are long-running or rate-limited async jobs. BullMQ gives retries, backoff, and concurrency control. Redis doubles as session/cache store and audio-cache metadata. |
| Object storage | **S3-compatible** (AWS S3 / Cloudflare R2 cloud; **MinIO** self-hosted) | Stores SCORM/cmi5 ZIP packages, generated TTS audio, uploaded policy docs, scenario media. Same SDK (`@aws-sdk/client-s3`) targets both modes. |
| LLM provider | **Provider-abstracted; default OpenAI GPT-4o** (Anthropic + Azure OpenAI adapters) | Dialogue generation, branch suggestion, quality review, free-text response evaluation, translation. Abstraction lets self-hosted orgs point at Azure OpenAI or a private endpoint for data residency. |
| TTS provider | **Provider-abstracted; default ElevenLabs** (Azure Speech adapter) | Character voice synthesis (standards.md: ElevenLabs Flash v2.5 ~75ms latency, voice design API). Adapter pattern for orgs that cannot use ElevenLabs. |
| Auth | **OAuth 2.0 / OIDC** (`openid-client`), local password fallback, **LTI 1.3** via `ltijs` patterns | standards.md mandates OAuth 2.0 (RFC 6749), OIDC, JWT (RFC 9068), and LTI 1.3 (built on OIDC). Short-lived access tokens (≤15 min) + refresh rotation. |
| Standards libraries | Custom SCORM/cmi5 packagers; **`@xapi/xapi`** for xAPI; `jose` for JWT/JWKS | No mature MIT SCORM-packager exists; the packager is built in-house (Phase 7) against ADL specs and tested with the SCORM Cloud reference LRS and ADL xAPI conformance suite. |
| Testing | **Vitest** (unit/integration) + **Playwright** (E2E, editor + player + accessibility) | Vitest shares Vite config with the frontend. Playwright drives the React Flow editor and the learner player, and runs `axe-core` checks for WCAG 2.2 AA (standards.md). |
| Code quality | **ESLint + Prettier + `tsc --noEmit`** | Standard TS toolchain; type checking is a Definition-of-Done gate. |
| Package manager / monorepo | **pnpm workspaces + Turborepo** | The runtime engine package is shared between the API (server-side validation/replay) and the exported player bundle. A monorepo with a shared `engine` package is the clean way to guarantee identical runtime behaviour. |
| Containerisation | **Docker + docker-compose** | Self-hosted target ships api + worker + postgres + redis + minio in one compose file. |
| Accessibility | **axe-core + WAI-ARIA 1.3 roles** in the player | standards.md: WCAG 2.2 AA (= ISO/IEC 40500:2025) required for regulated-industry procurement (Section 508, EN 301 549). |

### Project Structure

```
training-simulator-builder/
├── package.json                  # pnpm workspace root
├── pnpm-workspace.yaml
├── turbo.json
├── docker-compose.yml            # api + worker + postgres + redis + minio
├── Dockerfile.api
├── Dockerfile.worker
├── .env.example
├── packages/
│   ├── engine/                   # shared scenario runtime + graph model (server + browser)
│   │   ├── src/
│   │   │   ├── schema.ts         # ScenarioGraph TypeBox schema + types (JSONB shape)
│   │   │   ├── validate.ts       # graph validation (connectivity, cycles, reachability)
│   │   │   ├── runtime.ts        # ScenarioRuntime state machine (traversal, variables, scoring)
│   │   │   ├── variables.ts      # variable mutation + condition evaluation
│   │   │   └── index.ts
│   │   └── test/
│   ├── xapi-profile/             # xAPI verbs/activity types + statement builders
│   │   └── src/index.ts
│   └── config/                   # shared eslint/tsconfig/prettier
├── apps/
│   ├── api/                      # Fastify backend
│   │   ├── src/
│   │   │   ├── server.ts
│   │   │   ├── db/
│   │   │   │   ├── schema.ts      # Drizzle table definitions
│   │   │   │   ├── migrations/
│   │   │   │   └── client.ts
│   │   │   ├── plugins/          # auth, rbac, audit, openapi, errorHandler
│   │   │   ├── modules/
│   │   │   │   ├── orgs/         # organizations + users + roles
│   │   │   │   ├── projects/
│   │   │   │   ├── scenarios/    # CRUD + graph save/validate + versions
│   │   │   │   ├── characters/   # characters + voice profiles
│   │   │   │   ├── ai/           # generation jobs, RAG, providers
│   │   │   │   ├── voice/        # TTS providers + audio cache
│   │   │   │   ├── templates/
│   │   │   │   ├── collaboration/# comments + review workflows
│   │   │   │   ├── delivery/     # hosted learner player API + attempts
│   │   │   │   ├── export/       # SCORM/cmi5/xAPI/LTI packagers
│   │   │   │   ├── lti/          # LTI 1.3 launch + AGS
│   │   │   │   ├── lrs/          # xAPI statement dispatch
│   │   │   │   ├── analytics/
│   │   │   │   └── compliance/   # evidence reports
│   │   │   ├── providers/        # llm/, tts/, storage/ adapter interfaces
│   │   │   └── lib/
│   │   └── test/
│   ├── worker/                   # BullMQ processors
│   │   └── src/
│   │       ├── index.ts
│   │       └── jobs/             # aiGeneration, voiceSynthesis, exportBuild, xapiDispatch, analyticsRollup
│   ├── studio/                   # React authoring app (editor, dashboards)
│   │   └── src/
│   │       ├── editor/           # React Flow canvas + node inspectors
│   │       ├── ai/               # AI generation panels
│   │       ├── analytics/
│   │       └── lib/api-client.ts
│   └── player/                   # standalone learner player (also bundled into exports)
│       └── src/
│           ├── Player.tsx        # consumes engine ScenarioRuntime
│           ├── adapters/         # scorm12, scorm2004, cmi5, hosted, lti
│           └── a11y/
├── packages/templates/          # built-in scenario template JSON
│   ├── workplace-harassment.json
│   ├── workplace-safety.json
│   ├── data-privacy.json
│   └── code-of-conduct.json
└── docs/
    └── openapi.json              # generated
```

---

## Phase 1: Foundation — Monorepo, Database, Auth, OpenAPI

### Purpose
Establish the monorepo, the hybrid PostgreSQL schema, the Fastify server with auto-generated OpenAPI, multi-tenant auth/RBAC, and the audit log. After this phase the system can authenticate users, enforce roles, and persist organizations/projects — the substrate every later feature depends on.

### Tasks

#### 1.1 — Monorepo & tooling scaffold

**What**: Create the pnpm + Turborepo workspace with `engine`, `api`, `worker`, `studio`, `player` packages, shared ESLint/Prettier/tsconfig, and Docker compose.

**Design**:
- `pnpm-workspace.yaml` includes `packages/*` and `apps/*`.
- `turbo.json` pipelines: `build` (depends on `^build`), `test`, `lint`, `typecheck`.
- `docker-compose.yml` services: `postgres:16` (with `pgvector` image `pgvector/pgvector:pg16`), `redis:7`, `minio`, `api`, `worker`. Healthchecks on postgres/redis before api starts.
- `.env.example` with: `DATABASE_URL`, `REDIS_URL`, `S3_ENDPOINT`, `S3_BUCKET`, `S3_ACCESS_KEY`, `S3_SECRET_KEY`, `JWT_SECRET`, `LLM_PROVIDER`, `OPENAI_API_KEY`, `TTS_PROVIDER`, `ELEVENLABS_API_KEY`, `DEPLOYMENT_MODE` (`cloud`|`self_hosted`), `PUBLIC_URL`.

**Testing**:
- `Unit: turbo build` completes with empty stub packages, exit 0.
- `Integration: docker compose up` brings postgres + redis + minio healthy; `api` responds 200 on `GET /health`.
- `Unit: tsc --noEmit` passes across all packages.

#### 1.2 — Database schema (Drizzle, hybrid relational/JSONB)

**What**: Define all relational tables from data-model-suggestion-1, with the node graph stored as validated JSONB on `scenarios` per suggestion-3.

**Design**:
Relational tables (Drizzle): `organizations`, `users`, `roles`, `user_roles`, `projects`, `scenarios`, `scenario_versions`, `characters`, `voice_profiles`, `audio_cache`, `policy_documents`, `document_chunks` (pgvector `vector(1536)`), `ai_generation_jobs`, `scenario_templates`, `comments`, `review_requests`, `export_packages`, `lti_registrations`, `lti_launches`, `lrs_configurations`, `learners`, `learner_attempts`, `attempt_steps`, `xapi_statements`, `compliance_reports`, `audit_log`, `scenario_analytics`, `node_analytics`.

Key deviation from suggestion-1: `scenario_nodes`, `node_edges`, `scenario_variables`, `variable_mutations`, `edge_conditions` are NOT separate tables. Instead `scenarios.graph` is a `jsonb` column holding the full `ScenarioGraph` (defined in Phase 2). `scenario_versions.snapshot_data` stays JSONB. This is the hybrid model's core decision.

```ts
// apps/api/src/db/schema.ts (excerpt)
export const scenarios = pgTable("scenarios", {
  id: uuid("id").primaryKey().defaultRandom(),
  projectId: uuid("project_id").notNull().references(() => projects.id, { onDelete: "cascade" }),
  name: varchar("name", { length: 255 }).notNull(),
  status: varchar("status", { length: 50 }).notNull().default("draft"),
  version: integer("version").notNull().default(1),
  trainingObjective: text("training_objective"),
  passingScore: numeric("passing_score", { precision: 5, scale: 2 }),
  locale: varchar("locale", { length: 10 }).notNull().default("en-US"),
  graph: jsonb("graph").$type<ScenarioGraph>().notNull().default(emptyGraph()),
  isTemplate: boolean("is_template").notNull().default(false),
  createdBy: uuid("created_by").notNull().references(() => users.id),
  publishedAt: timestamp("published_at", { withTimezone: true }),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
  deletedAt: timestamp("deleted_at", { withTimezone: true }),
});
```
- Multi-tenancy: `organization_id` on tenant-rooted tables; a Drizzle query helper `scoped(orgId)` enforces it. PostgreSQL Row-Level Security policies enabled when `DEPLOYMENT_MODE=cloud`.
- All mutable tables carry `created_at`/`updated_at`; soft delete via `deleted_at`.

**Testing**:
- `Integration (real PG): drizzle migrate` against a throwaway DB creates all tables; `\d scenarios` shows `graph jsonb NOT NULL`.
- `Unit: emptyGraph()` returns a graph with one `start` node, validates against the graph schema.
- `Integration: insert scenario with org A, query as org B → 0 rows` (RLS / scoped helper).
- `Integration: delete project cascades to scenarios` (FK ON DELETE CASCADE).

#### 1.3 — Auth, RBAC, and audit plugin

**What**: OIDC + local auth issuing short-lived JWTs, role-based permission checks, and an audit-log interceptor.

**Design**:
- `POST /auth/login` (local: email+password → bcrypt verify → 15-min access JWT + refresh token in httpOnly cookie). `GET /auth/oidc/start` and `/auth/oidc/callback` via `openid-client`.
- JWT claims: `{ sub, org, roles[], exp }`, signed with `jose` (HS256 local, RS256 with JWKS for federation).
- RBAC: roles `admin | author | reviewer | publisher | learner`; permissions are string scopes (e.g. `scenario:write`, `scenario:publish`, `review:approve`). Fastify `preHandler` `requirePermission(scope)` decorator.
- Audit plugin: Fastify `onResponse` hook writes to `audit_log` for mutating routes — `{action, entity_type, entity_id, old_values, new_values, ip_address, user_agent}`.

**Testing**:
- `Unit: valid credentials → JWT with correct sub/org/roles`.
- `Unit: expired JWT → 401`.
- `Integration: author role calls POST /scenarios/:id/publish → 403` (needs `scenario:publish`).
- `Integration: any mutating call writes one audit_log row with old/new values`.
- `Integration (mocked OIDC): callback with valid code → session established`.

#### 1.4 — OpenAPI generation & error handling

**What**: Auto-generate `docs/openapi.json` (OpenAPI 3.1) from TypeBox route schemas; standardized RFC 9457 problem+json errors.

**Design**:
- `@fastify/swagger` configured for OpenAPI 3.1; every route declares TypeBox `schema`.
- Error handler maps `ValidationError` → 400 with `{type, title, status, detail, errors[]}` (problem+json). Unhandled → 500 with correlation id.
- `pnpm gen:openapi` writes `docs/openapi.json`.

**Testing**:
- `Unit: invalid request body → 400 problem+json with field path in errors[]`.
- `Integration: GET /openapi.json validates against OpenAPI 3.1 meta-schema`.
- `Integration: every registered route appears in the generated spec`.

---

## Phase 2: Scenario Graph Model & Runtime Engine

### Purpose
Build the shared `engine` package — the typed `ScenarioGraph` schema, a validator, and a deterministic runtime state machine. This is the heart of the product: it runs identically server-side (validation, scoring) and in the browser (authoring preview + exported player). Everything in later phases consumes it.

### Tasks

#### 2.1 — ScenarioGraph schema & types

**What**: A TypeBox schema defining the JSONB graph shape (nodes, edges, variables) shared across packages.

**Design**:
```ts
// packages/engine/src/schema.ts
export type NodeType = "start" | "dialogue" | "decision_point" | "consequence"
  | "branch_merge" | "debrief" | "end" | "free_response" | "media" | "information" | "delay";

export interface ScenarioNode {
  id: string;                      // nanoid
  type: NodeType;
  title?: string;
  position: { x: number; y: number };
  characterId?: string;
  dialogueText?: string;
  narratorText?: string;
  media?: { url: string; type: "image" | "video" | "audio" };
  feedbackText?: string;
  scoreValue?: number;
  timeLimitSeconds?: number;
  // free_response node:
  rubric?: { criterion: string; weight: number; idealAnswer: string }[];
}

export interface NodeEdge {
  id: string;
  source: string;                  // node id
  target: string;
  type: "choice" | "fallthrough" | "conditional" | "timeout" | "default";
  label?: string;                  // choice text shown to learner
  scoreDelta?: number;
  isCorrect?: boolean | null;
  feedbackText?: string;
  conditions?: EdgeCondition[];    // ANDed within group, ORed across groups
  mutations?: VariableMutation[];  // applied when traversed
  sortOrder: number;
}

export interface ScenarioVariable { name: string; type: "string"|"number"|"boolean"; default: string|number|boolean; }
export interface VariableMutation { variable: string; op: "set"|"increment"|"decrement"|"toggle"; value: string|number|boolean; }
export interface EdgeCondition { variable: string; operator: "eq"|"neq"|"gt"|"gte"|"lt"|"lte"|"contains"; value: string|number|boolean; group: number; }

export interface ScenarioGraph {
  schemaVersion: 1;
  nodes: ScenarioNode[];
  edges: NodeEdge[];
  variables: ScenarioVariable[];
  entryNodeId: string;
}
```
TypeBox `Type.Object` mirrors each interface so the graph is validated on every save.

**Testing**:
- `Unit: well-formed graph → schema validation passes`.
- `Unit: edge referencing missing node id → fails referential check (2.2)`.
- `Unit: node with type "free_response" but no rubric → flagged by validator (2.2)`.

#### 2.2 — Graph validator

**What**: `validateGraph(g): ValidationResult` checking structural integrity.

**Design**:
```ts
export interface ValidationIssue { severity: "error"|"warning"; code: string; nodeId?: string; edgeId?: string; message: string; }
export function validateGraph(g: ScenarioGraph): { ok: boolean; issues: ValidationIssue[] };
```
Checks: exactly one reachable `start`/entry node (error); all edges reference existing nodes (error); no orphan/unreachable nodes (warning); every non-terminal node has ≥1 outgoing edge (error); `end`/`debrief` terminality; `decision_point` has ≥2 choice edges (warning); self-loops rejected; cycle detection reports cycles not gated by a variable condition (warning, since intentional loops are allowed); `free_response` nodes have a rubric (error). Reachability via BFS from `entryNodeId`.

**Testing**:
- `Unit: graph with unreachable node → warning UNREACHABLE_NODE with nodeId`.
- `Unit: decision_point with one outgoing edge → warning`.
- `Unit: edge to deleted node → error DANGLING_EDGE`.
- `Unit: ungated cycle → warning POSSIBLE_INFINITE_LOOP`.
- `Unit: valid harassment-template graph → ok:true, no errors`.

#### 2.3 — Runtime state machine

**What**: `ScenarioRuntime` that drives a learner through the graph, evaluating conditions, applying mutations, accumulating score.

**Design**:
```ts
export interface RuntimeState {
  currentNodeId: string;
  variables: Record<string, string|number|boolean>;
  visited: { nodeId: string; edgeId?: string; enteredAt: number; scoreDelta: number }[];
  score: number;
  status: "in_progress" | "completed";
}
export class ScenarioRuntime {
  constructor(graph: ScenarioGraph, seedVars?: Record<string, unknown>);
  start(): RuntimeState;
  availableChoices(): NodeEdge[];                    // outgoing edges whose conditions pass
  choose(edgeId: string): RuntimeState;              // traverse: apply mutations, score, advance
  submitFreeResponse(text: string, evalResult: FreeResponseEvaluation): RuntimeState;
  isTerminal(): boolean;
  result(): AttemptResult;                           // total score, %, passed, path
}
```
- Condition evaluation: group conditions AND within `group`, OR across groups.
- Deterministic given the same choices — required so server replay matches client play (anti-cheat + analytics).
- `free_response` advancement uses an externally-supplied `FreeResponseEvaluation` (AI scoring happens in Phase 5; the engine just consumes the score so it stays pure/testable).

**Testing**:
- `Unit: linear 3-node path → choose() advances and accumulates scoreDelta`.
- `Unit: conditional edge gated on variable=true, var false → edge not in availableChoices`.
- `Unit: increment mutation on traversal → variable updated in state`.
- `Unit: reaching end node → isTerminal true, result() computes percentage = score/maxScore`.
- `Unit: same choice sequence on two runtimes → identical RuntimeState (determinism)`.

---

## Phase 3: Authoring Backend & Visual Editor

### Purpose
Deliver the core authoring experience: scenario CRUD with graph persistence/validation, versioning, and the React Flow visual editor with custom node types and an inspector panel. After this phase a non-technical author can build and preview a branching scenario by hand — the table-stakes capability of every competitor.

### Tasks

#### 3.1 — Scenario & project CRUD API

**What**: REST endpoints for projects and scenarios, including graph save with server-side validation.

**Design**:
- `POST /projects`, `GET /projects`, `GET/PATCH/DELETE /projects/:id`.
- `POST /projects/:id/scenarios`, `GET /scenarios/:id`, `PATCH /scenarios/:id` (metadata).
- `PUT /scenarios/:id/graph` — body `ScenarioGraph`; runs `validateGraph`; rejects on errors (400 with issues[]), accepts with warnings (returns `warnings[]`). Optimistic concurrency via `If-Match: <version>` header → 409 on mismatch.
- `GET /scenarios/:id/validate` — returns issues without saving.

**Testing**:
- `Integration: PUT graph with dangling edge → 400, issues[] contains DANGLING_EDGE, graph not persisted`.
- `Integration: PUT valid graph with warning → 200, persisted, warnings[] returned`.
- `Integration: PUT with stale If-Match → 409`.
- `Integration: author of org A cannot GET scenario of org B → 404`.

#### 3.2 — Scenario versioning

**What**: Immutable snapshots on publish and on demand.

**Design**:
- `POST /scenarios/:id/versions` body `{changelog}` → writes `scenario_versions{version_number, snapshot_data=graph, changelog, published_by}`, bumps `scenarios.version`.
- `GET /scenarios/:id/versions`, `GET /scenarios/:id/versions/:n` (returns snapshot), `POST /scenarios/:id/versions/:n/restore` (copies snapshot back into `scenarios.graph`).

**Testing**:
- `Unit: creating a version snapshots current graph verbatim`.
- `Integration: edit graph after versioning, restore → graph matches snapshot`.
- `Integration: version_number is monotonic per scenario (unique constraint)`.

#### 3.3 — React Flow visual editor

**What**: The `studio` authoring canvas with custom nodes, edge creation, inspector, and live validation.

**Design**:
- Custom React Flow node components per `NodeType` (distinct icon/colour: decision_point = diamond, dialogue = speech bubble, debrief/end = terminal).
- Drag from a palette to add nodes; connect handles to create `choice` edges; click edge to edit label/score/isCorrect/feedback in the inspector.
- Inspector panel (right) edits the selected node/edge; variable manager (modal) defines `ScenarioVariable[]` and per-edge conditions/mutations.
- Autosave: debounced (1.5s) `PUT /scenarios/:id/graph`; validation badge shows error/warning counts from the response; problem nodes highlighted.
- Minimap + zoom for large graphs (research.md: "dozens of decision points" must stay manageable).

**Testing**:
- `E2E (Playwright): drag two nodes, connect them, set choice label → autosave PUT fires, graph persisted`.
- `E2E: introduce dangling edge by deleting target → validation badge shows 1 error, node highlighted`.
- `E2E: define variable, add condition to an edge → persisted and reloads correctly`.
- `Unit: editor maps React Flow nodes/edges ↔ ScenarioGraph losslessly (round-trip)`.

#### 3.4 — Preview player in studio

**What**: Embed the `player` package to let authors play their scenario inside the editor.

**Design**:
- "Preview" button mounts `<Player graph={currentGraph} adapter="preview" />` in a drawer; uses `ScenarioRuntime` directly (no persistence), shows choices, feedback, running score, and debrief.

**Testing**:
- `E2E: build 3-node scenario, click Preview, make choices → reaches debrief, shows score`.
- `E2E: free_response node in preview shows a text box (AI eval stubbed)`.

---

## Phase 4: Characters, Voice & Compliance Templates

### Purpose
Add the character library, TTS voice synthesis with caching, and the four MVP compliance templates. After this phase scenarios have realistic multi-character dialogue with audio, and authors can start from a policy-shaped template instead of a blank canvas.

### Tasks

#### 4.1 — Character & voice-profile management

**What**: CRUD for `characters` and `voice_profiles`, org-shared library flag.

**Design**:
- `POST/GET/PATCH/DELETE /characters`; `POST/GET /voice-profiles`.
- Character carries appearance fields (`avatar_url`, `avatar_style`, demographics) and a `voice_profile_id`.
- Nodes reference `characterId`; player resolves character name/avatar/voice at runtime.

**Testing**:
- `Integration: create character with voice profile → GET returns joined voice profile`.
- `Integration: library character visible to all org authors; non-library only to creator`.

#### 4.2 — TTS provider abstraction & audio cache

**What**: Pluggable TTS with content-hash caching to avoid re-synthesizing unchanged dialogue.

**Design**:
```ts
interface TtsProvider {
  synthesize(text: string, voice: VoiceProfile): Promise<{ audio: Buffer; format: "mp3"; durationMs: number }>;
  listVoices(): Promise<ProviderVoice[]>;
}
```
- `ElevenLabsProvider` (default), `AzureSpeechProvider`. Selected by `TTS_PROVIDER`.
- `POST /voice/synthesize {nodeId|text, voiceProfileId}` → compute `sha256(text+voiceProfileId+speed+pitch)`; if `audio_cache` hit, return URL; else enqueue `voiceSynthesis` BullMQ job → provider → upload mp3 to S3 → insert `audio_cache` → return URL.

**Testing**:
- `Unit: identical text+voice → same text_hash → second call is cache hit (provider not called)`.
- `Integration (mocked ElevenLabs): synthesize → mp3 uploaded to MinIO, audio_cache row created`.
- `Integration: provider 429 → job retried with backoff`.

#### 4.3 — Compliance scenario templates

**What**: Four built-in templates (workplace harassment, workplace safety, data privacy, code of conduct) as seedable `ScenarioGraph` JSON.

**Design**:
- `packages/templates/*.json` — each a valid `ScenarioGraph` (passes `validateGraph` with no errors) with realistic branching, decision points, scoring, and a debrief.
- Seed loader inserts them into `scenario_templates` (`is_system=true`).
- `GET /templates?category=`, `POST /scenarios/from-template/:templateId` → deep-copies `template_data` into a new scenario, increments `usage_count`.

**Testing**:
- `Unit: each template JSON passes validateGraph with zero errors`.
- `Integration: create scenario from harassment template → new scenario graph equals template_data`.
- `Integration: from-template increments usage_count`.

---

## Phase 5: AI-Assisted Authoring (Differentiator)

### Purpose
Deliver the AI-native advantage: generate policy-grounded dialogue and branches from a brief or uploaded policy document, suggest response options, review dialogue quality, and evaluate free-text learner responses. This is the core differentiator from features.md and the reason the product exists.

### Tasks

#### 5.1 — LLM provider abstraction & generation jobs

**What**: Pluggable LLM client and the `ai_generation_jobs` async pipeline.

**Design**:
```ts
interface LlmProvider {
  complete(opts: { system: string; user: string; json?: boolean; maxTokens?: number }):
    Promise<{ text: string; inputTokens: number; outputTokens: number; model: string }>;
}
```
- `OpenAiProvider` (default GPT-4o), `AnthropicProvider`, `AzureOpenAiProvider`; selected by `LLM_PROVIDER`.
- `POST /scenarios/:id/ai/generate {job_type, prompt, context}` → inserts `ai_generation_jobs(status=pending)`, enqueues `aiGeneration` job. Worker runs the prompt, parses JSON output into graph fragments, records tokens + `cost_usd`, sets `status=completed`. `GET /ai/jobs/:id` polls.

**Testing**:
- `Integration (mocked LLM): generate job → completed with output_content and token counts`.
- `Unit: cost_usd computed from token counts × model price table`.
- `Integration: provider error → job status=failed with error_message`.

#### 5.2 — Policy document ingestion & RAG

**What**: Upload policy docs, extract text, chunk, embed into pgvector for grounded generation.

**Design**:
- `POST /policy-documents` (multipart) → store file in S3, enqueue extraction (pdf/docx/txt) → chunk (~500 tokens, overlap 50) → embed via `text-embedding-3-small` (1536-dim) → insert `document_chunks.embedding`.
- `retrieveContext(scenarioId, query, k=6)`: associate docs via `scenario_policy_documents`, cosine-similarity search (`vector_cosine_ops`) → top-k chunks injected into generation prompts.

**Testing**:
- `Integration: upload PDF → chunks created with embeddings, chunk_count set`.
- `Unit: retrieveContext returns top-k chunks ordered by similarity`.
- `Integration (mocked embeddings): generation prompt includes retrieved policy text`.

#### 5.3 — Dialogue/branch generation & quality review

**What**: Generation prompts that emit graph fragments grounded in policy, plus a quality-review pass.

**Design**:
- System prompt (dialogue_generation): *"You are an instructional designer creating a branching compliance training simulation. Using ONLY the provided policy excerpts, produce realistic, professionally-worded dialogue and decision points. Output JSON matching the ScenarioGraph fragment schema: nodes[], edges[]. Mark policy-correct choices isCorrect:true with positive scoreDelta and cite the policy clause in feedbackText."* User prompt: training objective + retrieved chunks + existing graph context.
- `branch_suggestion`: given a `decision_point` node, propose additional response options (edges) the author may have missed.
- `quality_review`: flags each node/edge as `realistic|unrealistic|policy_inconsistent|legally_risky` with rationale (features.md: "flag unrealistic, legally inaccurate, or policy-inconsistent content"). Returns issues the author accepts/rejects.
- Generated fragments are merged into the editor as **suggestions** the author approves — never auto-published (research.md: human review essential for compliance).

**Testing**:
- `Integration (mocked LLM): dialogue_generation returns valid ScenarioGraph fragment that passes validateGraph when merged`.
- `Unit: generated fragment node ids are remapped to avoid collisions on merge`.
- `Integration: quality_review on a graph with an off-policy line → flags that node`.

#### 5.4 — Free-text response evaluation (open-ended practice)

**What**: Semantic scoring of free-text learner answers against a node rubric (v1.1 conversational practice mode).

**Design**:
```ts
interface FreeResponseEvaluation { score: number; maxScore: number; passed: boolean;
  criterionScores: { criterion: string; awarded: number; rationale: string }[]; feedback: string; }
```
- `POST /delivery/attempts/:id/free-response {nodeId, text}` → LLM scores `text` against `node.rubric` with a structured-output prompt → returns `FreeResponseEvaluation`; runtime consumes it via `submitFreeResponse`. Stored in `attempt_steps.ai_evaluation`.

**Testing**:
- `Integration (mocked LLM): strong answer → high criterionScores, passed:true`.
- `Integration: weak answer → low score, actionable feedback`.
- `Unit: evaluation persisted to attempt_steps.ai_evaluation JSONB`.

---

## Phase 6: Hosted Learner Delivery & Analytics

### Purpose
Let organizations without an LMS deliver scenarios directly, track attempts step-by-step, and view a learner analytics dashboard. This makes the product usable end-to-end without any export, and produces the data that feeds compliance evidence.

### Tasks

#### 6.1 — Hosted delivery API & player adapter

**What**: Public learner endpoints and the hosted `player` adapter, with WCAG 2.2 AA player UI.

**Design**:
- `POST /delivery/scenarios/:id/start {learner ref}` → creates `learners` (if new) + `learner_attempts(status=in_progress)`, returns published `graph` (version snapshot) + `attemptId`.
- `POST /delivery/attempts/:id/step {nodeId, edgeId?, responseText?}` → server-side `ScenarioRuntime` replays the choice (authoritative scoring), writes `attempt_steps`.
- `POST /delivery/attempts/:id/complete` → finalizes score, `score_percentage`, `passed`, `completion_status`/`success_status`.
- Player UI: ARIA roles on choice buttons (`role=radiogroup`), keyboard navigation, captions for audio, `axe-core` clean.

**Testing**:
- `Integration: start→step→step→complete → attempt completed, total_score matches engine result`.
- `Integration: client-reported score ignored; server recomputes from edges (anti-cheat)`.
- `E2E (Playwright + axe): play full scenario via keyboard only, 0 critical a11y violations`.

#### 6.2 — Attempt tracking & resume

**What**: Persist per-step progress and support resume via suspend data.

**Design**:
- Each `step` updates `attempt_steps` and a `suspend_data` blob (current node + variables) for resume.
- `GET /delivery/attempts/:id/resume` → returns runtime state to continue.

**Testing**:
- `Integration: abandon mid-scenario, resume → continues from correct node with correct variables`.

#### 6.3 — Analytics rollup & dashboard

**What**: Aggregate path/decision metrics and render the author dashboard.

**Design**:
- Nightly + on-demand `analyticsRollup` BullMQ job computes `scenario_analytics` (total/completed attempts, avg score, pass rate, most_common_path) and `node_analytics` (visit_count, correct_choice_rate, edge_distribution, drop_off_count).
- `GET /scenarios/:id/analytics` returns both. Dashboard (studio) shows path-frequency heat on the graph, decision-point error rates, completion-time distribution (features.md "decision-point error rates, completion time").

**Testing**:
- `Integration: seed 100 attempts → rollup computes correct pass_rate and edge_distribution`.
- `Unit: most_common_path = modal node-id sequence across completed attempts`.
- `E2E: dashboard renders heatmap; high-error node visually flagged`.

---

## Phase 7: LMS Export — SCORM, xAPI, cmi5

### Purpose
Package scenarios for deployment to any compatible LMS. This is the table-stakes integration that makes the product enterprise-deployable (research.md: Cornerstone, SuccessFactors, Docebo, Moodle). The exported package embeds the `player` bundle running the same `engine`.

### Tasks

#### 7.1 — xAPI profile & statement builder

**What**: Define the simulator's xAPI profile and statement builders (standards.md: choose/publish a profile).

**Design**:
- `packages/xapi-profile`: verbs (`launched`, `progressed`, `answered`, `completed`, `passed`/`failed`), activity types (`scenario`, `decision-point`, `free-response`). Statement builder produces conformant `actor-verb-object` statements with `result` (score scaled 0..1, success, duration ISO-8601) and `context`.
- Statements logged to `xapi_statements` (sync_status=pending) and dispatched in Phase 8.

**Testing**:
- `Unit: built statements validate against xAPI 1.0.3 JSON Schema (Draft 2020-12)`.
- `Fixture: completed-attempt → statement set validates against ADL conformance fixtures`.

#### 7.2 — HTML5 standalone player bundle

**What**: A self-contained player bundle (Vite build of `apps/player`) that runs a graph from a bundled JSON with no network dependency.

**Design**:
- `pnpm build:player` outputs `player.html` + assets reading `scenario.json`, audio, media — all embedded in the package. Player calls a runtime `adapter` (injected per format) for tracking.

**Testing**:
- `E2E: open built player.html with a sample scenario.json → playable offline, debrief reached`.

#### 7.3 — SCORM 1.2 / 2004 packager

**What**: Build SCORM ZIP packages with `imsmanifest.xml` and the JS API adapter.

**Design**:
- `POST /scenarios/:id/export {format: scorm_12|scorm_2004}` → enqueues `exportBuild` job → assembles: player bundle + `scenario.json` + `imsmanifest.xml` (IMS Content Packaging) + a `scorm` adapter that calls `LMSInitialize`/`LMSSetValue(cmi.core.lesson_status|score.raw)`/`LMSCommit`/`LMSFinish` (1.2) or `cmi.completion_status`/`cmi.success_status`/`cmi.score.scaled` (2004) → ZIP to S3 → `export_packages` row.

**Testing**:
- `Unit: generated imsmanifest.xml validates against SCORM XSD`.
- `Integration: built ZIP opened in SCORM Cloud test → registers, reports completion + score`.
- `Unit: passing attempt → lesson_status=passed (1.2) / success_status=passed (2004)`.

#### 7.4 — cmi5 packager

**What**: cmi5 export with `cmi5.xml` and mandatory AU lifecycle statements.

**Design**:
- cmi5 adapter emits `initialized`, `completed`, `passed`/`failed`, `terminated` statements per the cmi5 AU spec; package includes `cmi5.xml` course structure. Reuses the xAPI statement builder.

**Testing**:
- `Unit: cmi5.xml validates against cmi5 XSD`.
- `Fixture: AU lifecycle emits the cmi5-mandated statement sequence in order`.

---

## Phase 8: LTI 1.3, LRS Dispatch & Collaboration

### Purpose
Add native LMS embedding (LTI 1.3 + Assignment & Grade Services), reliable xAPI delivery to external LRSs, and the multi-author collaboration workflow (v1.1). After this phase the product integrates natively with modern LMS platforms and supports team authoring.

### Tasks

#### 8.1 — LRS dispatch pipeline

**What**: Reliable delivery of `xapi_statements` to configured LRSs.

**Design**:
- `xapiDispatch` BullMQ job POSTs pending statements to `lrs_configurations` endpoints (basic or OAuth2 auth, credentials decrypted), marks `sync_status=sent`/`failed` with retry/backoff. `POST /lrs-configurations` to register an LRS.

**Testing**:
- `Integration (mocked LRS): pending statement → POSTed, sync_status=sent`.
- `Integration: LRS 500 → retried, then failed with sync_error after max attempts`.

#### 8.2 — LTI 1.3 launch & Assignment-Grade Services

**What**: Register LTI platforms, accept OIDC launches, post grades back.

**Design**:
- `lti_registrations` store platform issuer/client_id/keysets; JWKS endpoint exposes tool public keys (`jose`).
- `GET /lti/login` (OIDC initiation) → `POST /lti/launch` (validate signed JWT via platform JWKS, create `lti_launches` + `learners` from `sub`) → render player.
- On completion, AGS posts `score` + `activityProgress`/`gradingProgress` to the platform's line-item (LTI-AGS OpenAPI).

**Testing**:
- `Integration (mocked platform): valid launch JWT → learner created, player rendered`.
- `Integration: invalid JWT signature → 401, no launch record`.
- `Integration (mocked AGS): completion → score POSTed to line item`.

#### 8.3 — Collaboration: comments & review workflow

**What**: Threaded comments on nodes/edges and an author→reviewer→publisher review cycle.

**Design**:
- `POST/GET /scenarios/:id/comments` (optional `nodeId`/`edgeId`, threaded via `parent_comment_id`, `is_resolved`).
- `POST /scenarios/:id/reviews {assigned_to}` (status `pending`→`in_review`→`approved`/`rejected`/`revision_requested`); `approved` is a precondition for `publish` (RBAC: only `publisher` publishes).
- Studio: comment pins on the canvas; review status banner.

**Testing**:
- `Integration: reviewer approves → scenario can be published; without approval publish → 409`.
- `Integration: threaded reply resolves under parent; resolve sets resolved_by/at`.
- `E2E: author leaves comment on a node, reviewer replies and resolves`.

---

## Phase 9: Compliance Evidence, Packaging & Hardening

### Purpose
Deliver regulatory evidence reporting, finalize self-hosted packaging, and harden the platform (accessibility audit, security, performance) for production use in regulated industries.

### Tasks

#### 9.1 — Compliance evidence reports

**What**: Generate audit-ready completion/competency evidence linked to regulations.

**Design**:
- `POST /compliance-reports {report_type, regulation_ref, scenario_ids, date_range}` → aggregates `learner_attempts` + `xapi_statements` + `audit_log` into `generated_data`, renders a PDF to S3 (report_type: completion_summary | competency_evidence | regulatory_audit; regulation_ref: OSHA/FINRA/HIPAA/GDPR).

**Testing**:
- `Integration: report over date range → includes only attempts in range with correct pass counts`.
- `Unit: PDF generated with regulation_ref and per-learner completion evidence`.

#### 9.2 — Self-hosted packaging & docs

**What**: Production docker-compose, env validation, idempotent migrations, seed command.

**Design**:
- `docker compose -f docker-compose.yml up` brings the full stack; first-run runs migrations + seeds system templates. Startup validates required env (fails fast with a clear message). Migrations are forward-only and backward-compatible for customer upgrades.

**Testing**:
- `Integration: fresh compose up → migrate + seed → can create org, build scenario, export SCORM`.
- `Unit: missing required env → startup exits non-zero with named variable`.

#### 9.3 — Accessibility, security & performance hardening

**What**: WCAG 2.2 AA audit of the player, security review, and load validation.

**Design**:
- Player passes `axe-core` AA across all node types; full keyboard operability; captions for audio nodes (WCAG 2.2 AA, ARIA 1.3).
- Security: short-lived JWTs + refresh rotation, encrypted LRS credentials, rate limiting on AI/TTS endpoints, S3 signed URLs for media.
- Load: hosted delivery sustains target concurrent learners (research.md: mandatory-training spikes) — analytics on read replica, BullMQ concurrency caps.

**Testing**:
- `E2E (axe): every node type renders with 0 critical/serious violations`.
- `Integration: rate-limited AI endpoint → 429 after threshold`.
- `Load (k6): N concurrent hosted attempts → p95 step latency < target, no errors`.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (monorepo, DB, auth, OpenAPI)   ─── required by everything
    │
Phase 2: Graph Model & Runtime Engine               ─── requires Phase 1
    │
Phase 3: Authoring Backend & Visual Editor          ─── requires Phase 2
    │
    ├── Phase 4: Characters, Voice & Templates       ─── requires Phase 3
    │       │
    ├── Phase 5: AI-Assisted Authoring               ─── requires Phase 3 (RAG uses Phase 1 pgvector); can parallel Phase 4
    │
    └── Phase 6: Hosted Delivery & Analytics         ─── requires Phase 3 (+ Phase 5.4 for free-text mode)
            │
            ├── Phase 7: LMS Export (SCORM/xAPI/cmi5) ─── requires Phase 6 player; can parallel Phase 8
            │
            └── Phase 8: LTI 1.3, LRS, Collaboration  ─── LTI/LRS require Phase 7 xAPI profile; collaboration requires Phase 3 (can parallel)
                    │
Phase 9: Compliance, Packaging & Hardening           ─── requires Phases 6, 7, 8
```

Parallelism:
- Phases 4 and 5 can be developed concurrently once Phase 3 is complete.
- Phase 7 (export) and Phase 8.3 (collaboration) can be developed concurrently; Phase 8.1/8.2 depend on the Phase 7 xAPI profile.

---

## Definition of Done (per phase)

1. All tasks in the phase implemented.
2. All unit and integration tests pass (`pnpm test`); mocked-provider tests run in CI, real-dependency tests marked optional.
3. Linting and formatting pass (`pnpm lint`, Prettier clean).
4. Type checking passes (`pnpm typecheck` / `tsc --noEmit`) across all packages.
5. `docker compose up` builds and starts the stack; `GET /health` is 200.
6. The phase's feature works end-to-end (demonstrated by an E2E or integration test).
7. New config/env options added to `.env.example` and documented.
8. New API endpoints appear in the regenerated `docs/openapi.json` (OpenAPI 3.1).
9. Database changes ship as a reviewed, idempotent, backward-compatible `drizzle-kit` migration.
10. Any learner-facing UI added in the phase passes `axe-core` with no critical/serious violations (WCAG 2.2 AA).
11. Standards-bearing outputs validate against their schema/conformance suite (SCORM XSD, cmi5 XSD, xAPI 1.0.3, ADL conformance fixtures) where applicable.
```