---
name: user-story
description: >
  Use this skill whenever the user wants to generate Salesforce user stories
  and acceptance criteria from a requirements register. Accepts either a Google
  Doc URL (output from the requirements-extraction skill) or pasted REQ items.
  Supports full backlog generation or scoped runs (one epic or specific REQs).
  Outputs a canonical JSON file saved to Claude project storage for the story
  viewer. Google Sheet export is a separate on-demand step.

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
register. Saves a canonical JSON file to Claude project storage. The story
viewer opens automatically after each run for team review.

Do NOT create a Google Sheet during this skill run. Sheet export is a
separate skill triggered on demand after stories have been reviewed.

---

## Before You Start — Determine Scope

Check the prompt for scope instructions:

| Prompt says | What to do |
|---|---|
| Nothing / "all epics" | Generate stories for every REQ in the register |
| `epic="[name]"` | Generate stories for that named epic only |
| `reqs="REQ-001, REQ-004"` | Generate stories for those specific REQs only |

Also check for:
- **Project name** — needed for file naming
- **SF Cloud context** — needed for persona selection and OOB vs custom flags
  - If not stated, infer from the requirements content
  - If still unclear, flag as TBC in the Notes field and proceed

---

## Step 1 — Read the Requirements Register

### Input method A: Google Doc URL
If a Google Doc URL was provided:
1. Use the Google Drive MCP to fetch the document content
2. Parse out all REQ items with their epic grouping
3. Note the project name, cloud context, FLAGS, and AMBs from the document header
4. Filter to the requested scope before proceeding

### Input method B: Pasted REQ items
If REQ items were pasted directly into the conversation:
1. Accept any format — standard REQ format, plain bullets, or numbered list
2. Infer epic groupings from context if not explicitly stated
3. Ask for project name and cloud context if not provided and not inferable

### Sparse input
If fewer than 3 REQs are provided with no cloud context, ask for the missing
context before generating. Stories written without cloud context will have
inaccurate personas and unreliable OOB flags.

---

## Step 2 — Load Reference Files

Before generating any stories, read both reference files:

1. `references/salesforce-personas.md` — load the persona table for the relevant
   cloud(s). Use this to write accurate "As a..." lines.

2. `references/story-schema.json` — load the canonical JSON schema. You will
   need this at Step 7 to assemble the JSON output correctly.

---

## Step 3 — Generate User Stories

Process each REQ item in scope. For each requirement produce one primary story.
If a requirement contains multiple distinct user actions, split into sub-stories
(e.g. US-012a, US-012b) and note the parent REQ in the `req_ref` field.

### Story format

```
As a [specific Salesforce persona],
I want [to perform a specific action],
so that [I achieve a specific business outcome].
```

**Rules:**
- Persona must be specific and cloud-appropriate — use `references/salesforce-personas.md`
- Action describes what the user does, never how it is built
- Outcome describes business value, not system behaviour
- One story = one user need. If you find yourself using "and" in the action, split it

**Good:** `As a Service Agent, I want to see all of a customer's open cases and
recent interactions on a single screen so that I can resolve their issue without
switching between tabs.`

**Bad:** `As a user, I want the system to create a Flow that sends an email and
updates a field so that it works.`

---

## Step 4 — Generate Acceptance Criteria

For each story write a minimum of 2 and maximum of 5 Given/When/Then scenarios.

```
Given [the starting context or precondition]
When [the user performs an action, or an event occurs]
Then [the expected outcome is observed]
```

**Scenario order:**
1. Happy path — always first
2. Edge case or boundary condition
3. Error state or failure scenario (where relevant)

**Rules:**
- Each scenario tests one thing only
- "Then" statements must be observable and testable
- Do not describe implementation details in the criteria

---

## Step 5 — Assign Metadata

For each story assign the following fields.

### OOB or Custom
- **OOB** — achievable with standard Salesforce config (Flows, validation rules,
  standard objects, assignment rules, page layouts, approval processes)
- **Custom** — requires Apex, LWC, custom integrations, or significant platform extension
- **TBC** — genuinely unclear; add a note in the `salesforce.notes` field explaining
  what needs confirming

When uncertain, use TBC rather than guessing. A wrong OOB flag creates false
confidence in the project estimate.

### Size (T-shirt, Salesforce implementation effort)
| Size | Effort |
|---|---|
| XS | < 1 day — simple config, one or two clicks |
| S | 1–2 days — straightforward Flow, page layout, standard object config |
| M | 3–5 days — multi-step Flow, custom report type, standard approval process |
| L | 1–2 weeks — complex automation, integration config, custom LWC |
| XL | > 2 weeks or high uncertainty — flag for breakdown |

### Priority
| Value | Meaning |
|---|---|
| High | Launch is blocked without this — maps to Must Have |
| Medium | Important but a workaround exists for go-live — maps to Should Have |
| Low | Nice to have; include only if budget allows — maps to Could Have |

