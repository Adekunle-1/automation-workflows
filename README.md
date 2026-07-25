# n8n Workflows

This README is an overview of what is currently in this repository.

## Top-Level Workflows

| Workflow File                  | Purpose                                                                                                                                                                                                               | Business Impact                                                                                                                                                                                                                  |
| ------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `DigitalTradeBot.json`       | WhatsApp customer engagement agent. Ingests documents via webhook, generates vector embeddings for semantic search, maintains conversation history and customer profiles in PostgreSQL.                               | Enables conversational commerce on WhatsApp. Customers discover products and promo codes through natural language search against uploaded documents (catalogs, product sheets), reducing friction compared to manual browsing.  |
| `MentorshipProgram.json`     | Quarterly mentorship lifecycle automation. Tracks active mentor-mentee pairs through checkpoints, generates goal-submission tokens, auto-escalates non-responsive pairs at grace, reminder, and escalation intervals. | Ensures consistent engagement in mentorship programs; prevents ghosting; automatic escalation reduces manual follow-up burden.                                                                                                   |
| `LeaveApprovalEngine.json`   | 3-tier leave request workflow: employee submission → manager approval → HR validation with balance deduction. Each stage triggers notifications and state updates.                                                  | Centralizes leave requests, prevents over-allocation, creates audit trail, reduces spreadsheet chaos and manual balance tracking.                                                                                                |
| `ProcurementVisibility.json` | Procurement request intake with HTML-based approval decision links. Captures requirement dates, quantity, unit price, and purpose. Routes approved requests forward, rejected requests back to requester.             | Creates visibility into procurement lead times; requirement dates enable capacity planning; decision context reduces approval re-work.                                                                                           |
| `TaskUpdateEngine.json`      | Email-driven task management via authenticated HTML forms. Auto-generates 30-day tokens per task, prioritizes by urgency (overdue → due-soon → upcoming), sends periodic digests scored by deadline proximity.      | Reduces status-update friction (no login required), concentrates urgent tasks in visibility, prevents deadline slippage via automated priority scoring.                                                                          |

## MySquad Workflows Overview

MySquad is an SMS/WhatsApp-first customer engagement platform for SMEs. It orchestrates multi-channel conversations, AI-powered content generation, automated customer lifecycle workflows, and real-time notification delivery. MySquad workflows are grouped under the MySquad_workflows folder by role.

| Group                     | Count | What It Covers                                                                                            |
| ------------------------- | ----: | --------------------------------------------------------------------------------------------------------- |
| Entry_Channels            |     4 | Incoming channel handlers and onboarding entry points (Telegram, WhatsApp, onboarding save/flow).         |
| Core_Orchestration        |     4 | Context resolution, owner/customer orchestration, and central configuration/update control.               |
| Agent_Modules_And_Tools   |    13 | Specialist agent modules and integration tools (Google, Speedy, RSS, brand/vision, dispatch tooling).     |
| Catalogue_And_Pages       |     8 | Catalogue page delivery, manager actions, link generation, and order confirmation page actions.           |
| Operations_And_Monitoring |     9 | Reliability and lifecycle operations: reminders, escalations, follow-up, fallback, analytics, and alerts. |

## Notes

- Workflow files are sanitized outputs intended for safe sharing/import.
