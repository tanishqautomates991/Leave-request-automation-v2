# End-to-End Workflow Demo — V2 (n8n)

## Scenario: New/Unique Employee Request (Path 2)

This demo traces a single request from a new employee — since no cache history exists for them, the workflow routes through **Path 2 (Employee Database Fallback)**.

---

[![Step 1: Form Submission](docs/screenshots/step-01-google-form-submission.png)](docs/screenshots/step-01-google-form-submission.png)

*Step 1: Form Submission*
> Employee submits the leave request through the Google Form, initiating the automation pipeline.

[![Step 2: Response Logged in Google Sheets](docs/screenshots/step-02-data-in-google-sheets.png)](docs/screenshots/step-02-data-in-google-sheets.png)

*Step 2: Response Logged in Google Sheets*
> The response lands in the sheet, triggering the n8n webhook.

[![Step 3: Path 2 Routing & Employee Database Lookup](docs/screenshots/step-03-path2-employee-db-lookup.png)](docs/screenshots/step-03-path2-employee-db-lookup.png)

*Step 3: Path 2 Routing & Employee Database Lookup*
> No matching cache entry exists for this employee, so the workflow routes to Path 2 — this is the Employee Database lookup node's response confirming the record.

[![Step 4: Edit Fields (Path 2) Output](docs/screenshots/step-04-edit-fields-path2-output.png)](docs/screenshots/step-04-edit-fields-path2-output.png)

*Step 4: Edit Fields (Path 2) Output*
> The normalized payload assembled for the Agent — this is exactly what the Agent receives to perform its task.

[![Step 5: Full End-to-End Execution Diagram](docs/screenshots/step-05-full-execution-diagram.png)](docs/screenshots/step-05-full-execution-diagram.png)

*Step 5: Full End-to-End Execution Diagram*
> The complete execution path across the workflow canvas for this run, from the Agent node onward.

[![Step 6: Agent Tool — Cache Memory Update](docs/screenshots/step-06-agent-tool-cache-memory-update.png)](docs/screenshots/step-06-agent-tool-cache-memory-update.png)

*Step 6: Agent Tool — Cache Memory Update Output*
> The Agent's Cache Memory Update tool call output.

[![Step 7: Agent Tool — Document Analysis Output](docs/screenshots/step-07-agent-tool-document-analysis-output.png)](docs/screenshots/step-07-agent-tool-document-analysis-output.png)

*Step 7: Agent Tool — Document Analysis Output*
> The Agent's document-analysis tool call output, as returned from the document analysis sub-workflow.

[![Step 8: Document Analysis Sub-workflow Diagram](docs/screenshots/step-08-document-analysis-subworkflow-diagram.png)](docs/screenshots/step-08-document-analysis-subworkflow-diagram.png)

*Step 8: Document Analysis Sub-workflow Diagram*
> Inside the sub-workflow: Google Drive downloads the file and Gemini analyzes it, producing the factual description and authenticity judgment.

[![Step 9: Agent Tool — Approval-Link Output](docs/screenshots/step-09-agent-tool-approval-link-output.png)](docs/screenshots/step-09-agent-tool-approval-link-output.png)

*Step 9: Agent Tool — Approval-Link Output*
> The Agent's approval-link tool call output — the FastAPI backend responds and the sub-workflow classifies the result as `"success"` for this run.

[![Step 10: Approval-Link Sub-workflow Diagram](docs/screenshots/step-10-approval-link-subworkflow-diagram.png)](docs/screenshots/step-10-approval-link-subworkflow-diagram.png)

*Step 10: Approval-Link Sub-workflow Diagram*
> Inside the sub-workflow: the HTTP Request node calls the FastAPI backend and returns the classified result (success / permanent / transient) along with the approval and rejection links.

[![Step 11: Agent Tool — Manager Gmail](docs/screenshots/step-11-agent-tool-manager-gmail.png)](docs/screenshots/step-11-agent-tool-manager-gmail.png)

*Step 11: Agent Tool — Manager Gmail Output*
> The Agent's Manager Gmail tool call, sending the manager their notification.

[![Step 12: Agent Tool — Employee Gmail](docs/screenshots/step-12-agent-tool-employee-gmail.png)](docs/screenshots/step-12-agent-tool-employee-gmail.png)

*Step 12: Agent Tool — Employee Gmail Output*
> The Agent's Employee Gmail tool call, notifying the employee of their pending request.

[![Step 13: Agent Tool — Update Manager's Approval Record](docs/screenshots/step-13-agent-tool-update-manager-records.png)](docs/screenshots/step-13-agent-tool-update-manager-records.png)

*Step 13: Agent Tool — Update Manager's Approval Record Sheet*
> The Agent's tool call creating the new entry (Path 2) in the manager approval records sheet.

[![Step 14: Agent Final Node Output](docs/screenshots/step-14-agent-final-node-output.png)](docs/screenshots/step-14-agent-final-node-output.png)

*Step 14: Agent Final Node Output*
> The Agent node's final output, showing the forced `executed_every_step_successfully` and `workflow_reasoning_trail` fields.

[![Step 15: Manager's Email Inbox](docs/screenshots/step-15-manager-email-inbox.png)](docs/screenshots/step-15-manager-email-inbox.png)

*Step 15: Manager's Email Inbox*
> The notification email as it appears in the manager's inbox.

[![Step 16: Employee's Email Inbox](docs/screenshots/step-16-employee-email-inbox.png)](docs/screenshots/step-16-employee-email-inbox.png)

*Step 16: Employee's Email Inbox*
> The pending-confirmation email as it appears in the employee's inbox.

[![Step 17: Backend Confirmation Webhook Execution](docs/screenshots/step-17-backend-confirmation-webhook-execution.png)](docs/screenshots/step-17-backend-confirmation-webhook-execution.png)

*Step 17: Backend Confirmation Webhook Execution*
> The workflow execution triggered when the manager clicks the approval link — token validation and the confirmation webhook flow.

[![Step 18: Manager's Browser — Request Approved](docs/screenshots/step-18-manager-browser-approved.png)](docs/screenshots/step-18-manager-browser-approved.png)

*Step 18: Manager's Browser Screen*
> The confirmation screen the manager sees after approving the request.

[![Step 19: Idempotency Check — Repeated Click](docs/screenshots/step-19-idempotency-check-double-click.png)](docs/screenshots/step-19-idempotency-check-double-click.png)

*Step 19: Idempotency Verified on Repeated Click*
> The manager clicks the same approval link a second time; the backend detects the status is no longer "Approval Pending" and responds accordingly without modifying the sheet again.
