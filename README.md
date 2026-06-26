# n8n Workflows

This README is an overview of what is currently in this repository.

## Top-Level Workflows

| Workflow File                                             | Brief Description                                                                                                                 |
| --------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| Favour_-_Digital_Trade_Promotion_Agent_sanitized.json     | Scheduled digital trade promotion agent with file upload intake (for RAG), MinIO storage flow, and PostgreSQL-backed processing. |
| HR_-_MentorMe_sanitized.json                              | Large mentorship lifecycle workflow handling pairing cycles, token generation, and recurring mentor/mentee email sequences.       |
| Leave_Management_System_sanitized.json                    | Webhook-driven leave request and approval workflow with employee-facing forms and email notifications.                            |
| Muna_-_TreasAlert_Agent_sanitized.json                    | Treasury alert workflow combining scheduled checks, transaction validation logic, and reporting actions (sheet/PDF/alerts).       |
| Orange_Group_-_Task_Management_Agent_sanitized.json       | Task operations workflow with webhook update endpoints, database operations, AI-assisted task handling, and status notifications. |
| Winnifred_-_Stock_Auditor_Agent_sanitized.json            | Stock audit and image-based verification workflow with email/webhook triggers and exception routing.                              |
| Winnifred_-_Transaction_verification_Agent_sanitized.json | Transaction verification flow for payload parsing, sheet logging, and email-based review/confirmation steps.                      |

## MySquad Workflows Overview

MySquad workflows are grouped under the MySquad_workflows folder by role.

| Group                     | Count | What It Covers                                                                                            |
| ------------------------- | ----: | --------------------------------------------------------------------------------------------------------- |
| Entry_Channels            |     4 | Incoming channel handlers and onboarding entry points (Telegram, WhatsApp, onboarding save/flow).         |
| Core_Orchestration        |     4 | Context resolution, owner/customer orchestration, and central configuration/update control.               |
| Agent_Modules_And_Tools   |    13 | Specialist agent modules and integration tools (Google, Speedy, RSS, brand/vision, dispatch tooling).     |
| Catalogue_And_Pages       |     8 | Catalogue page delivery, manager actions, link generation, and order confirmation page actions.           |
| Operations_And_Monitoring |     9 | Reliability and lifecycle operations: reminders, escalations, follow-up, fallback, analytics, and alerts. |

## Notes

- Personal Projects remains ignored by git.
- Workflow files are sanitized outputs intended for safe sharing/import.
