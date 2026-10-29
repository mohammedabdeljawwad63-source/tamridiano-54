# Tamridiano (تمريضيانو)

Arabic (RTL) e-exam platform: turns text/PDF curriculum material into AI-generated
exams (MCQ, true/false, essay, short answer) with auto-grading and an AI tutor that
explains wrong answers in Arabic.

## Layout

- `app/main.py` — FastAPI app; mounts `/api` routers, `/static`, and an SPA
  catch-all that serves `index.html` for any non-`api/` path.
- `app/config.py` — env-driven settings. `APP_ROOT_PATH` defaults to empty (site at "/").
- `app/routers/` — `admin`, `catalogue`, `materials`, `exams`, `attempts`.
- `app/static/` — vanilla JS SPA. `app.js` holds hash routes; `index.html`
  loads assets with `?v=N` cache-busting.
- `app/services` / `app/llm.py` — LLM calls to the OpenAI-compatible provider.

## Running

```bash
pip install --user -r requirements.txt   # after a container reset
./dev.sh --daemon                        # supervised, restarts on crash
./dev.sh                                 # foreground (Ctrl+C to stop)
./run.sh                                 # bare uvicorn, no supervision
```

`dev.sh` reinstalls missing packages and restarts the server if it exits, which
is what keeps the site up on a dev box or bare VM. On a managed platform
(Render / Fly / docker compose) the platform supervises the process instead.

Leave `APP_ROOT_PATH` empty on a dedicated domain, so the site is served at `/`.
Set it (e.g. `APP_ROOT_PATH=/Tamridiano`) only behind a proxy that strips that
prefix. The HTML references assets relatively (`static/app.js`), so a non-empty
`ROOT_PATH` while serving at `/` makes the page load but every stylesheet and
script 404 — the site renders unstyled and looks broken while health checks
still pass. This was the default and it was caught only by running the built
Docker image.

## Capacity and staying up

The app is I/O bound — students wait on the database and the LLM, not the CPU —
so a few async workers serve many students. One 2-vCPU instance with 4 workers
holds roughly 600-800 concurrent students; scale by adding instances, not by
raising `WORKERS` past core count.

What limits throughput, in order:

1. **Blocking the event loop.** Anything CPU-heavy in an `async def` freezes
   every other request on that worker. PBKDF2 hashing (~45ms per login) and PDF
   extraction (~1s for a 500KB file) both did this; both now run via
   `asyncio.to_thread`. Keep new CPU work off the loop the same way.
2. **SQLite.** It serialises writes, so it is fine for one node but not for
   thousands of concurrent students. Set `DATABASE_URL` to PostgreSQL for a real
   exam day.
3. **LLM rate limits.** Generation is the slowest path and the provider throttles.
4. **Postgres connections.** Keep `instances x WORKERS x DB_POOL_SIZE` under
   `max_connections`.

Measured on this box: 400 concurrent logins complete 400/400 with health-check
latency staying under 70ms. Before the threading fix the same burst stalled a
worker for seconds, because every hash froze the loop it was waiting on.

## Auth

`app/auth.py` is the real implementation; `app/security.py` is a
backwards-compatible re-export shim. Two independent sessions:

- **Admin** (`tmr_admin` cookie) — passcode via `ADMIN_PASSCODE`, or Google
  sign-in for an address in `ADMIN_EMAILS`.
- **Student** (`tmr_user` cookie) — self-service account: the student registers
  with a name, a student number and a password, then signs in with the number and
  password. See "Student accounts" below. Google and email OTP remain available
  as optional alternatives.

Pick the dependency deliberately:

- `require_admin` — admin-only.
- `require_user` — signed-in student only.
- `require_identity` — student *or* passcode admin. Use this for anything an
  admin should be able to preview, such as `materials/{id}/file`; using
  `require_user` there locks the passcode admin out of the library.

Both session tokens are HMAC-signed with `SESSION_SECRET`; changing that value
signs everyone out.

## Deploying

`Dockerfile`, `docker-compose.yml` (Postgres + app + nginx), `deploy/render.yaml`,
and `deploy/fly.toml` are ready. See `DEPLOY.md`. For 2500+ concurrent students
the app must run on PostgreSQL, not SQLite — SQLite serialises writes.

## Conventions and gotchas

