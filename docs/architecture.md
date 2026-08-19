# Architecture

```
                         EMPLOYEE LEAVE REQUEST
                                  │
                                  ▼
                         Main Webhook / Intake
                                  │
                    ┌─────────────┴─────────────┐
                    │                           │
                 PATH 1                      PATH 2
              Cache Memory                Employee DB
               validation                  validation
                    │                           │
          ID no match → Path 2         ID not found → workflow
                    │                    terminates (no record,
          Details mismatch:              no notification)
          Gmail attempt (on error              │
          continues) → Invalid          Details mismatch:
          Entries log → End             same Gmail-attempt →
                    │                    Invalid Entries → End
                    │                           │
                    └─────────────┬─────────────┘
                                  │
                         Email Validator
                                  │
                           Edit Fields
                     (normalized Agent input)
                                  │
                                  ▼
                    n8n AI Agent  (+ fallback model)
                                  │
     ┌───────────┬────────────┬──┴───────────┬───────────────┐
     │           │            │               │               │
  Sheets       Gmail    Invalid Entries  Document Tool   Approval-Link
     │        (2 tools:    log tool           │           Sub-workflow
     │        manager /       │               ▼               │
     │        employee)       │        Document Analysis  HTTP → Render
     │           │            │         sub-workflow       backend, with
     │           │            │        (Drive → success:   continue-on-
     │           │            │         Gemini → respond;   error branching
     │           │            │         error: respond      → returns
     │           │            │         false directly)      approval_link,
     │           │            │               │               rejection_link,
     └───────────┴────────────┴───────────────┴───── execution_status
                                  │
                                  ▼
                     Agent output: forces
              executed_every_step_successfully +
                   workflow_reasoning_trail
                                  │
              ┌───────────────────┼───────────────────┐
              │                   │                   │
      Agent success +      Agent success but    Agent node itself
      all steps OK         a step failed         fails (model down)
              │                   │                   │
        workflow ends      Gmail → admin with    Gmail → admin
        silently           reasoning trail +     notifying failure
                            failure point
```

Separately, once a request is placed in Approval Pending state, the manager's click on the approval/rejection link is handled by the FastAPI backend and an idempotency check in Sheets (see the README's "Manager Approval & Idempotency" section) — this part is outside the Agent's scope.
