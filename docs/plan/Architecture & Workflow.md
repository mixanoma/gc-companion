# `gc-companion` Architecture & Workflow
Version: v1.0  
Owner: Michael L  
Purpose: Define deterministic, secure, scalable knowledge ingestion and automation architecture.

---

# 1. Design Principles

1. Local-first archive (raw artifacts never depend on GitHub)
2. GitHub repo stores structured knowledge + metadata
3. All raw media stored by SHA-256 hash (content-addressed)
4. No freeform scanning of raw storage
5. PR-only updates (no direct pushes to main)
6. Human-in-the-loop approval
7. Deterministic ingestion rules
8. Separation of:
   - Capture
   - Processing
   - Archive
   - Structured Knowledge
   - Automation

---

# 2. High-Level Architecture

iPad Capture
    ↓
iCloud Drive / BrainInbox
    ↓
Mac Sync
    ↓
BrainArchive/agent_inbox
    ↓
OpenClaw (VM)
    ↓
GitHub Repo (PR)
    ↓
Human Review
    ↓
Archive Finalization

---

# 3. Folder Structure

## 3.1 Host Machine (Canonical Archive)

BrainArchive/
  agent_inbox/              # Temporary ingestion folder (scanned by agent)
    media/
    email/
    permits/
    notes/

  raw/                      # Immutable content-addressed storage
    media_store/
      ab/cd/<sha256>.<ext>
    email_store/
    permit_store/

  derived/                  # Optional derived artifacts
    ocr_text/
    thumbnails/

---

## 3.2 GitHub Brain Repo

brain-repo/
  brain/
    00-Status.md
    01-Decisions.md
    02-Assumptions.yaml
    03-Schedule.yaml
    04-Risks.md
    05-Backlog.md

  sources/
    media/
      SRC-<date>-<slug>.yaml
    email/
    permits/

  indexes/
    media_index.yaml

  scripts/
    validate_sources.py
    validate_schedule.py

---

# 4. Media Storage Model (Hybrid + Hash-Based)

## 4.1 Content Addressed Storage (CAS)

All raw files stored by SHA-256:

raw/media_store/8f/93/8f93a7d8c15d5a7b2a0e8e4b9e73f0d4.png

Rules:
- Identity = SHA-256 hash of file bytes
- Sharded by first 2 bytes + next 2 bytes
- Immutable once stored
- No renaming after CAS insertion

---

## 4.2 What Lives in GitHub vs Archive

| Artifact Type | Location |
|---------------|----------|
| Small screenshots (<2MB) | Optional in repo |
| Large photos | Archive only |
| PDFs / plans | Archive only |
| Email attachments | Archive only |
| Structured notes | Repo |
| Metadata | Repo |

Repo never depends on filenames — only hash references.

---

# 5. Ingestion Workflow

## Step 1 — Capture
User captures on iPad:
- Screenshot
- Photo
- PDF
- Note export

Saved to:
iCloud Drive/BrainInbox/

---

## Step 2 — Sync
Mac syncs automatically to:
BrainArchive/agent_inbox/

---

## Step 3 — OpenClaw Processing

Agent performs:

1. Compute SHA-256
2. Detect file type
3. Insert into CAS store (if new)
4. Generate metadata file:
   - source_id
   - archive_path
   - sha256
   - classification
   - extracted facts
5. Update relevant structured files:
   - Schedule.yaml
   - Status.md
   - Decisions.md
6. Update media_index.yaml
7. Open Pull Request

---

## Step 4 — Human Review

User reviews:
- Diff
- Extracted facts
- Schedule changes
- Decision links

Approve or reject PR.

---

## Step 5 — Finalization

If merged:
- File moved from inbox to permanent archive location
- Metadata status set to "processed"
- Inbox cleared

If rejected:
- File remains in inbox or moved to rejected/

---

# 6. Metadata Standard (Source File Example)

```yaml
source_id: SRC-2026-02-16-ROOF-DIAPHRAGM
captured_at: 2026-02-16T18:42:00-08:00
origin: ios_screencap
content:
  sha256: 8f93a7d8...
  ext: png
  cas_path: BrainArchive/raw/media_store/8f/93/8f93a7d8....png
classification:
  category: inspection_issue
  sensitivity: low
extracted:
  summary: >
    Annotated screenshot showing missing shear nailing.
links:
  related_schedule_ids: [M-127]
state:
  status: pending
```

---

# 7. Retrieval Rules (Critical Safety)

The reasoning layer may request raw files only by:

- source_id
- sha256

Agent must:

- Resolve via metadata index
- Verify hash before use
- Provide only requested file or excerpt
- Log access

Agent must not:

- Scan entire archive
- Perform semantic search on raw media without explicit instruction
- Upload bulk folders to LLM

---

# 8. Gmail Ingestion (Label-Based Allow List)

Rules:

- OAuth scope: gmail.readonly
- Only ingest messages with label VIBE/INGEST
- Optional sender whitelist
- Only allow safe attachment types
- Raw .eml + attachments stored in archive
- Repo stores metadata + structured summary only

---

# 9. Inspection Snapshot Workflow

Preferred method:

- Screenshot or PDF snapshot of permit portal
- Ingest as media
- Extract inspection rows
- Diff against previous snapshot
- Update Schedule.yaml
- Open PR

No automated web scraping required.

---

# 10. Security Model

- OpenClaw runs inside Linux VM
- VM has access only to:
  - agent_inbox
  - raw store
  - brain repo clone
- No Full Disk Access
- GitHub bot account scoped to one repo
- Branch protection enabled
- No auto-merge
- All changes via PR

---

# 11. Core Invariants

1. Raw artifacts are immutable.
2. Structured knowledge is versioned via Git.
3. Hash = identity.
4. Metadata = index.
5. PR = gatekeeper.
6. Archive never scanned blindly.
7. Human always approves structural changes.

---

# 12. Mental Model

Library = BrainArchive/raw  
Index = sources/*.yaml  
Published Knowledge = brain/*.md + yaml  
Assistant = OpenClaw  
Ledger = GitHub  
Gatekeeper = You  

---

End of Document