
# TSU AI Helpdesk — `todo.md`

> **Mode:** Speed & simplicity MVP
> **Goal:** Ship a lean, working app that meets the spec with minimal dependencies and safe, iterative steps.
> **Definition of Done (DoD):** All P0 items checked; app can serve bilingual (KA/EN) Q&A with GPT-4o-mini, single-PDF per question (≤10MB), auth, rate-limits (5/day), storage cap (250MB auto-delete), chat history & deletion, basic admin stats, exports, and moderation fallback.

---

## Legend

* **Priority:** P0 (must), P1 (should), P2 (nice-to-have)
* **Status:** ☐ todo · ▣ in-progress · ☑ done
* Use `[ ]`, `[x]` to track progress.

---

## 0) Repo & Foundations

* [ ] **Create monorepo structure** (P0)

  * [ ] `tsu-ai-helpdesk/` with `frontend/`, `backend/`, `.gitignore`, `README.md`
* [ ] **Initialize Git & CI skeleton** (P1)

  * [ ] GitHub/GitLab repo
  * [ ] CI job: install, build, run tests (no deploy)
* [ ] **Node versions & package managers** (P0)

  * [ ] Pin Node LTS in `.nvmrc`
  * [ ] Use npm (default) across FE/BE
* [ ] **Environment templates** (P0)

  * [ ] `backend/.env.example` with:
    `OPENAI_API_KEY, DB_URL (sqlite file path), JWT_SECRET, ADMIN_EMAIL, ADMIN_PASSWORD, STORAGE_DIR, FRONTEND_ORIGIN`
  * [ ] `frontend/.env.example` with: `VITE_API_BASE`

---

## 1) Backend: Bootstrap (Express)

* [ ] **Scaffold Express app** (P0)

  * [ ] `backend/src/index.js` — CORS, JSON body parsing, health route `/api/health`
  * [ ] Nodemon dev script
* [ ] **Config loader** (P0)

  * [ ] `dotenv` load & validate required envs at boot (fail-fast)
* [ ] **Logger** (P0)

  * [ ] Minimal structured logger (console) with request-id middleware

**DoD:** `GET /api/health` returns `{status:'ok'}` from local dev.

---

## 2) Frontend: Bootstrap (Vite + React + Tailwind)

* [ ] **Vite React app** (P0)

  * [ ] Tailwind configured
  * [ ] Base layout & theme (light/dark toggle stub)
* [ ] **Health check component** (P0)

  * [ ] Calls `/api/health` and shows status

**DoD:** Frontend dev server up, health status visible.

---

## 3) Auth (Simple, JWT)

* [ ] **User model & password hashing** (P0)

  * [ ] bcrypt/argon2 hash
* [ ] **Auth routes** (P0)

  * [ ] `POST /api/auth/signup {email,password}`
  * [ ] `POST /api/auth/login {email,password}` → JWT (1 day)
  * [ ] `GET /api/auth/me` (JWT)
* [ ] **Auth middleware** (P0)

  * [ ] Verifies JWT, injects `req.user`
* [ ] **Admin bootstrap** (P1)

  * [ ] Seed admin user from env at boot if missing

**DoD:** Able to sign up, login, and call a protected route with JWT.

---

## 4) Database (SQLite for speed)

* [ ] **SQLite setup** (P0)

  * [ ] `better-sqlite3` with file path from env
* [ ] **Migrations** (P0)

  * [ ] Create tables:

    * [ ] `users(id, email unique, password_hash, created_at, last_login, storage_used_bytes default 0, daily_count default 0, daily_reset_at)`
    * [ ] `sessions(id, user_id, session_token, created_at, expires_at)` *(optional if JWT only)*
    * [ ] `chats(id, user_id, created_at)`
    * [ ] `messages(id, chat_id, role, content, timestamp, user_id nullable)`
    * [ ] `files(id, user_id, filename, disk_path, size_bytes, uploaded_at)`
    * [ ] `questions(id, user_id, message_id, file_id nullable, tags json, admin_phone_shown text, created_at)`
    * [ ] `feedback(id, question_id, user_id, rating int, comments text, created_at)`
    * [ ] `errors(id, component, error_message, stack, created_at)`
    * [ ] `usage_logs(id, user_id, event_type, metadata json, created_at)`
* [ ] **DB helper layer** (P0)

  * [ ] Minimal CRUD for users, chats, messages, files

**DoD:** DB file created; migration script runs idempotently.

---

## 5) Chat Loop (Echo → OpenAI)

* [ ] **Chat routes** (P0)

  * [ ] `POST /api/chat/start` → new chat row, returns chatId
  * [ ] `POST /api/chat/:chatId/message {text, fileId?}` (protected)

    * [ ] Validates limits (see §7)
    * [ ] Echo reply for now
    * [ ] Persists user/ai messages
  * [ ] `GET /api/chat/:chatId/history` (protected)
