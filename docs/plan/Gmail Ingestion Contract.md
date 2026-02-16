# Gmail Ingestion Contract
Version: v1.0  
Applies To: OpenClaw VM + Brain Repo (“Project Brain”)  
Owner: Michael L  
Goal: Safely ingest selected Gmail threads + attachments into the Brain as **structured knowledge**, without dumping raw email content into GitHub.

---

## 1) Core Principles

1. **Read-only by default**: use Gmail API scope `gmail.readonly`.
2. **Allow-list first**: ingest only messages explicitly labeled (e.g., `VIBE/INGEST`).
3. **Whitelist as a safety net**: optional sender/domain allowlist to prevent accidental ingestion.
4. **Raw stays out of GitHub**: store `.eml` + attachments in the **host archive**, not in the repo (unless explicitly allowed).
5. **Repo stores metadata + summaries**: hashes + archive paths + extracted facts + links into schedule/decisions.
6. **Deterministic retrieval**: re-open raw emails/attachments only by `source_id` or `sha256`, never “search my mailbox.”

---

## 2) Accounts & Auth

### 2.1 Recommended Setup
- Prefer a **dedicated Gmail account** for project communications OR forward/selectively label in your primary Gmail.
- OAuth client credentials stored **inside VM** (not in repo).
- Use only this scope unless you have a strong reason otherwise:
  - ✅ `https://www.googleapis.com/auth/gmail.readonly`

### 2.2 Disallowed Scopes (default)
- ❌ `gmail.modify`
- ❌ `gmail.send`
- ❌ `mail.google.com` (full access)

If you ever need auto-labeling or “mark processed,” that requires `gmail.modify` (higher risk). Recommended approach: keep read-only and do labeling manually.

---

## 3) Allow-List Labels

### 3.1 Required Label
- `VIBE/INGEST` is the canonical allow-list label.

### 3.2 Optional Labels
- `VIBE/HOLD` (explicitly exclude)
- `VIBE/PROCESSED` (human-applied if using read-only)

### 3.3 Hard Rule
Agent may only query:
- `label:VIBE/INGEST`

Agent must not accept free-form “search Gmail for …” prompts.

---

## 4) Whitelist (Optional but Recommended)

Whitelist is an additional filter applied AFTER label allow-list.

### 4.1 Whitelist Types
Preferred:
- Exact addresses: `architect@firm.com`, `inspections@county.gov`

Acceptable:
- Domains: `@sonomacounty.ca.gov`, `@architectfirm.com` (use carefully)

Not recommended:
- Display-name matching

### 4.2 Gating Logic
Ingest if and only if:

1) Message has label `VIBE/INGEST`
AND
2) (`from` is allowlisted OR `to/cc` contains allowlisted)
AND
3) Attachment rules pass (type/size)

---

## 5) What Gets Stored Where

### 5.1 Host Archive (Canonical Raw)
Store in:
- `BrainArchive/raw/email_store/<YYYY>/...`
- `BrainArchive/raw/media_store/...` (CAS by hash) for attachments

Raw artifacts:
- `.eml` (or raw MIME)
- attachments (PDFs/images/spreadsheets)
- optional: extracted OCR text, thumbnails in `BrainArchive/derived/`

### 5.2 GitHub Repo (Structured Knowledge)
Store:
- Metadata YAML records
- Extracted summaries (redacted)
- Links to schedule IDs / decisions / risks
- Attachment hashes and archive paths
- NO full raw email body by default

---

## 6) Attachment Handling Rules

### 6.1 Allowed Extensions (example default)
- `.pdf`, `.png`, `.jpg`, `.jpeg`, `.heic`, `.csv`, `.xlsx`, `.docx`, `.txt`

### 6.2 Blocked by Default
- `.zip`, `.rar`, `.7z`, `.dmg`, `.exe`, `.pkg`, `.js`, `.bat`, `.sh` (unless explicitly allowed)

### 6.3 Size Limits
- Default max attachment size: 25 MB (adjustable)
- Above limit → store raw only, do not process unless explicitly requested

### 6.4 Storage Method
Attachments are stored in CAS:
- Compute SHA-256
- Save as: `raw/media_store/ab/cd/<sha256>.<ext>`
- Record mapping in repo metadata

---

## 7) Redaction & Privacy Rules

### 7.1 Do Not Commit to GitHub
- Full email bodies (by default)
- Signature blocks
- Phone numbers, personal emails, addresses
- Account numbers, invoices with sensitive details
- Anything “sensitivity: high”