- **Path converters must be typed.** Dynamic routes use `/{id:int}`; otherwise a
  static sibling such as `/history` or `/admin/overview` gets swallowed by the
  dynamic route.
- **`ev.currentTarget` is null after `await`.** Every async click handler captures
  the element first (`const btn = ev.currentTarget;`) before any `await`, then
  passes `btn` to `busy()`.
- **Hash-only navigation does not reload the document.** To exercise a real page
  reload in browser testing, change the query string too (`?r=2#exam`).
- **Bump the `?v=N` query on both `styles.css` and `app.js`** in `index.html`
  whenever you touch either file, or browsers keep the cached copy.
- **Exam resume** uses `sessionStorage["tmr_attempt"]`; answer drafts use
  `localStorage["tmr_draft_<attemptId>"]` and are cleared on submit.
- **Resume responses never include correct answers** — only the attempt-taking
  payload. Submitted attempts are blocked from resume.
- **New `sections` columns need a default.** `created_at` is `NOT NULL` with no
  server default, so any raw `INSERT` must supply it; the year seeding in
  `app/db.py` does.
- **Serve originals inline, not as object URLs.** `materials/{id}/file` sends the
  real content type plus an RFC 5987 filename, so a phone routes it to its native
  PDF/Word viewer. iOS Safari renders a `blob:` PDF as a blank page, and a blob
  carries no filename into the receiving app — so the library navigates directly.
- **Uploads must be decoded, not assumed UTF-8.** Arabic `.txt` from Windows
  Notepad is often `cp1256`; naive `data.decode()` raises or produces mojibake
  that still looks non-empty. `app/extract.py` handles BOM/UTF-8/legacy
  code pages and rejects unusable text (scanned PDFs, garbage) with an Arabic
  message instead of storing it. Both upload paths go through it.
- **Chunk generation runs concurrently.** A multi-chunk PDF fans out one LLM
  call per chunk via `asyncio.gather`; sequential calls made an 80k-char PDF
  take ~180s, which admins read as "it hung". Partial failures are tolerated
  (partial exam) and only an all-chunk failure raises.

## Verification

- End-to-end API suite: `python3 /tmp/e2e_test.py` — 49 checks covering generation,
  start/resume dedup, submission, grading, AI explanations, and admin stats.
- Student account suite: `python3 /tmp/student_test.py` — 26 checks covering
  register, duplicate/short-password rejection, sign-in, Arabic-Indic digit and
  case folding, wrong-password and unknown-number handling, per-account result
  isolation, and a session whose account row was deleted. It writes to the real
  DB, so delete the `T*`/`ZZ*` accounts it creates afterwards.
- Auth/serving/years suite: 34 checks covering the sign-in gate, signed file
  serving (MIME, RFC 5987 name, Range, 401 without a session), year CRUD and
  reordering, and a full attempt lifecycle.
- Extraction/encoding suite: covers `cp1256`, `utf8`, `utf16`, scanned PDF,
  garbage bytes, and unsupported suffix, asserting 201 vs 422 per case.
- Browser flow: `#subjects` → exam → answer → reload → submit → `#result`,
  plus `#results`, `#library`, and `#admin` (passcode in `.env`).
- Reset state with `rm -f data/tamridiano.db*` and restart the server.
- LLM latency dominates generation: a 3-question exam from a large PDF takes
  ~20s, 40 mixed questions ~45s. Time budgets in tests must allow this.

## Front-end form bug (fixed)

- **`adminGenerate` count inputs must match `countBox` ids.** `countBox("truefalse",…)`
  renders `id="g-truefalse"`, but the submit handler read `$("#g-tf")`. That threw
  `TypeError: Cannot read properties of null (reading 'value')` *before* the fetch,
  so the button did nothing and no request reached the server — this was the
  "مش راضي يعمل اختبارات" report. Keep rendered ids and handler selectors in sync.
- A click that produces no request at all is a synchronous JS error, not a network
  or SSE problem. Instrument with `window.fetch` + `unhandledrejection` and read
  `#gen-err`; the stack points straight at the bad selector.
- **Library file preview requires admin.** `/api/materials/{id}/text` and
  `/api/materials/{id}/file` are `require_admin`, so `renderLibrary` checks
  `/admin/session` and only renders the "فتح" button for admins.

## Real-browser UI testing

