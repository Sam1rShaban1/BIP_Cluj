# AI Study Assistant

> A multi-agent, context-engineered study planning system built entirely on \*\*n8n\*\* + \*\*Ollama\*\*, with a second n8n workflow dedicated to closed-loop, LLM-as-judge evaluation of every plan it produces.

Two workflows, one product:

|Workflow|File|Purpose|
|-|-|-|
|**Study Planner**|`Study\_Planner.json`|Form intake → context engineering → multi-agent generation → Markdown report + ICS calendar → feedback capture|
|**Planner Evaluation**|`Planner\_Evaluation.json`|Polls unevaluated feedback → comment analysis → LLM judge scoring → persists evaluation rows|

\---

## 1\. Architecture

```
On form submission (formTrigger)
        │
        ├──\[IF has\_handbook]── pdf\_extractor (extractFromFile) ──┐
        │                                                        │
        └──\[IF has\_code]────── code\_extractor (extractFromFile) ─┤
                                                                   ▼
                                                          context (Code)
                                                  \[builds prompt + VARK tally]
                                                                   │
                                                                   ▼
                                                   plan\_agent  (Agent · qwen3:14b)
                                                   tools: MCP Client, get\_weather
                                                                   │
                                                                   ▼
                                                wellness\_agent (Agent · gemma4:26b-a4b-it-qat)
                                                                   │
                                                                   ▼
                                             sustainability\_agent (Agent · gemma4:26b-a4b-it-qat)
                                                                   │
                                                                   ▼
                                                  assemble\_report (Code) → save\_report (Code, writes .md)
                                                                   │
                                                                   ▼
                                               ics\_agent (Agent · qwen3:14b) → build\_ics (Code, writes .ics)
                                                                   │
                                                                   ▼
                                                   Feedback (Form: Rating, Comments)
                                                                   │
                                                                   ▼
                                          log\_feedback (Code) → save\_feedback\_row (Data Table: student\_feedback)
                                                                   │
                                                                   ▼
                                    bundle\_files (Code, base64) → Compression (zip) → Completion (Form, returnBinary)
```

Evaluation workflow (separate, independently triggered/scheduled):

```
get\_feedback (Data Table: student\_feedback, filter evaluated = isFalse, returnAll)
        │
        ▼
comment\_input (Code) ──► Comment Analysis Agent (Agent · gemma4:26b-a4b-it-qat)
        │
        ▼
judge\_input (Code, parses analysis JSON, builds judge prompt)
        │
        ▼
Judge Agent (Agent · gemma4:26b-a4b-it-qat)
        │
        ▼
combine (Code, parses judge JSON) ──► save\_evaluation\_row (Data Table: plan\_evaluations)
        │
        ▼
mark\_evaluated (Data Table: student\_feedback, sets evaluated = true)
```

\---

## 2\. Intake \& Conditional Extraction

The form (`On form submission`, `n8n-nodes-base.formTrigger`, titled *"Import File"*) collects:

* `Student Name` (required), `Email`
* `Study goal` (required, textarea)
* `Module handbook` (file, optional)
* `Source code` (file, optional)
* `Your level` — dropdown: `Beginner | Intermediate | Advanced`
* 7 VARK questions, each a dropdown of `Visual | Auditory | Reading/Writing | Kinesthetic`

File uploads are **optional**, so extraction is gated by two `IF` nodes rather than running unconditionally:

```js
// has\_handbook
{{ $('On form submission').first().binary?.Module\_handbook?.fileName }}  →  notEmpty
// has\_code
{{ $('On form submission').first().binary?.Source\_code?.fileName }}     →  notEmpty
```

Only when a file is actually present does the flow route into `pdf\_extractor` / `code\_extractor` (`n8n-nodes-base.extractFromFile`). Both branches converge back into `context` regardless of which files were uploaded, so the planner always runs even with zero attachments.

\---

## 3\. Context Engineering (`context` node)

This is the single point where every heterogeneous input — form fields, extracted PDF text, extracted source code, and seven separate dropdown answers — is normalized into **one prompt object**. Key implementation details:

* **Defensive extraction.** A `safe(fn, fallback)` helper wraps every `$('NodeName').first().json...` lookup, so a missing handbook or missing code file degrades to a placeholder string (`'(no handbook provided)'`, `'(no source code provided)'`) instead of throwing and killing the execution.
* **Context-window management.** Extracted source code is hard-truncated: `codeText.slice(0, 8000)`.
* **VARK aggregation.** The seven learning-style answers are tallied into a 4-bucket object (`Visual / Auditory / Reading-Writing / Kinesthetic`); the dominant style is whichever key hits `Math.max(...)`, with ties joined as `"Visual / Kinesthetic"`. If no VARK question was answered, `dominant = 'Unspecified'`.
* **Single structured prompt.** Everything is concatenated into one templated string with explicit section headers (`STUDY GOAL`, `SELF-ASSESSED LEVEL`, `LEARNING PREFERENCES`, `MODULE HANDBOOK`, `SOURCE CODE`) — this is what `plan\_agent` actually receives as `{{ $json.prompt }}`. The dominant learning style and `email` are also forwarded as separate fields for downstream use.

This pattern — merge once, hand a fully-assembled context object to the agent rather than letting the agent pull from multiple loosely-related upstream nodes — is what the README's "Context Engineering" claim resolves to in practice.

\---

## 4\. Multi-Agent Pipeline

