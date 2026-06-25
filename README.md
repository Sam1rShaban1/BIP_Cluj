# AI Study Assistant

> A multi-agent, context-engineered study planning system built entirely on **n8n** + **Ollama**, with a second n8n workflow dedicated to closed-loop, LLM-as-judge evaluation of every plan it produces.

Two workflows, one product:

|Workflow|File|Purpose|
|-|-|-|
|**Study Planner**|`Study_Planner.json`|Form intake → context engineering → multi-agent generation → Markdown report + ICS calendar → feedback capture|
|**Planner Evaluation**|`Planner_Evaluation.json`|Polls unevaluated feedback → comment analysis → LLM judge scoring → persists evaluation rows|

---

## Prerequisites

- **n8n** (self-hosted, v1.30+ with Data Tables support)
- **Ollama** running locally with models pulled:
  ```bash
  ollama pull qwen3:14b
  ollama pull gemma4:26b-a4b-it-qat
  ```
- Set this environment variable before starting n8n (required for `require('fs')` in Code nodes):
  ```bash
  export NODE_FUNCTION_ALLOW_BUILTIN=*
  ```

---

## 1. Architecture

```
On form submission (formTrigger)
        │
        ├──[Page 1] Student Name, Email, Timezone, Location
        ├──[Page 2] Study goal, Existing weekly commitments
        ├──[Page 3] Module handbook (file), Source code (file)
        ├──[Page 4] Your level (Beginner/Intermediate/Advanced)
        ├──[Page 5] VARK learning style (4 questions)
        ├──[Page 6] Calendar delivery preference
        │
        ▼
   form_data (Code) → parse_commitments (Code)
        │                    │
        ├──[IF has_handbook]── pdf_extractor ──┐
        │                                      │
        └──[IF has_code]──── code_extractor ───┤
                                               ▼
                                      context (Code)
                              [builds prompt + VARK tally + timezone]
                                               │
                                               ▼
                               plan_agent  (Agent · qwen3:14b)
                               tools: get_weather
                               [receives commitments in prompt]
                                               │
                                               ▼
                            wellness_agent (Agent · gemma4:26b-a4b-it-qat)
                                               │
                                               ▼
                          sustainability_agent (Agent · gemma4:26b-a4b-it-qat)
                                               │
                                               ▼
                               assemble_report (Code) → save_report (Code, writes .md)
                                               │
                                               ▼
                            ics_agent (Agent · qwen3:14b) → build_ics (Code, writes .ics)
                                               │
                                               ▼
                               Feedback (Form: Rating, Comments)
                                               │
                                               ▼
                          log_feedback (Code) → save_feedback_row (Data Table: student_feedback)
                                               │
                                               ▼
                  bundle_files (Code, base64) → Compression (zip) → Completion (Form, returnBinary)
```

Evaluation workflow (separate, independently triggered/scheduled):

```
get_feedback (Data Table: student_feedback, filter evaluated = isFalse, returnAll)
        │
        ▼
comment_input (Code) ──► Comment Analysis Agent (Agent · gemma4:26b-a4b-it-qat)
        │
        ▼
judge_input (Code, parses analysis JSON, builds judge prompt)
        │
        ▼
Judge Agent (Agent · gemma4:26b-a4b-it-qat)
        │
        ▼
combine (Code, parses judge JSON) ──► save_evaluation_row (Data Table: plan_evaluations)
        │
        ▼
mark_evaluated (Data Table: student_feedback, sets evaluated = true)
```

---

## 2. Intake & Conditional Extraction

The form collects across 6 pages:

* **Page 1:** `Student Name` (required), `Email`, `Timezone` (dropdown, 16 IANA timezones), `Location` (dropdown: London, Berlin, Paris, New York, etc.)
* **Page 2:** `Study goal` (textarea), `Existing weekly commitments` (textarea, one per line, e.g. `Monday 09:00-13:00 Lectures`)
* **Page 3:** `Module handbook` (file, optional), `Source code` (file, optional)
* **Page 4:** `Your level` — dropdown: `Beginner | Intermediate | Advanced`
* **Page 5:** 4 VARK learning style questions (each a dropdown of `Visual | Auditory | Reading/Writing | Kinesthetic`)
* **Page 6:** Calendar delivery preference

File uploads are **optional**, so extraction is gated by two `IF` nodes. Both branches converge back into `parse_commitments` → `context` regardless of which files were uploaded.

---

## 3. Context Engineering (`context` node)

Every heterogeneous input is normalized into **one prompt object**:

* **Defensive extraction.** `safe(fn, fallback)` wraps every upstream lookup — missing files degrade to placeholders.
* **Context-window management.** Source code is truncated to 8,000 chars.
* **VARK aggregation.** Four learning-style answers tallied into `Visual / Auditory / Reading-Writing / Kinesthetic`; dominant style extracted.
* **Commitments injection.** Parsed weekly commitments (from `parse_commitments`) are appended as a structured block the agent must work around.
* **Timezone passthrough.** The student's selected timezone is forwarded for ICS generation.

---

## 4. Multi-Agent Pipeline

### 4.1 `plan_agent` — Planning Agent (Qwen3 14B)