- Headless Chromium is at `/usr/bin/chromium`. Drive it over CDP with
  `websockets` (installed) when a UI bug depends on live DOM values: the plain
  browser tool cannot commit typed values into this SPA, so forms look empty and
  clicks appear to do nothing.
- `app.js` is an IIFE — its functions are not global. Drive the UI with real
  `.click()` / `dispatchEvent` calls on queried elements, not by calling functions.

## Student accounts (name + number + password)

- This is the primary sign-in. A student opens the gate, picks "حساب جديد",
  and registers with a name, a student number and a password. No email and no
  Google account are needed, which is what makes it work on a phone in class.
- `app/students.py` owns everything: `register`, `authenticate`,
  `normalise_number`, `hash_password`, `verify_password`, `change_password`, and
  `StudentError` (whose message is Arabic and safe to show the student).
- The account is the anchor for results. An attempt taken while signed in stores
  `attempts.student_account_id`, and `/api/attempts/history` reads the session,
  so a student only ever sees their own work.
- Endpoints live in `app/routers/auth.py`: `POST /api/auth/register`,
  `POST /api/auth/login`, `GET /api/auth/me`. `/api/auth/methods` reports
  `student_accounts` and `signup_enabled`.

### Passwords

- PBKDF2-HMAC-SHA256, per-user random salt, `PASSWORD_ITERATIONS` rounds
  (default 240000). Standard library only — no bcrypt/argon2 dependency.
- The iteration count is stored inside the hash string
  (`pbkdf2_sha256$rounds$salt$digest`), so it can be raised later without
  invalidating existing accounts.
- `verify_password` is constant-time (`hmac.compare_digest`) and treats a
  malformed stored hash as a failed login rather than crashing.

### Number and name handling

- `normalise_number` folds Arabic-Indic digits (٠-٩ and ۰-۹) to ASCII, then
  strips spaces, dashes and underscores. So `2024 0012`, `2024-0012` and
  `٢٠٢٤٠٠١٢` are all one account.
- Matching is case-insensitive via `func.lower(...)`, backed by a unique index.
  A losing race in `register` surfaces as a friendly Arabic message, not a 500.
- Length bounds come from `STUDENT_NUMBER_MIN`/`STUDENT_NUMBER_MAX` and
  `STUDENT_MIN_PASSWORD`.

### Gotchas

- **Login failures are deliberately vague.** An unknown number and a wrong
  password return the same Arabic message, and an unknown number still runs a
  dummy `hash_password` so response timing does not reveal which numbers exist.
  Do not "improve" the message — it is what stops account enumeration.
- **`STUDENT_MAX_FAILED_LOGINS` (default 8) locks the account** for
  `STUDENT_LOCKOUT_MINUTES` (default 15). A test that logs in wrongly more than
  that many times will then fail with the lockout message, not the usual one.
- **Set `STUDENT_SIGNUP_ENABLED=false` to close registration.** The endpoints
  return 403 and the gate hides the "حساب جديد" tab; existing accounts still log
  in.
- **A session outlives the account row.** Tokens are stateless, so deleting a
  student does not invalidate tokens already handed out. `_account_still_exists`
  in `app/auth.py` re-checks `students.id` on every request and resolves a
  vanished account to a signed-out visitor. Without it, the stale token passes
  every check and then dies on the `attempts.student_account_id` foreign key,
  turning an account deletion into 500s for that student.
- **The gate is not the security boundary.** `/api/attempts/history` and the
  attempt endpoints enforce the session server-side; hiding a button proves
  nothing.

## Optional sign-in: email OTP

Kept as an alternative. `app/otp.py` owns generation and verification,
`app/routers/auth.py` the endpoints, `app/mailer.py` the delivery.
- Codes are stored as an HMAC, never in clear text, and are single-use. Issuing a new
  code marks every outstanding one consumed, so only the newest is ever valid.
- `EMAIL_PROVIDER=console` logs the code instead of sending it — that is the default and
  the only mode that works without a provider key. `resend` and `brevo` send for real.
- An attempt is bound to the verified address, not a typed name. `/api/attempts/history`
  reads the session, so one student can never see another's results.

### Gotchas

- `OTP_RESEND_SECONDS` (default 60) throttles re-requests per address. A test that reuses
  a fixed email will fail with `429` on a second run inside that window; use a unique
  address per run or set `OTP_RESEND_SECONDS=0` for the process.
