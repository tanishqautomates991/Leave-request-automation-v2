# End-to-End Workflow Demo — V2 (n8n)

## Scenario: 6-Day Leave Request with Irrelevant Supporting Document

This demo traces a realistic edge case: an employee requests **6 days of leave** (mandatory supporting documentation required per policy) but uploads an **irrelevant document** (a school project PDF instead of a medical certificate or legitimate leave-supporting document).

The AI Agent is explicitly instructed to:
1. **Never assume document relevancy just because it's authentic.** Gemini will describe the document accurately and judge its authenticity, but the Agent must cross-check that description against the stated leave reason.
2. **Flag mismatches to the manager** rather than rejecting outright or approving blindly.
3. **Let the manager make the final call** on whether the document suffices.

This demonstrates the V2.2 refinement where document-relevancy judgment is a critical Agent responsibility, not an assumption based on document authenticity.

---

## Step-by-Step Walkthrough

### Step 1: Form Submission (6-Day Leave + Irrelevant Document)

![Step 1: Form Submission](placeholder-step-1-form.png)

*Employee submits leave request via Google Form:*
- **Leave Reason:** "Personal time to prepare for a family event"
- **Start Date:** Sept 1  
- **End Date:** Sept 6  
- **Leave Days:** 6 (exceeds 5-day threshold → mandatory documentation)
- **Supporting Document:** Uploads `school-project.pdf` (a school assignment, completely unrelated to the stated reason)

---

### Step 2: Response Logged in Google Sheets

![Step 2: Response Logged in Sheets](placeholder-step-2-sheets.png)

*Form response appears in Google Sheets, triggering the main webhook.*

---

### Step 3: Path 1 Cache Check (Returning Employee)

![Step 3: Path 1 Cache Check](placeholder-step-3-cache-check.png)

*The workflow identifies this as a returning employee (Path 1), retrieves cache history:*
- Employee ID matches  
- Email and name match  
- Previous status: "Approved" (June leave request)  
- Previous end date: July 20  
- **20-day gap check:** Sept 1 is >20 days after July 20 → ✅ PASS

---

### Step 4: Email Validator Passes

![Step 4: Email Validator](placeholder-step-4-email-validator.png)

*Email Validator node returns: Valid, deliverable email.*

---

### Step 5: AI Agent Execution Begins

![Step 5: Agent Node Starts](placeholder-step-5-agent-start.png)

*Agent receives normalized payload:*
```json
{
  "employee_id": "EMP-1234",
  "full_name": "John Doe",
  "email": "john.doe@company.com",
  "start_date": "2024-09-01",
  "end_date": "2024-09-06",
  "leave_days": 6,
  "leave_reason": "Personal time to prepare for a family event",
  "document_url": "https://drive.google.com/...",
  "current_date": "2024-08-15"
}
```

---

### Step 6: Document Analysis Sub-workflow Triggered

![Step 6: Document Analysis Sub-workflow](placeholder-step-6-doc-analysis-start.png)

*Agent calls the document-analysis sub-workflow with the file URL.*

**Sub-workflow flow:**
- Google Drive downloads `school-project.pdf`  
- Gemini analyzes the binary content  
- Returns: `{"document_analysis_successful": true, "document_analysis": "This is a school project about renewable energy sourcing, containing diagrams and a thesis statement. Document appears authentic and well-structured, signed by the school."}`

---

### Step 7: Agent Reads Gemini's Analysis — Key Moment

![Step 7: Agent Reasoning (Critical Step)](placeholder-step-7-agent-relevancy-check.png)

*Agent sees:*
- ✅ Document is **authentic** (signed by school, well-structured)  
- ❌ Document content is **irrelevant** (renewable energy project ≠ personal time for family event)

**Agent's reasoning:**
> "Gemini says this is an authentic school project about renewable energy. But the leave reason is 'personal time for family event.' This document has no connection to the stated reason. Even though it's authentic, it doesn't prove the need for leave. **document_relevancy = false**."

Agent outputs:
```json
{
  "leave_reason_summary": "Personal time for family event preparation",
  "urgency": "Medium",
  "document_relevancy": false,
  "document_authenticity": true
}
```

---

### Step 8: Cache Update — Status Set to "Pending"

![Step 8: Cache Update](placeholder-step-8-cache-update.png)

*Agent calls Cache Memory Update tool to set the request status to "Pending".*

---

### Step 9: Approval-Link Sub-workflow Called

![Step 9: Approval-Link Generation](placeholder-step-9-link-gen.png)

*Agent calls the FastAPI approval-link sub-workflow:*
- Request_ID: 42  
- Employee_ID: "EMP-1234"  
- **Result:** `execution_status: "success"`, approval_link and rejection_link generated

