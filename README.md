# AI-Powered Leave Request Automation — n8n



A second-generation rebuild of an employee leave automation system, redesigned from a Make.com workflow into a self-hosted n8n workflow with an AI Agent as the orchestration layer.



The project automates employee validation, leave-policy checks, document analysis, manager notification, approval tracking, and idempotent approval/rejection confirmation — while being explicitly designed to degrade gracefully when any external dependency (a free-tier backend, a document-analysis call, the model itself) is slow or fails.



## Key Highlights



- Self-hosted n8n replacing the original Make.com implementation.

- n8n AI Agent coordinates validation, tool usage, notifications, and workflow execution.

- Two validated employee input paths are normalized into a common Agent payload.

- Stateful processing using Google Sheets as a lightweight data store.

- Dedicated document-analysis sub-workflow using Google Drive + Gemini, keeping binary/base64 data outside the Agent context, with explicit success/error branching.

- Approval/rejection link generation is delegated to a dedicated retry-safe sub-workflow rather than being called directly from the Agent, to absorb a Render free-tier backend's cold-start latency.

- Custom FastAPI backend generates signed approval/rejection links and handles manager confirmation requests.

- Idempotent approval processing prevents duplicate updates from repeated manager-link clicks.

- Pre-Agent validation failures (details mismatch) always end the workflow cleanly and are logged, regardless of whether the rejection email itself was deliverable.

- Post-Agent routing distinguishes "Agent succeeded but a step failed," "Agent itself failed," and "everything succeeded" — the first two alert an admin via Gmail.

- A fallback model is attached directly to the Agent node.

- Google OAuth + Service Account authentication configured manually for self-hosted n8n.

- ngrok exposes self-hosted n8n webhook endpoints to external systems.

- Agent is constrained by explicit rules and does not make the final qualified leave approval decision; the manager does.



---



## Repository Structure



This repo is a standalone V2 portfolio project and does not replace the original Make.com-based V1 repo — both remain up as separate projects.



```

/

├── README.md ← this file

├── docs/

│ ├── architecture.md ← full architecture diagram

│ └── system-prompt.md ← live n8n Agent system prompt

└── workflows/ ← n8n workflow JSON exports (added separately)

```



The FastAPI backend (`main.py`, `requirements.txt`) is unchanged from V1 — no code or dependency changes were made for this version — and stays deployed on Render as-is. It lives in the V1 repo rather than being duplicated here: **[link to V1 repo/backend]**.



---



## Problem Statement



Employee leave processing commonly involves repetitive coordination between employees, HR/administration, managers, spreadsheets, email, and supporting documents.



The workflow needed to handle:



- employee identity validation

- duplicate/previous-request checks

- email validation

- leave-policy validation

- supporting-document handling

- manager approval requests

- employee notifications

- approval-state tracking

- repeated approval-link clicks

- invalid-entry logging

- unreliable/slow external dependencies (free-tier backend cold starts, document-analysis failures, model outages)



The original version was built in Make.com. Its AI usage was limited mainly to producing structured outputs such as leave-reason summaries, urgency, and document analysis.



As the workflow grew, similar downstream nodes had to be duplicated across multiple branches, resulting in increasingly complex branching.



---



## Why I Rebuilt It



The main goal of this version was hands-on practice with agentic workflow engineering.



Instead of using an LLM as an isolated processing step, I wanted to understand how an AI Agent could operate as a controlled orchestration layer inside a real automation workflow — and, as the build matured, how to make that orchestration resilient to the kind of failures real external services actually produce (slow cold starts, API errors, model outages).



The rebuild also allowed me to work with:



- self-hosted n8n

- AI Agent tool calling

- stateful workflows

- webhook-based integrations

- sub-workflows

- OAuth configuration

- service accounts

- API failure modes and rate limits

- backend integration

- idempotency

- separation of deterministic and probabilistic logic

- retry-safe wrapper workflows around unreliable dependencies

- distinguishing step-level failure from agent-level failure for operational alerting



The project therefore represents an architectural evolution rather than a completely different business use case.



---



## Architecture



The full architecture diagram lives in [`/docs/architecture.md`](./docs/architecture.md), alongside the system prompt. At a high level: the request is routed through Path 1 or Path 2, normalized, handed to the AI Agent, which orchestrates Sheets, Gmail, document analysis, and the approval-link sub-workflow before routing to a final admin-alert or silent-success state depending on execution outcome.



Separately, once a request is placed in Approval Pending state, the manager's click on the approval/rejection link is handled by the FastAPI backend and an idempotency check in Sheets (see "Manager Approval & Idempotency" below) — this part is outside the Agent's scope.



---



## AI Agent Responsibility