- `purge_expired` only deletes codes older than a day, so recently issued rows stay on
  purpose. Startup calls it via `_purge_dead_login_codes` in `app/db.py`.

## LLM provider

- `https://codecraftapi.com/v1`, OpenAI-compatible. The key lives in `.env` as
  `LLM_API_KEY` and is verified working.
- `gemini-3.7-flash` spends part of its token budget on reasoning. A small `max_tokens`
  can return `finish_reason: "length"` with empty content, which looks like a broken key
  but is only truncation — allow a few hundred tokens for any real answer.
- Verify the key with `app.llm.chat(...)` rather than raw curl, since that exercises the
  same client the generator uses.

## Data safety

- `init_db` writes a snapshot of the SQLite file to `data/backups/` on every start,
  keeping the newest `DB_BACKUP_KEEP` (default 10). `data/` is gitignored, so these
  never leave the machine.
- Deleting a section cascades to its subjects, and a subject cascades to its exams,
  questions, attempts and materials (`ON DELETE CASCADE` in `app/models.py`). Deleting
  one section can therefore erase a large amount of unrelated-looking content.
- SQLite keeps recent writes in `tamridiano.db-wal`. Reading the `.db` file alone can
  show older, still-complete data — that is how lost rows were recovered once. Copy
  `.db`, `-wal` and `-shm` together before touching anything.
- When cleaning test data by hand, scope deletes with `WHERE id = ?` or a test-only
  name prefix. An unqualified `DELETE FROM exams` wipes real exams too.

## SQLite and PostgreSQL differ in ways that only show up at deploy time

Raw SQL in `app/db.py` must satisfy both engines. SQLite is lenient where asyncpg
is strict, so a query that passes every local test can still crash startup on
PostgreSQL — and startup is the first thing that runs, so the site simply never
comes up. Two such bugs were found only by running against a real PostgreSQL:

- `created_at` was bound as an ISO string. SQLite stores it happily; asyncpg
  raises `expected a datetime.date or datetime.datetime instance, got 'str'`.
- The seeded years omitted `is_active`. The model's `default=True` is a
  Python-side default that raw SQL bypasses, and unlike the SQLite DDL path the
  PostgreSQL table has no column default, so it raised `NotNullViolationError`.

Rule: when writing raw SQL, pass real Python objects and list every
`NOT NULL` column explicitly. Then test the change against PostgreSQL, not just
SQLite:

```bash
docker run -d --name tmri-pgtest -e POSTGRES_USER=tamridiano \
  -e POSTGRES_PASSWORD=testpass -e POSTGRES_DB=tamridiano -p 55432:5432 postgres:16-alpine
DATABASE_URL="postgresql+asyncpg://tamridiano:testpass@127.0.0.1:55432/tamridiano" \
  python3 -m uvicorn app.main:app --port 12050
```

## Source archive download (temporary)

`tools/make_source_zip.py` writes `data/tamridiano-code.zip` - the project source
without `.env`, `data/` or `.git`. With `DOWNLOAD_TOKEN` set in `.env`, the app
serves it at `GET /source/<token>` (404 otherwise). This exists only so the code
can be retrieved without a GitHub account; remove the route once the repository
is the source of truth.

## Free-host deployment notes

`app/keepalive.py` requests the app's own `/api/health` every
`KEEP_ALIVE_INTERVAL_MINUTES` when `KEEP_ALIVE=true`, which stops a free Render
or Koyeb service from being suspended for inactivity. One worker is elected via
a file lock so the timer is not duplicated. It cannot wake a service that is
already asleep - that needs an external pinger.

`app/config.py::_asyncpg_url` rewrites a managed Postgres URL into the form
asyncpg accepts. Providers hand out `postgres://` with `sslmode=require` and
`channel_binding=require`; asyncpg rejects both (it spells sslmode as `ssl` and
has no channel binding), so pasting a provider URL unchanged makes the app fail
to start.

Two limits of free tiers to keep in mind: a Render free web service has an
ephemeral filesystem (SQLite data is lost on every restart or spin-down, so a
free managed Postgres such as Neon is required for persistence), and Render
blocks outbound SMTP ports 25/465/587, so student sign-in codes must use the
HTTP providers (resend, brevo) rather than gmail SMTP.
