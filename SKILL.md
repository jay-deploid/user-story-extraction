---
name: user-story
description: >
  Use this skill whenever the user wants to generate Salesforce user stories
  and acceptance criteria from a requirements register. Accepts either a Google
  Doc URL (output from the requirements-extraction skill) or pasted REQ items.
  Default scope is one epic per run. All-epics is an explicit override.
  Outputs canonical JSON to Claude project storage. The story viewer opens
  automatically after each run. Google Sheet export is a separate on-demand step.

  ALWAYS trigger this skill when the user:
  - Says "generate user stories", "write stories", "create the backlog"
  - Pastes REQ items and asks for stories or AC
  - Provides a requirements register URL and asks for stories
  - Asks to break requirements down into stories for a sprint or epic

  This is a Salesforce implementation context. Stories must reflect Salesforce
  personas, OOB vs custom signals, and cloud-specific patterns.
---

# User Story Skill

Generates Salesforce user stories and acceptance criteria from a requirements
register. Saves canonical JSON to Claude project storage. Epic-by-epic by
default — one focused run per epic for consistent quality.

Do NOT create a Google Sheet during this skill run. Sheet export is a
separate skill triggered on demand after stories have been reviewed.

---

## Before You Start — Determine Scope

### Default behaviour
Epic-by-epic is the default. If no scope is stated, prompt the user:

```
Which epic would you like to generate stories for?
[list epics from the requirements register]

Or say "all epics" to generate the full backlog in one run.
```

### Scope options

| Prompt | What to do |
|---|---|
| Nothing stated | Prompt the user to select an epic — do not default to all |
| `epic="[name]"` | Generate stories for that named epic only |
| `reqs="REQ-001, REQ-004"` | Generate stories for specific REQs only |
| `all epics` | Explicit override — generate every Phase 1 REQ in the register |
| `include_phase_2` | Include Phase 2 REQs in addition to Phase 1 |

### Phase behaviour
- Phase 1 REQs → always included
- Phase 2 REQs → skipped by default. Only included if `include_phase_2` is passed
- Phase TBC REQs → included but flagged in `salesforce.notes`

### Also check for
- **Project name** — needed for filename. Ask if not provided.
- **SF Cloud context** — needed for personas and OOB flags.
  Infer from requirements content if not stated. Flag as TBC if unclear.

---

## Step 1 — Read Existing State

Before reading anything else, check whether a JSON file already exists
in Claude project storage for this project.

### If a file exists
Read the `_state` block **only** — do not load the full stories array:

```json
"_state": {
  "last_story_id": "US-006",
  "last_epic_id": "EPIC-001",
  "last_run_epic": "Case Management & SLA",
  "last_run_date": "2026-06-05",
  "epics_complete": ["Case Management & SLA"],
  "epics_remaining": ["Routing", "Knowledge Base", "Agent Experience", "Reporting"]
}
```

Note the `last_story_id` and `last_epic_id` — new stories continue from these.
Note `epics_complete` — do not regenerate stories for epics already done.

### If no file exists
This is Run 1. Initialise with:
```json
"_state": {
  "last_story_id": "US-000",
  "last_epic_id": "EPIC-000",
  "last_run_epic": null,
  "last_run_date": null,
  "epics_complete": [],
  "epics_remaining": []
}
```

---

## Step 2 — Read the Requirements Register

### Read Project Context first
Before reading any REQs, explicitly read the PROJECT CONTEXT block from
the requirements register. This is the first section of the Google Doc.

Extract and hold in memory:
- **Client Goal** — used to write meaningful "so that" outcomes
- **End Users** — used to validate persona selection
- **Current Pain** — used to frame the problem each story solves
- **Success Criteria** — used to ensure stories point toward measurable outcomes
- **Constraints** — used to flag scope risks in `salesforce.notes`

Do not generate stories without this context. Stories written without
Project Context produce generic "so that" outcomes that don't reflect
the client's actual business goals.

### Input method A: Google Doc URL
1. Use the Google Drive MCP to fetch the document content
2. Read Project Context block first — hold in memory
3. Parse REQ items for the epic in scope with their confidence and phase tags
4. Note FLAGS and AMBs relevant to the epic in scope
5. Apply phase filter — skip Phase 2 unless `include_phase_2` was passed

### Input method B: Pasted REQ items
1. Accept any format — standard REQ format, plain bullets, or numbered list
2. Ask for Project Context if not provided — do not skip this step
3. Infer epic groupings from context if not stated
4. Ask for project name and cloud context if not inferable

### Confidence and phase inheritance
Carry confidence and phase from the source REQ into the story:
- High confidence REQ → story confidence starts at High
- Medium confidence REQ → story confidence starts at Medium
- Low confidence REQ → story confidence starts at Low, add validation
  scenario to AC, note in `salesforce.notes` that requirement needs
  client confirmation before build