All four agents are `@n8n/n8n-nodes-langchain.agent` nodes running in `/no\_think` mode (Qwen3's non-reasoning fast path) against locally-hosted Ollama models. Each is single-purpose and stateless w.r.t. the others — they only see what's explicitly piped to them.

### 4.1 `plan\_agent` — Planning Agent (Qwen3 14B)

**Tools attached:** `MCP Client` (`@n8n/n8n-nodes-langchain.mcpClientTool`, connected to a local MCP endpoint at `http://localhost:5678/mcp/...`) and `get\_weather` (`n8n-nodes-base.httpRequestTool` hitting Open-Meteo's `forecast` endpoint, hardcoded to Berlin coordinates `52.52, 13.41`, 7-day daily forecast).

**System prompt enforces, in order:**

1. Tool-calling discipline — call `get\_weather` and `get\_calendar\_commitments` (via MCP) *before* writing the plan; never schedule over an existing calendar commitment; weather is only used for outdoor-break/location suggestions.
2. Backwards planning from the handbook's actual assessment deadlines/weights — heavier or sooner assessments get more allotted time.
3. Three-tier difficulty calibration (Beginner/Intermediate/Advanced) controlling depth, not just tone.
4. VARK-driven *medium* selection (diagrams/video for Visual, talk-throughs for Auditory, written summaries for Reading/Writing, labs/coding-first for Kinesthetic) — explicitly scoped to *how* a session is delivered, never to the grounding rules on deadlines.
5. Source code, if present, is treated as hands-on material to work through, not passive reference.
6. Explicit hallucination guard: *"Do not invent deadlines not present in the handbook; if a deadline is unknown, say so."*

**Output contract:** clean Markdown — summary paragraph, `Weekly Milestones`, a `Detailed Schedule` table (`Day | Time block | Module/Topic | Task`), and a `Risks \& assumptions` list.

### 4.2 `wellness\_agent` (Gemma4 26B-A4B-IT-QAT)

Input is **only** `plan\_agent`'s output (`{{ $('plan\_agent').item.json.output }}`) — it has no access to the raw handbook or form data, by design, so its assessment is grounded purely in the *schedule that was actually produced*. It's instructed not to rewrite the plan, only to reference real day/time blocks from it, and to emit a fixed-shape output: load assessment (1–2 sentences) → `Burnout risk: Low/Medium/High` with reason → 2–4 concrete break suggestions tied to named slots.

### 4.3 `sustainability\_agent` (Gemma4 26B-A4B-IT-QAT)

Same input contract as the wellness agent (consumes `plan\_agent`'s output only). Constrained to module-specific suggestions — every bullet must name the actual module or activity it applies to (e.g., paperless workflow for a named module, batching on-campus days). Generic eco-tips are explicitly disallowed by the system prompt.

### 4.4 `ics\_agent` — Calendar Extraction Agent (Qwen3 14B)

Input is the **assembled report** (`{{ $('save\_report').first().json.report }}`), i.e., plan + wellness + sustainability combined. Its only job is structured extraction: read the prose/table and return a bare JSON array (`\[{"day":"Monday","start":"09:00","end":"11:00","title":"..."}]`, 24h times, `Monday`–`Sunday` literals), ignoring deadlines/milestones and emitting `\[]` if no concrete sessions exist. No prose, no code fences — this keeps the downstream parser trivial.

\---

## 5\. Report Assembly \& Calendar Generation

**`assemble\_report` (Code)** stitches the three agent outputs into one Markdown document with `## Wellness check` and `## Sustainability` sections under the plan, and tags it with the student's name.

**`save\_report` (Code)** persists that Markdown to disk (`/data/flagged/plan-<name>-<timestamp>.md`) and forwards both the file path and the raw text downstream.

**`build\_ics` (Code)** is the most defensive node in the workflow, because it has to turn an LLM's free-form JSON output into a calendar file that real clients will accept:

* `extractArray()` strips model artifacts (special tokens like `<|...|>`, markdown code fences) and recovers a JSON array three ways in sequence: direct `JSON.parse`, then a substring between the first `\[` and last `]`, then gives up and returns `\[]` — never throws.
* Weekday names are mapped to `Date.getDay()` indices; `nextDateForDow()` projects each session onto the *next* upcoming occurrence of that weekday from "now."
* Generates standards-compliant `VEVENT` blocks with `RRULE:FREQ=WEEKLY;COUNT=12` (each recurring session repeats for 12 weeks — roughly one semester), proper `DTSTAMP`/`UID`, and RFC-5545 text escaping (`esc()` escapes `\\`, `;`, `,`, newlines) so titles with punctuation don't corrupt the file.
* Output is written to disk as `.ics` and the path threaded forward alongside the report text (read back, by name, in `bundle\_files` — see below).

**`bundle\_files` (Code)** explicitly re-reads the `.ics` from disk via `fs.readFileSync` rather than trusting binary data to survive the intervening feedback-form step, base64-encodes both files (`study-plan.md`, `study-plan.ics`), and emits them as n8n binary properties for the `Compression` node to zip.

**`Completion` (Form, `respondWith: returnBinary`)** is the terminal response node — it serves the zip back to the student's browser with a static completion message instructing them to import the `.ics` into their calendar app.

\---

## 6\. Feedback Capture (n8n Data Tables)

The `Feedback` form (Rating 1–5 dropdown + free-text Comments) fires immediately after the planner finishes, *before* the file bundle is delivered, ensuring feedback collection isn't optional/skippable.

**`log\_feedback` (Code)** assembles the row that gets persisted — notably:

* `inputs`: the full `context.prompt` (handbook + level + VARK + source code), truncated to 12,000 chars, so the evaluator later has the *exact* grounding material the planner saw.
* `plan`: the full assembled Markdown report.
* `feedback\_id`: a deterministic composite key — `${ISOtimestamp}\_\_${slugified\_student\_name}`.
* `evaluated: false` — the flag the evaluation workflow polls on.

**`save\_feedback\_row`** writes this into the `student\_feedback` Data Table (n8n's built-in structured storage, not an external DB) with columns: `feedback\_id, created\_at, student, email, level, goal, inputs, plan, rating, comment, evaluated`.

\---

## 7\. Evaluation Workflow

Runs as a fully separate n8n workflow against the same `student\_feedback` table, designed to be re-triggered (manual/cron) without ever reprocessing already-scored feedback.

1. **`get\_feedback`** — Data Table `get` with filter `evaluated == isFalse`, `returnAll: true`. This is the entire "continuously processes only unevaluated feedback" claim, implemented as a single filtered query rather than a queue.
2. **`comment\_input` (Code)** — wraps the raw comment in a labeled block, or substitutes `"(the student left no comment)"` so the downstream agent never receives an empty/ambiguous prompt.
3. **`Comment Analysis Agent`** (Gemma4 26B) — sees *only* the comment, explicitly forbidden from judging the plan itself or inventing unstated opinions. Returns strict JSON: `comment\_present, sentiment (positive|neutral|negative|mixed|none), themes\[], specific\_issues, praise, summary`.
4. **`judge\_input` (Code)** — the connective tissue between the two agents. Implements a hand-rolled, brace-depth-counting JSON extractor (`extractJson`) that scans for balanced `{...}` substrings and tries parsing candidates from the *end* of the string backwards — robust against an LLM prepending stray text before the JSON it was told to emit alone. Builds the `judgePrompt` containing: original grounding inputs, the **untouched human rating**, the structured comment analysis, and the full plan text.
5. **`Judge Agent`** (Gemma4 26B) — scores four dimensions, 1–5 each, against an explicit rubric:

   * **grounding** — uses only handbook-stated deadlines/weights, flags unknowns, invents nothing
   * **calibration** — pacing/assumed knowledge matches self-assessed level
   * **actionability** — sessions are concrete; schedule is realistic
   * **honesty** — states trade-offs under time pressure, doesn't over-promise

   Explicitly instructed to treat the human rating/comment as evidence to weigh, but never to alter the recorded rating itself. Returns `{grounding, calibration, actionability, honesty, overall, notes}` as bare JSON.

6. **`combine` (Code)** — same brace-counting JSON extractor pattern as `judge\_input`, with a hard fallback (`{grounding:0,...,notes:'PARSE\_FAILED'}`) so a malformed model response never crashes the row write — it just gets a visibly-flagged zero score instead.
7. **`save\_evaluation\_row`** — writes a flattened row to the `plan\_evaluations` Data Table: `feedback\_id, evaluated\_at, student, student\_rating, comment\_present, comment\_sentiment, comment\_themes, comment\_summary, grounding, calibration, actionability, honesty, overall, notes`.
8. **`mark\_evaluated`** — Data Table `update`, filtered by `feedback\_id`, sets `evaluated = true` on the source row in `student\_feedback`, closing the loop so the next run won't reprocess it.

   This gives a permanent, queryable audit trail: every generated plan has a row in `student\_feedback` (human signal) joined 1:1 by `feedback\_id` to a row in `plan\_evaluations` (LLM-judge signal).

   \---

   ## 8\. Engineering Patterns Worth Calling Out

* **`safe(fn, fallback)` wrapper** — used throughout (`context`, `save\_report`, `log\_feedback`) to make every optional upstream field non-fatal. The workflow runs identically whether a student uploads zero, one, or two files.
* **Robust LLM-output parsing** — three independent JSON/array recovery strategies appear across `build\_ics`, `judge\_input`, and `combine`: strict parse → substring-by-delimiter parse → brace/bracket-depth-aware candidate scan, always with a typed fallback object/array rather than an exception. This is the actual mechanism behind the README's "hallucination avoidance" — it's not just a prompt instruction, it's parser-level defense against malformed model output.
* **Strict separation of concerns between agents** — `wellness\_agent` and `sustainability\_agent` never see the original handbook or form inputs, only `plan\_agent`'s output. This forces their critiques to be grounded in what was *actually scheduled*, not in re-deriving requirements from scratch.
* **Idempotent evaluation** — the `evaluated` boolean + filtered `get` + `update` round-trip means the evaluation workflow can be triggered on a cron with no dedupe logic required elsewhere.
* **Disk round-tripping over binary passthrough** — `build\_ics` writes to disk and `bundle\_files` reads it back by absolute path rather than threading n8n binary data through the feedback-form node, which avoids binary payloads getting dropped across a human-interaction step.

  \---

  ## 9\. Technology Stack

|Component|Technology|
|-|-|
|Workflow engine|n8n|
|LLM runtime|Ollama (local)|
|Planning / calendar-extraction model|`qwen3:14b`|
|Wellness / sustainability / evaluation models|`gemma4:26b-a4b-it-qat`|
|Agent orchestration|`@n8n/n8n-nodes-langchain.agent`|
|External tool calling|MCP Client (`mcpClientTool`), Open-Meteo (`httpRequestTool`)|
|Structured persistence|n8n Data Tables (`student\_feedback`, `plan\_evaluations`)|
|Report format|Markdown|
|Calendar format|ICS (RFC 5545, weekly `RRULE`, 12-week recurrence)|
|Packaging|`compression` node (zip), `returnBinary` form completion|
|Glue logic|JavaScript (Code nodes)|

\---

## 10\. Repository Structure

```
.
├── Study\_Planner.json        # main intake → multi-agent → report/ICS → feedback workflow
├── Planner\_Evaluation.json    # standalone LLM-judge evaluation workflow
├── README.md
└── assets/
```

\---

## 11\. Future Work

* RAG over a vector store for handbook retrieval instead of full-text injection (would lift the current 8,000-char source-code truncation ceiling)
* LMS integration for direct handbook/deadline ingestion
* Adaptive rescheduling when the wellness agent flags burnout
* A dashboard over the `plan\_evaluations` table for longitudinal quality tracking
* Multi-language plan generation
* Email delivery of the report/ICS bundle instead of (or alongside) the form's `returnBinary` response

\---

## Author

**Alessia Cordea**

**Manas Samant**

**Samir Shabani**

**Kumar Arpit**

AI • Workflow Automation • n8n • LLM Applications • Context Engineering

