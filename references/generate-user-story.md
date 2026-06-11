# Generate User Stories

Reference file for the user-story skill.
Defines the format, rules, and examples for writing user stories and acceptance criteria
in a Salesforce implementation context.

---

## Usage

Trigger this skill by stating scope in your prompt:

```
Generate user stories
Generate user stories — all epics
Generate user stories — epic="Case Management & SLA"
Generate user stories — reqs="REQ-001, REQ-004, REQ-007"
```

Provide one of the following as input:
- A Google Doc URL from the requirements-extraction skill output
- Pasted REQ items in any standard format

---

## User Story Format

Every story must follow this structure exactly:

```
As a [specific Salesforce persona],
I want [to perform a specific action],
so that [I achieve a specific business outcome].
```

### Rules

- **Persona** — must be specific and cloud-appropriate. Never use "User" or "Stakeholder".
  See `references/salesforce-personas.md` for the full list by cloud.
- **Action** — describes what the user wants to do, not how it will be built.
  Never reference Flows, Apex, LWC, or any technical implementation.
- **Outcome** — describes the business value to the user, not a system behaviour.
- **Scope** — one story = one user need. If "and" appears in the action, split into two stories.

### Good example
```
As a Service Agent, I want to see all of a customer's open cases and recent
interactions on a single screen so that I can resolve their issue without
switching between tabs.
```

### Bad examples
```
As a user, I want the system to work better.
❌ Persona too generic, outcome too vague.

As a Sales Rep, I want a Flow that triggers on Opportunity stage change
so that the system updates automatically.
❌ Action describes the solution, not the user need.

As a Service Agent, I want to view cases and update SLAs and send emails
so that I can manage my workload.
❌ Three needs in one story — split it.
```

---

## Acceptance Criteria Format

Every story must have a minimum of 2 and maximum of 5 scenarios.

```
Given [the starting context or precondition]
When [the user performs an action, or an event occurs]
Then [the expected outcome is observed]
```

### Scenario order
1. **Happy path** — the primary success scenario. Always first.
2. **Edge case** — a boundary condition or alternate valid path.
3. **Error state** — what happens when something goes wrong. Include where relevant.

### Rules
- Each scenario tests exactly one thing
- "Then" statements must be observable and testable by a QA tester
- Do not describe implementation details in any part of the scenario
- Write scenarios from the user's perspective, not the system's

### Good example
```
Given I am logged in as a Service Agent and a case is assigned to me
When I open the case record
Then I can see the customer's full contact details, case history, and last 5 interactions
on the same page without navigating away

Given a case has been open for more than 4 hours without a response
When the SLA breach timer reaches zero
Then the case is automatically escalated to the Tier 2 queue and I receive
an in-app notification

Given I try to close a case without entering a resolution note
When I click the Close Case button
Then I see a validation error preventing closure until a resolution note is added
```

### Bad example
```
Given the system is working
When the user does something
Then it works correctly
❌ Not testable, too vague, no real scenario.
```

---

## Story Splitting Guide

Split a requirement into multiple stories when:

| Signal | Action |
|---|---|
| "and" appears in the action | Split into two stories |
| Two different personas are involved | One story per persona |
| The story would be sized XL | Break into smaller deliverable chunks |
| Happy path and error path are very different flows | Consider splitting |
| A setup step is required before the main action | Create a dependency story |

When splitting, use sub-story IDs: US-012a, US-012b, US-012c.
All sub-stories reference the same parent REQ in the REQ Ref column.