* [ ] **Frontend Chat UI** (P0)

  * [ ] Message list (role: user/ai/system)
  * [ ] Composer (textarea + send)
  * [ ] Loading indicator (“AI is typing…”)
* [ ] **OpenAI integration** (P0)

  * [ ] Replace echo with GPT-4o-mini completion
  * [ ] `.env` `OPENAI_API_KEY` required
  * [ ] Error handling: fallback string & log

**DoD:** Logged-in user can send a message and receive AI response; history displays.

---

## 6) PDF Upload (Single per Question, ≤10MB)

* [ ] **Backend file upload** (P0)

  * [ ] `POST /api/files/upload` (JWT) — multer to `STORAGE_DIR/uploads`
  * [ ] Enforce MIME `application/pdf`
  * [ ] Enforce size ≤ 10MB (reject with spec message)
  * [ ] Record in `files` table; update `storage_used_bytes`
* [ ] **Auto-delete oldest files over 250MB** (P0)

  * [ ] Before accept, compute total + new → delete oldest until ≤ 250MB
  * [ ] Return list of deleted filenames to show as system messages
* [ ] **Frontend file attach** (P0)

  * [ ] `<input type="file" accept="application/pdf" />`
  * [ ] Show selected file name
  * [ ] Upload on send; obtain `fileId` for chat message
* [ ] **PDF text extraction** (P1)

  * [ ] `pdf-parse` best-effort extraction
  * [ ] If no text detected: flag `needs_ocr` (skip OCR in MVP)
  * [ ] Snippet (first ~200 words) available for prompt context

**DoD:** Can attach **one** PDF to a message; enforced size; storage cap with FIFO deletion & system messages.

---

## 7) Limits & Policies

* [ ] **Daily question limit (5/day/account)** (P0)

  * [ ] Track `daily_count`, `daily_reset_at`
  * [ ] Reset if `now - daily_reset_at >= 24h`
  * [ ] Block with system message on limit reached
* [ ] **Bilingual support (KA/EN)** (P0)

  * [ ] UI copy available in KA/EN (manual toggle or auto based on browser)
  * [ ] Prompt template instructs model to answer in the user’s question language
* [ ] **Tone & content rules** (P0)

  * [ ] Post-process responses to ensure standard length & neutral tone (heuristic)
* [ ] **No onboarding, blank start** (P0)

**DoD:** Limits enforced; UI and responses work in Georgian/English.

---

## 8) Retrieval Context (Simple, No External Vector DB)

* [ ] **Public data stub** (P1)

  * [ ] Minimal keyword search over a folder of curated TSU HTML/PDF snippets *(optional for MVP speed)*
* [ ] **Attach PDF snippet to prompt** (P0)

  * [ ] If message has `fileId`, include extracted snippet + simple “source: filename” footer in AI message

**DoD:** AI can reference uploaded PDF snippet; simple “Sources” list appended.

---

## 9) Moderation & Fallback

* [ ] **Basic moderation** (P0)

  * [ ] Blocklist JSON (words/phrases)
  * [ ] `checkModeration(text) → allow/deny`
  * [ ] If denied: system message per spec
* [ ] **Low-confidence fallback** (P1)

  * [ ] If PDF snippet empty AND no public data hits → show admin phone mapping (env-based single number in MVP)

**DoD:** Inappropriate content blocked; low confidence shows phone number.

---

## 10) Admin Basics

* [ ] **Admin auth** (P1)

  * [ ] Admin-only JWT via separate login or claim on user
* [ ] **Stats endpoint** (P1)

  * [ ] `GET /api/admin/usage` — daily counts, total users, per-user question count, top 20 normalized questions
* [ ] **Frontend admin page** (P1)

  * [ ] `/admin` simple table view

**DoD:** Admin can see usage stats in a basic page.

---

## 11) History, Deletion, Export

* [ ] **Message delete** (P0)

  * [ ] `DELETE /api/chat/:chatId/message/:messageId` (owner only)
* [ ] **File delete** (P0)

  * [ ] `DELETE /api/files/:fileId` (owner only) — update storage counters
* [ ] **Export** (P1)

  * [ ] `POST /api/chat/:chatId/export` → JSON (messages + metadata) and presigned-like local links to files
  * [ ] Frontend “Export chat” button (downloads JSON or ZIP)

**DoD:** Users can delete messages/files; can export chat JSON.

---

## 12) UX Polish (MVP)

* [ ] **Timestamps** (P0)
* [ ] **Typing indicator** (P0)
* [ ] **Error toasts** (P0)
* [ ] **System messages for limits & deletions** (P0)
* [ ] **Light/Dark toggle** (P1)

**DoD:** Smooth chat feel, clear system feedback.

---

## 13) Logging & Error Handling

* [ ] **Usage logs** (P0)

  * [ ] Log: question submitted (user_id, chat_id, ts)
