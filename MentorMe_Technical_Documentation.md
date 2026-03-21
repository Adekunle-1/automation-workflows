# MentorMe Program — Technical Documentation
> **Internal Use Only** | Version 1.0 | Orange Group

---

## Table of Contents
1. [System Overview](#1-system-overview)
2. [Database Schema](#2-database-schema)
3. [Quarterly Cycle Logic](#3-quarterly-cycle-logic)
4. [Automation Flows (n8n)](#4-automation-flows-n8n)
5. [Email Templates](#5-email-templates)
6. [Web Interface Spec](#6-web-interface-spec)
7. [AI Feedback Generation Logic](#7-ai-feedback-generation-logic)
8. [Escalation Logic](#8-escalation-logic)
9. [HR Dashboard Spec](#9-hr-dashboard-spec)
10. [Setup & Deployment Checklist](#10-setup--deployment-checklist)
11. [Open Items & Future Enhancements](#11-open-items--future-enhancements)

---

## 1. System Overview

### Architecture

```
Google Sheet (Pair source)
       ↓  [Manual import / webhook sync]
PostgreSQL Database
       ↓
n8n Automation Engine
  ├── Scheduled triggers (weekly)
  ├── Webhook triggers (user actions)
  └── Sub-workflows (email, AI, logging)
       ↓
Email (SMTP / SendGrid)     Web Interface (hosted)
       ↓                           ↓
  Mentor / Mentee            Mentor / Mentee
                                   ↓
                            HR Dashboard
```

### Core Principle
Every action a user takes (confirm meeting, submit goals, give feedback) is triggered from a **tokenised link in an email**. There is no login. The token in the URL identifies the person and the cycle. This removes all friction for end users.

### Technology Stack

| Component | Tool | Purpose |
|-----------|------|---------|
| Automation | n8n (self-hosted) | Workflow orchestration |
| Database | PostgreSQL | All operational data |
| Email | SMTP / SendGrid | All outbound communication |
| Web Interface | Custom (HTML/JS or Next.js) | User-facing forms + HR dashboard |
| AI | Groq (LLM) via n8n | Personalised feedback question generation |
| Source of truth | Google Sheets | Initial pair import |

---

## 2. Database Schema

### 2.1 `mentorship_pairs`
Stores all mentor-mentee pairings. Source: Google Sheet import.

```sql
CREATE TABLE IF NOT EXISTS mentorship_pairs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  mentee_name VARCHAR(255) NOT NULL,
  mentee_email VARCHAR(255) NOT NULL,
  mentor_name VARCHAR(255),
  mentor_email VARCHAR(255),
  cohort_start_date DATE NOT NULL,
  status VARCHAR(50) DEFAULT 'active',  -- active | paused | completed
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

**Notes:**
- Pairs with no mentor assigned are still inserted with `mentor_name` and `mentor_email` as NULL
- `cohort_start_date` drives all quarterly scheduling
- When a mentor is manually assigned later, updating this record will trigger onboarding for that pair

---

### 2.2 `quarterly_cycles`
One record per pair per quarter. Created at the start of each quarter.

```sql
CREATE TABLE IF NOT EXISTS quarterly_cycles (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  pair_id UUID REFERENCES mentorship_pairs(id) ON DELETE CASCADE,
  quarter INT NOT NULL CHECK (quarter BETWEEN 1 AND 4),
  cycle_start_date DATE NOT NULL,
  current_week INT DEFAULT 1,
  meeting_confirmed_mentee BOOLEAN DEFAULT FALSE,
  meeting_confirmed_mentor BOOLEAN DEFAULT FALSE,
  meeting_confirmed_at TIMESTAMPTZ,
  meeting_date DATE,
  status VARCHAR(50) DEFAULT 'pending',
  -- Status values:
  -- pending        → waiting for meeting confirmation
  -- confirmed      → both parties confirmed meeting
  -- feedback_sent  → feedback emails dispatched
  -- complete       → feedback received from both
  -- escalated      → week 10+ with no confirmation
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(pair_id, quarter)
);
```

---

### 2.3 `goals`
Goals submitted by mentee and mentor at the start of each quarter.

```sql
CREATE TABLE IF NOT EXISTS mentorship_goals (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  pair_id UUID REFERENCES mentorship_pairs(id) ON DELETE CASCADE,
  cycle_id UUID REFERENCES quarterly_cycles(id) ON DELETE CASCADE,
  quarter INT NOT NULL,
  submitted_by VARCHAR(10) NOT NULL CHECK (submitted_by IN ('mentee', 'mentor')),
  goal_1 TEXT,
  goal_2 TEXT,
  goal_3 TEXT,
  additional_context TEXT,   -- mentor: "how I plan to support"
  submitted_at TIMESTAMPTZ DEFAULT NOW()
);
```

**Notes:**
- Mentee submits up to 3 career/development goals
- Mentor submits their support plan (not goals per se, but intentions)
- These are fed directly into the AI prompt when generating feedback questions

---

### 2.4 `feedback`
Stores all feedback responses per person per cycle.

```sql
CREATE TABLE IF NOT EXISTS mentorship_feedback (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  cycle_id UUID REFERENCES quarterly_cycles(id) ON DELETE CASCADE,
  pair_id UUID REFERENCES mentorship_pairs(id) ON DELETE CASCADE,
  quarter INT NOT NULL,
  submitted_by VARCHAR(10) NOT NULL CHECK (submitted_by IN ('mentee', 'mentor')),
  responses JSONB NOT NULL,
  -- Structure of responses JSONB:
  -- [
  --   { "question": "...", "type": "scale|text", "answer": "..." },
  --   ...
  -- ]
  ai_questions_used JSONB,   -- snapshot of questions that were generated
  submitted_at TIMESTAMPTZ DEFAULT NOW()
);
```

---

### 2.5 `access_tokens`
Tokenised links for all web interactions. No login required.

```sql
CREATE TABLE IF NOT EXISTS mentorship_tokens (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  token VARCHAR(128) UNIQUE NOT NULL,
  pair_id UUID REFERENCES mentorship_pairs(id) ON DELETE CASCADE,
  cycle_id UUID REFERENCES quarterly_cycles(id),
  person_email VARCHAR(255) NOT NULL,
  person_role VARCHAR(10) NOT NULL CHECK (person_role IN ('mentee', 'mentor')),
  token_type VARCHAR(50) NOT NULL,
  -- Token types:
  -- goal_submission
  -- meeting_confirmation
  -- feedback_submission
  used BOOLEAN DEFAULT FALSE,
  expires_at TIMESTAMPTZ NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

---

### 2.6 `escalations`

```sql
CREATE TABLE IF NOT EXISTS mentorship_escalations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  cycle_id UUID REFERENCES quarterly_cycles(id) ON DELETE CASCADE,
  pair_id UUID REFERENCES mentorship_pairs(id) ON DELETE CASCADE,
  week_triggered INT NOT NULL,
  escalation_sent_at TIMESTAMPTZ DEFAULT NOW(),
  resolved BOOLEAN DEFAULT FALSE,
  resolved_at TIMESTAMPTZ
);
```

---

## 3. Quarterly Cycle Logic

### 3.1 Quarter Schedule

Given `cohort_start_date`:

| Quarter | Starts | Focus |
|---------|--------|-------|
| Q1 | Week 1 (cohort_start_date) | Career Assessment & Objectives |
| Q2 | Week 13 (+12 weeks) | Skill & Leadership Development |
| Q3 | Week 25 (+24 weeks) | Skill & Leadership Development (continued) |
| Q4 | Week 37 (+36 weeks) | Measure of Success & Culture |

### 3.2 Within-Quarter Week Logic

```
Week 1  → Send intro email + goal submission links
Week 2  → Begin weekly meeting reminder emails
Week 3-9 → Continue weekly reminders (if not yet confirmed)
Week 10 → Escalate to HR if not confirmed
Week 11-12 → Final reminders + late feedback collection
End of Q → Close cycle, prep for next quarter
```

### 3.3 Meeting Confirmation Logic

```
Meeting confirmed = meeting_confirmed_mentee = TRUE 
                  AND meeting_confirmed_mentor = TRUE

When both = TRUE:
  1. Set meeting_confirmed_at = NOW()
  2. Set status = 'confirmed'
  3. Generate AI feedback questions for this pair
  4. Send feedback email to mentee (separate link)
  5. Send feedback email to mentor (separate link)
  6. Stop sending meeting reminder emails
```

### 3.4 Feedback Completion Logic

```
When feedback received from BOTH mentee AND mentor:
  1. Set cycle status = 'complete'
  2. Update HR dashboard data
```

---

## 4. Automation Flows (n8n)

### Flow 1: Weekly Scheduler
**Trigger:** Cron — every Monday at 8:00 AM (Africa/Lagos)

```
1. Query all active pairs from mentorship_pairs
2. For each pair:
   a. Calculate current week based on cohort_start_date
   b. Determine current quarter
   c. Check cycle status in quarterly_cycles
   d. Route to appropriate action:
      - Week 1       → Create cycle record + send intro email + goal links
      - Week 2-9     → If status = 'pending', send meeting reminder
      - Week 10      → If status = 'pending', escalate to HR
      - Week 11-12   → Send late reminder
      - New quarter  → Create new quarterly_cycle record
```

### Flow 2: Goal Submission Webhook
**Trigger:** POST to `/webhook/goals`

```
1. Validate token from URL
2. Check token not expired and not used
3. Save goals to mentorship_goals table
4. Mark token as used
5. Return success page
```

### Flow 3: Meeting Confirmation Webhook
**Trigger:** GET to `/webhook/confirm-meeting?token=xxx`

```
1. Validate token
2. Identify person_role (mentee or mentor)
3. Update quarterly_cycles:
   - If mentee: set meeting_confirmed_mentee = TRUE
   - If mentor: set meeting_confirmed_mentor = TRUE
4. Check if BOTH now confirmed
5. If YES:
   a. Update status = 'confirmed'
   b. Generate AI feedback questions (sub-workflow)
   c. Create feedback tokens for both parties
   d. Send feedback emails to both
6. Return confirmation page
```

### Flow 4: Feedback Submission Webhook
**Trigger:** POST to `/webhook/feedback`

```
1. Validate token
2. Save responses to mentorship_feedback table
3. Mark token as used
4. Check if partner has also submitted
5. If both submitted: update cycle status = 'complete'
6. Return thank you page
```

### Flow 5: AI Question Generation (Sub-workflow)
**Trigger:** Called by Flow 3 after meeting confirmed

```
1. Fetch pair's goals from mentorship_goals
2. Fetch quarter focus from config
3. Build prompt:
   - Quarter context + expected outcomes
   - Mentee's specific goals
   - Mentor's support plan
4. Call LLM (Groq) to generate questions
5. Return structured JSON:
   [
     { "question": "...", "type": "scale" },
     { "question": "...", "type": "text" }
   ]
6. Store questions in feedback table (ai_questions_used)
```

### Flow 6: Escalation Sender
**Trigger:** Called by Weekly Scheduler at Week 10

```
1. Query all cycles where:
   - current_week >= 10
   - status = 'pending'
   - No existing escalation record
2. Build escalation report (list of pairs)
3. Send email to HR_EMAIL
4. Insert records into mentorship_escalations
5. Update cycle status = 'escalated'
```

---

## 5. Email Templates

### Email 1: Program Introduction (Week 1)

**To:** Mentee and Mentor (separate emails)
**Subject:** Welcome to the Orange Group MentorMe Program 🎯

```
Hi [Name],

Welcome to the Orange Group MentorMe Program!

You've been paired with [Partner Name] as your [mentor/mentee] 
for this cohort. This program runs across 4 quarters and is 
designed to support your growth and development at Orange Group.

Here's what to expect:
- Quarterly 1:1 meetings with your [mentor/mentee]
- Goal setting at the start of each quarter
- Structured feedback after each meeting
- A dedicated HR team tracking your progress

To get started, please submit your goals for Q1 using the 
button below. This takes less than 5 minutes.

[SUBMIT YOUR Q1 GOALS →]

Questions? Reach out to HR at [HR_EMAIL].

Regards,
Orange Group HR Team
```

---

### Email 2: Weekly Meeting Reminder

**To:** Mentee and Mentor (separate emails)
**Subject:** MentorMe Q[X] — Have you had your meeting? 📅

```
Hi [Name],

This is your Week [N] reminder for the MentorMe program.

Have you had your Q[X] meeting with [Partner Name] yet?

If yes, please confirm below so we can send you your 
feedback form. Both you and [Partner Name] need to confirm.

[✅ CONFIRM MEETING HAPPENED →]

If you haven't met yet, please schedule your meeting soon. 
This quarter's focus is: [Quarter Focus Area].

Regards,
Orange Group HR Team
```

---

### Email 3: Feedback Request

**To:** Mentee and Mentor (separate emails, different questions)
**Subject:** MentorMe Q[X] Feedback — Share Your Experience

```
Hi [Name],

Your meeting with [Partner Name] has been confirmed. 

Please take a few minutes to complete your Q[X] feedback. 
Your responses help us track your progress and improve 
the program for everyone.

[COMPLETE YOUR FEEDBACK →]

This form is personalised to your goals and will take 
approximately 5-8 minutes to complete.

Regards,
Orange Group HR Team
```

---

### Email 4: HR Escalation

**To:** HR_EMAIL
**Subject:** ⚠️ MentorMe Escalation — Week 10 Unconfirmed Meetings

```
Hi HR Team,

The following mentor-mentee pairs have not confirmed 
their Q[X] meeting as of Week 10. Please follow up.

[TABLE: Mentee | Mentor | Quarter | Weeks Overdue]

You can view the full dashboard at: [DASHBOARD_URL]

This is an automated alert from the MentorMe system.
```

---

## 6. Web Interface Spec

### Page 1: Goal Submission (`/goals?token=xxx`)

**Sections:**
- Header: "Q[X] Goal Setting — [Name]"
- Quarter context blurb (what this quarter is about)
- For Mentee:
  - Goal 1 (required): text input
  - Goal 2 (optional): text input
  - Goal 3 (optional): text input
  - "Anything else you'd like your mentor to know?" (optional textarea)
- For Mentor:
  - "How do you plan to support [Mentee Name] this quarter?" (textarea)
  - "What skills or areas will you focus on?" (textarea)
- Submit button
- On submit: thank you screen ("Goals saved! Look out for your meeting reminder each week.")

---

### Page 2: Meeting Confirmation (`/confirm?token=xxx`)

**Layout:** Single-screen, minimal
- "Hi [Name]! Did you have your Q[X] meeting with [Partner Name]?"
- Optional: "When did you meet?" (date picker)
- Big CTA: "✅ Yes, we met!"
- On submit: "Thank you! Once [Partner Name] also confirms, we'll send you your feedback form."

---

### Page 3: Feedback Form (`/feedback?token=xxx`)

**Layout:** Stepped form
- Header: "Q[X] Feedback — [Name]"
- Questions rendered dynamically from AI-generated JSON
- Question types:
  - **Scale (1-5):** Rendered as radio buttons or star selector
  - **Text:** Rendered as textarea
- Progress indicator (Question 3 of 7)
- Submit button
- On submit: "Thank you for your feedback! Your responses have been recorded."

---

### Page 4: HR Dashboard (`/dashboard`) — Password protected

**Sections:**

**Overview cards:**
- Total pairs | Confirmed this quarter | Feedback received | Escalations

**Pairs table:**
- Mentee | Mentor | Quarter | Week | Meeting Status | Feedback Status | Actions

**Feedback viewer:**
- Click any pair → see their goals + feedback responses side by side

**Export:**
- Download CSV of all feedback for selected quarter

**Escalation log:**
- List of all escalations with resolution status

---

## 7. AI Feedback Generation Logic

### Prompt Structure

```
You are an HR assistant generating feedback questions for a 
mentorship program. Generate exactly 6-8 questions.

QUARTER: Q[X] — [Focus Area]
QUARTER EXPECTATIONS:
[Paste quarter-specific expectations from framework]

MENTEE GOALS FOR THIS QUARTER:
1. [goal_1]
2. [goal_2]
3. [goal_3]

MENTOR'S SUPPORT PLAN:
[additional_context from mentor]

PERSON RECEIVING THIS FORM: [mentee | mentor]

INSTRUCTIONS:
- For mentee: ask about their own progress against their goals
- For mentor: ask about how they supported the mentee's goals
- Mix scale questions (1-5) and open text questions
- Be specific — reference their actual goals, not generic platitudes
- Keep questions concise and conversational
- Return ONLY valid JSON, no preamble:

[
  { "question": "...", "type": "scale" },
  { "question": "...", "type": "text" }
]
```

### Per-Quarter Context Injected

**Q1 — Career Assessment & Objectives:**
```
Expected outcomes: Mentee has set 1-3 realistic goals aligned with 
org objectives. A one-year action plan exists. First 1:1 meeting 
has been productive. Mentor has helped establish development plan.
```

**Q2 & Q3 — Skill & Leadership Development:**
```
Expected outcomes: Mentee is actively developing skills using the 
70/20/10 framework (70% on-the-job, 20% through relationships, 
10% formal learning). Mentor has identified training resources, 
helped expand network, and provided feedback on intangibles 
(attitude, work ethic, enthusiasm).
```

**Q4 — Measure of Success:**
```
Expected outcomes: Mentee can assess achievement of Q1 goals. 
Both parties can reflect on the program's impact. Culture, 
engagement, and productivity improvements are visible.
```

---

## 8. Escalation Logic

### Trigger Conditions
- `current_week >= 10`
- `status = 'pending'` (not yet confirmed)
- No existing escalation record for this cycle

### Escalation Behaviour
1. HR email sent with table of unconfirmed pairs
2. Record inserted into `mentorship_escalations`
3. Cycle status updated to `'escalated'`
4. Weekly reminders continue until Week 12
5. At Week 12, final reminder sent and cycle closes regardless

### HR Email Contains
- Mentee name + email
- Mentor name + email
- Quarter number
- Number of weeks overdue
- Link to dashboard

---

## 9. HR Dashboard Spec

### Authentication
- Simple username/password (HR team only)
- Session-based, no OAuth required at MVP

### Data Displayed

#### Summary Cards
```
Active Pairs | Meeting Confirmed (%) | Feedback Complete (%) | Open Escalations
```

#### Main Table Columns
```
Mentee Name | Mentor Name | Quarter | Current Week | 
Meeting Status | Mentee Feedback | Mentor Feedback | Actions
```

#### Colour Coding
- 🟢 Complete
- 🟡 In progress / pending
- 🔴 Escalated / overdue

#### Feedback Viewer
- Side-by-side: Mentee goals vs Mentor support plan
- Side-by-side: Mentee feedback vs Mentor feedback
- Questions shown alongside answers

#### Export
- Filter by quarter
- Download CSV
- Columns: all fields from feedback + pair info

---

## 10. Setup & Deployment Checklist

### Database
- [ ] Postgres instance provisioned
- [ ] Run all 6 `CREATE TABLE IF NOT EXISTS` statements
- [ ] Add indexes on `pair_id`, `cycle_id`, `token`

### n8n
- [ ] Import workflow JSON
- [ ] Set environment variables:
  - `POSTGRES_CONNECTION_STRING`
  - `SMTP_HOST`, `SMTP_USER`, `SMTP_PASS`
  - `GROQ_API_KEY`
  - `HR_EMAIL`
  - `WEB_BASE_URL` (base URL of web interface)
- [ ] Activate all workflows
- [ ] Test webhook URLs are publicly accessible

### Google Sheet Import
- [ ] Export current pairings as CSV
- [ ] Import into `mentorship_pairs` table
- [ ] Set `cohort_start_date` for this cohort
- [ ] Manually assign missing mentors and update records

### Web Interface
- [ ] Deploy goal submission page
- [ ] Deploy meeting confirmation page
- [ ] Deploy feedback form page
- [ ] Deploy HR dashboard (with auth)
- [ ] Test all tokenised flows end-to-end

### Pre-launch
- [ ] Send test intro email to HR team
- [ ] Confirm all links work in email clients
- [ ] Confirm feedback form renders correctly on mobile
- [ ] Brief HR team on dashboard usage

---

## 11. Open Items & Future Enhancements

### Open Items (Must resolve before launch)
| # | Item | Owner |
|---|------|-------|
| 1 | Confirm `cohort_start_date` for current cohort | HR |
| 2 | Provide HR escalation email address | HR |
| 3 | Assign missing mentors (Adaku, Mutiat, Nelson) | HR |
| 4 | Confirm SMTP credentials / email provider | Engineering |
| 5 | Confirm web hosting environment | Engineering |

### Future Enhancements
- **Mentor-side goal visibility:** Allow mentors to see mentee goals before the meeting (currently submitted independently)
- **Progress tracking:** Show mentees their own progress across all 4 quarters on a personal page
- **Cohort analytics:** Compare engagement rates across cohorts over time
- **Calendar integration:** Optional Google Calendar invite generation when meeting is confirmed
- **Nudge intelligence:** If a mentee confirms but mentor hasn't after 48hrs, send a targeted nudge to mentor only
- **Multi-language support:** Localisation for non-English speaking team members
