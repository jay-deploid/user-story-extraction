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

Decomposition is **two passes over every REQ**. Do not decide the type
first and split accordingly — split first, and let the type fall out of
what the two passes did.

### Pass 1 — Split on the text

Break the REQ into the distinct user needs it actually states.

| Signal | Action |
|---|---|
| "and" appears in the action | Split into two stories |
| Two different personas are involved | One story per persona |
| Two directions are described (a push and a pull/request) | One story per direction |
| Multiple filter, report or channel dimensions are listed | One story per dimension |

Test: delete a clause — is something the client asked for now missing?

Pass 1 produces **siblings**. They are usually independent and carry no
dependencies between them.

### Pass 2 — Expand on implementation depth

Run this on **every** need from Pass 1, including REQs that Pass 1 left
whole. This is a Salesforce judgement, not a reading exercise.

| Signal | Action |
|---|---|
| A setup or config step is required before the main action | Create a dependency story |
| The feature needs several distinct configurations to function at all | One story per configuration |
| There is an admin-side build and an end-user-side experience | Split admin setup from user action |
| The story would be sized XL | Break into smaller deliverable chunks |
| The fallback or error path is a materially different build | Split it out |

Test: delete a story — does the feature still function end to end?

Pass 2 produces a **sequence**. Populate `dependencies` to encode build
order. A Pass 2 expansion whose stories have no dependencies between
them is usually a sign the split was textual, not architectural.

### Resulting type

| Pass 1 | Pass 2 | Type |
|---|---|---|
| no split | no expansion | **A — Atomic** |
| split | no expansion | **B — Compound** |
| no split | expansion | **C — Implementation Depth** |
| split | expansion on ≥1 branch | **B+C** |

A REQ that reads as atomic can still be pure Type C. Type C splits are
not visible in the requirement text — that is what distinguishes them.

When splitting, use sub-story IDs: US-012a, US-012b, US-012c.
All sub-stories reference the same parent REQ in `req_ref`, however many
passes produced them.

---

## Worked Examples

### Type B — compound in the text

```
REQ-023 | Team leads should be able to reassign cases between agents,
         and agents should be able to request reassignment of a case
         they cannot resolve.
```

```
REQ-023 → Type B → 2 stories
  Reason: Two personas, two opposing directions — a push (lead reassigns)
          and a pull (agent requests). Both stated explicitly in the text.
  Story 1: As a Senior Agent / Team Lead, I want to reassign a case to a
           different agent so that work moves off a blocked agent quickly
  Story 2: As a Service Agent, I want to request reassignment of a case I
           cannot resolve so that it reaches someone with the right expertise
```

Both stories are siblings — either could ship alone and be useful, so
`dependencies` stays empty on both.

### Type C — depth not visible in the text

```
REQ-021 | Cases should be routed automatically to the right agent
         based on their skills and current availability.
```

This passes the atomic test on the page — one implied persona, one
action, one outcome, no "and". A text-level read produces one story.
It cannot be built, or tested, as one story.

```
REQ-021 → Type C → 5 stories (Omni-Channel Routing)
  Reason: Skills-based routing requires channel setup, capacity model,
          skill mapping, agent-side work acceptance, and supervisor
          oversight — each independently configurable and testable.
  Story 1: Admin configures Service Channel and presence statuses
  Story 2: Admin defines agent capacity and routing configuration
  Story 3: Admin maps skills to cases and enables skills-based routing
  Story 4: Agent receives, accepts and declines routed work in the console
  Story 5: Service Manager monitors queue backlog and reassigns stuck work
  Dependency chain: 1 → 2 → 3 → 4, 5
```

Story excerpt showing the fields that carry the decomposition:

```json
{
  "id": "US-021d",
  "req_ref": "REQ-021",
  "title": "As a Service Agent I want to accept or decline work pushed to me so that I keep control of my workload when I am mid-conversation",
  "tshirt_size": "S",
  "salesforce": {
    "build_type": "OOB",
    "feature": "Omni-Channel Widget (Service Console)",
    "notes": "Requires the Service Console app; decline reasons need a picklist if reporting on declines is in scope."
  },
  "dependencies": ["US-021c"]
}
```

Nothing in REQ-021 mentions presence statuses, capacity, decline windows
or supervisor reassignment. Type B pulls apart what is written; Type C
adds what the platform requires.

### Type B+C — both passes fire

```
REQ-024 | Cases should be routed by skill, and managers should be able
         to override the routing manually.
```

```
REQ-024 → Type B+C → 6 stories
  Pass 1: 2 needs — automated skills-based routing, manual manager override
  Pass 2: skills-based routing → 5 stories (Omni-Channel Routing);
          manual override stays atomic
  Story 1–5: as REQ-021 above
  Story 6: As a Service Manager, I want to manually reassign a routed case
           so that I can respond to escalations routing cannot anticipate
  Dependency chain: 1 → 2 → 3 → 4, 5; 6 depends on 3
```

Classifying this REQ as B alone loses four stories. Classifying it as C
alone loses the override. Running both passes is what catches it.