The Agent receives a normalized JSON payload from the final "Edit Fields" node of either input path:



```

{{ JSON.stringify($json) }}

```



It determines the incoming path from the supplied payload and follows the corresponding rules. The Agent's job ends after full processing plus sending the employee their final status notification — it does not handle the manager's later approval/rejection click; that confirmation callback is handled separately by the backend.



**The Agent handles**



- path-specific workflow logic

- leave-policy validation after upstream email validation

- previous leave-status checks

- leave-reason summarization

- urgency classification

- document relevancy/authenticity interpretation

- tool selection and sequencing

- manager notification

- employee notification

- cache/state updates

- manager-record updates

- invalid-entry logging

- approval-link generation through the retry-safe backend sub-workflow tool



**The Agent does not**



- calculate leave days independently

- replace webhook values with cached values

- invent missing information

- approve a qualified leave request

- reject a request because a document appears suspicious or irrelevant



A valid request is placed into Pending / Approval Pending state and sent to the manager for the final decision.



---



## Input Paths



### Path 1 — Returning Employee



```

Main Webhook

→ Cache Memory Check

→ Employee ID match? (no → diverts to Path 2)

→ Full Name / Department / Email match?

no → Gmail rejection-notification attempt (continue-on-error)

→ both success and failure branches converge into

Invalid Entries log → workflow ends

yes ↓

→ Email Validator

→ Edit Fields (Path 1)

→ AI Agent

```



The cache is used for identity/history checks.



For returning employees, the previous request status can additionally trigger the 20-day rule:



- **"Pending"** → employee is informed to wait

- **"Approved"** → next leave must start at least 20 days after the previous end date

- **"Rejected"** → the 20-day rule does not apply



### Path 2 — Employee Database Fallback



```

Cache miss

→ Employee Database

→ Employee ID found?

no → workflow terminates immediately (no matching record,

so no notification is sent)

yes ↓

→ Full Name / Department / Email match?

no → same mechanism as Path 1: Gmail rejection-notification

attempt (continue-on-error) → Invalid Entries log →

workflow ends

yes ↓

→ Email Validator

→ Edit Fields (Path 2)

→ AI Agent

```



The Agent receives only the validated/usable employee context produced by the upstream path.



---



## Leave Policy Rules



The Agent applies the configured rules using the webhook-provided leave data.



- Start Date must be before End Date.

- Start Date must be at least 2 calendar days after the current date supplied by the workflow.

- Leave Days must not exceed 30 days.

- Supporting documentation is optional for fewer than 5 leave days.

- Supporting documentation is mandatory for 5 or more days.

- Path 1 approved-history requests must satisfy the 20-day gap rule.



The Agent never calculates or overrides the webhook's Leave Days value.



---



## Invalid Entries & Rejection Handling



**Pre-Agent (deterministic) rejections** — details-mismatch cases in either path:



A Gmail node attempts to notify the employee of the rejection, with continue-on-error enabled. Whether that email actually delivers or not, both branches converge into the same Invalid Entries log step, and the workflow ends cleanly either way. An email-validator check was deliberately not added before this send — its API cost is roughly the same as just attempting the Gmail send directly, so pre-validating for this low-value case isn't worth it.



The **Invalid Entries** sheet records: Employee ID, Full Name, Department, Email Address, Leave Type, Start Date, End Date, Reason for Leave, Supporting Document, Leave Days, Rejection Reason.



**Agent-driven rejections** — the Agent is instructed to call the same Invalid Entries logging tool in both Path 1 and Path 2 whenever a request fails a validation rule or the email check:



- **Invalid email** — the Email Validator output comes back invalid, non-deliverable, or disposable (either path).

- **Rule violations** — e.g. the Path 1 20-day gap rule is violated, or a mandatory supporting document is missing.



Each call must carry a strictly 5–8 word, factual, accurate rejection reason as its parameter, and the Agent must explain in 10–15 words immediately afterward why that tool was called. Once this tool is called, the request is considered rejected and the workflow stops there.



---



## Document Analysis Sub-workflow



The Agent does not receive raw binary or Base64 document content directly from Google Drive.



```

AI Agent

│ file URL

▼

HTTP Request Tool

│

▼

Document Analysis Workflow

│

├── Google Drive downloads file

│ │

│ success ──→ Gemini analyzes document ──→ Respond to Webhook:

│ │ { document_analysis_successful: true,

│ │ document_analysis: "<30-40 word

│ │ factual description string>" }

│ │

│ error ───────────────────────────────→ Respond to Webhook:

│ { document_analysis_successful: false }

▼

document_analysis result returned to Agent

```



This design avoids sending large Base64 strings through the Agent context, and ensures a Drive-download failure returns a clean, minimal error signal instead of breaking the sub-workflow.



