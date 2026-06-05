# Salesforce Persona Reference

Use this file to assign accurate, cloud-specific user roles to the "As a..." part of every story.
Never use generic roles like "User" or "Stakeholder".

---

## Sales Cloud

| Persona | When to use |
|---|---|
| Sales Rep | Prospecting, pipeline management, opportunity tracking, activity logging |
| Sales Manager | Team reporting, forecasting, pipeline reviews, approvals |
| Sales Director | Executive dashboards, territory management, revenue forecasting |
| Sales Operations | Process config, data hygiene, CRM administration |
| SDR / BDR | Lead qualification, outbound sequences, handoff to AE |
| Account Executive | Opportunity management, quoting, close plans |
| System Admin | Config, user management, automation rules |

---

## Service Cloud

| Persona | When to use |
|---|---|
| Service Agent | Case handling, customer communication, knowledge lookup |
| Senior Agent / Team Lead | Escalations, quality review, queue management |
| Service Manager | SLA reporting, team performance, escalation policy |
| Customer | Self-service actions — only when Experience Cloud is also in scope |
| System Admin | Queue config, routing rules, SLA setup, template management |
| Knowledge Manager | Article creation, review workflows, publication |

---

## Experience Cloud

| Persona | When to use |
|---|---|
| Portal Customer | Self-service case submission, order tracking, knowledge search |
| Partner User | Deal registration, lead sharing, MDF management |
| Community Member | Collaboration, peer support, forum participation |
| Portal Admin | Content management, member management, branding |
| System Admin | Template config, permission sets, SSO setup |

---

## Agentforce / Einstein

| Persona | When to use |
|---|---|
| Service Agent | Receiving handoffs from the AI agent, reviewing transcripts |
| End Customer | Interacting with the AI agent via chat, messaging, or voice |
| Agent Designer | Configuring topics, actions, and guardrails in Agent Builder |
| Service Manager | Monitoring agent performance, reviewing deflection rates |
| System Admin | Deploying agents, managing channels, data grounding config |

---

## Revenue Cloud / CPQ

| Persona | When to use |
|---|---|
| Sales Rep | Creating quotes, configuring products, submitting for approval |
| Deal Desk | Reviewing and approving complex or discounted quotes |
| Finance / Billing Team | Invoice management, revenue recognition, payment tracking |
| Contract Manager | Contract creation, amendments, renewals |
| System Admin | Product catalogue config, pricing rules, approval chain setup |

---

## Marketing Cloud / Account Engagement

| Persona | When to use |
|---|---|
| Marketing Manager | Campaign planning, audience segmentation, performance reporting |
| Email Marketing Specialist | Email build, A/B testing, list management |
| Demand Gen Marketer | Lead nurture programs, scoring rules, MQL definition |
| Sales Rep | Receiving MQL alerts, reviewing engagement history |
| Marketing Ops | Integration management, data sync, scoring model config |
| System Admin | Connector setup, user permissions, data hygiene |

---

## Multi-cloud

When a story spans clouds, use the primary persona and note the cross-cloud
context in the story itself.

Example:
`As a Sales Rep, I want to see a customer's open support cases on the Account
page so that I can have informed conversations without switching to Service Cloud.`