**Tools attached:** `get_weather` (Open-Meteo forecast, contextual to student's city).

**System prompt enforces:**

1. Works around existing weekly commitments (embedded in prompt).
2. Backwards planning from assessment deadlines/weights.
3. Three-tier difficulty calibration (Beginner/Intermediate/Advanced).
4. VARK-driven medium selection (diagrams for Visual, talk-throughs for Auditory, etc.).
5. Source code treated as hands-on material.
6. Hallucination guard: *"Do not invent deadlines not present in the handbook."*

**Output contract:** clean Markdown — summary, `Weekly Milestones`, `Detailed Schedule` table, `Risks & assumptions`.

### 4.2 `wellness_agent` (Gemma4 26B)

Input is only `plan_agent`'s output. Assesses burnout risk and suggests breaks tied to actual schedule slots.

### 4.3 `sustainability_agent` (Gemma4 26B)

Same input contract. Module-specific eco-suggestions only — generic tips disallowed.

### 4.4 `ics_agent` — Calendar Extraction Agent (Qwen3 14B)

Input is the assembled report. Returns a **structured JSON object** with two fields:

```json
{
  "events": [{"day":"Monday","start":"09:00","end":"11:00","title":"Algorithms review"}],
  "weeks": 8
}
```

- `events`: array of study sessions with day, start/end times (24h), and title
- `weeks`: plan duration extracted from the plan text

---

## 5. Report Assembly & Calendar Generation

**`assemble_report`** stitches three agent outputs into one Markdown document.

**`save_report`** persists to disk (`/data/flagged/plan-<name>-<timestamp>.md`).

**`build_ics`** generates a standards-compliant `.ics` file:

* Parses structured JSON output from `ics_agent` (with fallback for plain arrays).
* **Default end time:** if the LLM omits `end`, defaults to start + 1 hour.
* **ISO week anchoring:** `dateForDow()` finds this week's Monday, then offsets to the target day — events anchor to the correct week, not "next" from today.
* **Timezone support:** `DTSTART` and `DTEND` include `TZID` from the student's selected timezone.
* **Configurable duration:** `RRULE:FREQ=WEEKLY;COUNT=<weeks>` uses the plan duration extracted by `ics_agent` (not hardcoded 12).
* **Input validation:** invalid weekday names, missing times, and malformed entries are silently skipped.
* **Error handling:** file I/O wrapped in try/catch — disk errors return a null path instead of crashing.
* **RFC-5545 escaping:** `esc()` handles `\`, `;`, `,`, newlines.

**`bundle_files`** re-reads the `.ics` from disk, base64-encodes both files, and emits them as n8n binary properties for zipping.

---

## 6. Feedback Capture (n8n Data Tables)

The `Feedback` form (Rating 1–5 + Comments) fires after the planner finishes.

**`log_feedback`** assembles the row — including the full `context.prompt` (truncated to 12,000 chars) and the full assembled report.

**`save_feedback_row`** writes to the `student_feedback` Data Table.

---

## 7. Evaluation Workflow

Runs separately, polls unevaluated feedback:

1. **`get_feedback`** — Data Table query, `evaluated == isFalse`.
2. **`comment_input`** — wraps comment in labeled block.
3. **`Comment Analysis Agent`** — extracts sentiment, themes, summary.
4. **`judge_input`** — builds judge prompt with grounding inputs + comment analysis.
5. **`Judge Agent`** — scores `grounding`, `calibration`, `actionability`, `honesty` (1–5 each).
6. **`combine`** — parses judge JSON with fallback for malformed output.
7. **`save_evaluation_row`** — writes to `plan_evaluations` Data Table.
8. **`mark_evaluated`** — sets `evaluated = true` on source row.

---

## 8. Engineering Patterns

* **`safe(fn, fallback)` wrapper** — every optional upstream field is non-fatal.
* **Robust LLM-output parsing** — three recovery strategies: direct parse → substring parse → fallback, always with typed defaults.
* **Strict agent separation** — wellness/sustainability agents never see the original handbook.
* **Idempotent evaluation** — `evaluated` boolean + filtered get/update, no dedupe logic needed.
* **Disk round-tripping** — `build_ics` writes, `bundle_files` reads back by path, avoiding binary passthrough issues.

---

## 9. Technology Stack

|Component|Technology|
|-|-|
|Workflow engine|n8n (v1.30+)|
|LLM runtime|Ollama (local)|
|Planning / calendar model|`qwen3:14b`|
|Wellness / sustainability / evaluation|`gemma4:26b-a4b-it-qat`|
|Agent orchestration|`@n8n/n8n-nodes-langchain.agent`|
|Weather API|Open-Meteo (`httpRequestTool`)|
|Structured persistence|n8n Data Tables (`student_feedback`, `plan_evaluations`)|
|Report format|Markdown|
|Calendar format|ICS (RFC 5545, weekly RRULE, configurable duration)|
|Packaging|`compression` node (zip), `returnBinary` form completion|
|Glue logic|JavaScript (Code nodes)|

---

## 10. Repository Structure

```
.
├── Study_Planner.json        # main intake → multi-agent → report/ICS → feedback workflow
├── Planner_Evaluation.json   # standalone LLM-judge evaluation workflow
└── README.md
```

---

## 11. Future Work

* RAG over a vector store for handbook retrieval instead of full-text injection
* LMS integration for direct handbook/deadline ingestion
* Adaptive rescheduling when the wellness agent flags burnout
* A dashboard over the `plan_evaluations` table for longitudinal quality tracking
* Multi-language plan generation
* Email delivery of the report/ICS bundle

---

## Authors

**Alessia Cordea** · **Manas Samant** · **Samir Shabani** · **Kumar Arpit**

AI · Workflow Automation · n8n · LLM Applications · Context Engineering
