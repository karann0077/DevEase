# requirements.md — AI Developer Assistant (Documentation Helper + Debugging Aids)

## Project overview
A web-based AI Developer Assistant that:
1. Generates and maintains high-quality documentation (docstrings, file summaries, README, mental-model maps, micro-videos).  
2. Provides debugging aids (stacktrace/log correlation, minimized repro generation, time-travel variable timeline, and sandbox-verified patches with confidence scoring).

**Target users:** students, beginners, professional developers, and engineering teams.

---

## Goals
- Reduce time-to-understand for new contributors (improve onboarding).  
- Reduce mean time-to-repair (MTTR) for runtime failures.  
- Provide safe, auditable AI-assisted code edits (human-in-loop).  
- Ship a demoable MVP for the hackathon with core flows end-to-end.

---

## Primary personas
- **Junior dev / intern:** needs fast mental models and simple docs.  
- **Mid-level dev:** wants concise function/class explanations and test suggestions.  
- **Senior / maintainer:** needs reliable patch candidates with evidence & confidence.  
- **Team lead / manager:** wants metrics (docs coverage, patch success rate).

---

## Scope

### In-scope (MVP)
- GitHub repo connect (OAuth read-only) and repo ZIP upload.  
- Auto-generate docstrings for selected functions.  
- Auto-generate README preview for repository.  
- Explain pane with three depths: ELI5, Developer, Expert.  
- Paste stacktrace → correlated file:line suggestions.  
- Generate minimized failing input (delta minimizer).  
- Propose patch diff and run in an isolated Docker sandbox (targeted single-test run).  
- Basic patch confidence metric (tests pass/fail + linter delta).  
- Human approval required before creating PRs (draft PR creation optional).

### Out-of-scope (post-MVP)
- Continuous, full-scale multi-repo indexing across enterprise orgs.  
- Default on-prem LLM hosting (optional later).  
- Deep CI/CD pipeline integration (stretch goals).

---

## Functional requirements

### FR-001: Repository ingestion
- Accept GitHub OAuth login (read-only scope default).  
- Accept repo zip uploads for offline/demo use.  
- Store repo metadata: repo_id, latest_commit_hash, languages, file list.  

**Acceptance:** user connects repo and sees file list + basic metadata.

---

### FR-002: Semantic index & chunking
- Parse files using AST-aware parsing (Tree-sitter preferred) to extract functions, classes, and logical blocks.  
- Chunk large code regions with sliding-window token overlap.  
- Generate embeddings for chunks and insert into vector DB with metadata (repo_id, file_path, function_name, line_range, commit_id).  
- De-duplicate identical chunks by content-hash.

**Acceptance:** top-10 relevant chunks returned for a representative query.

---

### FR-003: Explain / Docstring generation
- For selected function/file, generate:
  - 1–2 sentence summary.
  - A docstring in requested style (Google / NumPy / RST).
  - 3 succinct mental-model bullets (why, tradeoffs, edge cases).
- Support three explanation depths: **ELI5**, **Developer**, **Expert**.
- Provide preview UI showing diff (docstring inserted) and an "Insert Docstring" action (local patch or PR draft).

**Acceptance:** docstring preview shown; user can accept to insert or create PR.

---

### FR-004: README generator
- Generate a README with sections: Project summary, Setup & Installation, Usage examples, API summary (if applicable), Contributing notes.
- Allow export/copy or commit as a draft.

**Acceptance:** README preview displays with actionable setup steps.

---

### FR-005: Mental-model map
- Auto-generate a node/edge diagram of key components (API, Auth, DB, Workers, Cache) with one-line descriptions per node.
- Each node links to relevant files or directories in the repo.

**Acceptance:** mental-model visualization renders and nodes link to source files.

---

### FR-006: Error correlation & root-cause suggestion
- Accept pasted stacktrace / build log and optional screenshot.  
- Parse stack frames and error messages, map frames to repository files.  
- Retrieve relevant code chunks (RAG) and return candidate file:line(s) with confidence scores and 1-line root cause suggestions.  
- Provide short reproducible-check steps and suggested tests to add.

**Acceptance:** system returns plausible top-3 candidate files for demo stacktrace.

---

### FR-007: Delta minimizer (reproduction shrinker)
- Given a large failing input or test harness, iteratively reduce input size while preserving failure (ddmin-like algorithm).  
- Run reducer in sandbox; cache intermediate results to avoid repeated runs.

**Acceptance:** produces a smaller input that reproduces the failure for demo cases.

---

### FR-008: Patch generation & sandbox verification
- Generate unified diff patch (git-style) based on LLM suggestion + AST-aware edits.  
- Apply patch in ephemeral branch inside sandbox container.  
- Run targeted tests (either specified tests or auto-generated unit tests) and collect results, stdout/stderr, and coverage snippet.  
- Stream logs to UI and show final test verdict.

**Acceptance:** patch applied in sandbox; test run result (pass/fail) and logs visible.

---

### FR-009: Patch Confidence Auditor
- Compute a confidence score using weighted signals:
  - Test pass/fail before/after (strongest signal).
  - Linter/static analyzer delta (errors introduced/resolved).
  - Diff size and files touched (smaller is safer).
  - Presence and validity of evidence citations (file:line) used by LLM.
  - Historical patch success metrics (if available).
