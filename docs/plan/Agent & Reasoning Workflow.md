# Agent & Reasoning Workflow
Version: v1.0  
Applies To: OpenClaw VM + Brain Repo Architecture  
Owner: Michael L  

---

# 1. Purpose

Define the operational contract between:

- The Reasoning Layer (LLM)
- The Agent Runtime (OpenClaw)
- The GitHub Brain Repo
- The Raw Archive (CAS Store)
- The Human Gatekeeper

This document ensures:

- Deterministic behavior
- Safe automation
- Auditability
- Clear authority boundaries
- No uncontrolled file access

---

# 2. System Roles

## 2.1 Human (Owner)

Authority:
- Approves or rejects Pull Requests
- Controls ingestion labeling (Gmail, inbox)
- Defines policy rules
- Owns schedule authority

Human is the final gatekeeper.

---

## 2.2 Reasoning Layer (LLM)

Capabilities:
- Analyze structured repo content
- Propose changes
- Request raw artifacts (by ID or hash)
- Detect inconsistencies
- Generate structured updates

Limitations:
- Cannot access filesystem directly
- Cannot modify repo directly
- Cannot push or merge
- Cannot browse archive
- Cannot expand search scope beyond provided inputs

LLM proposes. It does not execute.

---

## 2.3 Agent Runtime (OpenClaw VM)

Responsibilities:
- Monitor agent_inbox
- Compute SHA-256 hashes
- Insert artifacts into CAS store
- Generate metadata YAML
- Update structured repo files
- Open Pull Requests
- Enforce policy constraints
- Retrieve raw files only via metadata index
- Log all operations

Limitations:
- No direct push to main branch
- No auto-merge
- No broad filesystem scanning
- No dynamic expansion of Gmail queries
- No override of retrieval constraints

Agent executes but does not govern.

---

## 2.4 GitHub Repo (Ledger)

Purpose:
- Structured knowledge store
- Version history
- Audit log
- Review surface (PRs)

No raw large media stored unless explicitly allowed.

---

## 2.5 Raw Archive (CAS Store)

Purpose:
- Immutable content-addressed storage
- Identity via SHA-256
- Long-term source preservation

Archive is never scanned blindly.
All access must be deterministic via metadata.

---

# 3. Core Workflow Types

There are three major workflow categories:

1. Ingestion Workflow
2. Reasoning / Update Workflow
3. Retrieval Workflow

---

# 4. Ingestion Workflow

## 4.1 Trigger

File appears in:
BrainArchive/agent_inbox/

Or email labeled VIBE/INGEST.

---

## 4.2 Agent Actions

1. Compute SHA-256
2. Detect file type
3. Insert into CAS store (if not duplicate)
4. Create metadata file:
   sources/<type>/<source_id>.yaml
5. Update media_index.yaml
6. Extract structured facts
7. Propose updates to:
   - Schedule.yaml
   - Status.md
   - Decisions.md
8. Open Pull Request

---

## 4.3 Human Review

Human reviews:
- Extracted facts
- Schedule impact
- Decision links
- Metadata correctness

If approved → merge.
If rejected → artifact remains pending or marked rejected.

---

# 5. Reasoning & Update Workflow

## 5.1 Trigger

- Manual prompt
- Scheduled analysis run
- PR validation pass

---

## 5.2 Reasoning Layer Capabilities

LLM may:

- Analyze schedule dependencies
- Detect missing prerequisites
- Propose timeline shifts
- Identify conflicting assumptions
- Suggest new risk entries

LLM outputs structured proposal.

---

## 5.3 Agent Execution Rules

Agent must:

- Validate schema
- Ensure no illegal file modifications
- Confirm no forbidden fields altered
- Stage changes on new branch
- Open PR

Agent must never:

- Modify archive
- Delete historical metadata
- Rewrite source hashes
- Merge automatically

---

# 6. Retrieval Workflow

## 6.1 Allowed Retrieval Requests

LLM may request raw file by:

- source_id
- sha256

Example:
"Re-open SRC-2026-02-16-ROOF-DIAPHRAGM"

---

## 6.2 Agent Retrieval Rules

Agent must:

1. Resolve via metadata index
2. Locate CAS path
3. Verify file hash matches metadata
4. Provide only requested file or excerpt
5. Log retrieval event

Agent must not:

- Scan entire media_store
- Perform semantic search over archive
- Upload multiple unrelated files
- Retrieve by vague query ("find similar images")

Retrieval must be deterministic.

---

# 7. Cross-File Dependency Updates

When schedule change proposed:

Agent must:

1. Identify impacted nodes
2. Apply deterministic propagation rules
3. Annotate PR with:
   - Cause
   - Impact chain
   - Affected milestones

LLM proposes propagation.
Agent validates structural integrity.
Human approves final state.

---

# 8. State Machine

Each source artifact transitions:

captured  
→ pending  
→ parsed  
→ PR_open  
→ merged  
→ archived  
→ indexed  

Rejected artifacts transition to:

pending → rejected

No artifact may skip states.

---

# 9. Policy Enforcement

## 9.1 Never Allowed

- Auto-merge PRs
- Archive scanning without metadata reference
- Deleting source metadata
- Rewriting hash identity
- Expanding Gmail queries beyond label filter
- Writing outside shared directories

---

## 9.2 Always Required

- PR for structural change
- Hash verification before raw retrieval
- Log entries for:
  - CAS insert
  - Raw retrieval
  - PR creation
  - Schedule modification

---

# 10. Logging Requirements

Logs stored inside VM:

/var/log/openclaw/
  ingestion.log
  retrieval.log
  pr_events.log
  schedule_changes.log

Every raw access must be recorded.

---

# 11. Human Authority Boundary

Only human may:

- Merge PR
- Approve schedule timeline shifts
- Change dependency logic
- Update classification policy
- Modify ingestion rules

Agent assists.
LLM reasons.
Human governs.

---

# 12. Failure & Recovery

If VM compromised:
- Destroy VM
- Recreate from ISO
- Re-clone repo
- Archive remains intact
- No structured data lost

VM is disposable.
Archive and repo are authoritative.

---

# 13. Core Invariants

1. Hash = identity.
2. Metadata = index.
3. Archive = immutable.
4. Repo = versioned knowledge.
5. PR = gatekeeper.
6. Human = authority.

---

End of Document