# 📬 Job Application Tracker — Gmail × Notion Automation

> **Automated pipeline that scans your inbox daily, classifies job-related emails, and logs structured records into Notion — with zero manual input.**

---

## Pitch

Manually tracking job applications across dozens of emails is error-prone and time-consuming. This Make.com scenario eliminates that friction entirely: every morning, it searches your Gmail inbox for job-related signals, extracts structured data, applies intelligent status classification, and upserts clean records directly into a Notion database — all while staying within a minimal operation budget.

Built to demonstrate real-world automation engineering: API orchestration, noise filtering, data normalization, and cost-conscious scheduling.

---

## Key Features

| Feature | Details |
|---|---|
| 🔍 **Advanced Gmail Query Filtering** | Uses Gmail's search syntax to target relevant emails by keyword (`postulación`, `vacante`, `apply`, `application`, `interview`) while explicitly excluding noise sources (e.g., promotional senders, self-sent emails) |
| 🧹 **Data Cleaning with `ifempty` Logic** | Applies conditional fallback functions before writing to Notion — if a company name cannot be extracted from the subject line, a normalized default value is substituted, ensuring no field is left blank or malformed |
| 🏷️ **Dynamic Status Mapping** | Classifies each email automatically: detects the keyword `interview` to flag the record as **Interview Invitation**; all other matches default to **Application Confirmed** — enabling a real-time status pipeline |
| ⏱️ **Scheduled Daily Execution** | Runs once per day on a fixed schedule, batching all matches from the previous 24-hour window in a single scenario run to minimize operation consumption |
| 🔗 **Notion API Integration** | Creates or updates database entries with structured fields: Company, Position, Status, Date Received, and Source Email — ready for filtering, sorting, and Kanban-style tracking |
| 🛡️ **Noise Suppression** | Multi-layered filter logic prevents false positives from newsletters, automated marketing, and self-sent drafts from polluting the dataset |

---

## Flow Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        DAILY TRIGGER                            │
│              (Scheduled — once per day)                         │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                     GMAIL SEARCH MODULE                         │
│  Query: (postulación OR vacante OR apply OR application OR      │
│          interview) AND -from:me AND -from:temu.com AND ...     │
│  Window: last 24 hours                                          │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                     EMAIL ITERATOR                              │
│  Loops through each matching message                            │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    DATA CLEANING LAYER                          │
│  • Extract: Subject, Sender Name, Date                          │
│  • ifempty(senderName, "Unknown Company")                       │
│  • Normalize subject → position title candidate                 │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    STATUS CLASSIFICATION                        │
│  contains(subject, "interview")                                 │
│       ├── TRUE  → Status: "Interview Invitation"                │
│       └── FALSE → Status: "Application Confirmed"              │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                   NOTION UPSERT MODULE                          │
│  • Target Database: Job Applications                            │
│  • Fields written: Company · Position · Status · Date · Email  │
│  • Strategy: Create new entry per matched email                 │
└─────────────────────────────────────────────────────────────────┘
```

**Step-by-step breakdown:**

1. **Trigger** — The scenario activates once daily via Make.com's built-in scheduler.
2. **Gmail Search** — Executes an advanced query string against the Gmail API, filtering for job-related keywords and excluding known noise sources within the last 24 hours.
3. **Iterator** — Each returned message is processed individually through the pipeline.
4. **Data Cleaning** — Raw email metadata is parsed and normalized. The `ifempty` function guards every critical field against null values before downstream modules consume the data.
5. **Status Router** — A text-match condition on the email subject determines the application stage and sets the `Status` property accordingly.
6. **Notion Write** — A finalized, structured record is sent to the Notion API and logged into the tracking database.

---

## Using the Blueprint

This repository includes the Make.com scenario exported as a `.json` blueprint file.

### Prerequisites

- A [Make.com](https://make.com) account (Free tier is sufficient for daily use)
- A Gmail account with API access authorized via Make.com
- A Notion workspace with a database containing the following properties:

| Property Name | Type |
|---|---|
| `Company` | Title |
| `Position` | Text |
| `Status` | Select (`Application Confirmed`, `Interview Invitation`) |
| `Date Received` | Date |
| `Source Email` | Email |

### Import Steps

1. Download `blueprint.json` from this repository.
2. In Make.com, navigate to **Scenarios → Create a new scenario**.
3. Click the **three-dot menu (⋯)** → **Import Blueprint**.
4. Upload `blueprint.json` and confirm.
5. Open each module and **reconnect your credentials**:
   - **Gmail module**: Authorize with your Google account.
   - **Notion module**: Connect your Notion integration token and select your target database.
6. In the Gmail Search module, review and adjust the query string to match your language and preferred noise filters.
7. Set the scheduler to your preferred daily execution time.
8. Run once manually to validate the pipeline end-to-end.
9. Activate the scenario.

> **Note:** Make.com's free plan includes 1,000 operations/month. This scenario consumes approximately 3–10 operations per run depending on the number of matched emails, meaning daily execution remains well within the free tier for most users.

---

## Why This Matters

### Cost-Efficiency by Design

Most automation workflows are built without considering operational cost — modules fire on every event, regardless of frequency or volume. This scenario deliberately inverts that pattern:

- **Batch scheduling over event-driven triggers**: Instead of reacting to every incoming email, the scenario runs once per day and processes all matches in a single execution cycle. This reduces operation count by an order of magnitude compared to per-email triggers.
- **Query-level noise filtering**: By pushing complexity into the Gmail query string itself — rather than handling it with extra filter modules inside Make — the scenario avoids consuming operations on irrelevant emails. Filtering happens *before* data enters the pipeline.

### Defensive Data Engineering

Real-world email data is inconsistent. Subject lines vary, senders omit names, and formatting differs across companies and locales. This scenario accounts for that:

- **`ifempty` guards** ensure no Notion record is created with missing required fields.
- **Fallback values** are semantically meaningful (e.g., `"Unknown Company"`) rather than empty strings, preserving database integrity and filter usability in Notion.
- **Status defaulting** means an unclassified email never produces an invalid state — it always resolves to a known, valid value.

### Engineering Mindset Over No-Code Mindset

This is not a drag-and-drop workflow. It reflects deliberate systems thinking:

- Separation of concerns between search, transformation, classification, and persistence.
- Idempotent-friendly design (each email produces exactly one record).
- Minimal external dependencies and no custom code required — the entire logic runs on Make.com's native modules.

---

## Tech Stack

- **Automation Platform**: [Make.com](https://make.com) (formerly Integromat)
- **Email Source**: Gmail API (via Make.com OAuth connector)
- **Database**: Notion API (via Make.com Notion connector)
- **Logic Layer**: Make.com native functions — `ifempty`, `contains`, text parsing operators

---

## Author

**José Joaquín**
Automation & Backend Engineering · Open to opportunities
[GitHub](https://github.com/jjrc163) · [LinkedIn](https://linkedin.com/in/josejoaquinrojas)

---

*This project was built as part of a personal automation portfolio to demonstrate practical API integration, workflow design, and engineering-first thinking applied to everyday productivity problems.*