---

## Step 3 — Load Reference Files

Read both reference files before proceeding:

1. `references/salesforce-personas.md` — load the persona table for the
   relevant cloud(s). Required for accurate "As a..." lines.

2. `references/generate-user-story.md` — load story format rules,
   decomposition types, and the decomposition plan format. Required
   before Step 4.

---

## Step 4 — Decompose Requirements

Before writing any stories, produce a decomposition plan for every REQ
in scope. Do not write a single story until this plan is complete.

Classify each REQ as one of three types using the rules in
`references/generate-user-story.md`:

**Type A — Atomic**
One persona, one action, one outcome. Passes the atomic test.
Produces 1 story.

**Type B — Compound**
Compound signals visible in the REQ text (multiple personas, actions,
directions, or filter dimensions). Produces 2+ stories.

**Type C — Implementation Depth**
Reads as simple but requires multiple stories to configure and deliver
correctly in Salesforce. Apply Salesforce implementation knowledge —
not text analysis — to identify these. Produces 2+ stories.

### Decomposition plan output format
```
DECOMPOSITION PLAN
══════════════════
REQ-001 → Type A → 1 story
  Reason: [one line]

REQ-012 → Type B → 2 stories
  Reason: [one line]
  Story 1: [brief description]
  Story 2: [brief description]

REQ-019 → Type C → 5 stories (Omni-Channel routing)
  Reason: [one line]
  Story 1–5: [brief descriptions]

Total stories this run: X
══════════════════
```

Output this plan before writing any stories. Proceed only after the
plan is complete.

---

## Step 5 — Generate User Stories

Work through the decomposition plan. For each story in the plan:

### Story format
```
As a [specific Salesforce persona],
I want [to perform a specific action],
so that [I achieve a specific business outcome].
```

### Rules
- Persona — cloud-appropriate, from `references/salesforce-personas.md`
- Action — what the user does, never how it is built
- Outcome — business value grounded in the Project Context.
  Use Client Goal and Current Pain to make the "so that" meaningful,
  not generic. `"so that I can do my job better"` is not acceptable.
- One story = one user need

### Populate `salesforce.feature`
For every story, name the specific Salesforce feature being configured
or built. Use your Salesforce knowledge — do not leave this blank.

Examples:
- Email-to-Case, Omni-Channel Routing, Entitlements & Milestones
- Web-to-Lead, Opportunity Stages, Forecasting Categories
- Scheduling Policy, Work Order Management, FSL Mobile App
- Household Model, Action Plans, Recurring Donations

### Detect dependencies
Where a story in a Type C decomposition must be completed before
another can be built or tested, populate the `dependencies` field
with the preceding story ID.

Example: "Configure presence statuses" must be complete before
"Configure routing rules" — add the presence status story ID to
the routing story's `dependencies` array.

---

## Step 6 — Generate Acceptance Criteria

For each story write a minimum of 2 and maximum of 5 Given/When/Then
scenarios following the rules in `references/generate-user-story.md`.

### Scenario order
1. Happy path — always first
2. Edge case or boundary condition
3. Error state or failure scenario
4. Validation scenario — required for any story from a Low confidence REQ

### AC format in JSON
Map each scenario correctly:
- Given/When/Then → `format: "bdd"` with `given`, `when`, `then` fields
- Checklist statement → `format: "checklist"` with `statement` field

Mix both formats within a story where appropriate — checklist for
simple verifiable states, BDD for flow-based scenarios.

---

## Step 7 — Assign Metadata

For each story assign all required fields:

### Priority
| Value | Meaning |
|---|---|
| High | Launch is blocked without this |
| Medium | Important but workaround exists for go-live |
| Low | Nice to have; include only if budget allows |

Default to Medium if not determinable. Note it in `salesforce.notes`.

### T-shirt size (Salesforce implementation effort)
| Size | Effort |
|---|---|
| XS | < 1 day — simple config |
| S | 1–2 days — standard Flow, page layout |
| M | 3–5 days — multi-step Flow, custom report type |
| L | 1–2 weeks — complex automation, integration, custom LWC |
| XL | > 2 weeks or high uncertainty — flag for breakdown |

XL stories must have a note in `salesforce.notes` flagging breakdown needed.

### Build type
- **OOB** — standard Salesforce config only
- **Custom** — requires Apex, LWC, or custom integration
- **TBC** — unclear without further discovery

When uncertain use TBC — a wrong OOB flag creates false confidence
in the project estimate.

### Tags
Populate the `tags` array with: cloud name, build type, phase,
and any relevant feature area (e.g. "SLA", "Email", "Mobile").

---

## Step 8 — Quality Check

Before assembling JSON, verify every story:

- [ ] Decomposition plan was produced before any stories were written
- [ ] "As a" uses a specific cloud-appropriate persona
- [ ] "I want" describes a user action, not a technical implementation
- [ ] "So that" is grounded in Project Context — not generic
- [ ] Minimum 2 AC scenarios, happy path is first
- [ ] Low confidence REQs have a validation AC scenario
- [ ] `salesforce.feature` is populated on every story
- [ ] `salesforce.build_type` is OOB / Custom / TBC — not guessed
- [ ] XL stories have a breakdown note in `salesforce.notes`
- [ ] Dependencies are populated where story order matters
- [ ] Phase 2 stories are absent unless `include_phase_2` was passed
- [ ] Story IDs continue sequentially from `_state.last_story_id`
- [ ] Epic IDs continue sequentially from `_state.last_epic_id`
- [ ] Every story references its source REQ in `req_ref`
- [ ] `_state` block is ready to update with this run's values

---

## Step 9 — Assemble and Save Canonical JSON

### 9.1 — Assemble the JSON

Build the full JSON object per `references/story-schema.json`.

Default values for fields not determinable at generation time:
- `story_points` → `null`
- `status` → `"draft"`
- `reviewed_by` → `null`
- `reviewed_at` → `null`
- `comments` → `[]`

Populate the `epics` array for this run:
```json
{
  "id": "EPIC-00X",
  "name": "[Epic name from requirements register]",
  "description": "[One sentence drawn from requirements context]",
  "phase": 1,
  "story_count": X,
  "salesforce_cloud": "[Cloud]"
}
```

Recalculate the full `summary` block across all stories in the file
after merging — not just the stories from this run.

### 9.2 — Update the `_state` block

After assembling stories, update `_state` with this run's values:

```json
"_state": {
  "last_story_id": "[highest story ID from this run]",
  "last_epic_id": "[highest epic ID from this run]",
  "last_run_epic": "[epic name just completed]",
  "last_run_date": "[today's date YYYY-MM-DD]",
  "epics_complete": ["[all epics done including this run]"],
  "epics_remaining": ["[epics from register not yet run]"]
}
```

`epics_remaining` is calculated by reading the epic list from the
requirements register and removing all epics in `epics_complete`.

### 9.3 — Save to Claude project storage

Filename convention:
```
[projectname]-stories.json
```
Lowercase, hyphens, no spaces or special characters.
Examples: `acme-corp-service-cloud-stories.json`

**Epic-scoped run (default):** merge new stories and epics into
the existing file. Preserve all existing stories. Update `_state`
and `summary`.

**Full run (`all epics`):** overwrite any existing file.

### 9.4 — Error handling
If the project storage write fails:
1. Do not silently drop output
2. Output full JSON in a code block in chat
3. Tell the user the save failed and ask them to copy it manually

---

## Step 10 — Post-run Summary

```
✅ Stories generated — [Epic Name]

Project: [ProjectName]
JSON: [projectname]-stories.json

This run:
Stories generated: X | REQs processed: X
Type A: X | Type B: X | Type C: X
OOB: X | Custom: X | TBC: X
High: X | Medium: X | Low: X confidence

Full backlog to date:
Total stories: X | Phase 1: X | Phase 2: X

Epics complete: [list]
Epics remaining: [list]

XL stories to break down: [IDs or "none"]
Type C decompositions applied: [feature names or "none"]
Low confidence stories needing validation: [IDs or "none"]

━━━━━━━━━━━━━━━━━━━━━━
Next run — copy and paste:
Generate user stories — epic="[next epic name]"
Project: [ProjectName]
Doc: [requirements register URL]
━━━━━━━━━━━━━━━━━━━━━━
```

---

## Edge Cases

**Requirement is solution-framed** (e.g. "Build a Flow that sends an email"):
Reframe as a user need. Note in `salesforce.notes` that the original was
solution-framed and has been rewritten.

**Requirement maps to multiple stories (Type B or C)**: Use sub-story IDs
(US-012a, US-012b) if splitting a single REQ. All reference same `req_ref`.

**Low confidence REQ**: Generate story on best-judgment, set confidence to
Low, add validation AC scenario, note in `salesforce.notes` that the
requirement needs client confirmation before build commences.

**Epic already in `epics_complete`**: Do not regenerate. Tell the user this
epic has already been run and ask if they want to re-run it (which will
replace the existing stories for that epic only).

**No project name provided**: Use `client-unknown` in the filename. Flag
in post-run summary and ask the consultant to confirm.

**No Project Context available**: Do not generate stories. Ask the user
to provide the requirements register URL or paste the Project Context block.
Stories without context produce low-quality outcomes.

**`include_phase_2` passed**: Include Phase 2 REQs but tag them clearly
in the JSON with `"phase": 2`. Ensure the summary counts reflect this.
