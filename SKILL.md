---
name: user-story
description: >
  Use this skill whenever the user wants to generate Salesforce user stories
  and acceptance criteria from a requirements register. Accepts either a Google
  Doc URL (output from the requirements-extraction skill) or pasted REQ items.
  Default scope is one epic per run. All-epics is an explicit override.
  Outputs canonical JSON to Claude project storage. Google Sheet export is a separate on-demand step.

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

2. `references/generate-user-story.md` — load story format rules, the
   two-pass decomposition rules with worked examples, and the
   decomposition plan format. Required before Step 4.

---

## Step 4 — Decompose Requirements

Before writing any stories, produce a decomposition plan for every REQ
in scope. Do not write a single story until this plan is complete.

Decomposition runs as **two passes over every REQ**. Do not classify the
REQ up front and then split accordingly — split first, and let the type
fall out of what the two passes did. Full rules and worked examples are
in `references/generate-user-story.md`.

### Pass 1 — Split on the text
Read the REQ and break it into the distinct user needs it actually
states. Compound signals: multiple personas, actions, directions
(push vs request), or filter dimensions. Test: delete a clause — is
something the client asked for now missing?

Pass 1 produces sibling needs. They are usually independent of each
other and carry no dependencies between them.

### Pass 2 — Expand on implementation depth
Run this on **every** need from Pass 1, including REQs that Pass 1 left
whole. Ask: what does it actually take to configure, deliver and test
this in Salesforce? Apply Salesforce implementation knowledge — not
text analysis. Test: delete a story — does the feature still function
end to end?

Pass 2 produces a sequence. Populate `dependencies` to encode the build
order — a Pass 2 expansion with no dependencies between its stories is
usually a sign the split was textual, not architectural.

### Resulting type
The type is a label for what happened, used in the run summary. It is
not stored in the JSON.

| Pass 1 | Pass 2 | Type |
|---|---|---|
| no split | no expansion | **A — Atomic** |
| split | no expansion | **B — Compound** |
| no split | expansion | **C — Implementation Depth** |
| split | expansion on ≥1 branch | **B+C** |

A REQ that reads as atomic can still be pure Type C — REQ text that
states one need may still need five stories to deliver. That case is
the reason Pass 2 runs on unsplit REQs.

### Decomposition plan output format
```
DECOMPOSITION PLAN
══════════════════
REQ-001 → Type A → 1 story
  Reason: [one line]

REQ-012 → Type B → 2 stories
  Reason: [one line — what the text split on]
  Story 1: [brief description]
  Story 2: [brief description]

REQ-019 → Type C → 5 stories (Omni-Channel Routing)
  Reason: [one line — what the platform requires]
  Story 1–5: [brief descriptions]
  Dependency chain: [e.g. 1 → 2 → 3 → 4, 5]

REQ-024 → Type B+C → 6 stories
  Pass 1: 2 needs — [brief], [brief]
  Pass 2: [which need expanded] → 5 stories (Omni-Channel Routing)
  Story 1–6: [brief descriptions]
  Dependency chain: [e.g. 1 → 2 → 3 → 4, 5; 6 independent]

Total stories this run: X
══════════════════
```

Output this plan before writing any stories. Proceed only after the
plan is complete.

---

## Step 5 — Generate Directly to JSON

Work through each story in the decomposition plan. For every story,
reason first then write the JSON object directly. No prose output.

### Before writing each JSON object, internally reason:

**Persona**
Which specific persona from `references/salesforce-personas.md` fits
this story? Is it cloud-appropriate? Is it specific enough?

**"I want" action**
Does this describe what the user does — not how it is built?
Does it contain "and"? If so, this should have been split in Step 4.

**"So that" outcome**
Is this grounded in the Project Context — Client Goal and Current Pain?
Generic outcomes like "so that I can do my job better" are not acceptable.
Use what the client actually said.

**`salesforce.feature`**
What specific Salesforce feature is being configured or built?
Never leave this blank. Use Salesforce knowledge to name it precisely.
Examples: Email-to-Case, Omni-Channel Routing, Entitlements & Milestones,
Web-to-Lead, Forecasting Categories, Scheduling Policy, Action Plans.

**`salesforce.build_type`**
OOB — standard config only. Custom — requires Apex, LWC, integration.
TBC — unclear without further discovery. When uncertain use TBC.
A wrong OOB flag creates false confidence in the project estimate.

**Acceptance criteria**
Plan the scenarios before writing them:
- Scenario 1: happy path — always first
- Scenario 2: edge case or boundary condition
- Scenario 3: error state or failure (where relevant)
- Scenario 4: validation scenario — required if source REQ is Low confidence
Use `format: "bdd"` for flow-based scenarios, `format: "checklist"`
for simple verifiable states. Mix both within a story.

**Priority**
High — launch blocked without this. Medium — workaround exists.
Low — nice to have. Default to Medium if not determinable.

**T-shirt size**
XS < 1 day | S 1–2 days | M 3–5 days | L 1–2 weeks | XL > 2 weeks.
Size against Salesforce implementation effort. XL must have a breakdown
note in `salesforce.notes`.

