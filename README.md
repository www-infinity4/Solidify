# Solidify

**The versioned project database and promotion interface for turning live experiments into tested, explainable releases.**

Solidify receives selected work from Infinity HI-FI, Wave-style collaboration canvases, Omni Gemini agents, StarQuest production tools, and other Infinity projects. It records exactly what was selected, freezes every dependency, coordinates validation, collects approval, and publishes an immutable release manifest.

Solidify does not generate the creative work itself. It is the trusted boundary between **something that worked once** and **a project that can be rebuilt, reviewed, deployed, and rolled back**.

## Promotion model

```text
Experiment
   ↓ select
Candidate
   ↓ build
Verified ──→ Rejected
   ↓ approve      ↑ deterministic failure
Approved ──→ Blocked
   ↓ publish      ↑ missing authority or dependency
Released
   ↓ deploy
Active deployment
```

## Responsibilities

- Stable project, Candidate, build, validation, approval, release, and deployment IDs
- Content-addressed dependencies and assets
- Idempotent build and promotion requests
- Expiring worker leases
- Deterministic validation reports
- Human approval receipts
- Immutable release manifests
- Environment-specific deployments
- Rollback without deleting history
- Provenance across human, agent, code, media, and model inputs
- Rights, visibility, security, and policy gates

## Non-responsibilities

Solidify does not:

- replace Git source control;
- store large audio or video blobs directly in ordinary database rows;
- silently publish a Candidate;
- give agents broader authority than the initiating user;
- remove failed builds or rejected decisions from history;
- treat infrastructure failure as proof that creative work is bad;
- overwrite an active release before validation and approval complete.

## Core records

```text
Project
Candidate
CandidateInput
Dependency
AssetReference
Build
BuildStep
WorkerLease
ValidationRun
ValidationCheck
ValidationReport
ApprovalRequest
ApprovalReceipt
Release
ReleaseManifest
Deployment
Environment
Rollback
AuditEvent
```

## Database schema

The prototype can use SQLite. Production may use PostgreSQL, D1, or another transactional database that preserves the same constraints.

```sql
CREATE TABLE projects (
  project_id TEXT PRIMARY KEY,
  slug TEXT NOT NULL UNIQUE,
  name TEXT NOT NULL,
  description TEXT,
  owner_id TEXT NOT NULL,
  visibility TEXT NOT NULL CHECK (visibility IN ('private','shared','public')),
  active_release_id TEXT,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);

CREATE TABLE candidates (
  candidate_id TEXT PRIMARY KEY,
  project_id TEXT NOT NULL REFERENCES projects(project_id),
  idempotency_key TEXT NOT NULL,
  source_commit TEXT,
  source_snapshot_hash TEXT NOT NULL,
  target_environment TEXT NOT NULL,
  status TEXT NOT NULL CHECK (status IN (
    'queued','claimed','running','retry_wait','blocked',
    'rejected','verified','approved','released','failed','cancelled'
  )),
  created_by TEXT NOT NULL,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL,
  UNIQUE(project_id, idempotency_key)
);

CREATE TABLE candidate_inputs (
  input_id TEXT PRIMARY KEY,
  candidate_id TEXT NOT NULL REFERENCES candidates(candidate_id),
  input_type TEXT NOT NULL,
  source_uri TEXT NOT NULL,
  content_hash TEXT NOT NULL,
  version TEXT,
  rights_state TEXT,
  metadata_json TEXT NOT NULL DEFAULT '{}'
);

CREATE TABLE builds (
  build_id TEXT PRIMARY KEY,
  candidate_id TEXT NOT NULL REFERENCES candidates(candidate_id),
  attempt INTEGER NOT NULL,
  status TEXT NOT NULL,
  started_at TEXT,
  completed_at TEXT,
  output_manifest_hash TEXT,
  error_class TEXT,
  error_message TEXT,
  UNIQUE(candidate_id, attempt)
);

CREATE TABLE worker_leases (
  lease_id TEXT PRIMARY KEY,
  build_id TEXT NOT NULL REFERENCES builds(build_id),
  worker_id TEXT NOT NULL,
  lease_token_hash TEXT NOT NULL,
  expires_at TEXT NOT NULL,
  renewed_at TEXT,
  released_at TEXT
);

CREATE TABLE validation_checks (
  check_id TEXT PRIMARY KEY,
  candidate_id TEXT NOT NULL REFERENCES candidates(candidate_id),
  build_id TEXT REFERENCES builds(build_id),
  check_name TEXT NOT NULL,
  check_version TEXT NOT NULL,
  status TEXT NOT NULL CHECK (status IN ('pending','running','passed','failed','blocked','skipped')),
  severity TEXT NOT NULL CHECK (severity IN ('info','warning','required','critical')),
  report_uri TEXT,
  report_hash TEXT,
  started_at TEXT,
  completed_at TEXT
);

CREATE TABLE approval_receipts (
  approval_id TEXT PRIMARY KEY,
  candidate_id TEXT NOT NULL REFERENCES candidates(candidate_id),
  approver_id TEXT NOT NULL,
  decision TEXT NOT NULL CHECK (decision IN ('approved','rejected','changes_requested')),
  candidate_manifest_hash TEXT NOT NULL,
  comment TEXT,
  decided_at TEXT NOT NULL
);

CREATE TABLE releases (
  release_id TEXT PRIMARY KEY,
  project_id TEXT NOT NULL REFERENCES projects(project_id),
  candidate_id TEXT NOT NULL UNIQUE REFERENCES candidates(candidate_id),
  version TEXT NOT NULL,
  manifest_uri TEXT NOT NULL,
  manifest_hash TEXT NOT NULL UNIQUE,
  created_by TEXT NOT NULL,
  created_at TEXT NOT NULL,
  UNIQUE(project_id, version)
);

CREATE TABLE deployments (
  deployment_id TEXT PRIMARY KEY,
  project_id TEXT NOT NULL REFERENCES projects(project_id),
  release_id TEXT NOT NULL REFERENCES releases(release_id),
  environment TEXT NOT NULL,
  status TEXT NOT NULL CHECK (status IN ('queued','deploying','active','failed','rolled_back','superseded')),
  previous_deployment_id TEXT,
  deployed_by TEXT NOT NULL,
  started_at TEXT NOT NULL,
  completed_at TEXT
);

CREATE TABLE audit_events (
  event_id TEXT PRIMARY KEY,
  project_id TEXT NOT NULL REFERENCES projects(project_id),
  candidate_id TEXT,
  release_id TEXT,
  deployment_id TEXT,
  actor_type TEXT NOT NULL,
  actor_id TEXT NOT NULL,
  action TEXT NOT NULL,
  event_hash TEXT NOT NULL,
  previous_event_hash TEXT,
  details_json TEXT NOT NULL DEFAULT '{}',
  created_at TEXT NOT NULL
);
```

