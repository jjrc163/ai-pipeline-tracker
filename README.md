# 📬 AI-Powered Job Application Tracker — Gmail × Gemini × Notion × Telegram

> **An intelligent automation pipeline that reads your inbox, uses a Gemini AI model to extract and classify job data, logs structured records into Notion, and sends real-time Telegram alerts for interview invitations — all without a single line of server code.**

---

## Pitch

Most job seekers track applications in spreadsheets they forget to update, or miss interview invitations buried in their inbox. This Make.com scenario solves both problems simultaneously: it watches your Gmail for job-related emails, delegates understanding to a Gemini AI model, persists clean structured records into a Notion database, and fires an instant Telegram notification the moment an interview is detected.

The result is a fully automated, zero-maintenance hiring pipeline — built entirely on no-code APIs, consuming only **18 operations per execution** on Make.com's free tier.

---

## Key Features

| Feature | Details |
|---|---|
| 📨 **Event-Driven Gmail Trigger** | Watches for new emails matching `subject:(application OR applying OR interview)` plus full-text body scan — fires automatically on each new match, no polling required |
| 🤖 **Gemini AI Extraction** | Sends email subject + snippet to **Gemini 3.1 Flash Lite** with a strict system prompt that forces pure JSON output — extracts `cargo` (position), `empresa` (company), and `status` with no hallucinated fields |
| 🔒 **Constrained Status Classification** | The AI is constrained by prompt engineering to output exactly one of three valid states: `Applied`, `Interview`, or `Rejected` — no ambiguous or invented statuses can enter the pipeline |
| 🧩 **Typed JSON Parsing** | Gemini's raw text response is parsed against a named Make.com data structure (`Estructura_Vacantes_IA`) with three typed fields, enforcing schema integrity before any downstream write |
| 🗂️ **Notion Database Logging** | Creates a new page in a structured Notion database with fields: **Nombre** (position title), **Empresa**, **Status** (kanban-grouped select), and **Fecha** (date received) |
| 🔔 **Conditional Telegram Alert** | A route filter (`Solo Entrevistas`) checks if `status == "Interview"` — only then fires a Telegram bot message with position, company, and status, delivering real-time alerts exclusively for high-priority events |
| ⚙️ **Error-Resilient Scenario Config** | `autoCommit: true`, `maxErrors: 3`, and `autoCommitTriggerLast: true` ensure partial failures don't roll back committed records and the trigger cursor advances correctly |

---

## Module Architecture

| Step | Module | Details |
|:---:|---|---|
| **1** | 📨 **Gmail** — Watch New Emails | Query: `subject:(application OR applying OR interview)` + full body scan · Format: Full · Limit: 10 · Captures: subject, snippet, fromName, fromEmail, internalDate |
| ↓ | | |
| **2** | 🤖 **Gemini AI** — gemini-3.1-flash-lite | Input: `{{subject}}` + `{{snippet}}` · System prompt enforces JSON-only output · Returns: `{ "cargo": "...", "empresa": "...", "status": "..." }` · Allowed statuses: `Applied` / `Interview` / `Rejected` |
| ↓ | | |
| **3** | 🧩 **JSON Parse** — Estructura_Vacantes_IA | Deserializes Gemini's raw text into typed fields: `cargo` (text), `empresa` (text), `status` (text) · Enforces schema before any downstream write |
| ↓ | | |
| **4** | 🗂️ **Notion** — Create Page | Writes: Nombre → `cargo` · Empresa → `empresa` · Status → `status` · Fecha → `internalDate` |
| ↓ *(if status == Interview)* | | |
| **5** | 🔔 **Telegram** — Send Message | Filter `"Solo Entrevistas"` passes only when `status == "Interview"` · Sends: 🚀 cargo · 🏢 empresa · 📍 status |

**Step-by-step breakdown:**

1. **Gmail Trigger** — An instant trigger watches for new incoming emails matching job-related keywords in both subject and body. Fetches full content including subject, snippet, sender metadata, and internal date.

