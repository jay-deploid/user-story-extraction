# Google Sheets Output Reference

Instructions for creating the user story backlog Google Sheet at the end of the skill.

---

## MCP Connection

Use the Google Drive MCP connector already configured on the user's account:
```
URL: https://drivemcp.googleapis.com/mcp/v1
```

---

## Sheet Structure

Create a single sheet named `Backlog`. Row 1 is the header row.
Each user story is one data row from Row 2 onwards.

| Col | Header | Description |
|---|---|---|
| A | Story ID | US-001, US-002 — sequential across all epics |
| B | Epic | Business function epic name (e.g. Case Management & SLA) |
| C | REQ Ref | Source requirement(s) this story was generated from (e.g. REQ-012) |
| D | SF Cloud | Salesforce cloud tag (e.g. Service Cloud, Sales Cloud, Multi-cloud) |
| E | User Role | The "As a..." persona (e.g. Service Agent, Sales Rep) |
| F | User Story | Full story: "As a [role], I want [action] so that [outcome]" |
| G | Acceptance Criteria | Full AC with each scenario on its own line: Given / When / Then |
| H | OOB or Custom | OOB / Custom / TBC |
| I | Size | XS / S / M / L / XL |
| J | Priority | Must Have / Should Have / Could Have / Won't Have |
| K | Notes | Dependencies, open questions, flags, breakdown notes |
| L | Status | Default: "Draft" for all new stories |

---

## Column guidance

### Story ID (Col A)
- Format: `US-001`, `US-002` — sequential across the whole sheet
- For epic-scoped runs on an existing backlog, start IDs from where the last sheet left off

### User Story (Col F)
- Strict format: `As a [role], I want [action] so that [outcome].`
- Action = what the user does, not how it is built
- Outcome = business value, not system behaviour

### Acceptance Criteria (Col G)
- Minimum 2 scenarios, maximum 5
- Each scenario on its own line:
  ```
  Given [context]
  When [action]
  Then [outcome]
  ```
- Happy path first, then edge case, then error state

### OOB or Custom (Col H)
- **OOB** — standard Salesforce clicks/config only
- **Custom** — requires Apex, LWC, or custom integration
- **TBC** — unclear without further discovery; always add a note in Col K

### Size (Col I)
- XS: < 1 day | S: 1–2 days | M: 3–5 days | L: 1–2 weeks | XL: > 2 weeks
- Size against Salesforce implementation effort, not general software dev
- Flag XL stories in Col K — they should be broken down before sprint planning

### Priority (Col J)
- Must Have: blocks launch | Should Have: workaround exists | Could Have: nice to have | Won't Have: out of scope
- Default to Should Have if not determinable; note it in Col K

### Notes (Col K)
- Flag dependencies on other stories (e.g. "Requires US-004 first")
- Flag open questions that block build
- Note if the story needs to be broken down further
- Reference source AMB or FLAG from the requirements register where relevant
- Leave blank if nothing to add

---

## Creating the Sheet via MCP

### Step 1 — Assemble all row data first
Build the complete array of rows in memory before making any MCP calls.
Do not make incremental calls row by row.

### Step 2 — Create the Google Sheet
Use the Google Drive MCP `create_file` tool:
- `name`: see file naming convention below
- `mimeType`: `application/vnd.google-apps.spreadsheet`
- Target: root of user's Google Drive (no parent folder ID needed)

### Step 3 — Populate the sheet
Write the header row first, then all data rows.

### Step 4 — Return the link
Share the full Google Sheets URL with the user immediately after creation.

---

## File naming convention

```
[ProjectName] — User Story Backlog — [YYYY-MM-DD]
```
Epic-scoped:
```
[ProjectName] — User Story Backlog — [EpicName] — [YYYY-MM-DD]
```

Examples:
- `Acme Corp — User Story Backlog — 2025-11-14`
- `Acme Corp — User Story Backlog — Case Management & SLA — 2025-11-14`

If no project name is available, use `[Client Unknown]` and flag in the post-run summary.

---

## Error handling

If the Sheet creation fails:
1. Do not silently drop the output
2. Tell the user the write failed
3. Output the full story data as a markdown table in chat so nothing is lost
4. Suggest they copy it manually or retry