- Present High / Medium / Low score plus top evidence pointers (file:line).

**Acceptance:** confidence score computed and shown for each candidate patch.

---

### FR-010: Human-in-loop PR flow
- Require explicit user approval before creating or pushing changes.  
- If user consents and has write-permission, create a branch, commit patch, and open a draft PR with auto-generated title & description and included test results + confidence summary.  
- Store audit trail of action and user consent.

**Acceptance:** draft PR created in GitHub for authorized user.

---

## Non-functional requirements (NFRs)
- **NFR-001 (Latency):** Explain requests under 10s for short snippets (MVP target with hosted LLMs).  
- **NFR-002 (Security):** Automatic secrets redaction before sending content to external LLMs/APIs.  
- **NFR-003 (Isolation):** Sandboxes enforce CPU/memory/time limits and default to no external network egress.  
- **NFR-004 (Scalability):** Support horizontally scalable indexing workers and sandbox job workers.  
- **NFR-005 (Auditability):** Tamper-evident audit logs of user approvals, prompts (redacted), and PR creation.  
- **NFR-006 (Usability):** Core flows (explain → patch → sandbox) must be achievable in ≤ 3 clicks during demo.

---

## Constraints & assumptions
- MVP uses hosted LLM APIs (OpenAI / Anthropic) to accelerate delivery.  
- Vector DB: Pinecone or Weaviate for MVP.  
- Users provide read access to connected repositories.  
- Use prepared lightweight demo repositories to avoid long dependency installs during sandbox runs.

---

## Security & privacy requirements
- Mask credentials, tokens, and long hex strings using regex redaction before sending to external APIs.  
- Show user an exact preview of content that will be sent to LLM and require explicit consent (per-repo / per-analysis).  
- Follow least-privilege OAuth scopes: request `repo:read` by default; request `repo:write` only when user triggers PR creation.  
- Provide an enterprise option for private/on-prem inference and private vector storage (post-MVP).  
- Encrypt sensitive data at rest and in transit (TLS + AES-256 or cloud-provider default).

---

## Data & storage
- PostgreSQL — metadata, users, audit logs.  
- Redis — caching and background job queues.  
- Vector DB — embeddings and semantic index.  
- S3-compatible object storage — sandbox artifacts, logs, test reports.  
- Store only redacted prompt/context snapshots in logs; full unredacted content must never be stored unless user explicitly allows.

---

## Acceptance criteria (MVP-level)
- Connect at least one GitHub repo and generate a README preview.  
- Generate a docstring for a selected function and show a patch preview to the user.  
- Paste a sample stacktrace and return a plausible top candidate file:line for demo test case.  
- Generate a patch and run a sandbox test; show test pass/fail and confidence score.  
- Display “what AI saw” preview and record user consent before external API calls.

---

## Success metrics
- Doc usefulness human rating ≥ 3.5 / 5 on sample functions.  
- Patch sandbox pass rate ≥ 60% on demo dataset.  
- Demonstrable reduction in manual debugging time in recorded demo scenarios.  
- Increase in docs coverage percentage per repository after tool use.

---

## Dependencies
- OpenAI / Anthropic API keys and quota.  
- Vector DB account (Pinecone / Weaviate / Qdrant).  
- GitHub OAuth App registration.  
- Docker host or Kubernetes cluster for sandboxing.

---

## Milestones / roadmap (short)
- **MVP (48–72h hackathon):** Repo connect, explain pane (ELI5/Dev/Expert), docstring generation, README preview, stacktrace correlation, patch generation + single-test sandbox run.  
- **Post-MVP (weeks):** Delta minimizer improvements, time-travel variable viewer, evidence-backed PR drafts, multi-repo knowledge graph, on-prem LLM option.

---

## Risks & mitigations
- **LLM hallucinations:** require evidence pointers, automatically validate referenced file:line existence, and enforce sandbox verification before accepting fixes.  
- **Secrets leakage:** automated redaction + explicit consent; do not store unredacted prompts persistently.  
- **Slow sandbox runs:** mitigate with prebuilt base images, dependency caching, and running targeted tests only during demo.

---

## Appendix — example acceptance tests

### Docstring example
- Given: selected Python function `def add(a, b): return a + b`  
- When: user clicks “Generate Docstring”  
- Then: UI shows Google-style docstring and an “Insert Docstring” preview button.

### Debugging example
- Given: failing pytest that expects `login()` → 401 but returns 200, with stacktrace  
- When: user pastes stacktrace and clicks “Diagnose”  
- Then: system returns candidate `auth.py:142` with confidence ≥ 70% and a suggested patch; user runs patch in sandbox and verifies result.

---

## Notes for maintainers
- Keep this `requirements.md` as the authoritative specification and update it with status markers (e.g., "Implemented: FR-003") as features complete.  
- Link `design.md` for architecture diagrams and `prompts.md` for LLM prompt templates and examples.  
- Record model versions and prompt templates used per output in audit logs for traceability.