If priority is not determinable from context, default to Medium and note it in
`salesforce.notes`.

### Phase
- **1** — explicitly in scope for initial delivery, or clearly core to the solution
- **2** — deferred, mentioned as "later" or "next stage", or unlikely given complexity
- **TBC** — cannot be determined from source material

### Confidence
- **High** — requirement was clearly stated and confirmed
- **Medium** — inferred with reasonable certainty or stated but not confirmed
- **Low** — fragmented, contradicted, or raised in passing

---

## Step 6 — Quality Check

Before assembling the JSON, verify each story:

- [ ] "As a" uses a specific Salesforce persona, not a generic role
- [ ] "I want" describes a user action, not a technical implementation
- [ ] "So that" describes a business outcome, not a system state
- [ ] Minimum 2 AC scenarios per story, happy path is first
- [ ] OOB/Custom assignment is defensible — not guessed
- [ ] XL stories have a note in `salesforce.notes` flagging breakdown needed
- [ ] Stories referencing FLAGS or AMBs have a note in `salesforce.notes`
- [ ] Story IDs are sequential and unique across the full backlog
- [ ] Every story references its source REQ in `req_ref`

---

## Step 7 — Assemble and Save Canonical JSON

Assemble all generated stories into the canonical JSON structure per
`references/story-schema.json`. This file is the single source of truth
for the story viewer and all downstream exports.

### 7.1 — Assemble the JSON

Build the full JSON object. Every field in the schema must be present.
Rules for fields not determinable at generation time:

- `story_points` → `null`
- `status` → `"draft"` for all stories on first generation
- `reviewed_by` → `null`
- `reviewed_at` → `null`
- `comments` → `[]`
- `dependencies` → `[]` unless a dependency is explicitly known

For acceptance criteria, map each scenario using the correct format:

- Given/When/Then scenarios → `format: "bdd"` with `given`, `when`, `then` fields
- Checklist-style statements → `format: "checklist"` with `statement` field

Populate the `summary` block by counting across all assembled stories:

```json
"summary": {
  "total_stories": X,
  "phase_1": X,
  "phase_2": X,
  "by_status": { "draft": X, "approved": 0, "flagged": 0, "deferred": 0 },
  "by_confidence": { "high": X, "medium": X, "low": X },
  "by_epic": { "[Epic Name]": X }
}
```

Populate the `epics` array from the epic groupings identified in Step 1:

```json
"epics": [
  {
    "id": "EPIC-001",
    "name": "[Epic name]",
    "description": "[One sentence from the requirements context]",
    "phase": 1,
    "story_count": X,
    "salesforce_cloud": "[Cloud]"
  }
]
```

### 7.2 — Save to Claude project storage

Save the assembled JSON to Claude project storage using this filename convention:

```
[projectname]-stories.json
```

Use lowercase and hyphens, no spaces or special characters.

Examples:
- `acme-corp-service-cloud-stories.json`
- `techstart-sales-cloud-stories.json`

If no project name was provided, use `client-unknown-stories.json`.

**Full run:** overwrite any existing file with the same name.
**Epic-scoped run:** merge new stories into the existing file — preserve
stories from other epics already present. Update the `summary` block to
reflect the full file after merging.

### 7.3 — Error handling

If the project storage write fails:
1. Do not silently drop the output
2. Output the full JSON in a code block in chat so nothing is lost
3. Tell the user the save failed and ask them to copy it manually or retry

---

## Step 8 — Post-run Summary

After saving the JSON, return this summary in chat:

```
✅ Stories generated and saved

Project: [ProjectName]
JSON saved: [projectname]-stories.json

Stories generated: X
Epics covered: [list]
Phase 1: X | Phase 2: X | TBC: X
OOB: X | Custom: X | TBC: X
High: X | Medium: X | Low: X priority

XL stories flagged for breakdown: [IDs or "none"]
Open TBC items: [key items or "none"]

```

Do NOT include a Google Sheets link. The Sheet is created separately
on demand via the sheet-export skill after the team has reviewed stories.

---

## Edge Cases

**Requirement is solution-framed** (e.g. "Build a Flow that sends an email"):
Reframe as a user need. Note in `salesforce.notes` that the original was
solution-framed and has been rewritten.

**Requirement maps to multiple stories**: Create sub-stories (US-012a, US-012b),
all referencing the same parent REQ in `req_ref`.

**Ambiguity on the source REQ**: Generate the story on best-judgment, note the
ambiguity in `salesforce.notes`, set confidence to Low.

**No project name provided**: Use `client-unknown` in the filename, flag
in the post-run summary, and ask the consultant to confirm the project name.

**Epic-scoped run on an existing backlog**: Merge into the existing JSON.
Note in the summary that story IDs have been checked for conflicts and
confirm the highest existing ID so the consultant can verify sequencing.

