# Project Management & Schedule Dependency
Version: v1.0  
Owner: Michael L  
Scope: Brain repo schedule logic + agent (OpenClaw) behavior

---

## 1. What this document defines

- How your project schedule is represented in the repo
- How dependencies are encoded and validated
- How the agent and reasoning layer collaborate to propose updates
- What changes require human approval
- How external events (inspections/permits/emails) map into schedule updates

---

## 2. Files of record

- `brain/03-Schedule.yaml` — schedule + dependencies (source of truth)
- `brain/00-Status.md` — current state narrative (derived from schedule + events)
- `brain/01-Decisions.md` — decision log (reasons for major schedule changes)
- `brain/02-Assumptions.yaml` — schedule assumptions (durations, lead times, constraints)
- `sources/*/*.yaml` — event/source metadata (inspection snapshots, emails, media)

---

## 3. Task model

A task is a node in a dependency graph.

Each task has:
- **ID** (unique, stable)
- **Name**
- **Category** (inspection / permit / procurement / construction / etc.)
- **Status** (pending / scheduled / complete / blocked)
- **Duration** (working days; inspections may be 0)
- **Constraints** (earliest start, hard deadline)
- **Dependencies** (must complete before this task can start)
- **Notes** (human context)

Minimal example (shape only):

- `id`
- `name`
- `category`
- `status`
- `duration_days`
- `dependencies[]`
- `earliest_start` (optional)
- `hard_deadline` (optional)

---

## 4. Dependency rules

### 4.1 Hard dependencies
If `A -> B` then:
- B cannot start until A is complete
- If A slips, B slips (unless B has slack)

### 4.2 No cycles
- The dependency graph must be **acyclic**
- Agent must validate and refuse PRs that introduce cycles

### 4.3 All references must resolve
- Every dependency ID must exist in `03-Schedule.yaml`
- Missing IDs are a schema error

---

## 5. Propagation rules (how schedule changes ripple)

When a task changes (status/date/duration), the agent must:

1. Identify all downstream tasks (transitive dependents)
2. Recompute earliest start dates based on:
   - dependency completion dates
   - constraints (earliest_start, hard_deadline)
3. Compute impact:
   - which tasks shift
   - projected completion date shift
   - critical path change (if applicable)
4. Summarize the chain of impact in the PR description

The agent may propose these changes, but they are always gated by PR review.

---

## 6. Critical path policy

- “Critical path” = longest dependent chain determining completion date
- Agent may calculate and report:
  - current critical path
  - tasks entering/leaving critical path
  - slack estimates (if modeled)
- Agent must not silently reclassify “critical” without showing the impact in PR.

---

## 7. Event-to-task updates

External events are ingested as sources (email, permit snapshot, screenshot, PDF).

Examples:
- Inspection passed -> mark inspection task `complete`
- Inspection failed -> keep inspection task incomplete + insert corrective task(s)
- Permit revision request -> insert review/rework tasks + dependencies
- Vendor lead time change -> adjust procurement task duration/start

Rules:
- Every event-driven schedule change must link back to a `sources/...yaml` record
- Agent must cite which event triggered which task updates

---

## 8. Allowed agent actions (always via PR)

Agent MAY:
- Propose status changes (pending/scheduled/complete/blocked)
- Propose date shifts based on deterministic propagation
- Propose new tasks when justified by events (e.g., correction tasks)
- Propose dependency links when clearly implied (inspection prerequisites)
- Flag conflicts (missing prereqs, deadline violations, orphan tasks)
- Generate impact chain summaries

Agent MUST:
- Validate schema (IDs, cycles, references)
- Provide reasoning notes in PR
- Keep changes minimal and localized to affected nodes

---

## 9. Forbidden agent actions

Agent must NEVER:
- Merge PRs automatically
- Push directly to `main`
- Delete tasks without explicit instruction
- Rewrite historical IDs (IDs are stable)
- Modify hard deadlines without explicit human approval
- Remove source metadata or alter hashes that define artifact identity

---

## 10. Task insertion policy

When a new task is introduced (e.g., rework), the agent must:
- Create a new unique ID (stable)
- Attach it to the graph with explicit dependencies
- Document why it exists (source link)
- Avoid retroactively changing completed history; prefer add-on corrective nodes

---

## 11. Valid status transitions

Allowed:
- `pending -> scheduled -> complete`
- `pending -> blocked`
- `blocked -> pending` (once blocking prereq resolves)

Not allowed (without explicit corrective workflow):
- `complete -> pending` (instead insert a corrective/rework task)

---

## 12. PR requirements for schedule changes

Every schedule PR must include:
- Summary of the trigger (event or request)
- Tasks changed (IDs + names)
- Dependency impact chain (what moved because of what)
- Critical path impact (if any)
- Source reference(s) (`sources/...yaml`)

---

## 13. Conflict detection (agent must block PR if invalid)

Agent must detect and prevent PR creation if:
- Cycles exist
- Missing referenced IDs exist
- Date constraints are contradictory
- Hard deadlines are violated without an explicit human-approved override note
- Orphan tasks exist (optional warning vs hard error, based on policy)

---

## 14. Human authority boundary

Only the human may approve:
- Hard deadline changes
- Major dependency restructuring
- Timeline acceleration (starting earlier than previously planned)
- Any override of deterministic propagation results

Agent proposes. Human approves.

---

## 15. Key invariants

1. Schedule graph is acyclic.
2. All changes go through PR.
3. All schedule updates have a source reference or explicit human instruction.
4. IDs are stable; history is additive (corrections add nodes).
5. Agent never self-approves; human remains the gatekeeper.