---

### Step 10: Manager Email Sent (With Document Relevancy Flag)

![Step 10: Manager Email](placeholder-step-10-manager-email.png)

*Agent sends Manager Gmail notification:*

> **Subject:** Leave Approval Request - John Doe (6 Days)
>
> Dear Manager,
>
> A new leave request requires your approval.
>
> **Employee:** John Doe (EMP-1234)  
> **Leave Reason:** Personal time to prepare for a family event  
> **Duration:** Sept 1–6, 2024 (6 days)  
> **Urgency:** Medium  
> **Request Status:** Pending Your Approval
>
> **Document Uploaded:** Yes  
> **Document Analysis:**  
> The uploaded document is an authentic school project on renewable energy sourcing. However, **this document does not appear relevant to the stated leave reason** (personal family event). While the document is genuine, it does not provide supporting evidence for the requested leave. **Please review and decide whether to accept, request a different document, or reject the request.**
>
> **[APPROVE](#)** | **[REJECT](#)**
>
> Best regards,  
> Leave Automation System

---

### Step 11: Manager Records Updated

![Step 11: Manager Records Update](placeholder-step-11-manager-records.png)

*Agent calls "Update approval status in manager's records":*
- Path 1 → updates existing entry  
- Request_ID: 42  
- Status: "Approval Pending" (hardcoded, untouched by Agent)

---

### Step 12: Employee Notified of Pending Status

![Step 12: Employee Notification](placeholder-step-12-employee-notified.png)

*Agent sends Employee Gmail:*

> Dear John,
>
> Your leave request for Sept 1–6 has been received and is now **pending your manager's approval**.
>
> Leave Reason: Personal time to prepare for a family event  
> Leave Days: 6  
> Urgency: Medium
>
> You will receive a final notification once your manager has reviewed your request.
>
> Best regards,  
> Leave Management System

---

### Step 13: Manager Reviews and Approves (Accepting Responsibility)

![Step 13: Manager Decision](placeholder-step-13-manager-decision.png)

*Manager clicks the APPROVE link despite the document relevancy flag.* The manager is aware the document is not technically related but decides to approve on other grounds (e.g., employee history, verbal confirmation, company policy discretion).

**FastAPI backend processes:**
- Token validation → ✅ Valid  
- Status check → "Approval Pending"  
- Update → Status changed to "Approved"  
- Response: `{"status": "ok"}`

---

### Step 14: Idempotency Verified

![Step 14: Idempotency Check](placeholder-step-14-idempotent.png)

*Manager accidentally clicks the APPROVE link again.*

**FastAPI backend processes:**
- Token validation → ✅ Valid  
- Status check → "Approved" (no longer "Approval Pending")  
- **Result:** No update  
- Response: `{"status": "already_processed"}`

---

### Step 15: Final Approval Notification Sent to Employee

![Step 15: Employee Final Confirmation](placeholder-step-15-employee-approved.png)

*Employee receives final confirmation:*

> Dear John,
>
> **Your leave request has been APPROVED.**
>
> Leave Dates: Sept 1–6, 2024  
> Status: Approved
>
> Enjoy your leave!
>
> Best regards,  
> Leave Management System

---

## Key Takeaways

This scenario demonstrates three critical aspects of the V2 Agent design:

1. **Document Relevancy is an Active Judgment Call**  
   The Agent reads Gemini's factual description ("authentic school project") and explicitly compares it to the leave reason ("personal family event"). Authenticity ≠ relevancy.

2. **Flags Don't Reject**  
   Instead of auto-rejecting an irrelevant document, the Agent flags it in the manager's email and **lets the manager decide.** The manager remains in control of final approval.

3. **Agent Role is Orchestration, Not Decision-Making**  
   The Agent collects information (document analysis, policy validation), surfaces flags (document not relevant), and routes to decision-makers (manager). It never overrides human judgment.

---

## Workflow Execution Summary

| Step | Status | Duration |
|------|--------|----------|
| Cache Check | ✅ Pass | < 1s |
| Email Validator | ✅ Pass | < 1s |
| Document Analysis | ✅ Complete | 2–3s (Gemini API) |
| Agent Orchestration | ✅ Complete | 1–2s |
| Link Generation | ✅ Complete | < 1s |
| Manager Email | ✅ Sent | < 1s |
| Employee Email | ✅ Sent | < 1s |
| **Total** | **✅ Success** | **~5–8s** |

**Agent Output Fields:**
- `executed_every_step_successfully`: ✅ true  
- `workflow_reasoning_trail`: Document flagged as irrelevant; manager notified and approved anyway
-
