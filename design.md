# design.md

# DevEase — System Design Document

## 1. Project summary
DevEase AI is a web-based platform that helps developers understand codebases faster and resolve runtime issues with confidence. It combines automatic documentation generation and powerful debugging aids (stacktrace correlation, minimal repro creation, runtime replay, and sandbox-validated patches) into a single workflow. The emphasis is on reproducibility, evidence-backed fixes, and human-in-the-loop safety.

## 2. Objectives and measurable targets
- Produce readable, context-accurate docstrings and README drafts for selected code units.  
- Convert a raw stacktrace + logs into a ranked set of candidate code locations with ≥70% accuracy on demo cases.  
- Generate and verify patches in an isolated environment; aim for a sandbox-pass rate ≥65% on curated examples.  
- Shrink large failing inputs to compact repros automatically (ddmin-like heuristics).  
- Provide auditable, redacted prompts and outputs for compliance.

## 3. System overview (high level)
Users interact through a single-page web UI. Primary flows:
1. Repo ingest & indexing → builds semantic search index and symbol map.  
2. Documentation flow → select code → request explanation/docstring → preview & insert patch.  
3. Debugging flow → paste stacktrace/log/screenshot → ranked candidates → propose patch → run sandbox → present results and confidence.  

Core subsystems:
- Frontend UI
- API layer
- Indexing & retrieval (vector store)
- LLM orchestration (RAG)
- Sandbox execution cluster
- Background workers (queue)
- Persistent stores (metadata, artifacts, embeddings)
- Auditing & observability

## 4. Logical architecture
## System Architecture Diagram

```
[User Browser]
      |
[Frontend (React + Monaco)]
      |
[API Gateway / FastAPI]
      |
+-----+---------------------------+--------------------+
|     |                           |                    |
|  Indexer Worker            LLM Orchestrator      Sandbox Runner
|  (clone, chunk, embed)     (RAG + prompt manager) (apply patch + run tests)
|     |                           |                    |
|  Vector DB -------------------->|                    |
|     |                          DB & Audit            |
+-----+---------------------------+--------------------+
      |
[Postgres (metadata)]   [Redis (cache & queue)]   [S3/MinIO (artifacts)]
```



## 5. Component responsibilities

### 5.1 Frontend
- Provide code explorer with syntax-highlighted viewer (Monaco) and selection tools.  
- Explain pane with tiers (simple → developer → deep) and docstring preview.  
- Debug workspace: paste stacktrace, upload screenshot, show candidate file-lines, timeline control, patch preview and sandbox control.  
- Consent UI showing precisely which text will be sent to an external model (redacted preview).  
- Real-time updates via WebSocket for long-running jobs.

### 5.2 API layer
- Authentication and session management.  
- Expose endpoints for: repo ingest, explain, debug, run-sandbox, create-pr-draft, index status, job streaming.  
- Request validation, rate limiting, and per-org quota enforcement.

### 5.3 Indexer & ingestion
- Clone/sync repository (shallow clones, incremental fetch).  
- Parse source files with language-aware parser, extract logical units (functions, classes, modules).  
- Chunk large units into token-bounded slices with overlap.  
- Deduplicate using content hash.  
- Batch embeddings and write to vector store with metadata (repo, path, commit, line ranges).

### 5.4 Retrieval & RAG orchestrator
- On query, retrieve top-K vectors filtered by repo and language.  
- Construct context package (retrieved chunks, recent commits, linter outputs).  
- Use prompt templates to call model and request evidence pointers (file:line) and structured responses.

### 5.5 LLM prompt manager
- Maintain versioned prompt templates for docstring, explain, patch generation, test generation, and repro minimization.  
- Add instruction patterns to force citation of supporting lines.  
- Log prompt version and sanitized context to audit logs.

### 5.6 Debugging correlator & delta-minimizer
- Parse stacktrace frames and map filenames/line numbers to indexed chunks.  
- Score candidates by semantic similarity, frame proximity, and recent commit relevance.  
- Minimize failing input using iterative reduction (binary/ddmin) while validating reproduction in sandbox.

