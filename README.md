# user-story

A Claude skill that turns a requirements register into Salesforce user stories
and acceptance criteria, saved as canonical JSON.

Built for Salesforce implementation work: stories use cloud-specific personas,
flag OOB vs custom build, and decompose requirements the way the platform
actually needs them delivered — not just the way they were written.

---

## Repo contents

| File | Purpose |
|---|---|
| `SKILL.md` | The skill itself — trigger rules, the 8-step process, edge cases |
| `references/generate-user-story.md` | Story and AC format rules, two-pass decomposition, worked examples |
| `references/salesforce-personas.md` | Persona tables by cloud — Sales, Service, Experience, Agentforce, Revenue, Marketing |
| `references/story-schema.json` | Canonical JSON schema for the output file |

## Where it sits

```
requirements-extraction  →  user-story  →  sheet-export
   (Google Doc register)     (this skill)    (approved stories only)
                                  ↓
                       canonical JSON in project storage
```

This skill deliberately does **not** create a Google Sheet. Export is a
separate, on-demand step so stories can be reviewed first.

---

## How it works, step by step

### 0. Trigger and scope

```
Generate user stories — epic="Routing"
Project: AcmeCorp
Doc: https://docs.google.com/document/d/.../edit
```

Scope defaults to **one epic per run**. If no epic is stated the skill stops
and asks rather than guessing or generating everything.

| Prompt | Scope |
|---|---|
| *(nothing stated)* | Prompts you to choose an epic |
| `epic="Routing"` | That epic only |
| `reqs="REQ-021, REQ-023"` | Those requirements only |
| `all epics` | The full Phase 1 backlog in one run |
| `include_phase_2` | Adds Phase 2 REQs to whatever scope is set |

Also needed: a **project name** (used in the output filename) and a **cloud
context** (used to select personas). Cloud is inferred from the requirements
if not stated, or marked TBC.

### 1. Read existing state

Checks project storage for an existing file and reads only its `_state` block:

```json
"_state": {
  "last_story_id": "US-020",
  "last_epic_id": "EPIC-001",
  "epics_complete": ["Case Management & SLA"],
  "epics_remaining": ["Routing", "Knowledge Base", "Reporting"]
}
```

This is what lets runs stack: story IDs continue from `last_story_id`, and an
epic already in `epics_complete` is not silently regenerated.

### 2. Read the requirements register

**Project Context is read first**, before any requirement — Client Goal, End
Users, Current Pain, Success Criteria, Constraints. This is a hard gate: with
no Project Context the skill refuses to generate, because the "so that" clause
has nothing real to point at.

Then the REQs in scope are parsed. Phase 1 always; Phase 2 only on request;
Phase TBC included but flagged. Confidence carries from REQ to story — a Low
confidence REQ produces a Low confidence story with an extra validation
scenario and a note that the client must confirm before build.

### 3. Load reference files

`salesforce-personas.md` and `generate-user-story.md`, both before any writing.

### 4. Decompose — two passes

No stories are written yet. A decomposition plan is produced for every REQ
first, and shown before generation starts.

- **Pass 1 — split on the text.** Break the REQ into the distinct user needs it
  actually states. Signals: two personas, an "and" in the action, two
  directions, multiple filter dimensions. Produces siblings, usually
  independent.
- **Pass 2 — expand on implementation depth.** Runs on every need from Pass 1,
  including REQs Pass 1 left whole. Asks what it takes to configure, deliver
  and test this in Salesforce. Produces a sequence with dependencies.

The type is the outcome of the two passes, not a decision made up front:

| Pass 1 | Pass 2 | Type |
|---|---|---|
| no split | no expansion | **A — Atomic** |
| split | no expansion | **B — Compound** |
| no split | expansion | **C — Implementation Depth** |
| split | expansion on ≥1 branch | **B+C** |

Example — a requirement that reads as atomic but is not:

```
REQ-021 | Cases should be routed automatically to the right agent
         based on their skills and current availability.

REQ-021 → Type C → 5 stories (Omni-Channel Routing)
  Story 1: Admin configures Service Channel and presence statuses
  Story 2: Admin defines agent capacity and routing configuration
  Story 3: Admin maps skills to cases and enables skills-based routing
  Story 4: Agent receives, accepts and declines routed work
  Story 5: Service Manager monitors backlog and reassigns stuck work
  Dependency chain: 1 → 2 → 3 → 4, 5
```

**This is the review point.** Correcting a plan is cheap; correcting forty
written stories is not.

### 5. Generate directly to JSON

Each story is reasoned through — persona, action, outcome, Salesforce feature,
build type, AC scenarios, priority, T-shirt size, dependencies, tags — then
written as a complete JSON object in one pass. No prose drafts.

Stories split from one REQ get sub-IDs (`US-021a`, `US-021b`…) all carrying the
same `req_ref`, so traceability back to the requirement survives the split.

### 6. Quality check

A checklist run against the JSON: specific cloud-appropriate personas, "so that"
grounded in Project Context, minimum two AC scenarios with happy path first,
`salesforce.feature` never blank, `build_type` always set, XL stories carrying a
breakdown note, dependencies matching the pass that produced each story, IDs
sequential from `_state`. Issues are fixed in the JSON directly.

### 7. Assemble and save

Written to project storage as `[project]-stories.json` — one canonical file
per project, not per epic, so each run merges into the same place.

- **Epic run** merges into the existing file, preserving earlier epics
- **`all epics`** overwrites
- If the write fails, the full JSON is output in chat rather than lost

`_state` and the summary block are recalculated across the whole file, not just
this run.

### 8. Post-run summary

Run counts (stories, REQs, type breakdown, OOB/Custom/TBC, confidence spread),
the running backlog total, epics complete and remaining, and three attention
lists: XL stories needing breakdown, Pass 2 expansions applied, and Low
confidence stories needing client validation. Ends with a copy-paste command
for the next epic.

---

## Where it stops and asks

By design, rather than guessing:

- No epic scope stated
- No Project Context available
- An epic already marked complete in `_state`

## Output shape

One story, abbreviated:

```json
{
  "id": "US-021d",
  "req_ref": "REQ-021",
  "epic_id": "EPIC-002",
  "title": "As a Service Agent I want to accept or decline work pushed to me so that I keep control of my workload when I am mid-conversation",
  "acceptance_criteria": [
    { "id": "AC-021d-01", "format": "bdd", "given": "...", "when": "...", "then": "..." }
  ],
  "priority": "High",
  "confidence": "High",
  "tshirt_size": "S",
  "status": "draft",
  "salesforce": {
    "build_type": "OOB",
    "feature": "Omni-Channel Widget (Service Console)",
    "notes": "Requires the Service Console app."
  },
  "dependencies": ["US-021c"],
  "tags": ["Service Cloud", "OOB", "Phase 1", "Routing"]
}
```

Full schema in `references/story-schema.json`.