2. **Gemini AI Call** — The email subject and Gmail snippet are sent to Gemini 3.1 Flash Lite via Make.com's native connector. A carefully engineered system prompt instructs the model to act as a strict data extractor and return *only* a JSON object — no preamble, no explanation, no markdown fences. The model is explicitly forbidden from inventing status values beyond the three allowed options.

3. **JSON Parse** — Gemini's text output is deserialized against a typed Make.com data structure (`Estructura_Vacantes_IA`). This step decouples raw AI output from downstream modules, ensuring `cargo`, `empresa`, and `status` are accessible as proper typed variables.

4. **Notion Page Creation** — A new record is written to the job applications Notion database. The `Status` field maps directly to Notion's kanban-grouped select: `Applied` → To-do, `Interview` → In progress, `Rejected` → Complete.

5. **Conditional Telegram Alert** — A route filter evaluates `status == "Interview"`. If true, a Telegram bot message is dispatched immediately with position, company name, and status. Non-interview emails complete silently with no notification.

---

## Screenshots

> Live evidence of the pipeline running end-to-end in production.

<table>
  <tr>
    <td align="center" width="50%">
      <img src="screenshots/ss1.png" alt="Make.com scenario canvas showing the full 5-module pipeline" width="100%"/>
      <br/>
      <sub><b>① Make.com Scenario Canvas</b><br/>Complete pipeline view — Gmail → Gemini → JSON → Notion → Telegram</sub>
    </td>
    <td align="center" width="50%">
      <img src="screenshots/ss2.png" alt="Gemini module configuration showing the extraction prompt" width="100%"/>
      <br/>
      <sub><b>② Gemini Prompt & JSON Extraction</b><br/>System prompt enforcing strict JSON output with cargo, empresa and status fields</sub>
    </td>
  </tr>
  <tr>
    <td align="center" width="50%">
      <img src="screenshots/ss3.png" alt="Notion module mapping JSON fields to database columns" width="100%"/>
      <br/>
      <sub><b>③ Notion Module Field Mapping</b><br/>Each JSON field wired to its corresponding Notion database column</sub>
    </td>
    <td align="center" width="50%">
      <img src="screenshots/ss4.png" alt="Notion database populated with job application records" width="100%"/>
      <br/>
      <sub><b>④ Notion Database — Live Output</b><br/>Auto-populated records with status, company and position extracted by the AI</sub>
    </td>
  </tr>
</table>

---

## Operation Budget

| Module | Credits | Notes |
|---|:---:|---|
| Gmail: Watch New Emails | 1 | Fixed cost per run |
| Gemini: Create Completion | 6 | AI inference — highest single cost |
| JSON: Parse | 4 | Per field deserialization |
| Notion: Create Page | 2 | One record write |
| Telegram: Send Message | 5 | Conditional — only on Interview emails |
| **Standard email (Applied / Rejected)** | **13** | Telegram step skipped |
| **Interview email** | **18** | All 5 modules fire |

> Make.com's free plan includes **1,000 ops/month**. At 13–18 credits per matched email, the pipeline comfortably handles dozens of applications per month with headroom for manual test runs. Since the Telegram module is gated behind the `"Solo Entrevistas"` filter, most executions stay at the lower 13-credit cost.

---

## Notion Database Schema

| Property | Type | Values |
|---|---|---|
| `Nombre` | Title | Position title extracted by Gemini |
| `Empresa` | Rich Text | Company name extracted by Gemini |
| `Status` | Status (Select) | `Applied` · `Interview` · `Rejected` |
| `Fecha` | Date | Email's internal Gmail timestamp |

The `Status` field uses Notion's native kanban grouping: Applied → *To-do*, Interview → *In progress*, Rejected → *Complete*.

---

## Using the Blueprint

### Prerequisites