**Dependencies**
Does this story come from a Pass 2 expansion that depends on another
story being complete first? If yes, populate `dependencies` with the
preceding story ID. Pass 1 siblings are usually independent — leave
`dependencies` empty unless one genuinely blocks the other.

**Tags**
Cloud name, build type, phase, and relevant feature area.

### Then write the complete JSON story object in one pass:

```json
{
  "id": "US-00X",
  "req_ref": "REQ-XXX",
  "epic_id": "EPIC-00X",
  "epic_name": "[Epic name]",
  "title": "As a [persona] I want [action] so that [outcome]",
  "persona": "[persona]",
  "i_want": "[action]",
  "so_that": "[outcome grounded in Project Context]",
  "description": "[1-2 sentences expanding on the story context]",
  "acceptance_criteria": [
    {
      "id": "AC-00X-01",
      "format": "bdd",
      "given": "[context]",
      "when": "[action]",
      "then": "[observable outcome]"
    },
    {
      "id": "AC-00X-02",
      "format": "checklist",
      "statement": "[verifiable state]"
    }
  ],
  "phase": 1,
  "priority": "[High / Medium / Low]",
  "confidence": "[High / Medium / Low — inherited from source REQ]",
  "story_points": null,
  "tshirt_size": "[XS / S / M / L / XL]",
  "status": "draft",
  "salesforce": {
    "build_type": "[OOB / Custom / TBC]",
    "feature": "[specific Salesforce feature name]",
    "notes": "[any delivery notes, TBC reasons, or breakdown flags]"
  },
  "flags": [],
  "ambiguities": [],
  "dependencies": [],
  "tags": ["[Cloud]", "[OOB/Custom]", "Phase [X]"],
  "created_at": "[ISO timestamp]",
  "updated_at": "[ISO timestamp]",
  "reviewed_by": null,
  "reviewed_at": null,
  "comments": []
}
```

Repeat for every story in the decomposition plan before moving to Step 6.

---

## Step 6 — Quality Check JSON

After all story objects are written, validate against this checklist.
Fix any issues in the JSON directly — do not rewrite in prose.

- [ ] Every "As a" uses a specific cloud-appropriate persona
- [ ] Every "I want" describes a user action not a technical implementation
- [ ] Every "So that" is grounded in Project Context — not generic
- [ ] Every story has minimum 2 AC scenarios, happy path is first
- [ ] Every Low confidence story has a validation AC scenario
- [ ] `salesforce.feature` is populated on every story — none blank
- [ ] `salesforce.build_type` is OOB / Custom / TBC on every story
- [ ] XL stories have a breakdown note in `salesforce.notes`
- [ ] Dependencies populated on every Pass 2 expansion where build order matters
- [ ] Pass 1 siblings do not carry dependencies unless one genuinely blocks the other
- [ ] Phase 2 stories absent unless `include_phase_2` was passed
- [ ] Story IDs continue sequentially from `_state.last_story_id`
- [ ] Epic IDs continue sequentially from `_state.last_epic_id`
- [ ] Every story references its source REQ in `req_ref`
- [ ] `confidence` inherited from source REQ on every story

---

## Step 7 — Assemble and Save Canonical JSON

### 7.1 — Assemble the JSON

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

### 7.2 — Update the `_state` block

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

### 7.3 — Save to Claude project storage

Filename convention:
```
[projectname]-stories.json
```
Lowercase, hyphens, no spaces or special characters.
Examples: `acme-corp-stories.json`, `northwind-service-stories.json`

**One file per project, not per epic.** The filename must be identical
on every run for the same project — it is what Step 1 looks up and what
the merge below writes into. Do not include the epic name, epic number,
or run date: an epic-specific filename creates a new file per run, and
`_state`, the running backlog totals, and `sheet-export` all silently
lose sight of the earlier epics.

**Epic-scoped run (default):** merge new stories and epics into
the existing file. Preserve all existing stories. Update `_state`
and `summary`.

**Full run (`all epics`):** overwrite any existing file.

### 7.4 — Error handling
If the project storage write fails:
1. Do not silently drop output
2. Output full JSON in a code block in chat
3. Tell the user the save failed and ask them to copy it manually

---

## Step 8 — Post-run Summary

```
✅ Stories generated — [Epic Name]

Project: [ProjectName]
[projectname]-stories.json

This run:
Stories generated: X | REQs processed: X
Type A: X | Type B: X | Type C: X | Type B+C: X
OOB: X | Custom: X | TBC: X
High: X | Medium: X | Low: X confidence

Full backlog to date:
Total stories: X | Phase 1: X | Phase 2: X

Epics complete: [list]
Epics remaining: [list]

XL stories to break down: [IDs or "none"]
Pass 2 expansions applied: [feature names or "none"]
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

**Requirement maps to multiple stories (Type B, C or B+C)**: Use sub-story
IDs (US-012a, US-012b) if splitting a single REQ. All reference the same
`req_ref`, however many passes produced them.

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