### 5.7 Patch generator & confidence auditor
- Produce unified diff patches from LLM output.  
- Apply patch inside sandbox branch and run targeted tests.  
- Compute confidence by combining: sandbox pass result, linter delta, diff size, test coverage effect, and evidence quality.

### 5.8 Sandbox runner
- Launch ephemeral container (Docker; k8s Jobs in production).  
- Apply patch, install dependencies in isolated venv, run specified test commands, capture artifacts.  
- Enforce CPU/memory/time limits and restrict external network by default.  
- Stream logs to the frontend and persist results to object storage.

### 5.9 Persistence & cache
- Postgres for users, repos, job metadata, audit logs.  
- Redis for caching embeddings lookups, rate-limits, and as message broker for workers.  
- Vector DB (managed or self-hosted) for embeddings.  
- Object store for logs, test reports, and generated documentation.

### 5.10 Observability & security
- Metrics (Prometheus), dashboards (Grafana), error monitoring (Sentry), structured logs (Loki/ELK).  
- Audit trail for every AI action: prompt version, sanitized context hash, user confirmation id, model id, response id.

## 6. Data models (selected)

### User
- id, email, name, role, language_preference, created_at

### Repo
- id, owner, git_url, default_branch, last_indexed_commit, visibility, indexing_status

### Chunk
- id, repo_id, file_path, start_line, end_line, token_count, hash, created_at

### Embedding metadata
- chunk_id, vector_id, namespace, inserted_at

### Patch
- id, repo_id, author_id, diff_text, created_at, sandbox_job_id, confidence_score, evidence_json

### Job
- id, type, status, progress, started_at, finished_at, artifacts_uri

### Audit
- id, user_id, action_type, redacted_context_id, model_version, timestamp

## 7. API surface (examples)

- `POST /api/repos/connect`  
  Body: `{ "git_url": "...", "auth_token": "..." }` → returns `{ "repo_id", "job_id" }`

- `POST /api/explain`  
  Body: `{ "repo_id", "file_path"?, "code_snippet"?, "detail_level": "simple|dev|deep" }` → returns `{ "summary", "docstring", "evidence": [{path, start_line, end_line}] }`

- `POST /api/debug`  
  Body: `{ "repo_id", "stacktrace", "logs"?, "screenshot_id"? }` → returns `{ "candidates": [{path, line, score}], "suggested_patch": null | diff }`

- `POST /api/run-sandbox`  
  Body: `{ "repo_id", "patch_diff", "commands": ["pytest tests/..."], "timeout_sec": 300 }` → returns `{ "job_id" }` then use WebSocket `/ws/jobs/{job_id}` to stream logs.

- `POST /api/pr-draft` (requires explicit write consent)  
  Body: `{ "repo_id", "branch_base", "patch_diff", "title", "body" }` → returns `{ "pr_url" }`

## 8. Indexing and chunking strategy
- Extract functions and class bodies as primary chunk units.  
- For large bodies, apply sliding-window chunking with overlap to preserve context.  
- Compute sha256 on normalized code text to identify duplicates.  
- Batch size for embeddings tuned to provider limits (e.g., 100–512).  
- Maintain mapping from chunk → commit to enable incremental updates; reindex only changed files on new commit.

## 9. Prompt design (guidelines & templates)
- Use system-role instructions to define output format (JSON, evidence array).  
- Always request precise citations (file:line ranges) for assertions.  
- Limit context token length—include highest relevance retrieved chunks and static analysis summary.  
- Example docstring template (conceptual):  
  - System: "You are an expert software doc writer."  
  - User: "Given this code snippet, return a Google-style docstring and up to 3 evidence lines (path:start-end) that justify the summary."  

Store templates with a version id and include in audit entries.

## 10. Delta minimizer (approach)
- Iteratively remove or shorten parts of failing input then re-run the test until failure no longer reproduces; backtrack to last failing input.  
- Use intelligent heuristics to prioritize likely irrelevant blocks (large sections of input, optional fields, long strings).  
- Parallelize candidate checks across sandbox workers but limit concurrency to control cost.  
- Cache test outcomes for specific input hashes to avoid redundant runs.