- A [Make.com](https://make.com) account (Free tier supported)
- A Gmail account authorized via Make.com's Google connector
- A Google AI Studio API key (for the Gemini connector in Make.com)
- A Telegram bot token + your personal Chat ID
- A Notion workspace with a database matching the schema above

### Import Steps

1. Download `blueprint.json` from this repository.
2. In Make.com, go to **Scenarios → Create a new scenario**.
3. Click the **three-dot menu (⋯)** → **Import Blueprint** → upload `blueprint.json`.
4. Reconnect all credentials:
   - **Gmail module**: Authorize with your Google account.
   - **Gemini module**: Connect your Google AI Studio API key.
   - **JSON Parse module**: Verify `Estructura_Vacantes_IA` data structure is recognized. If not, recreate it manually with three text fields: `cargo`, `empresa`, `status`.
   - **Notion module**: Select your Notion integration and target database.
   - **Telegram module**: Connect your bot token and confirm your Chat ID.
5. Run once manually using **Run once** to validate the full pipeline end-to-end.
6. Activate the scenario.

> **Tip:** To test the Gemini prompt in isolation, paste a sample email subject and snippet directly into the Gemini module and verify the output is valid JSON before running the full scenario.

---

## Why This Matters

### AI-in-the-Loop, Not AI-as-a-Crutch

Using Gemini for entity extraction instead of regex patterns or keyword filters is a deliberate architectural choice. Job email formats vary wildly across companies, ATS platforms, and languages. A rule-based parser would require constant maintenance. A constrained LLM prompt handles format variation naturally while remaining deterministic in its output schema — because the system prompt enforces the contract, not the parsing code.

The prompt engineering here is non-trivial: the model is instructed with explicit prohibition language ("ESTRÍCTAMENTE PROHIBIDO inventar otros estados"), a golden rule for each status value, and a rigid output format. This is prompt engineering used as a **data contract**, not a chat interface.

### Cost Efficiency by Design

Every architectural decision was made with operation consumption in mind:

- **Gemini Flash Lite** is used instead of a heavier model — sufficient for structured extraction tasks, with a fraction of the latency and API cost.
- **The Telegram alert is conditional** — it only fires for `Interview` statuses, avoiding a wasted operation on every `Applied` or `Rejected` email.
- **Gmail's native query syntax** filters noise at the source (keywords in both subject and body), so Gemini never processes irrelevant emails.
- **18 ops for a full multi-email run** stays well within the free tier even with daily execution.

### Production-Grade Error Handling

The scenario metadata reflects deliberate resilience configuration:

- `autoCommit: true` — each successfully processed email commits independently. A failure on email #3 doesn't roll back emails #1 and #2.
- `autoCommitTriggerLast: true` — the Gmail trigger cursor only advances after the last bundle is committed, preventing duplicate processing on retry.
- `maxErrors: 3` — the scenario tolerates up to 3 consecutive errors before halting, providing a buffer for transient API failures such as network timeouts or Gemini rate limits.

### Engineering Mindset Over No-Code Mindset

This pipeline demonstrates systems thinking applied to automation:

- **Separation of concerns**: Gmail handles search, Gemini handles understanding, JSON handles typing, Notion handles persistence, Telegram handles alerting. Each module has exactly one responsibility.
- **Schema-first design**: The `Estructura_Vacantes_IA` data structure acts as a formal interface between the AI layer and the storage layer.
- **Conditional routing**: Not every email produces every action — the filter is a lightweight substitute for a full router, keeping the scenario linear and readable.

---

## Tech Stack

| Layer | Tool |
|---|---|
| Automation Platform | [Make.com](https://make.com) (formerly Integromat) |
| Email Source | Gmail API — Make.com native connector |
| AI Extraction | Google Gemini 3.1 Flash Lite — Make.com Gemini connector |
| Data Parsing | Make.com JSON module with typed data structure |
| Database | Notion API — Make.com native connector |
| Alerting | Telegram Bot API — Make.com native connector |

---

## Author

**José Joaquín Rojas Chacón**
Automation & Backend Engineering · Open to opportunities

[![GitHub](https://img.shields.io/badge/GitHub-jjrc163-181717?logo=github&logoColor=white)](https://github.com/jjrc163)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-josejoaquinrojas-0A66C2?logo=linkedin&logoColor=white)](https://linkedin.com/in/josejoaquinrojas)

---

*Built as part of a personal automation portfolio to demonstrate practical AI integration, multi-API orchestration, prompt engineering, and engineering-first thinking applied to real productivity workflows.*