**Division of labor between Gemini and the Agent is deliberate and narrow:** Gemini's only job is to describe the document's content in plain factual terms (30-40 words) and judge its authenticity with reasoning — it never connects the document to the leave request itself. The Agent is solely responsible for reading that factual description and explicitly cross-checking it against the stated leave reason to determine `document_relevancy`. A positive/authentic document description is not, by itself, evidence of relevancy — the Agent must not assume a match; it must actually compare content to stated reason. This distinction was added after a live-test bug where the Agent marked an illness-related leave request's document as relevant purely because the analysis was positive, without checking that the document actually matched the stated reason.



If `document_analysis_successful` comes back `false` (Drive download failed), the Agent does not invent `document_relevancy`/`document_authenticity` or make any approval/rejection call itself — it flags this in the Manager email instead, asking the manager to review the document directly.



---



## Approval-Link Generation (Retry-Safe Sub-workflow)



The Agent no longer calls the custom FastAPI backend directly. That direct HTTP tool was disconnected and replaced with a separate n8n workflow given to the Agent as a tool, using the same input fields (Request_ID, Employee_ID) but returning three outputs instead of two: `approval_link`, `rejection_link`, and `execution_status` (`success` or `failed`).



**Why:** the FastAPI backend is deployed on Render's free tier, so inactivity causes the server to be unresponsive for roughly 50–60 seconds on cold-start requests. Manual retry/wait logic can't be attached directly to a tool connected to the Agent, so this was moved into a dedicated sub-workflow instead.



**How it works:**



```

Parent workflow (Agent tool call)

│ Request_ID, Employee_ID

▼

HTTP Request → Render backend endpoint (continue-on-error enabled)

│

├── success ──→ Respond to Webhook:

│ { approval_link, rejection_link, execution_status: "success" }

│

└── error ──→ Respond to Webhook:

{ execution_status: "failed" }

```



Because the Agent's tool node naturally waits for the sub-workflow's output, the delayed cold-start response is absorbed transparently — the retry/wait behavior "just works" without needing custom wait logic inside the Agent's own tool call.



---



## Manager Approval & Idempotency



Once a request reaches Approval Pending, approval and rejection links (generated as above) are sent to the manager. When a manager clicks a link:



```

Manager

→ FastAPI

→ token validation

→ Backend Confirmation Webhook

→ Get Row from Manager Approval Records

```



The workflow checks the current status using the same "Request_ID".



**First valid click**



```

Approval Pending

→ update approval status

→ Respond to Webhook: "Ok"

```



**Repeated click**



```

Status != Approval Pending

→ do not modify the sheet

→ Respond to Webhook: "Already processed"

```



The backend uses this response to show either "The request has been submitted" or "The request is already processed" — preventing duplicate state updates caused by repeated manager-link clicks. This confirmation flow is handled entirely outside the Agent's scope.



**Manager-records update (Agent-side, step 7.5):** when the Agent calls the "Update the approval status in manager's records" tool, the Status column is hardcoded server-side to the exact value `Approval Pending` by the tool itself — the Agent never sets or alters Status. The Agent still fills in `Request_ID` and the supporting document accurately when calling the tool; it just never touches Status.



**Manager email formatting:** a live-test bug showed the manager email missing its greeting and breaking mid-sentence with poor spacing. The Agent's manager notification is now required to start with an appropriate greeting (e.g. "Dear Manager,") and use clean paragraph spacing with no random mid-sentence line breaks.



---



## Agent Output Routing & Fallback Handling



When the Agent node finishes, two fields are force-extracted from its output:



- `executed_every_step_successfully` (boolean)

- `workflow_reasoning_trail`



These don't require every *tool call* to have succeeded — they reflect whether the Agent itself completed its run.



| Case | What happens |

|---|---|

| Agent successful **and** `executed_every_step_successfully = true` | Nothing further — workflow ends silently |

| Agent successful **but** a step failed | Gmail node informs the admin of the Agent's reasoning trail and where execution failed |

| Agent node itself fails (e.g. model down) | Separate Gmail node notifies the admin of the failure |



A fallback model is connected directly to the Agent node itself (no separate mechanism), reducing the chance of a full Agent-level failure.



---



## Deterministic vs Agentic Logic



| Layer | Responsibility |

|---|---|

| n8n workflow | routing, lookups, deterministic integrations, retry-safe wrapper sub-workflows |

| AI Agent | orchestration, rule application, interpretation, tool sequencing |

| Gemini | document analysis / language interpretation |

| Google Sheets | state and records |

| FastAPI | signed approval-link handling |

| Manager | final approval/rejection |

| Admin (via Gmail alerts) | notified on partial or full Agent failure |