## 11. Patch Confidence scoring (formula)
Combine weighted signals into a single score (0–100):
- sandbox_test_pass_weight (e.g., 50%) — full points if targeted tests pass after patch  
- linter_delta_weight (e.g., 15%) — fewer warnings => higher score  
- diff_size_weight (e.g., 10%) — smaller diffs preferred  
- evidence_quality_weight (e.g., 15%) — presence and closeness of cited lines  
- historical_success_weight (e.g., 10%) — model past-patch reliability

Map numeric score to categories: High (≥80), Medium (50–79), Low (<50). Always present underlying signals to the user.

## 12. Security & privacy controls
- Display exact redacted snippet preview before any external model call; require explicit consent button.  
- Apply regex-based masking for tokens, keys, PEM blocks, and long hex strings prior to logging or external transmission.  
- Use minimal OAuth scopes; store tokens encrypted and rotate periodically.  
- Network policy: sandbox containers default to no outbound network; allow exceptions with explicit consent and strict controls.  
- Provide a per-org data retention policy and an API to purge all indexed data.

## 13. Observability & operational metrics
- Track: API latency, LLM call latency & token usage, embedding throughput, indexing job durations, sandbox job durations/success rate, queue lengths.  
- Configure alerts for: worker backlog growth, unexpected error spikes, high token usage, sandbox abuse.  
- Store prompts and sanitized context hashes for troubleshooting and compliance.

## 14. Testing strategy

### Unit tests
- Parser correctness, chunking, deduping logic, prompt template rendering, API validation.

### Integration tests
- Indexing pipeline end-to-end against small sample repos; retrieval → RAG call → expected fields.

### E2E tests
- Frontend flows: connect repo → explain → draft patch → run sandbox (mocked or local).

### Property checks
- Ensure deterministic chunking for identical inputs; ensure prompt versions are recorded.

### Load & performance
- Simulate indexing of large repo (100k+ LOC) for throughput tuning.  
- Measure sandbox concurrency and enforce quotas.

## 15. Deployment & infra
- Dev: docker-compose for local development (Postgres, Redis, MinIO, local vector DB emulator).  
- Staging/Prod: Kubernetes cluster with separate node pools (API, workers, sandbox, optional GPU pool for self-hosted models).  
- IaC: Terraform for cloud resources (VPC, databases, object store, managed vector DB if used).  
- CI: GitHub Actions for tests, lint, build, and deploy pipelines.  
- Secrets: secure store (Vault / cloud KMS).

## 16. Operational constraints & quotas
- Per-org token and sandbox-minute quotas to control costs.  
- Rate limits on API endpoints (configurable by plan).  
- Max sandbox concurrency per org to prevent noisy neighbors.

## 17. Roadmap (phased)
- Phase 0 (MVP): basic ingest, explain via hosted LLM, stacktrace-to-candidate mapping, simple sandbox runner.  
- Phase 1: vector store retrieval & RAG, delta minimizer, evidence enforcement, confidence scoring.  
- Phase 2: time-travel replay visuals, multi-repo impact analysis, PR draft automation.  
- Phase 3: on-prem model hosting, fine-tuning with approved corpora, enterprise compliance features.

## 18. Acceptance criteria (demoable)
- Generate a contextual README for sample repo and show mental-model map.  
- Given a prepared failing stacktrace, identify the correct file:line and produce a patch that passes sandbox tests for the demo repo.  
- Show consent flow and audit entry for an LLM call.  
- Display confidence breakdown for a generated patch.

## 19. Appendix A — example artifacts to include in repo
- `docker-compose.yml` for local services.  
- `backend/` skeleton with FastAPI endpoints stubbed.  
- `indexer/` prototype showing Tree-sitter chunking logic.  
- `prompts/` versioned prompt templates.  
- `docs/sample-repos/` with curated buggy cases and expected fixes.  
- `scripts/demo.sh` to run a recorded fallback demo.

## 20. Appendix B — glossary
- **Chunk**: a contiguous code segment (function/class or sliding window) used for embedding.  
- **RAG**: retrieval-augmented generation — use of vector retrieval to provide context to an LLM.  
- **Sandbox**: isolated environment to apply patches and run tests safely.  
- **Evidence pointer**: a (file_path, start_line, end_line) tuple referencing source lines used by the model to justify output.

---



