# Gantt Charts & Critical Path Reminders
Version: v1.0  
Owner: Michael L  
Scope: Brain Repo + OpenClaw + Notification Layer

---

## 1. Objective

Provide:

- Visual timeline (Gantt-style)
- Critical path visibility
- Proactive reminders for key risk points
- Automated alerts when schedule risk increases
- Human-controlled escalation

This system must remain:

- Deterministic
- PR-driven
- Human-governed
- Low-noise
- Not “nagware”

---

## 2. Gantt Chart Strategy

You have three realistic implementation layers.

---

### Option A — Generated Static Gantt (Recommended Phase 1)

Agent generates:

- `brain/visualizations/gantt.md`
- Mermaid-based Gantt diagram
- Updated via PR

Example structure (conceptually):

- Parse `03-Schedule.yaml`
- Compute start/finish dates
- Render Mermaid Gantt
- Commit diagram file
- Open PR

Pros:
- Fully versioned
- Deterministic
- No external SaaS
- Lives in repo
- Viewable in GitHub

Cons:
- Not interactive

Best for:
- Audit
- Snapshot timeline
- PR-based review

---

### Option B — Local Gantt Dashboard (Phase 2)

Agent generates:

- JSON export
- Small static HTML dashboard
- Local-only visualization served from VM

Pros:
- Interactive
- Not dependent on SaaS
- Still controlled

Cons:
- Slightly more infrastructure

---

### Option C — External Tool Sync (Not Recommended Initially)

Examples:
- Notion
- ClickUp
- MS Project export

Risks:
- Loss of deterministic control
- Sync drift
- Dual source of truth

Avoid until core system is stable.

---

## 3. Critical Path Monitoring

Critical path should be computed by agent from dependency graph.

Agent must calculate:

- Longest dependent chain
- Slack for non-critical tasks
- Tasks entering/exiting critical path
- Total project completion projection

Agent stores:

- Current critical path list
- Completion forecast
- Last change date

---

## 4. Reminder System Layers

You can implement reminders at three levels.

---

### Layer 1 — PR Warnings (Lowest Risk)

When a change impacts:

- Critical path
- Hard deadline
- Delay > X days
- Slack < Y days

Agent includes in PR:

- ⚠ Critical path impact
- ⚠ Hard deadline violation risk
- ⚠ Slack below threshold

No external notifications.
Only visible during review.

---

### Layer 2 — Summary Status File

Agent maintains:

`brain/00-Status.md`

Includes:

- Current completion projection
- Critical path tasks
- Upcoming deadlines (next 14/30 days)
- Blocked tasks
- Unscheduled tasks

You check this in repo or Obsidian.

Low noise.
High clarity.

---

### Layer 3 — Active Notifications (Optional, Controlled)

Agent may trigger reminders when:

- A critical task is due within N days
- A dependency remains blocked > threshold
- A hard deadline approaches
- An inspection must be scheduled

Delivery options:

- Email summary
- iOS calendar event creation
- Reminder app sync
- Slack/Push (if added later)

Important rule:

No reminder without deterministic trigger rule.

---

## 5. Reminder Trigger Rules (Example Policy)

Trigger alert if:

- Hard deadline within 14 days AND task incomplete
- Critical path task slack < 3 days
- Inspection pending > 7 days after eligible
- Procurement lead time exceeds buffer
- Task blocked > 10 days

These thresholds must live in:

`02-Assumptions.yaml`

So they are adjustable.

---

## 6. Human Override Controls

Human may:

- Suppress specific task alerts
- Extend deadline intentionally
- Add buffer manually
- Override critical classification

Agent must log overrides and avoid re-raising same alert unless new condition emerges.

---

## 7. Slack & Buffer Modeling (Optional Advanced Phase)

Add to tasks:

- buffer_days
- float_days
- risk_score

Agent may:

- Compute slack
- Flag tasks with zero slack
- Detect “buffer erosion”

This is advanced but powerful.

---

## 8. Escalation Model

Level 0:
- Silent computation only

Level 1:
- PR warnings only

Level 2:
- Status file summary

Level 3:
- Weekly digest email

Level 4:
- Immediate alert for deadline breach

Start at Level 1 or 2.

Avoid Level 4 unless system proves reliable.

---

## 9. Visualization + Reminder Flow

Event occurs
→ Schedule updated
→ Agent recalculates dependency graph
→ Critical path recomputed
→ Slack evaluated
→ Threshold rules applied
→ PR includes warnings
→ Status.md updated
→ Optional notification triggered

---

## 10. Recommended Rollout Plan

Phase 1:
- Mermaid Gantt generation
- Critical path calculation
- Status.md summary
- PR warnings only

Phase 2:
- Slack computation
- Upcoming deadline list
- Weekly digest email

Phase 3:
- Calendar sync
- Real-time alerts
- Buffer modeling

---

## 11. Design Guardrails

- No duplicate reminders
- No alerts without threshold rule
- No auto-rescheduling without PR
- No infinite alert loops
- Always include reason + impacted tasks

---

## 12. Minimal Implementation Starting Point

Implement first:

- Deterministic Gantt render from YAML
- Critical path list in Status.md
- PR-based warning annotations

Everything else can layer on later.

---

## 13. Core Principle

The schedule is authoritative.
The agent calculates.
The repo records.
The PR gates.
The human decides.
Reminders assist — they do not command.