## State rules

- A Candidate cannot become `verified` until every required check passes.
- A Candidate cannot become `approved` without a receipt bound to its exact manifest hash.
- Changing an input after approval creates a new Candidate.
- A release is immutable after creation.
- A deployment points to one release and one environment.
- Rollback creates a new deployment event pointing to a prior valid release.
- `rejected` means a deterministic requirement failed.
- `blocked` means required authority, information, infrastructure, or human action is unavailable.
- Expired leases may be reclaimed; completed build steps remain idempotent.

## Candidate API

```http
POST   /v1/projects
GET    /v1/projects/{projectId}
POST   /v1/projects/{projectId}/candidates
GET    /v1/candidates/{candidateId}
POST   /v1/candidates/{candidateId}/claim
POST   /v1/candidates/{candidateId}/lease/renew
POST   /v1/candidates/{candidateId}/builds
POST   /v1/candidates/{candidateId}/checks
POST   /v1/candidates/{candidateId}/approvals
POST   /v1/candidates/{candidateId}/releases
POST   /v1/releases/{releaseId}/deployments
POST   /v1/deployments/{deploymentId}/rollback
```

Mutation requests carry an idempotency key. Promotion requests also carry the expected Candidate manifest hash so a stale approval cannot publish changed inputs.

## Infinity HI-FI integration

Infinity HI-FI owns live playback sessions, media clocks, reactive audio graphs, working Wave blips, and temporary render jobs. It submits to Solidify:

- selected blip IDs;
- media and source content hashes;
- audio parameters and mix-graph version;
- code commit and runtime contract;
- Omni Gemini agent/model versions;
- rights and visibility state;
- required validation checks;
- target StarQuest preview environment.

Solidify returns Candidate state, validation reports, approval requests, release manifests, deployment state, and rollback history.

## Storage boundaries

| Data | Storage |
| --- | --- |
| Project and promotion state | Solidify transactional database |
| Large audio/video/images | Object storage referenced by URI and hash |
| Source code | Git repository and immutable commit |
| Collaborative working state | Wave/Yjs service |
| Logs and traces | Observability store linked by IDs |
| Release manifest | Immutable object plus indexed database record |

## Security

- Every mutation requires authenticated project authority.
- Worker leases use hashed, expiring tokens.
- Agents may submit results but cannot approve their own release unless explicitly authorized by policy.
- Approval binds to a manifest hash, not merely a Candidate name.
- Object references are validated for permitted scheme, ownership, and content hash.
- Database credentials never enter a blip, prompt, build log, or release manifest.
- Audit events are append-only and hash-linked.
- Publishing and rollback remain explicit authorized actions.

## Prototype plan

1. Create the SQLite schema and migrations.
2. Implement Project and Candidate creation.
3. Add idempotency and Candidate input hashing.
4. Implement build claims and expiring leases.
5. Store validation checks and reports.
6. Bind approval receipts to Candidate manifest hashes.
7. Generate immutable release manifests.
8. Add private preview deployment and rollback.
9. Connect one Infinity HI-FI Candidate end to end.
10. Test duplicate messages, expired workers, blocked dependencies, failed checks, stale approvals, and rollback.

## Acceptance criteria

- Repeating the same Candidate request does not create duplicates.
- A crashed worker's lease expires and work can resume safely.
- Failed required checks prevent promotion.
- Blocked dependencies remain distinguishable from rejected work.
- Approval for one manifest cannot approve a changed manifest.
- Released manifests cannot be edited.
- Rollback does not erase the failed or superseded deployment.
- Infinity HI-FI can submit a mix Candidate and receive a private StarQuest preview release.

## Status

**Database and API architecture specification.** The first implementation should remain small, transactional, auditable, and provider-independent.

## Working principle

> Experiments may be fluid. Releases must be exact.