* [ ] **Errors table** (P0)

  * [ ] Uncaught route errors captured & saved (component, message, stack, ts)
* [ ] **Feedback hook** (P1)

  * [ ] `POST /api/feedback {questionId, rating, comments}`

**DoD:** Errors and key usage events recorded.

---

## 14) Security Notes (MVP)

* [ ] **Password hashing** (P0)
* [ ] **JWT secrets & expiry** (P0)
* [ ] **File access control** (P0)

  * [ ] Only owners can download their PDFs (local disk path not guessable)
* [ ] **CORS** (P0)

  * [ ] Restrict to `VITE_API_BASE` in env
* [ ] **No TLS in MVP** (P0)

  * [ ] Document risk in README, keep code toggle-ready for TLS behind proxy

**DoD:** Core safeguards in place; risks documented.

---

## 15) Testing Plan Execution

* **Unit Tests** (P0)

  * [ ] Upload validation (type/size)
  * [ ] Rate limit logic
  * [ ] Storage FIFO deletion logic
  * [ ] Auth password hashing & JWT verify
  * [ ] Message CRUD
* **Integration Tests** (P1)

  * [ ] Auth → Chat → AI response path
  * [ ] Upload + message with `fileId` → extraction → prompt context
  * [ ] Export endpoint returns valid JSON
* **E2E Smoke (manual acceptable for MVP)** (P0)

  * [ ] Signup/login
  * [ ] Ask 5 questions → 6th blocked
  * [ ] Upload PDFs until >250MB → oldest auto-deleted, system messages displayed
  * [ ] Moderation block works
  * [ ] KA/EN responses correct language
* **Load (basic)** (P2)

  * [ ] 50 concurrent chat messages round-trip locally

**DoD:** All P0 tests pass locally; E2E smoke validated.

---

## 16) Frontend Wiring Details

* [ ] **State model**

  * [ ] `auth` (token, user)
  * [ ] `chats` (currentChatId, messages[])
  * [ ] `ui` (loading, theme, toasts)
* [ ] **API client**

  * [ ] Injects JWT, handles 401 → logout
* [ ] **Routes**

  * [ ] `/login`, `/chat`, `/admin` (guarded)
* [ ] **No orphaned components**

  * [ ] All buttons wired to real endpoints

**DoD:** Clicking through UI performs real actions.

---

## 17) Prompts & Model Use

* [ ] **Prompt template** (P0)

  * [ ] System: “You are TSU helpdesk… formal/neutral, bilingual…”
  * [ ] Include PDF snippet and short “Sources” list
  * [ ] Instruct: answer in user message language
* [ ] **Moderation pre-check** (P0)
* [ ] **Fallback to admin phone** (P1)

**DoD:** Responses adhere to tone/length; sources appended.

---

## 18) Ops & Manual Deployment

* [ ] **Single-host deployment** (P0)

  * [ ] Build FE → serve with Express static
  * [ ] `app.use(express.static('../frontend/dist'))`
  * [ ] `app.get('*', send index.html)`
* [ ] **Procfile or start script** (P0)
* [ ] **Backup notes** (P1)

  * [ ] Periodic copy of SQLite file and `STORAGE_DIR`

**DoD:** App runs on a single VM/service (Render/Railway/DO).

---

## 19) Documentation

* [ ] **README** (P0)

  * [ ] Quickstart (dev)
  * [ ] Env vars
  * [ ] Run scripts FE/BE
  * [ ] Known limitations (no TLS, OCR skipped)
* [ ] **API doc (brief)** (P1)

  * [ ] Routes, sample requests/responses
* [ ] **Admin usage** (P1)

**DoD:** New dev can boot in <15 minutes.

---

## 20) Acceptance Gates (Ship Checklist)

* [ ] Login, chat, history visible
* [ ] 5 Q/day limit enforced with correct message
* [ ] PDF upload ≤10MB; reject otherwise
* [ ] Storage cap 250MB with FIFO deletion + system messages
* [ ] AI replies KA/EN appropriately
* [ ] Moderation blocks prohibited requests
* [ ] Admin page shows usage stats
* [ ] Delete message & delete file work and reflect in UI and storage
* [ ] Export chat JSON works
* [ ] README complete; `.env.example` complete

---

## 21) Post-MVP (Queue, optional)

* [ ] OCR for scanned PDFs (Tesseract)
* [ ] Proper vector search (PG FTS or Lite vector lib)
* [ ] Better confidence heuristics + per-topic admin phones
* [ ] Rate limiting per-user per-day using Redis
* [ ] TLS termination config

---

### Notes & Risk Acknowledgments

* Running without TLS is insecure; plan to enable behind a proxy for any public traffic.
* Uploaded PDFs may contain sensitive data; deletion flows must be reliable.
* OpenAI latency dominates perceived performance; keep streaming & typing indicator responsive.
