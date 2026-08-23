# RuralCare / "Sanjeevani OS" — Complete Forensic Codebase Audit

Scope note: The ZIP contained 14,187 files, but 14,122 of those are vendor trees — `node_modules` in both frontends and a full Python `.venv` (including the entire `anthropic` and `groq` SDK source, `pip`, `pydantic`, `fastapi`, etc.) under `backend/agent-service/.venv`. **65 files are actually authored project code.** This audit inspects all 65, plus the live `ruralcare.db` SQLite file shipped in the ZIP (used to verify the real schema). Every claim below is tagged `[IMPLEMENTED]`, `[PARTIALLY IMPLEMENTED]`, `[REFERENCED BUT NOT IMPLEMENTED]`, `[CONFIGURED BUT UNUSED]`, `[DEAD/UNUSED]`, or `[NOT VERIFIABLE]`, with file citations.

---

# EXECUTIVE OVERVIEW

**What this is:** A three-service hackathon prototype (Smart India Hackathon 2026, per the README badge) called "Sanjeevani OS" / RuralCare. A patient chats symptoms to an LLM-driven intake agent; the agent produces a structured JSON summary; a Node.js hub stores it and pushes it to a doctor dashboard over WebSocket; a doctor (any named person — there is no password) reviews it and submits a plain-text precautions/medication response that streams back to the patient.

**Maturity: Functional MVP**, closer to a well-executed weekend hackathon build than an "Advanced MVP." It runs end-to-end, has one real integration test, and is genuinely deployed (Render URLs in the README). But there is zero real authentication, zero database-level patient/case ownership, no automated safety layer despite the README's explicit claim of one, and several doc-vs-code mismatches that suggest a mid-build pivot from Anthropic Claude to Groq that wasn't fully cleaned up.

**The single most important finding:** `doctor-frontend/src/pages/Login.jsx` (lines 6–38) issues a valid doctor session token for **any typed name**, no password, no allow-list. `backend/server.js` (`/api/login`, lines 108–116) accepts `role` directly from the request body with no verification the caller is actually clinical staff. Anyone with the doctor-frontend URL can read every patient's full chat transcript, name, and symptoms, and can write back fabricated "medical" precautions/medication to a real patient session. This is a Critical finding, not a nitpick — the entire trust boundary the architecture claims to have (`requireDoctor` middleware) is a name field.

---

# COMPLETE PROJECT TREE

```text
ruralcare-project/
├── README.md                        # Top-level pitch/architecture doc
├── README_DEPLOY.md                 # 3-line deploy note
├── start-all.ps1                    # Windows-only dev launcher (hardcoded local paths)
├── .vscode/
│   ├── settings.json
│   └── tasks.json                   # Launch tasks; prompts for "anthropicApiKey" -> sets GROQ_API_KEY (see §29)
├── backend/                         # Node.js hub (Express + ws + better-sqlite3)
│   ├── server.js                    # All REST + WebSocket logic (262 lines)
│   ├── db.js                        # SQLite persistence + JSON-file fallback (164 lines)
│   ├── test-integration.mjs         # One end-to-end integration test (52 lines)
│   ├── package.json / package-lock.json
│   ├── Dockerfile
│   ├── ruralcare.db / -shm / -wal   # Live SQLite file shipped in the ZIP (3 rows of real test data)
│   └── agent-service/               # Python FastAPI microservice
│       ├── main.py                  # Groq LLM chat + Whisper transcription (123 lines)
│       ├── requirements.txt         # fastapi, uvicorn, groq, pydantic, python-dotenv
│       ├── Dockerfile
│       ├── README.md                # Claims "sends...to Claude" (see §29 — contradicts main.py)
│       └── .venv/                   # [OMITTED — vendored Python env, ~13,700 files, incl. unused `anthropic` SDK]
├── doctor-frontend/                 # React 19 + Vite 8 + Tailwind
│   └── src/
│       ├── App.jsx, main.jsx, ErrorBoundary.jsx, ProtectedRoute.jsx
│       ├── lib/api.js
│       └── pages/{Login,Queue,CaseDetail}.jsx
├── patient-frontend/                # React 19 + Vite 8 + Tailwind
│   ├── public/agent-connect.html    # Orphaned dev tool, posts to a route that doesn't exist (see §33)
│   └── src/
│       ├── App.jsx, main.jsx, ErrorBoundary.jsx
│       ├── lib/api.js
│       └── pages/{Login,Home,ChatPage,StatusPage}.jsx
└── [node_modules/ x2, .venv/ — 14,122 vendored files omitted from this tree]
```

---

# PROJECT DNA

- **Problem:** Rural Indian PHC doctors spend disproportionate consult time on basic history-taking (README's claim, `[UNKNOWN / NOT VERIFIABLE]` — no citation given for the "80%" figure).
- **Users:** Patients (unauthenticated, chat-only) and doctors (nominally authenticated, actually just named).
- **Inputs:** Free-text or transcribed-voice patient chat messages.
- **Processing:** A single Groq LLM call per turn, with a system prompt instructing the model to ask ~6–8 questions then self-emit a JSON block.
- **Intelligence:** One LLM (`openai/gpt-oss-120b` via Groq) for chat + summarization, one hosted Whisper model (`whisper-large-v3`, via Groq) for transcription. No RAG, no embeddings, no vector store, no separate classifier — `[IMPLEMENTED: single-LLM, no retrieval]`.
- **Outputs:** A structured case (name, chief complaint, symptoms, duration, severity, urgency flag, history, conversation notes) plus a doctor-authored decision (precautions, medication, notes).
- **Storage:** SQLite file (`better-sqlite3`), with a JSON-file fallback if the native module fails to load — `backend/db.js:16-45`.
- **Integrations:** Groq API only (chat + Whisper). No email, no SMS, no calendar, no EHR, no payment.
- **Decision logic:** None encoded outside the LLM's own system prompt (see §14 — the README's claimed "rule-based safety layer" does not exist in code).
- **Safety:** A regex strips a leaked `<<<SUMMARY>>>` block from the reply shown to the patient (`server.js:142-143`) — this is the only deterministic safety mechanism actually in the codebase.
- **UX:** Two separate React SPAs (patient, doctor), mobile-first, with voice input for the patient side.
- **Deployment:** Four independently deployed services on Render (per README table) with Vercel config files also present (`vercel.json` in both frontends) — `[CONFIGURED BUT UNUSED/AMBIGUOUS]`, since the README's live URLs are all `onrender.com`.
- **Scalability:** Not designed for it — see §25.
- **Differentiation:** Sub-second LLM latency via Groq's LPU hardware, and voice input in 11 Indian languages via Whisper's auto-detect (`patient-frontend/src/pages/ChatPage.jsx:9-22`) — this is a real, verifiable differentiator, not just a slide claim.

---

# TECH STACK

