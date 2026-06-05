---
name: user-story
description: >
  Use this skill whenever the user wants to generate Salesforce user stories
  and acceptance criteria from a requirements register. Accepts either a Google
  Doc URL (output from the requirements-extraction skill) or pasted REQ items.
  Supports full backlog generation or scoped runs (one epic or specific REQs).
  Outputs a Google Sheet with one row per story, ready for Jira or ADO import,
  and a canonical JSON file saved to Claude project storage for the story viewer.

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
register. Outputs a Google Sheet backlog ready for Jira or ADO import, and
a canonical JSON file saved to Claude project storage.

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
  - If still unclear, flag as TBC in the Notes column and proceed

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

Before generating any stories, read all three reference files:

1. `references/salesforce-personas.md` — load the persona table for the relevant
   cloud(s). Use this to write accurate "As a..." lines.

2. `references/google-sheets-output.md` — load the column structure, formatting
   rules, and MCP output instructions. You will need this at Step 7.

3. `references/story-schema.json` — load the canonical JSON schema. You will
   need this at Step 9 to assemble the JSON output correctly.

---

## Step 3 — Generate User Stories

Process each REQ item in scope. For each requirement produce one primary story.
If a requirement contains multiple distinct user actions, split into sub-stories
(e.g. US-012a, US-012b) and note the parent REQ in the REQ Ref column.

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

For each story assign the following. Refer to `references/google-sheets-output.md`
for full guidance on each field.

### OOB or Custom
- **OOB** — achievable with standard Salesforce config (Flows, validation rules,
  standard objects, assignment rules, page layouts, approval processes)
- **Custom** — requires Apex, LWC, custom integrations, or significant platform extension
- **TBC** — genuinely unclear; add a note in the Notes column explaining what needs confirming

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

### Priority (MoSCoW)
- **Must Have** — launch is blocked without this
- **Should Have** — important but a workaround exists for go-live
- **Could Have** — nice to have; include only if budget allows
- **Won't Have** — explicitly out of scope this phase

If priority is not determinable from context, default to Should Have and note it.

---

## Step 6 — Quality Check

Before writing to Sheets, verify each story:

- [ ] "As a" uses a specific Salesforce persona, not a generic role
- [ ] "I want" describes a user action, not a technical implementation
- [ ] "So that" describes a business outcome, not a system state
- [ ] Minimum 2 AC scenarios per story, happy path is first
- [ ] OOB/Custom assignment is defensible — not guessed
- [ ] XL stories are flagged in Notes for breakdown
- [ ] Stories referencing FLAGS or AMBs have a note in the Notes column
- [ ] Story IDs are sequential and unique across the whole sheet
- [ ] Every story references its source REQ in the REQ Ref column

---

## Step 7 — Write to Google Sheets

Read `references/google-sheets-output.md` for the full column structure,
MCP connection details, file naming convention, and error handling.

Key points:
- Assemble all rows in memory before making any MCP calls
- File name: `[ProjectName] — User Story Backlog — [YYYY-MM-DD]`
- If epic-scoped: `[ProjectName] — User Story Backlog — [EpicName] — [YYYY-MM-DD]`
- Target: root of user's Google Drive
- Return the Google Sheets URL after creation
- If the write fails, output as a markdown table in chat — never silently drop output

---

## Step 8 — Post-run Summary

After the Sheet is created, return a brief summary in chat:

```
✅ User story backlog created: [link]

Stories generated: X
Epics covered: [list]
OOB: X | Custom: X | TBC: X
Must Have: X | Should Have: X | Could Have: X

XL stories flagged for breakdown: [IDs or "none"]
Open TBC items before build: [key items or "none"]
```

---

## Step 9 — Assemble and Save Canonical JSON

After the post-run summary, assemble all generated stories into the canonical
JSON structure and save to Claude project storage. This file is the single
source of truth for the story viewer and all downstream exports.

### 9.1 — Assemble the JSON

Build the JSON object per `references/story-schema.json`. Every field in the
schema must be present. Rules for fields not determinable at this stage:

- `story_points` — set to `null` (assigned during sprint planning)
- `status` — set to `"draft"` for all stories on first generation
- `reviewed_by` — set to `null`
- `reviewed_at` — set to `null`
- `comments` — set to `[]`

For acceptance criteria, use the following format per AC item:

- Given/When/Then scenarios → `format: "bdd"` with `given`, `when`, `then` fields
- Checklist statements (if any) → `format: "checklist"` with `statement` field

Populate the `summary` block by counting across all stories before writing:

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

### 9.2 — Save to Claude project storage

Save the assembled JSON to Claude project storage using the following filename
convention:

```
[ProjectName]-stories.json
```

Examples:
- `acme-corp-service-cloud-stories.json`
- `techstart-sales-cloud-stories.json`

Use lowercase, hyphens instead of spaces, no special characters. If no project
name was provided, use `client-unknown-stories.json`.

If an existing `[ProjectName]-stories.json` already exists in project storage
from a previous run, overwrite it. The JSON is always the latest complete run.
For epic-scoped runs, merge the new stories into the existing file rather than
overwriting — preserve stories from other epics that are already present.

### 9.3 — Confirm in chat

After saving, append a single line to the post-run summary:

```
📦 Canonical JSON saved: [ProjectName]-stories.json ([X] stories)
```

### Error handling

If the project storage write fails:
1. Do not silently drop the output
2. Output the full JSON in a code block in chat so nothing is lost
3. Note the save failed and ask the user to copy it manually or retry

---

## Edge Cases

**Requirement is solution-framed** (e.g. "Build a Flow that sends an email"):
Reframe as a user need. Note in the Notes column that the original was
solution-framed and has been rewritten.

**Requirement maps to multiple stories**: Create sub-stories (US-012a, US-012b),
all linked to the same REQ ref.

**Ambiguity on the source REQ**: Generate the story on best-judgment, note the
ambiguity in the Notes column, mark Priority as TBC until resolved.

**No project name provided**: Use `[Client Unknown]` in the file name and flag
in the post-run summary.

**Epic-scoped run on an existing backlog**: Note in the summary that story IDs
should be checked against the existing sheet to avoid conflicts. When saving
JSON, merge into the existing project JSON rather than overwriting — preserve
all stories from other epics.