### 7.2 Allowed in Repo (Redacted)
- Subject
- Sender domain (and optionally address if non-sensitive)
- Date
- High-level summary
- Extracted structured facts (dates, inspection codes, permit #s)
- Attachment hashes + archive paths

### 7.3 Sensitivity Classification
Metadata must include:
- `sensitivity: low | medium | high`

Suggested policy:
- `high`: financial, identity, legal docs → archive only, metadata only, extra redaction
- `medium`: vendor quotes, contracts → metadata + redacted summary, attachments archive-only
- `low`: permit notifications, inspection scheduling → metadata + summary, attachments as needed (usually archive-only)

---

## 8) Ingestion Workflow (End-to-End)

### Step A — Human Selection
- You label thread/message with `VIBE/INGEST` (manual curation).

### Step B — Agent Fetch
Agent fetches only labeled items:
- Query: `label:VIBE/INGEST`

### Step C — Normalize & Store Raw
- Save raw `.eml` to `BrainArchive/raw/email_store/...`
- For each attachment:
  - compute SHA-256
  - store in CAS media_store
  - record `sha256`, `cas_path`

### Step D — Extract Structured Knowledge
Agent extracts:
- key dates
- action items
- permit/inspection codes
- deadlines
- referenced documents

### Step E — Write Repo Updates
Agent writes:
- `sources/email/<source_id>.yaml`
- updates `brain/00-Status.md` and other relevant files
- opens PR

### Step F — Human Review
You review PR:
- confirm summary correctness
- verify no sensitive content leaked
- merge or reject

### Step G — Post-Merge Handling
If read-only scope: you manually remove `VIBE/INGEST` label and optionally apply `VIBE/PROCESSED`.

If modify scope is enabled (optional): agent can remove label/add processed label (higher risk).

---

## 9) Deterministic Retrieval Rules (Re-open Raw Later)

### 9.1 What the Reasoning Layer May Request
- Re-open email/attachment by `source_id` or `sha256`

### 9.2 What the Agent Must Do
1) Resolve `source_id` → archive path(s)
2) Verify hash integrity
3) Provide only requested excerpt/artifact
4) Log access event (who/what/why/when)

### 9.3 Disallowed
- “Search Gmail for related stuff”
- “Scan mailbox for anything about X”
- “Download all attachments from this person”

---

## 10) Metadata Schema

### 10.1 Repo File Location
- `sources/email/<source_id>.yaml`

### 10.2 Example Record
```yaml
source_id: GMAIL-2026-02-16-001
gmail:
  message_id: "18c6fabc123..."
  thread_id: "18c1aaaa999..."
  label: "VIBE/INGEST"
  subject: "Inspection scheduling update"
  from_address: "inspections@sonomacounty.ca.gov"
  from_domain: "sonomacounty.ca.gov"
  to_domains: ["example.com"]
  date_utc: "2026-02-16T23:12:05Z"

archive:
  eml_path: "BrainArchive/raw/email_store/2026/2026-02-16_GMAIL-2026-02-16-001.eml"
  eml_sha256: "aa11bb22..."

attachments:
  - filename: "inspection_status.pdf"
    ext: "pdf"
    bytes: 512345
    sha256: "8f93a7d8..."
    cas_path: "BrainArchive/raw/media_store/8f/93/8f93a7d8....pdf"

classification:
  category: "permit_inspection"
  sensitivity: "low"

extracted:
  summary: >
    County inspection office confirms next available slot and notes prerequisite completion.
  key_values:
    inspection_code: "127"
    status: "pending"
    next_action: "schedule inspection"
    suggested_due: "2026-02-25"
links:
  related_schedule_ids: ["M-127"]
  related_decision_ids: []
state:
  status: "processed"
  pr_number: 123


## 11) Logging Requirements

Agent must log:

- Gmail fetch events (timestamp, message IDs retrieved, label used)
- Raw email archive writes (eml path + SHA-256)
- Attachment hash calculations and CAS insert events
- Duplicate detection events (when hash already exists)
- PR creation events (branch name, PR number, changed files)
- Raw retrieval events (who/what requested, source_id/sha256, timestamp)

### Recommended Log Files

/var/log/openclaw/
  - gmail_ingestion.log
  - retrieval.log
  - pr_events.log
  - ingestion_integrity.log

### Log Integrity Rules

- Logs are append-only
- No log rewriting
- Log rotation enabled
- Log files excluded from Git repo
- Optional: periodic off-VM log snapshot for audit backup

---

## 12) Safety Defaults

Default configuration must enforce:

- Label allow-list required (`VIBE/INGEST`)
- Gmail OAuth scope limited to `gmail.readonly`
- Attachment type allowlist enforced
- Attachment size limits enforced
- Raw email bodies NOT committed to repo
- Metadata-only commits for emails
- Deterministic raw retrieval only (by source_id or sha256)
- No free-form Gmail search
- No archive folder scanning
- No direct push to main branch
- No auto-merge of PRs
- Human approval required for structural updates

These defaults remain active unless explicitly overridden in a documented policy change.