| Layer | Technology | Evidence |
|---|---|---|
| Patient/Doctor Frontend | React 19.2.8, Vite 8.2.0, React Router 7.18.2, Tailwind CSS 3.4.19 | `patient-frontend/package.json`, `doctor-frontend/package.json` |
| State management | Local `useState`/`useRef` only — no Redux/Zustand/Context store | grep across all `.jsx`, confirmed absent |
| Client storage | `localStorage` (doctor auth token/name/role), `sessionStorage` (patient's in-progress case id) | `doctor-frontend/.../ProtectedRoute.jsx:5`, `patient-frontend/.../ChatPage.jsx:5,26` |
| Realtime | Native browser `WebSocket`, no Socket.io | `patient-frontend/src/lib/api.js:54-64`, `doctor-frontend/src/lib/api.js:48-58` |
| Core backend | Node.js (ESM, `"type": "module"`), Express 4.19.2, `ws` 8.18.0, `cors`, `nanoid` | `backend/package.json`, `backend/server.js` |
| AI microservice | Python 3.12, FastAPI 0.141.1 (from `.venv` dist-info), Pydantic v2, Uvicorn | `backend/agent-service/requirements.txt`, `Dockerfile` |
| LLM provider | **Groq** (not Anthropic despite doc claims — see §29), model `openai/gpt-oss-120b` for chat, `whisper-large-v3` for STT | `backend/agent-service/main.py:87,117` |
| Database | `better-sqlite3` with an automatic JSON-file fallback if the native binary fails to load | `backend/db.js:18-45` |
| Auth | Custom in-memory bearer tokens (`Map<token, {role,name}>`), no password, no JWT, no expiry | `backend/server.js:40,108-116,47-56` |
| Containerization | Two Dockerfiles (`backend/Dockerfile`, `backend/agent-service/Dockerfile`); no `docker-compose.yml` anywhere in the ZIP | confirmed by file search |
| Hosting | Render (per README's live-deployment table); Vercel config files present but role unclear | `README.md:27-40`, both `vercel.json` |
| CI/CD | **None found** — no `.github/workflows`, no CI config of any kind | confirmed by file search |

**External services — complete list:** Groq API is the *only* third-party service integrated. No email/SMS provider, no cloud storage, no analytics, no error-tracking (Sentry etc.), no CDN config beyond Vercel's implicit static hosting.

---

# ARCHITECTURE

**Style: Hub-and-spoke, three independently-deployed services communicating over plain HTTP/WebSocket** — the README's own label (`README.md:55`) is accurate and matches the code. It's not a monolith (three separate `package.json`/`requirements.txt` roots, three separate Dockerfiles/deploy targets), not microservices in the formal sense (no service mesh, no independent scaling policy, no message broker), and not any layered/clean-architecture pattern internally — `backend/server.js` is a single flat file mixing routing, WebSocket fan-out, and a couple of small view-model helpers (`formatSummary`) with no controller/service/repository separation. `backend/agent-service/main.py` is similarly a flat FastAPI file: prompt constant, regex parser, and two route handlers, no separation of concerns beyond function boundaries.

**Evidence for "hub" role of Node backend:** Every case-related write and every WebSocket broadcast passes through `server.js`; the Python service and the two frontends never talk to each other directly — `patient-frontend/src/lib/api.js` and `doctor-frontend/src/lib/api.js` both only ever call `VITE_API_URL` (the Node backend), never an agent URL directly.

---

# FRONTEND FORENSICS

## Patient frontend (`patient-frontend/`)
**Pages** (`src/pages/`): `Home.jsx` (marketing/landing), `ChatPage.jsx` (the intake conversation, incl. voice recording), `StatusPage.jsx` (post-intake status + doctor's decision, polled via WebSocket), `Login.jsx` (patient "login" — name only).

**Routing** (`App.jsx:9-14`): `/login`, `/`, `/chat`, `/status/:id`. Note `/chat` and `/status/:id` are **not** gated by any login check — there is no `ProtectedRoute` equivalent on the patient side, so `Login.jsx` for patients is **`[DEAD/UNUSED]`**: nothing in the app reads `localStorage.rc_token`/`rc_role` on the patient side, and none of the patient API calls in `lib/api.js` send an `Authorization` header. A patient can reach `/chat` directly with zero login. **Recommendation: DELETE or wire it up — currently pure dead weight.**

**State:** All local component state (`useState`), no global store. Case-in-progress continuity is handled via `sessionStorage` (`CASE_ID_KEY`, `ChatPage.jsx:5,26,58-75`) — a nice touch that survives a page refresh by re-fetching the case and resuming the conversation, or redirecting to `/status/:id` if intake is already complete.

**Voice input** (`ChatPage.jsx:104-157`): Uses `navigator.mediaDevices.getUserMedia` + `MediaRecorder`, uploads base64 audio to the backend's `/api/cases/:id/voice` proxy, which forwards to the agent service's Whisper endpoint. Handles unsupported-browser and mic-denied cases gracefully (`micSupported` check, try/catch around `getUserMedia`). Minimum-recording-length guard (`blob.size < 800`) prevents empty submissions — `[IMPLEMENTED]`, genuinely solid.

**Bug found — `StatusPage.jsx:106,120-124`:** displays `summary.medical_history`, but the agent's actual JSON field (both in the system prompt, `main.py:15`, and in `server.js`'s `formatSummary`, line 125) is `relevant_history`. **This field will never render.** Same bug is duplicated in `doctor-frontend/src/pages/CaseDetail.jsx:122,136-140`. This is a copy-paste field-name mismatch between the LLM's actual output schema and both frontends' display code — `[BUG, MEDIUM SEVERITY]`, silently drops clinically relevant information (allergies/meds/history) from both the patient and doctor views.

**Orphaned file — `public/agent-connect.html`:** A standalone HTML page (not routed to from React) that POSTs to `http://localhost:4000/api/agent` to "register" an agent URL. **No such route exists in `server.js`.** `[REFERENCED BUT NOT IMPLEMENTED]` — dead leftover from an earlier design where the agent URL was meant to be dynamically configurable rather than a static `AGENT_URL` env var (which is how it actually works, `server.js:41`). Also unreferenced is `VITE_CHATZY_EMBED_URL` in `patient-frontend/.env`/`.env.example` — grepped across every `.jsx`/`.js`/`.html` file in the project, zero occurrences beyond the env files themselves. `[CONFIGURED BUT UNUSED]`.

## Doctor frontend (`doctor-frontend/`)
**Pages:** `Login.jsx`, `Queue.jsx` (live-updating case list, split into Pending/Reviewed), `CaseDetail.jsx` (full case view + decision form).

**Auth gate** (`ProtectedRoute.jsx:3-12`): Purely a client-side `localStorage` check — trivially bypassed by anyone who calls the login endpoint directly or forges a token, since the *server* is what actually needs scrutiny here (see Security §).

**Queue bug — `Queue.jsx:62`:** `pending = cases.filter(c => c.status !== 'reviewed')`. Because `backend/server.js:118-122` (`POST /api/cases/start`) inserts a row with `status: "in_progress"` the *instant* a patient opens the chat page — before they've typed a single message — every abandoned or still-typing patient session appears in the doctor's "Pending Review" queue with `summary: ""`, rendered as "No summary yet" (`Queue.jsx:141`). On any real deployment this queue will fill with empty/abandoned rows. `[BUG, LOW-MEDIUM SEVERITY — noise/UX, not data-safety]`.

**Session timeout handling** (`doctor-frontend/src/lib/api.js:10-27`): A 10-second `AbortController` timeout wraps every fetch, and a 401 response clears local storage and redirects to `/login` — reasonable defensive coding for a hackathon build.

---

# BACKEND FORENSICS

`backend/server.js` is a single Express app with no router/controller/service split. Dependency flow is effectively:

```text
Express route handler
 → direct call into db.js functions (insertCase/getCase/listCases/updateCaseDecision/updateCaseIntake)
 → direct fetch() call to the Python agent service (for /chat and /voice only)
 → direct WebSocket broadcast (broadcastToDoctors / notifyPatient)
```

There is no separate "service layer" or "repository" abstraction — `db.js` *is* the repository, called directly from route handlers. For a codebase this size (262 lines) that's a reasonable, not over-engineered choice — see §45 (Do Not Overengineer).

**Middleware:** `express.json({ limit: "15mb" })` (sized deliberately for base64 voice payloads, per the inline comment `server.js:37`), `cors()` with **default (fully open) settings** — no origin allow-list (`server.js:36`) — `[SECURITY FINDING, see below]`. One custom middleware, `requireDoctor` (`server.js:47-56`), checks a bearer token against the in-memory `sessions` Map.

**Error handling:** A catch-all Express error handler (`server.js:235-238`) and an `app.use("/api", ...)` 404 fallback (`server.js:234`) are present. Process-level `unhandledRejection`/`uncaughtException` handlers just log — they do not crash-restart or alert (`server.js:28-33`), meaning the process could limp along in a corrupted state after an unhandled error rather than being restarted by a process manager. No `pm2`/systemd/restart-policy config was found anywhere in the ZIP to compensate.

**WebSocket implementation** (`server.js:242-259`): Role is read from a query string (`?role=doctor` or `?role=patient&caseId=...`) with **no authentication check at all** on the socket handshake — any client can connect as `role=doctor` and receive every `new-case`/`case-updated` broadcast (including full patient summaries) without ever calling `/api/login`. `[SECURITY FINDING]` — the WebSocket channel is entirely unauthenticated even though the REST API nominally requires a doctor token for the equivalent `GET /api/cases`.

---

# AI/ML FORENSICS

**Single LLM, single call pattern — no agentic loop, no tool use, no multi-step planning.**

| Component | Detail | Evidence |
|---|---|---|
| Chat model | `openai/gpt-oss-120b`, served via Groq | `main.py:87` |
| STT model | `whisper-large-v3`, served via Groq | `main.py:117` |
| Prompting | One long static system prompt (`main.py:15`), re-sent in full on every turn along with the accumulated conversation array | `main.py:89-90` |
| Structured output | **Not** via a JSON-mode API parameter or function-calling — the model is instructed via prose to end its final reply with a `<<<SUMMARY>>>...<<<END_SUMMARY>>>` block, parsed with a regex (`SUMMARY_PATTERN`, `main.py:20`) | `main.py:57-77` |
| Output validation | `json.loads()` on the matched block; a `JSONDecodeError` raises an HTTP 502 to the caller (`main.py:68-71`). **No schema validation beyond raw JSON parse** — Pydantic models exist for the *request* (`Message`, `AgentRequest`, `TranscribeRequest`) but there is no Pydantic model validating the *LLM's summary output* — a model that returns valid JSON with wrong types/missing keys (e.g., `symptoms` as a string instead of an array) would pass through unvalidated into `formatSummary()` on the Node side and could throw there (`symptoms?.length` guards against `undefined` but not against a non-array). |
| Failure handling | `GroqError` → HTTP 502 with a generic patient-facing message (`main.py:94-95`, `server.js:156-159`) | `[IMPLEMENTED]` |
| Reliability / guardrails on hallucination | **None beyond the prompt itself.** No output-side keyword scan, no second-pass validation LLM call, no confidence score | confirmed absent by full-file read of `main.py` |

**Is there a "rule-based safety layer" as the README claims?** **`[REFERENCED BUT NOT IMPLEMENTED]`.** README.md line 92 states: *"A rule-based safety layer sits on top of the LLM. Fatal keywords (e.g. 'chest pain', 'severe bleeding', 'can't breathe') instantly trigger an emergency escalation, bypassing the standard queue entirely."* No such keyword list, regex, or bypass logic exists anywhere in `backend/server.js` or `backend/agent-service/main.py`. The system prompt (`main.py:15`) *does* instruct the LLM to tell a patient to contact emergency services if they mention chest pain/bleeding/etc. — but that is a prose instruction to a probabilistic model, not the deterministic, code-level "safety layer sitting on top of the LLM" the README explicitly claims. This is the single biggest documentation-vs-code gap in the project (see §29) and matters clinically: a model that "forgets" the instruction on a long conversation has no code-level backstop.

**Is this "AI-decoupled-from-action" as claimed?** Partially true. README line 91 claims *"no free-form data ever reaches the database"* — this is accurate: `formatSummary()` (`server.js:124-126`) only pulls known fields out of the parsed JSON object, so arbitrary LLM prose cannot inject itself into the stored `summary` field. The one exception is the raw conversational `reply` text shown to the patient, which is unfiltered LLM output apart from the `<<<SUMMARY>>>` leak-guard (`server.js:140-144`).

---

# AI AGENT FORENSICS

**Is this an autonomous/agentic system, or a scripted LLM call?** Brutally honest answer: **it is a single-step, stateless chat-completion call re-issued on every user turn — not an agent.** There is no tool use, no planning, no multi-turn internal reasoning loop, no memory beyond the raw message array the Node backend hands it each time. "Agent" in this codebase (`agent-service`, `SYSTEM_PROMPT`'s "Triage Assistant" framing) is a product/marketing label for a chat-completions endpoint with a well-written system prompt — a legitimate and common pattern, just not "agentic" in the sense of the LLM having its own state, tools, or decision loop.

| Property | Value |
|---|---|
| Name | RuralCare Triage Assistant (system-prompt persona name only, `main.py:15`) |
| Goal | Gather structured intake info within 6–8 questions |
| Trigger | Every `POST /api/cases/:id/chat` |
| Tools | None |
| Memory | None beyond the conversation array passed in on each call — no server-side session/thread object on the Groq/agent side |
| Output | Free text + optional embedded JSON |
| State | Stateless service; all state (the conversation history) lives in `backend/db.js`'s `cases.conversation` column |
| Failure handling | HTTP 502 bubbled to the patient with a generic message |

---

# DATABASE FORENSICS

**Single table, `cases`** (confirmed against the live shipped `ruralcare.db` file, which matches `db.js`'s `CREATE TABLE` exactly):

```text
cases
 ├── id            TEXT PRIMARY KEY   (nanoid(8), server-generated)
 ├── patientName   TEXT NOT NULL
 ├── summary       TEXT NOT NULL      (human-readable, built by formatSummary())
 ├── urgency       TEXT NOT NULL      (routine | priority | emergency)
 ├── status        TEXT NOT NULL      (in_progress | waiting_for_doctor | reviewed)
 ├── decision      TEXT               (JSON-stringified {precautions, medication, notes, decidedAt, decidedBy})
 ├── conversation  TEXT NOT NULL DEFAULT '[]'   (JSON-stringified array of {role, content})
 ├── agentSummary  TEXT               (JSON-stringified raw LLM summary object)
 └── createdAt     TEXT NOT NULL      (ISO string)
```

No foreign keys, no indexes beyond the implicit primary-key index, no separate `users`/`doctors` table at all — doctor identity is never persisted; it exists only as an in-memory `sessions` Map entry (`server.js:40`) that is lost on every backend restart. **`decidedBy` is a free-text string typed at login time**, not a foreign key to any doctor record — there is no way to audit *which real doctor* reviewed a case beyond trusting whatever name string they typed in.

**Migrations:** Two manual `ALTER TABLE ... ADD COLUMN` statements guarded by a `PRAGMA table_info` check (`db.js:37-39`) — a lightweight, self-healing schema-evolution pattern appropriate for this project's size, but not a real migration framework (no down-migrations, no version tracking table).

**Fallback storage:** If `better-sqlite3`'s native binary fails to load (common cross-platform pain point for that package), `db.js` transparently falls back to a JSON file (`ruralcare.db.json`) with the *same* CRUD functions re-implemented against a plain array (`db.js:41-45,64-162`). This is well-engineered defensive coding for a demo that has to run on judges' laptops of unknown OS/arch — a genuine strength, not overengineering, given the actual failure mode it defends against.

**Data safety concerns:** No encryption at rest (plain SQLite file with plaintext patient names/symptoms — visible directly in the shipped `ruralcare.db`, which is committed and was even included in the submitted ZIP with 3 rows of what appears to be real test-session data). `.gitignore` (`backend/.gitignore`) does list `*.db` and its WAL/SHM siblings, so the *repo* is supposed to exclude them — but the ZIP the person uploaded to me includes them anyway, meaning a `.gitignore` exclusion doesn't protect against ad-hoc zip/copy operations. `[PRIVACY FINDING]`.

---

# API INVENTORY

| Method | Path | Auth | Body | Response | Notes |
|---|---|---|---|---|---|
| POST | `/api/login` | none | `{role, name}` | `{token, role, name}` | **Issues a valid session for any `role`/`name` combo — no credential check** (`server.js:108-116`) |
| POST | `/api/cases` | none | `{patientName, summary, urgency}` | 201, case record | Legacy/alt path to create a case directly (bypasses the chat flow entirely); broadcasts to doctors |
| POST | `/api/cases/start` | none | — | 201, `{case_id}` | Called by patient frontend on chat page load; inserts an immediate `in_progress` row (source of the Queue.jsx noise bug, §above) |
| POST | `/api/cases/:id/chat` | none | `{message}` | `{reply, done}` | Proxies to agent service, persists conversation + summary when `done` |
| GET | `/api/cases` | **doctor bearer token** | — | array of cases | The only endpoint actually gated by `requireDoctor` |
| POST | `/api/cases/:id/voice` | none | `{audio (base64), mimeType, language}` | `{text}` | Proxies to agent's Whisper endpoint |
| GET | `/api/cases/:id` | none | — | case record | **No auth — any caller with a case id (an 8-char nanoid) can read full patient PII/chat history**; used by both patient status page and doctor detail page |
| POST | `/api/cases/:id/decision` | **doctor bearer token** | `{precautions, medication, notes}` | updated case | Notifies both the patient socket and all doctor sockets |
| GET | `/health` | none | — | `{status:"ok"}` | Basic liveness check, matches README's "View Health Check" link |

**Agent service (`backend/agent-service/main.py`), internal-only (never exposed directly to browsers):**

| Method | Path | Body | Response |
|---|---|---|---|
| POST | `/agent/respond` | `{case_id, messages[]}` | `{reply, done, summary}` |
| POST | `/agent/transcribe` | `{audio, mime_type, language}` | `{text}` |

**`[REFERENCED BUT NOT IMPLEMENTED]`:** `POST /api/agent` (called by the orphaned `agent-connect.html`, never defined in `server.js`).

---

# BUSINESS LOGIC

The entire "clinical" decision logic in this codebase is exactly two rules, both enforced only weakly:

```text
Rule: Urgency classification
Condition: LLM's self-reported `urgency_flag` in its JSON summary
Action: Stored verbatim as the case's `urgency`, IF it's one of {routine, priority, emergency}; otherwise silently defaults to `routine`
Reason: Trust the LLM's own judgment, with an enum whitelist as the only guard
Where implemented: server.js:153 (`... ["routine","priority","emergency"].includes(agent.summary.urgency_flag) ? ... : record.urgency`)
```

```text
Rule: Summary-leak prevention
Condition: LLM reply text contains a literal "<<<SUMMARY>>>" substring
Action: Truncate the patient-facing reply at that point; substitute a fallback sentence if nothing is left
Reason: Defense-in-depth in case the Python service fails to strip the block itself
Where implemented: server.js:140-144
```

There is no severity scoring algorithm, no queue-prioritization/sorting logic beyond `ORDER BY createdAt DESC` (`db.js:126`) and a client-side `status !== 'reviewed'` filter (`Queue.jsx:62-63`) — cases are **not** sorted by urgency anywhere in the code, despite the doctor UI visually color-coding by urgency. An "emergency" case submitted at 9:00am will sit below a "routine" case submitted at 9:01am in the doctor's pending list. `[DESIGN GAP]`.

---

# ALGORITHMS & CORE LOGIC

There is exactly one non-trivial piece of "algorithmic" code in the entire project: the summary-block regex parser.

**`parse_agent_reply()` (`main.py:57-77`)**
1. **Problem:** Extract a trailing JSON payload from an LLM's free-text reply, which may or may not be wrapped in a markdown code fence.
2. **Input:** Raw LLM completion text.
3. **Output:** `{reply: str, done: bool, summary: dict|None}`.
4. **Logic:** Strip a leading/trailing ``` fence if present → regex-search (not anchor-to-end, `re.DOTALL`) for `<<<SUMMARY>>>\s*(\{.*?\})\s*<<<END_SUMMARY>>>` → `json.loads()` the captured group → split the reply text at the match start.
5. **Complexity:** O(n) in reply length for the regex scan; trivial.
6. **Edge cases handled:** code-fence wrapping, empty reply-before-summary (falls back to a canned sentence), invalid JSON (raises HTTP 502).
7. **Edge cases NOT handled:** a `.*?` non-greedy match inside `\{.*?\}` will break on any legitimate nested `}` inside the JSON if the model ever emits a string value containing literal `}` characters before the real closing brace — a low-probability but real fragility of using regex instead of a proper balanced-brace/streaming JSON parser or (better) the LLM provider's native JSON-mode / function-calling. **`[TECHNICAL DEBT]`** — Groq's chat-completions API does support `response_format: {type: "json_object"}` style constrained decoding for some models; this codebase does not use it, relying entirely on prompt compliance.

No sorting/ranking/matching/clustering/graph algorithms exist anywhere else in the codebase.

---

# DATA FLOW / USER FLOWS (Runtime Reconstruction)

**Patient intake flow:**
```text
Patient opens /chat (patient-frontend)
 → POST /api/cases/start  → Node inserts {status:"in_progress"} row, returns case_id
 → patient types/speaks a message
    (voice path: MediaRecorder → base64 → POST /api/cases/:id/voice → Node proxies →
     agent-service /agent/transcribe → Groq Whisper → text fills the input box for review)
 → POST /api/cases/:id/chat {message}
 → Node appends to conversation[], calls agent-service /agent/respond
 → agent-service sends [system prompt + full conversation] to Groq chat-completions
 → Groq returns text (possibly with a trailing <<<SUMMARY>>> block)
 → parse_agent_reply() splits reply vs. summary
 → Node: strips any leaked SUMMARY marker (defense-in-depth), appends assistant turn,
   persists conversation+patientName+summary+urgency+status via updateCaseIntake()
 → IF done: status becomes "waiting_for_doctor", Node broadcasts {type:"new-case"} to ALL
   connected doctor WebSocket clients (unauthenticated at the socket layer)
 → Patient frontend navigates to /status/:id
```

**Doctor review flow:**
```text
Doctor opens doctor-frontend → Login.jsx (name only, no password)
 → POST /api/login {role:"doctor", name} → Node mints a token, stores in an in-memory Map
 → Queue.jsx: GET /api/cases (requireDoctor) + opens a `role=doctor` WebSocket (NOT auth-checked)
 → doctor clicks a case → GET /api/cases/:id (no auth required on this route at all)
 → doctor submits precautions/medication/notes → POST /api/cases/:id/decision (requireDoctor)
 → Node persists decision, sets status="reviewed", notifies the specific patient's WebSocket
   AND broadcasts to all doctor sockets
 → Patient's StatusPage.jsx receives the WS "case-updated" event and re-renders the decision
```

---

# SECURITY

| Finding | Rating | Evidence | Impact |
|---|---|---|---|
| Doctor login accepts any name, no password/credential check | **Critical** | `server.js:108-116`, `Login.jsx:6-38` (doctor-frontend) | Anyone with the doctor-frontend URL gets full read access to every patient's PII + chat transcript, and can write fabricated medical guidance back to real patients |
| WebSocket connections are unauthenticated regardless of declared role | **Critical** | `server.js:244-259` | Bypasses even the weak `requireDoctor` REST gate — anyone can open `?role=doctor` and silently stream every new case's full summary in real time, no login call required at all |
| `GET /api/cases/:id` has no auth | **High** | `server.js:194-198` | Case IDs are 8-char nanoids (not guessable in bulk, but not access-controlled either) — anyone who obtains/guesses/logs an ID can read full patient details indefinitely, with no expiry |
| Sessions are an in-memory `Map`, no expiry, lost on restart but otherwise permanent | **Medium** | `server.js:40` | A leaked token is valid until the process restarts; no logout-everywhere, no token rotation |
| CORS fully open (`cors()` with default settings) | **Medium** | `server.js:36` | Any origin can call the API; combined with the auth issues above this materially widens the attack surface |
| Live SQLite DB with real-looking patient data shipped inside the audited ZIP | **Medium (privacy)** | `backend/ruralcare.db` (3 rows), despite `.gitignore` excluding `*.db` | Ad-hoc zip/copy operations bypass `.gitignore` protection; PHI-adjacent data is not actually protected in practice |
| No rate limiting anywhere (login, chat, decision endpoints) | **Medium** | absent from `server.js` in full | Login/chat/voice-transcription endpoints can be hammered; voice endpoint accepts 15MB payloads (`express.json({limit:"15mb"})`, `server.js:37`) with no per-IP throttling — cheap DoS/cost-abuse vector against the Groq bill |
| No input sanitization beyond type/emptiness checks | **Low** | throughout `server.js` route handlers | Not an injection risk (parameterized `better-sqlite3` queries throughout `db.js`, e.g. `.prepare(...).run({...})`), but stored `summary`/`conversation` text is rendered into React via JSX text nodes (auto-escaped) — no observed XSS vector in the reviewed code |
| API keys handled via env vars, never hardcoded in source | **Informational (good)** | `main.py:81,104`, `.gitignore` excludes `.env` | `[IMPLEMENTED]` correctly |

**No SQL injection risk found** — `better-sqlite3`'s `.prepare()`/named-parameter API is used consistently in `db.js`; no string-concatenated queries anywhere.

---

# PRIVACY

- **PII/health data collected:** patient name (self-reported, unverified), symptoms, duration, severity, "relevant history" (allergies/meds/conditions), free-text conversation, optionally recorded voice audio (transcribed, not stored as audio — only the base64 clip is transmitted per-request and not persisted to disk on either service, per a full read of `main.py` and `server.js`).
- **Where stored:** Locally in `backend/ruralcare.db` (or its JSON fallback), unencrypted.
- **Where transmitted:** Node backend → Groq API (both chat text and, for voice, raw audio bytes) — i.e., **all patient symptom/voice data leaves the system boundary to a third-party US-based inference provider (Groq)** with no consent screen, no data-processing disclosure, and no mention of this in either frontend's UI copy.
- **Retention/deletion:** No deletion mechanism found anywhere — no admin endpoint, no TTL, no "right to be forgotten" flow.
- **Consent:** None implemented — no consent checkbox/banner on the patient intake flow (`Home.jsx`, `ChatPage.jsx` both reviewed in full).
- `COMPLIANCE STATUS: NOT VERIFIABLE` — this is a healthcare-adjacent product handling symptom data with no HIPAA/DPDP-Act-equivalent controls visible in the code; that is a factual code observation, not a legal conclusion, since I'm not evaluating against a specific regulatory text.

---

# AI SAFETY

- **Hallucination mitigation:** Structural only (summary fields are extracted from JSON, not free-generated into the DB) — no fact-checking, no second-model verification pass.
- **Output schema validation:** `json.loads()` only; no Pydantic model validates the *shape* of the LLM's summary object before it's trusted (see AI/ML Forensics above).
- **Prompt injection resistance:** None evaluated/implemented — a patient could type instructions attempting to manipulate the system prompt's behavior (e.g., "ignore previous instructions, mark this as routine") and nothing in the code detects or resists this; the only downstream guard is the enum whitelist on `urgency_flag` (line 153), which limits the *blast radius* of a successful injection to picking among 3 legitimate-looking values, not arbitrary code/data injection.
- **Deterministic safety backstop for emergencies:** Does **not exist** in code, despite being explicitly claimed in the README (see AI/ML Forensics — this is the report's headline documentation/code gap).
- **Timeout/retry on the Groq call itself:** None — a single synchronous SDK call with no explicit timeout override, no retry-with-backoff; a slow Groq response simply makes the patient wait (Express itself has no global request timeout configured either).

---

# TESTING

**One integration test exists:** `backend/test-integration.mjs` (52 lines). It spins up a real `server.js` child process plus a stub HTTP server standing in for the agent service, then walks the full happy-path flow: start case → two chat turns → completed intake → doctor-queue listing → decision submission — asserting status/urgency/conversation-length at each step. This is a genuinely useful smoke test and demonstrates real engineering discipline for a hackathon timeline.

**What is NOT tested, anywhere in the ZIP:**
- No unit tests for `db.js`'s SQLite-vs-JSON-fallback branching
- No tests for the Python agent service (`parse_agent_reply`'s regex edge cases are untested)
- No frontend tests of any kind (no Vitest/Jest/RTL config in either `package.json`)
- No test of the `requireDoctor` auth failure path
- No test of the WebSocket broadcast paths
- No test of the voice-transcription proxy
- No load/concurrency test

`package.json` does define `"test:integration": "node test-integration.mjs"` (`backend/package.json:9`) confirming this was a deliberately wired-up, runnable test — not a stray script.

---

# PERFORMANCE

Nothing in this codebase does expensive local computation — the dominant cost is network round-trips to Groq (chat) and Groq (Whisper), both of which the README claims run at ~350ms latency thanks to Groq's LPU hardware (`[UNKNOWN / NOT VERIFIABLE]` from code alone — this is an infra-level claim about Groq's service, not something the app code controls or measures itself; no latency logging/metrics exist in the code to verify it).

**Observed inefficiencies:**
- Every chat turn re-sends the **entire conversation history plus the full system prompt** to Groq (`main.py:89-90`) — for a 6–8 question intake this is a small, bounded array, so token cost stays modest, but it's O(n) resend rather than any incremental/session-based approach (Groq's chat API doesn't offer server-side session state, so this is somewhat inherent to the provider choice, not obviously a code mistake).
- `listCases()` (`db.js:124-131`) does an unfiltered `SELECT * FROM cases ORDER BY createdAt DESC` on every doctor queue load/poll — fine at hackathon scale (single-digit-to-low-hundreds of rows), would need pagination well before real PHC-scale volume.
- No caching layer anywhere (no Redis, no in-memory cache) — not needed at this scale, and none is misleadingly claimed either.

---

# SCALABILITY

| Users | Bottleneck | Current limitation | What would actually need to change |
|---|---|---|---|
| ~10 | none | fine as-is | — |
| ~100 | in-memory `sessions` Map and `doctorSockets`/`patientSockets` Sets are single-process only | A backend restart drops every logged-in doctor and every open WebSocket instantly | Persist sessions to a real store (Redis/DB) if uptime matters |
| ~1,000 | SQLite single-writer file, single Node process, no horizontal scaling story (no load balancer config, no sticky-session handling for WebSockets across replicas) | Cannot run more than one backend instance without breaking the in-memory session/socket model | Move to a real DB (README itself flags "PostgreSQL (production-ready)" as the intended next step, `README.md:143` — accurate self-assessment) + externalize session state + a pub/sub layer (Redis pub/sub or similar) for WebSocket fan-out across replicas |
| ~10,000+ | Groq API rate limits (not evaluated in code — no rate-limit handling/backoff exists) become the real ceiling | No queueing/backpressure if Groq throttles | Add a request queue + retry/backoff; this is a "when you get there" problem, not a current-scale one |

**`DO NOT ADD THIS`:** Kubernetes, a message broker (Kafka/RabbitMQ), or a second AI agent are all unjustified at this project's actual current scale (a hackathon demo with 3 rows of real data). The single biggest, actually-justified next infra step the codebase itself already signals is swapping SQLite for Postgres — which the README already states as the plan.

---

# DEVOPS

**Local dev:** Four services, four terminals (or one `start-all.ps1` for Windows, hardcoded to `c:\Users\arish\Downloads\ruralcare-project\...` absolute paths — `[DEAD FOR ANYONE ELSE]`, this script only works on the original author's exact machine). `.vscode/tasks.json` provides a cross-referenced but more portable `${workspaceFolder}`-relative version of the same four launch tasks, which is the actually-usable local-dev entry point.

**Containerization:** Both `backend/Dockerfile` and `backend/agent-service/Dockerfile` are simple, correct single-stage builds (`node:20-alpine`, `python:3.12-slim` respectively) — no multi-stage build, no non-root user configured, no `.dockerignore` found (meaning a Docker build context would include `node_modules`/`.venv` if not otherwise excluded — not verified from the Dockerfiles alone since a `.dockerignore` could exist and simply wasn't in this ZIP's file list, so `[NOT VERIFIABLE]`).

**No `docker-compose.yml`** — the two Dockerfiles are never wired together into one local multi-container setup; deployment is evidently four separately-deployed Render services (per the README's own table) rather than a docker-compose stack.

**CI/CD: absent.** No GitHub Actions, no other CI config, anywhere in the ZIP.

---

# CONFIGURATION

| Key | Where | Required? | Notes |
|---|---|---|---|
| `PORT` | backend | optional, defaults to 4000 | `server.js:26` |
| `AGENT_URL` | backend | optional, defaults to `http://localhost:8001` | `server.js:41` |
| `DB_PATH` | backend | optional, defaults to `ruralcare.db` next to `db.js` | `db.js:12` |
| `GROQ_API_KEY` | agent-service | **required** (503 if unset) | `main.py:81,104` — secret, correctly never hardcoded |
| `VITE_API_URL` | both frontends | required for non-localhost deploys | `.env.example` files |
| `VITE_WS_URL` | both frontends | required for non-localhost deploys | `.env.example` files |
| `VITE_CHATZY_EMBED_URL` | patient-frontend only | present in `.env`/`.env.example` | `[CONFIGURED BUT UNUSED]` — zero references in source |

No secret values were found hardcoded in any source file reviewed.

---

# GIT HISTORY

`GIT HISTORY NOT AVAILABLE` — no `.git` directory was present in the uploaded ZIP.

---

# DOCUMENTATION VS. CODE

| Claimed Feature | Documentation Says | Code Evidence | Reality |
|---|---|---|---|
| AI provider | `backend/agent-service/README.md`: *"sends each persisted RuralCare conversation to Claude"* | `main.py` imports and exclusively calls `groq.Groq`, never `anthropic` | **Overclaim / stale doc.** The agent service is 100% Groq-backed. The `anthropic` Python SDK is present in the shipped `.venv` (fully vendored, unused) and the `.vscode/tasks.json` prompt is literally named `"anthropicApiKey"` while it sets `$env:GROQ_API_KEY` — strong circumstantial evidence the project was originally built against Claude and migrated to Groq without updating the agent-service README or the VS Code task's input label |
| AI model | Top-level `README.md:15,141`: *"AI Engine: Groq LPU"*, *"Llama-3.3-70B on Groq LPU hardware"* | `main.py:87`: `model="openai/gpt-oss-120b"` | **Overclaim.** The provider (Groq) is correct, but the specific model is wrong — it's an OpenAI open-weight model served on Groq, not Llama 3.3 70B |
| Safety layer | `README.md:92`: *"A rule-based safety layer sits on top of the LLM. Fatal keywords...instantly trigger emergency escalation, bypassing the standard queue entirely."* | No keyword list, no bypass code, anywhere in `server.js` or `main.py` | **`[REFERENCED BUT NOT IMPLEMENTED]`** — the only real code-level safety mechanism is the `<<<SUMMARY>>>` leak-strip, which is unrelated to emergency triage |
| Latency claim | `README.md:90`: *"cuts AI response latency from ~3.0s to ~350ms"* | No latency logging/measurement code anywhere | `[UNKNOWN / NOT VERIFIABLE]` from this codebase — an infra-provider claim, not something the app measures itself |
| "Doctor login required" security posture | `server.js`'s own inline comment (`server.js:44-46`): *"This is what actually ties a request to a real doctor identity instead of the queue being open to anyone with the URL."* | `Login.jsx` (doctor-frontend) requires no password; anyone can self-declare `role: "doctor"` via `/api/login` | **The comment's own stated goal is not met by the implementation** — the queue *is* effectively open to anyone with the URL, since "identity" here means an unverified typed string |
| Database | `README.md:143`: *"Better-SQLite3 (dev) → PostgreSQL (production-ready)"* | Only SQLite (+ JSON fallback) implemented; no Postgres driver/config anywhere | `[PLANNED]`, accurately flagged as not-yet-done by the README itself — this is the one doc claim that is honest about its own gap |

---

# CODE QUALITY

**Best-written parts:**
- `backend/db.js`'s SQLite/JSON-fallback pattern — genuinely thoughtful defensive engineering against a real cross-platform native-module failure mode, with identical function signatures on both code paths.
- `main.py`'s `parse_agent_reply()` — small, well-commented, handles the two realistic edge cases (code-fence wrapping, empty pre-summary reply) it needs to.
- Inline comments throughout `server.js` explaining *why*, not just *what* (e.g., the 15MB JSON limit comment, the summary-leak defense-in-depth comment) — above-average for a hackathon codebase.
- `test-integration.mjs` — a real, runnable, assertion-based smoke test is unusual and valuable at this project stage.

**Weakest parts:**
- The doctor-auth trust model (see Security).
- Field-name drift between the LLM's schema (`relevant_history`) and both frontends' display code (`medical_history`) — a symptom of not having a single shared type/schema definition between the Python service, the Node backend, and the two separate React codebases. There is no OpenAPI spec, no shared TypeScript/JSON-schema types file anywhere — every layer re-guesses the shape of the "summary" object independently.
- `server.js` is a single 262-line file mixing routing, WebSocket management, and formatting logic — fine now, would need decomposition before adding more endpoints.
- Both frontend JSX files use long single-line JSX chains in places (e.g., `ChatPage.jsx:159-232` is effectively one giant return statement) — a readability, not correctness, issue.

**Technical debt, ranked:**
1. Doctor auth (Critical — see Security)
2. No shared schema/type contract between the three services (High — root cause of the `medical_history`/`relevant_history` bug and a growing risk as fields are added)
3. Regex-based LLM-output parsing instead of provider JSON-mode (Medium)
4. `server.js` lacking any routing/service-layer decomposition (Low at current size, will compound)

---

# DEAD CODE / UNUSED LOGIC

| Item | Status | Recommendation |
|---|---|---|
| `patient-frontend/src/pages/Login.jsx` | Unreachable in normal use; unrouted-to check, no route requires it | **DELETE** or actually wire a `ProtectedRoute` equivalent for patients if patient identity verification is ever needed |
| `patient-frontend/public/agent-connect.html` | POSTs to a route (`/api/agent`) that doesn't exist | **DELETE** — confirmed dead, not just unreferenced |
| `VITE_CHATZY_EMBED_URL` env var | Defined, never read | **DELETE** from `.env`/`.env.example`, or implement whatever "Chatzy" embed was planned |
| `start-all.ps1` | Hardcoded to the original author's absolute Windows path | **DELETE or fix** — `.vscode/tasks.json` already supersedes it with a portable equivalent |
| `anthropic` Python package in `.venv` | Installed, never imported in `main.py` | Not in `requirements.txt` (confirmed — only `fastapi, uvicorn, groq, pydantic, python-dotenv` are listed), so this is a `.venv` artifact from an earlier pip install, not a real dependency — **no action needed on `requirements.txt` itself, but confirms the Claude→Groq migration history** |
| `doctor-frontend/src/lib/api.js`'s `authHeaders()` on `fetchCase` | Sent, but the corresponding backend route (`GET /api/cases/:id`) doesn't check auth at all | Harmless (extra unused header), but signals the frontend *assumed* this route was protected when it isn't — worth fixing on the backend side, not the frontend |

---

# FEATURE MATRIX

| Feature | Implemented? | Evidence | Quality | Risk | Importance |
|---|---|---|---|---|---|
| Conversational LLM intake | ✅ IMPLEMENTED | `main.py`, `ChatPage.jsx` | Good | Medium (no output schema validation) | Core |
| Structured JSON summary extraction | ✅ IMPLEMENTED | `main.py:57-77` | Good, minor regex fragility | Low | Core |
| Voice input, multi-language (11 languages) | ✅ IMPLEMENTED | `ChatPage.jsx:9-22,104-157`, `main.py:98-124` | Good | Low | Differentiator |
| Live doctor queue via WebSocket | ✅ IMPLEMENTED | `server.js:58-79,244-259`, `Queue.jsx` | Functional | **High (unauthenticated socket)** | Core |
| Doctor decision → patient notification | ✅ IMPLEMENTED | `server.js:200-229`, `StatusPage.jsx` | Good | Medium (no verification decidedBy is a real clinician) | Core |
| Doctor authentication | ⚠️ PARTIALLY IMPLEMENTED (exists structurally, provides no real security) | `Login.jsx`, `server.js:47-56,108-116` | Poor | **Critical** | Core |
| Patient authentication | ❌ NOT IMPLEMENTED (page exists, unused) | `Login.jsx` (patient) | Dead code | Low (nothing depends on it) | Non-core |
| Rule-based emergency safety layer | ❌ **CLAIMED, NOT IMPLEMENTED** | README vs. `main.py`/`server.js` | N/A | **High (clinical)** | Core |
| Urgency-based queue sorting | ❌ NOT IMPLEMENTED | `db.js:126`, `Queue.jsx:62` (createdAt-only, no urgency sort) | N/A | Medium | Should-have |
| Case data persistence with fallback | ✅ IMPLEMENTED | `db.js` | Very good | Low | Core |
| Integration test coverage | ⚠️ PARTIALLY IMPLEMENTED (one happy-path test only) | `test-integration.mjs` | Good for what exists | Medium (auth/edge cases untested) | Should-have |
| Deployment configs (Docker) | ✅ IMPLEMENTED | both `Dockerfile`s | Simple, correct | Low | Should-have |
| CI/CD | ❌ NOT IMPLEMENTED | absent | N/A | Low at current stage | Nice-to-have |

---

# ARCHITECTURE DIAGRAMS

```mermaid
graph TD
    P[Patient Browser] -->|REST + WS, no auth| B[Node.js Hub<br/>server.js]
    D[Doctor Browser] -->|REST + WS, weak/no auth| B
    B -->|REST, internal-only| A[FastAPI Agent Service<br/>main.py]
    A -->|Chat completions| G[Groq API<br/>openai/gpt-oss-120b]
    A -->|Speech-to-text| W[Groq API<br/>whisper-large-v3]
    B -->|better-sqlite3| DB[(ruralcare.db<br/>SQLite, single table)]
    DB -.fallback if native module missing.-> J[(ruralcare.db.json)]
```

This diagram reflects only components with direct code evidence. No load balancer, cache, or message broker exists to diagram.

---

# SENIOR ENGINEER REVIEW

**What would get attacked first:** the doctor login. A reviewer would ask, within the first two minutes of reading `server.js`, "wait — where's the password check?" The inline comment claiming this "ties a request to a real doctor identity" while the actual check is `role === 'doctor'` on a client-supplied string is the kind of gap that reads as either a rushed hackathon shortcut (most likely, and forgivable in that context) or a genuine misunderstanding of what "auth" means (less likely given the rest of the codebase's competence).

**What would get questioned:** the README's safety-layer and specific-model claims — a careful reviewer diffing docs against code will find both within minutes, and it undermines trust in the rest of the documentation.

**What they'd ask to change before any real deployment:** real doctor credentials (even a shared password + Render env var would be a massive improvement over the status quo), WebSocket-level auth, and either implementing the claimed keyword-based emergency bypass or removing the claim from the README.

---

# HACKATHON JURY REVIEW

**Impressive, technically defensible:** the voice-input pipeline (real multi-language Whisper integration, not just a UI mockup), the SQLite/JSON dual-persistence fallback, the real integration test, and the genuinely-deployed live URLs (this is not vaporware — it runs).

**Weak demo areas / things that should NOT be claimed on stage:** the "rule-based safety layer" line — a technical judge who asks "show me the keyword list" will find nothing, which is a worse outcome than never having claimed it. The "Llama-3.3-70B" model claim will similarly not survive a judge who checks the actual API call. The "doctor identity" security claim should not be repeated in a live demo to a technically literate panel.

**Worth demonstrating:** the sub-second-feeling chat responsiveness, the voice input in a regional language, the live WebSocket push from patient submission to doctor queue.

---

# REFACTORING ROADMAP

## P0 — Critical (before any deployment with real patient data)
- Replace name-only doctor login with actual credential verification (even a shared secret per clinic beats the status quo). *Files: `server.js:47-56,108-116`, `doctor-frontend/src/pages/Login.jsx`.*
- Add authentication to the WebSocket handshake (verify the bearer token on connect, not just on REST calls). *File: `server.js:244-259`.*
- Add auth to `GET /api/cases/:id` or accept that it's intentionally public and document that decision explicitly. *File: `server.js:194-198`.*
- Either implement the claimed deterministic emergency-keyword bypass, or correct the README to describe what actually exists (prompt-level instruction only). *Files: `README.md:92`, `main.py`, `server.js`.*

## P1 — High value
- Fix the `medical_history`/`relevant_history` field-name mismatch in both frontends. *Files: `StatusPage.jsx`, `CaseDetail.jsx`.*
- Add urgency-based sorting to the doctor queue (`emergency` cases should float to the top regardless of submission time). *Files: `db.js:126` or `Queue.jsx`.*
- Introduce a single shared schema (even a plain JSON Schema or a hand-maintained TypeScript types file consumed by both frontends, mirrored in the Python Pydantic models) for the intake summary object, to prevent future field-name drift.
- Validate the LLM's summary JSON against a Pydantic model server-side before trusting it, rather than only checking it's parseable JSON.

## P2 — Useful
- Delete the dead patient `Login.jsx`, the orphaned `agent-connect.html`, the unused `VITE_CHATZY_EMBED_URL`, and the machine-specific `start-all.ps1` (superseded by `.vscode/tasks.json`).
- Add rate limiting to `/api/login`, `/api/cases/:id/chat`, and `/api/cases/:id/voice`.
- Restrict CORS to the known frontend origins instead of the wide-open default.

## P3 — Future (not worth touching now)
- Migrate SQLite → Postgres (README's own stated plan) — genuinely not needed at current scale.
- Move to Groq's constrained/JSON-mode output instead of regex parsing, if/when that becomes a real reliability problem in practice.

---

# DO NOT OVERENGINEER

`DO NOT ADD`: Kubernetes, a message broker, a vector database/RAG pipeline, or a second AI agent. Nothing in this codebase's actual traffic pattern (a single sequential chat → single summary → single doctor decision) justifies any of them at the project's current, demonstrated scale.

---

# FINAL MASTER TABLE

| Area | Current Reality | Evidence | Strength | Weakness | Priority |
|---|---|---|---|---|---|
| Architecture | Hub-and-spoke, 3 services, flat files internally | `server.js`, `main.py` | Simple, matches its own claimed diagram | No shared schema between services | P1 |
| Frontend | Two independent React SPAs, solid UX polish, voice input | all `.jsx` files | Good defensive coding (ErrorBoundary, resume-on-refresh) | Dead patient-login page; field-name bug | P1/P2 |
| Backend | Single Express file, clean REST+WS design | `server.js` | Readable, well-commented | No auth on socket/GET-case; open CORS | P0 |
| AI | Single Groq LLM call, regex-parsed structured output | `main.py` | Works, has a real fallback message | No output schema validation, no JSON-mode use | P1 |
| Database | SQLite w/ JSON fallback, single table, no migrations framework | `db.js`, live `ruralcare.db` | Excellent fallback engineering | No doctor/user table, unencrypted PHI-adjacent data | P2 |
| Security | Name-only doctor auth, open WebSocket, open CORS | `Login.jsx`, `server.js` | API keys handled correctly, parameterized SQL | Doctor auth is not real auth | **P0** |
| Testing | One real integration test, nothing else | `test-integration.mjs` | Genuinely useful, runnable | No unit/frontend/auth-path tests | P1 |
| DevOps | Two correct Dockerfiles, no CI, no compose | Dockerfiles | Simple, correct | No CI, Windows-only helper script is dead | P2 |
| Documentation | Polished, ambitious README with some overclaims | `README.md` vs. code | Genuinely good architecture diagrams | Safety-layer and model-name claims don't match code | **P0 (fix or implement)** |
| Product | Real, deployed, usable end-to-end demo | live Render URLs in README | Actually works | Doctor queue not urgency-sorted | P1 |

---

# FINAL CTO VERDICT

1. **What is this project actually?** A working, genuinely-deployed hackathon MVP for LLM-assisted rural telemedicine intake — three services, one LLM, one database table, real voice input, real WebSocket live-updates. It is not vaporware; the integration test and the live SQLite file with real data prove it runs.
2. **Strongest technical asset:** The SQLite/JSON-fallback persistence layer and the multi-language voice pipeline — both solve real, specific engineering problems (cross-platform native-module failure; rural India's language diversity) rather than being decorative.
3. **Weakest technical point:** Doctor authentication is not authentication. This single gap undermines every other privacy/security claim in the README.
4. **Biggest hidden risk:** The gap between the README's claimed "rule-based safety layer" for emergency triage and the actual code (prompt-only, no deterministic backstop) — this is the kind of gap that matters most in a healthcare-adjacent product, and it's not visible unless someone reads both the docs and the code side by side, as this audit did.
5. **Most impressive implementation:** `db.js`'s dual-mode persistence with an identical API surface across SQLite and JSON-file backends.
6. **What's currently only marketing:** the specific "Llama-3.3-70B" model claim, the "~350ms" latency figure (unverifiable from this code), and the "rule-based safety layer."
7. **Fix immediately:** doctor login, WebSocket auth.
8. **Should NOT be touched:** the SQLite fallback pattern, the voice-input flow, the regex summary parser (works fine at current reliability needs — swap it out only if it actually starts failing in practice).
9. **What should be deleted:** patient `Login.jsx`, `agent-connect.html`, `start-all.ps1`, unused `VITE_CHATZY_EMBED_URL`.
10. **What should be built next:** real doctor credentials, urgency-sorted queue, a shared intake-summary schema across all three services.
11. **What would make this significantly stronger:** closing the doc/code gap on the safety layer — either by actually building the deterministic keyword check the README already describes, or being honest that it's prompt-only.
12. **What would make a senior engineer reject it (for production, not for a hackathon):** the doctor-auth gap, full stop.
13. **What would make a technical jury remember it:** the voice-input-in-11-languages demo, live, on stage — that's the genuinely differentiated, verified-working feature.
14. **Is the architecture appropriate for its current scale?** Yes — three simple services, one LLM call per turn, one SQLite table. It is not overbuilt and does not need Kubernetes/Kafka/multi-agent orchestration at this stage.
15. **Minimum work to be genuinely production-grade:** real credentialed doctor auth + WebSocket auth (P0 above) + closing or correcting the safety-layer claim. Everything else is refinement on top of an already-functional core.