The manager's final decision is therefore not delegated to the LLM.



---



## Technology Stack



**Automation & Orchestration**

- n8n — self-hosted

- n8n AI Agent (with fallback model)

- Webhooks

- HTTP Request tools

- n8n sub-workflows (document analysis, approval-link generation)



**AI**

- Google Gemini API



**Google Workspace**

- Google Sheets

- Google Drive

- Gmail

- Google OAuth 2.0

- Google Service Account



**Backend**

- Python

- FastAPI

- Signed/tokenized approval links

- Render deployment (free tier — cold-start latency handled via sub-workflow)



**Infrastructure**

- ngrok for externally reachable webhook endpoints

- Environment variables and local secrets



---



## Authentication Design



Because the n8n instance is self-hosted, Google integrations were configured directly rather than relying on a managed platform's built-in application credentials.



Two authentication approaches are used:



**OAuth Client** — used where user-account access is required, particularly Gmail.



**Service Account** — used for selected automated Google Workspace operations such as Sheets and Drive.



This required creating and configuring the Google Cloud credentials independently.



---



## What I Learned



This rebuild was primarily a practical exercise in moving from conventional automation toward agentic systems.



Key learning areas:



- designing an AI Agent around explicit execution boundaries

- using tools instead of embedding every operation inside the prompt

- structuring stateful workflows

- separating deterministic validation from probabilistic reasoning

- designing idempotent webhook callbacks

- handling multiple input paths with normalized Agent context

- building specialized sub-workflows

- controlling LLM context size

- integrating a custom backend with n8n

- configuring OAuth and service accounts for self-hosted automation

- working with API rate limits, failures, and external dependencies

- understanding the additional operational responsibility introduced by self-hosting

- wrapping unreliable free-tier dependencies in retry-safe sub-workflows instead of building retry logic directly into Agent-connected tools

- distinguishing step-level failure from Agent-level failure for operational alerting

- using continue-on-error branching to keep a workflow ending cleanly even when a downstream side-effect (like a notification email) can't be guaranteed to succeed



---



## Project Evolution



### Version 1 — Make.com



The original implementation used Make.com as the primary orchestration platform.



AI was used mainly for:

- leave-reason summary

- urgency

- document relevancy/authenticity



As more business logic was added, repeated downstream actions had to be implemented across multiple branches.



### Version 2 — Self-hosted n8n



The new implementation changes the orchestration model:

- Make.com → self-hosted n8n

- isolated AI steps → tool-using AI Agent

- repeated branch logic → centralized Agent orchestration

- direct document processing → dedicated document-analysis sub-workflow

- manager confirmation → explicit idempotent webhook flow

- managed platform credentials → manually configured Google OAuth/service-account setup



### Version 2.1 — Reliability & Failure-Handling Refinements



Once the core V2 design was working, a round of hardening addressed failure modes surfaced during testing:

- direct Agent → FastAPI calls replaced with a retry-safe approval-link sub-workflow to absorb Render free-tier cold starts

- explicit success/error branching added to the document-analysis sub-workflow

- pre-Agent details-mismatch rejections made resilient to undeliverable emails via continue-on-error branching

- Agent output routing added to distinguish full success, partial step failure, and full Agent failure, alerting an admin in the latter two cases

- a fallback model attached to the Agent node



### Version 2.2 — Live-Test Fixes



A few issues surfaced only once real requests ran through the live agent:

- manager emails were missing their greeting and had inconsistent spacing — the prompt now mandates a proper greeting and clean paragraph formatting

- the Agent was found assuming document relevancy from a positive authenticity result instead of actually cross-checking the document's content against the stated leave reason — the prompt now draws a hard line between Gemini's role (factual 30-40 word description + authenticity judgment) and the Agent's own responsibility (relevancy judgment via explicit comparison)

- the Agent was found capable of touching the Status column on the manager-records update tool, even though that value is hardcoded server-side — the prompt now explicitly forbids the Agent from setting or altering Status there



The business problem remained the same; the engineering objective changed.



---



## Demo



['docs/demo-v2-path2-walkthrough.md'](.docs/demo-v2-path2-walkthrough.md)



---



## Repository Context



This project is an evolution of an earlier GitHub implementation built with Make.com.



The earlier repository contains:

- Make.com scenario blueprints

- the FastAPI backend ("main.py")

- the deployed backend on Render



The new n8n implementation reuses and integrates the backend rather than treating it as a completely separate system.



---



## Project Scope



This project is a learning-driven production-style automation prototype rather than a production HRIS replacement.



Its purpose is to demonstrate the engineering of:



automation + AI agents + external APIs + state + backend integration + failure-aware workflow design



rather than to replace a complete enterprise leave-management platform.


