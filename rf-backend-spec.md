# RF Portfolio — Backend Spec (Express API)

Give this file to Codex together with `admin-dashboard.html` (admin design) and the portfolio prototype (public design). The API runs on a Linux VPS. The public site is Next.js on Vercel. The admin is a separate React app.

## 1. Goals
- One owner (Rifki) edits content. Visitors only read.
- The public site must keep working if the API or VPS is down (Next.js caches pages; the API is only called at build/revalidate time).
- Publishing is an explicit action, like a commit: edits are saved as drafts, then published as a versioned snapshot.
- Keep it small and boring: easy to run, easy to back up.

## 2. Stack
| Concern | Choice |
|---|---|
| Runtime | Node.js LTS, TypeScript (strict) |
| HTTP | Express 5 |
| DB | MySQL 8 or MariaDB on the same VPS, listening on localhost only, via `mysql2`; migrations + queries with Drizzle ORM. SQLite is a valid lighter alternative; only the driver and schema dialect change. |
| Validation | Zod (every body, query and param) |
| Auth | Single owner, `argon2` (argon2id) password hash, server-side session in SQLite, `httpOnly` cookie, TOTP 2-step (`otplib`) |
| Security middleware | `helmet`, `cors` (allowlist), `express-rate-limit`, `hpp` |
| Uploads | `multer` (memory, size cap) + `sharp` (convert to WebP) |
| Logging | `pino` (no secrets, no passwords) |
| Process | PM2 or systemd, Nginx reverse proxy, HTTPS (Let's Encrypt) |

Install the latest stable versions at setup time. Do not copy versions from this file.

## 3. Folder structure
```
src/
  server.ts            app bootstrap, middleware order
  config.ts            env parsing (zod), fail fast
  db/ schema.ts migrate.ts client.ts
  auth/ session.ts password.ts totp.ts
  routes/ public.ts admin-auth.ts admin-content.ts admin-media.ts admin-publish.ts contact.ts
  services/ snapshot.ts revalidate.ts media.ts
  middleware/ requireAuth.ts csrf.ts rateLimit.ts error.ts
data/ uploads/
```

## 4. Environment variables
```
NODE_ENV=production
PORT=4000
DATABASE_URL=mysql://rf_app:<password>@127.0.0.1:3306/rf_portfolio
ANALYTICS_SALT_SECRET=<long random>
UPLOAD_DIR=./data/uploads
PUBLIC_SITE_ORIGIN=https://example.com
ADMIN_ORIGIN=https://admin.example.com
SESSION_SECRET=<long random>
OWNER_EMAIL=...
REVALIDATE_URL=https://example.com/api/revalidate
REVALIDATE_SECRET=<long random>
```
Never commit `.env`. Secrets are server-side only.

## 5. Data model (SQLite)
All tables have `id` (integer pk), `created_at`, `updated_at`.

- **projects**: `slug` (unique), `title`, `description`, `tech`, `categories` (JSON array of keys), `status` (`draft|published`), `featured` (bool), `featured_order` (int), `is_example` (bool), `image_id` (nullable fk), `live_url` (nullable), `repo_url` (nullable), `results` (nullable, only real verified outcomes).
- **experience**: `role`, `company`, `period`, `description`, `sort`, `is_example`.
- **certificates**: `name`, `issuer`, `year`, `credential_url` (nullable), `sort`, `is_example`.
- **stack_groups**: `name`, `sort`. **stack_items**: `group_id`, `name`, `logo_key` (nullable), `sort`.
- **assistant_presets**: `question`, `answer`, `go_section` (nullable: about, stack, projects, experience, certificates, contact), `sort`.
- **messages**: `name`, `email`, `phone` (nullable, stored normalized as E.164, e.g. `+62812…`), `body`, `consent` (bool), `read` (bool), `ip_hash`, `created_at`.
- **media**: `filename`, `mime` (always `image/webp`), `width`, `height`, `bytes`, `alt`.
- **settings**: key/value (`availability`, `site_title`, social links). `availability` is empty by default and is only shown when set.
- **users**: single row. `email`, `password_hash`, `totp_secret`, `failed_logins`, `locked_until`.
- **sessions**: `token_hash`, `user_id`, `expires_at`, `csrf_token`, `ip`, `user_agent`.
- **snapshots**: `version` (int, increasing), `payload` (JSON), `commit_hash`, `summary` (JSON diff), `published_at`.
- **audit_log**: `action`, `entity`, `entity_id`, `meta` (JSON), `created_at`.
- **page_views_daily**: `day`, `path`, `views`, `uniques` (unique key `day+path`). **referrers_daily**: `day`, `source`, `views`. **visitors_daily**: `day`, `visitor_hash` (unique key `day+visitor_hash`; rows older than 2 days are deleted). See section 13.

Valid category keys (must match the site filters): `php, javascript, mobile, website, web-app, desktop, api, devops`. Reject unknown keys.

Draft edits write to the content tables. **Published state = latest row in `snapshots`.** The public API only ever serves the snapshot, never live tables.

## 6. API

### Public (read-only, no auth, cacheable)
| Method | Path | Notes |
|---|---|---|
| GET | `/api/public/content` | Latest snapshot payload: profile, projects (published only), experience, certificates, stack, assistant presets, settings. Send `ETag` + `Cache-Control: public, max-age=60, stale-while-revalidate=600`. |
| GET | `/api/public/projects/:slug` | One published project or 404. |
| GET | `/media/:filename` | Static WebP from `UPLOAD_DIR` (serve via Nginx in production, long cache). |
| POST | `/api/contact` | Contact form: `name` (1–80, required), `email` (valid, required), `phone` (optional, normalize to E.164, default country +62, reject if not 8–15 digits), `message` (10–2000, required), `consent` (must be true). Honeypot field. Strict rate limit (e.g. 3 per hour per IP). Store only, no HTML. Optionally email the owner a notification. |
| POST | `/api/track` | Page-view beacon from the public site (section 13). Tiny body, always returns 204, rate limited, no cookies. |

### Auth
| Method | Path | Notes |
|---|---|---|
| POST | `/api/admin/login` | Step 1: email + password. Returns `{ needs2fa: true }`. Same error for unknown email and wrong password. |
| POST | `/api/admin/login/2fa` | Step 2: 6-digit TOTP. On success set session cookie. |
| POST | `/api/admin/logout` | Destroys the session. |
| GET | `/api/admin/me` | Returns `{ email, csrfToken }` or 401. |

Rules: 5 login attempts per 15 min per IP, and a per-account lock (increasing delay) after repeated failures. Session cookie: `httpOnly`, `Secure`, `SameSite=Lax`, domain scoped to the parent domain so `admin.` and `api.` share it, 12 h expiry, rotate on login. Hash tokens in the DB.

### Admin content (all require session + CSRF header `X-CSRF-Token` on non-GET)
For each of `projects`, `experience`, `certificates`, `stack-groups`, `assistant-presets`:
- `GET /api/admin/<resource>` list (include draft state)
- `POST /api/admin/<resource>` create
- `PATCH /api/admin/<resource>/:id` update (partial)
- `DELETE /api/admin/<resource>/:id`
- `PUT /api/admin/<resource>/order` reorder (array of ids)

Extra:
- `GET /api/admin/messages`, `PATCH /api/admin/messages/:id` (`read`), `DELETE /api/admin/messages/:id`
- `GET /api/admin/analytics?range=14d|30d` → `{ today, last7, total, daily: [{day, uniques, views}], topPages: [...], referrers: [...] }`
- `GET/PUT /api/admin/settings`
- `POST /api/admin/media` (multipart, field `file`, max 5 MB, accept jpeg/png/webp only, verify magic bytes, convert to WebP, strip metadata, resize to max 1600 px wide, return `{ id, url, width, height }`). `DELETE /api/admin/media/:id`.

### Publish
| Method | Path | Notes |
|---|---|---|
| GET | `/api/admin/publish/preview` | Diff between draft tables and latest snapshot. Shape: `{ changes: [{ kind: "new"|"modified", entity, slug, fields: [{ name, from, to }] }], revalidate: ["/", "/projects", "/projects/<slug>"] }`. This feeds the admin "Review changes" dialog. |
| POST | `/api/admin/publish` | In one DB transaction: build the snapshot, assign next `version` and a short `commit_hash`, store it, write `audit_log`. Then call the Vercel revalidate hook (`REVALIDATE_URL`, header `x-revalidate-secret`) with the paths. If revalidate fails, the snapshot stays published and the response includes `revalidated: false` so the admin can retry. |
| GET | `/api/admin/snapshots` | History list (version, commit_hash, summary, published_at). |
| POST | `/api/admin/snapshots/:version/rollback` | Make that snapshot the latest again (creates a new snapshot row, never deletes history). |

## 7. Validation and content rules
- Strip or reject HTML in all text fields. The frontend escapes output, but the API must not store markup.
- URLs (`live_url`, `repo_url`, `credential_url`) must be `https://` and valid.
- `slug`: lowercase `a-z0-9-`, unique, auto-generated from title, editable.
- Honesty rules carried over from the design: entries with `is_example = true` are shown with an EXAMPLE label on the site. Verification links appear only when a URL exists. `availability` is not shown unless the owner sets it. `results` is shown only when filled.
- Max lengths: title 120, description 600, tech 200, answer 800.

## 8. Security checklist
- Middleware order: `helmet` → `cors` (allowlist: `PUBLIC_SITE_ORIGIN`, `ADMIN_ORIGIN`, `credentials: true`) → body parsers with size limits → rate limiters → routes → error handler.
- CSRF token required on all state-changing admin requests.
- Parameterized queries only (Drizzle). No string-built SQL.
- Generic error messages to clients; details only in logs.
- Uploads: never trust the filename or MIME header; generate random filenames; serve from a separate path with `X-Content-Type-Options: nosniff`.
- Do not expose the admin routes on the public origin. Optionally restrict `/api/admin` by IP at Nginx.
- Hash IPs before storing them in `messages`.

## 9. Deployment (Linux VPS)
- Nginx reverse proxy to `localhost:4000`, HTTPS via certbot, gzip, `client_max_body_size 6m`.
- UFW: allow 22 (key-only SSH), 80, 443 only. Disable password SSH. Run the app as a non-root user.
- PM2 or systemd with auto-restart.
- MySQL: bind to `127.0.0.1` only (never open 3306 in UFW). Create a dedicated user with privileges only on this database. Nightly cron: `mysqldump --single-transaction` plus `uploads/` to a second location, keep 14 days, test a restore once.
- Health endpoint `GET /healthz` returns `{ ok: true, db: true }` (no secrets).

## 10. Frontend integration notes
- Next.js reads `GET /api/public/content` at build time and with on-demand revalidation (`/api/revalidate` route that checks `x-revalidate-secret`). If the API is unreachable at build, fail the build rather than publish empty content.
- Real routes replace hashes: `/projects`, `/projects/[slug]`, filter via `?category=web-app`.
- The admin app calls the API with `credentials: "include"` and sends `X-CSRF-Token`.

## 11. Acceptance checks
1. Login rejects a wrong password and a wrong code with the same shape of error; the 4th rapid attempt is throttled.
2. Editing a project does not change the public output until Publish is run.
3. Publish returns a diff that matches what the admin dialog showed.
4. Stopping the API does not break the public site.
5. An uploaded `.png` is stored as `.webp` and the original is not kept; an uploaded `.svg` or renamed `.html` is rejected.
6. Posting HTML in a title is rejected or stored as plain text and rendered escaped.
7. Rollback restores the previous published content and keeps history.

## 12. Suggested build order for Codex
1. Project setup, config, DB schema, migrations, seed (the sample data from the prototype).
2. Public read endpoints from a hand-made snapshot.
3. Auth (password + TOTP + sessions + CSRF + rate limits) with tests.
4. Admin CRUD for projects, then the other sections.
5. Media upload pipeline.
6. Publish preview + publish + revalidate + rollback.
7. Contact form + messages.
8. Deployment files (Nginx, systemd/PM2, backup script) and a README.

## 13. Visitor analytics and contact privacy

### Counting visitors (first-party, privacy-friendly)
- The Next.js site pages are cached on Vercel, so the API never sees a normal visit. Count visits with a small **client-side beacon**: on each route change the site sends `navigator.sendBeacon('/api/track', { path, referrer })`.
- Server side, never store the raw IP or a cookie. Build a **daily anonymous hash**: `sha256(ANALYTICS_SALT_SECRET + dailySalt + ip + userAgent)`, where `dailySalt` changes every day. Insert into `visitors_daily`; if the row is new, increment `uniques`. Always increment `views`. Delete `visitors_daily` rows older than 2 days.
- This gives *approximate* unique visitors per day. It cannot follow a person across days, by design.
- Ignore obvious bots (known crawler user agents), requests with `DNT: 1` if you choose to honor it, and anything from the admin origin. Rate limit the endpoint.
- Aggregate only: store page, referrer host, and day. Do not store full referrer URLs or query strings.
- The admin Overview reads `GET /api/admin/analytics` and shows today / 7 days / 14 days, a daily bar chart, top pages and referrers.
- Ready-made alternative: self-host Umami (open source, cookie-free) and link to it from the admin. Check which databases the current Umami version supports before choosing it alongside MySQL.

### Contact form and personal data
- The form collects name, email, optional phone/WhatsApp, and a message. These are personal data. Show a short privacy note under the form (what is collected, why, how long it is kept, how to ask for deletion) and require the consent checkbox.
- Retention: auto-delete messages after a fixed period (e.g. 12 months) with a daily job, and let the owner delete any message from the admin.
- Never log message bodies, emails or phone numbers in application logs.
- Add a spam control: honeypot, rate limit, and optionally a CAPTCHA service such as Cloudflare Turnstile.
- The admin shows a **WhatsApp** link (`https://wa.me/<digits>`) next to the email button when a phone number exists.
- This is a technical design note, not legal advice. Check Indonesia's personal data protection rules (UU PDP) for your obligations.

### Extra acceptance checks
8. A visit increments `views` and `uniques` once per day per visitor, and the same visitor on the next day is counted as new.
9. No cookie is set by `/api/track`, and no raw IP appears in the database or logs.
10. A message with an invalid phone number is rejected with a clear error; a message without a phone number is accepted.
11. Without the consent checkbox, the contact request is rejected.
