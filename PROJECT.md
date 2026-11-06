# HR Hiring Form — Technical Decisions

This is UrbanGabru’s HR onboarding app. HR builds a form, shares a public link, and a candidate submits personal details plus documents. The interesting part for an interview is not the form itself. It is the set of choices around security, data integrity, and how the public candidate flow stays separate from the staff dashboard.

Stack: React 19 + Vite on the frontend, Node.js + Express on the API, MySQL 8 (`hr_hiring_form`), AWS S3 for files, JWT for staff login.

## 1. Layered API, not one big route file

Every feature follows the same path:

`route` → `controller` → `service` → `repository` → MySQL

- Routes only declare the URL, middleware, and validation schema.
- Controllers read the request and send the response.
- Services hold the rules (who can do what, what a valid form looks like, when to roll back).
- Repositories run SQL and nothing else.

That split is what lets auth, validation, and SQL change independently. A new field does not require touching the HTTP layer.

Errors are a single `ApiError` type, caught by one error middleware. Async route handlers are wrapped so a rejected promise becomes a JSON error instead of an unhandled crash.

## 2. Two audiences, two trust levels

Staff and candidates are not the same kind of user.

| Audience | How they get in | What they can touch |
|----------|-----------------|---------------------|
| Admin / HR | Email + password, JWT | Forms, submissions, documents, and (admin only) users |
| Candidate | Public slug URL, no account | One form, their own draft, their own uploads |

The public routes never accept a staff token as a shortcut, and the staff routes never trust a slug. Role checks run twice: Express `requireRole` on the API, and a React `ProtectedRoute` so the Users page is not even rendered for HR. The UI check is convenience. The API check is the real gate.

Passwords are bcrypt hashes (cost 10). The database never stores the plain password. Login compares the hash, then issues a JWT (`sub`, `role`, `email`, `name`) with a short expiry. Every protected request must send `Authorization: Bearer …`. Expired and malformed tokens return 401 with different messages, so the client can send the user back to login.

## 3. The database is relational where it must be, flexible where the form is

Seven tables: `roles`, `users`, `forms`, `form_fields`, `submissions`, `uploaded_documents`, `audit_logs`.

Fixed relationships use foreign keys and UUIDs (`CHAR(36)`), so a submission cannot point at a form that does not exist, and deleting a form cascades to its fields and submissions.

The parts that change per form are JSON:

- `answers_json` stores `{ fieldId, valueText | valueJson }` so a new question does not need a new column.
- `options_json` stores dropdown choices and the allowed MIME types for a file field.
- `branding_json` and `settings_json` hold optional form config.

Answers are keyed by **field id**, not by label. Renaming "Aadhaar Number" does not orphan old responses.

When HR edits a form, fields are replaced inside a transaction: delete the old rows, insert the new ones, commit, or roll back the whole edit. Existing field ids are reused on insert, so previous `answers_json` still points at the same question. The MySQL pool (`mysql2`) hands out connections, and `transaction()` always `commit`s, `rollback`s, and `release`s the connection.

The UAN migration shows the same care. It inserts "UAN Number" after "Aadhaar Number" and shifts `sort_order`. It does not rewrite existing field ids, and it skips forms that already have the field, so it is safe to run twice.

Slugs are generated from the title plus a short UUID suffix. Two forms titled "Software Engineer" cannot collide on the public URL.

All queries use placeholders (`?`). Values are never concatenated into SQL.

## 4. Files never sit in MySQL, and the bucket stays private

Documents (PAN, Aadhaar, cancelled cheque, salary slips) go to S3. MySQL stores only metadata: original name, S3 key, MIME type, size, checksum, and which submission and field they belong to.

Upload rules, enforced on the server:

- Allow-list of MIME types (PDF, JPEG, PNG) and a max size.
- Filename stripped to a safe character set before it becomes part of the object key.
- Object key includes form id, submission id, and a UUID, so two candidates uploading `pan.pdf` do not overwrite each other.
- Duplicate check on submission + filename + size, so a double-click does not create a second object.
- Optional server-side encryption (AES-256 or KMS) from env, not hardcoded.

Staff do not get a permanent public URL. Download goes through an authenticated API route that returns a **presigned URL** with a short life. The browser can preview the file; the bucket itself stays private.

There are two upload paths: a presigned `PUT` straight to S3, and a server-side upload through Multer. Both write the same metadata row.

## 5. The public form survives a refresh

A candidate does not create an account. On first interaction the app creates a `draft` submission and stores that id in `localStorage` under the form slug. Refreshing the page continues the same draft instead of starting a blank one.

The form is rendered from the field list the API returns, grouped by `group_label` (Personal Details, Identity, Bank, references, emergency contact). It is not a hardcoded page of inputs. Adding UAN, or any custom field in the builder, shows up on the public form without a frontend release for that question.

`react-hook-form` tracks values and validates as the candidate types. Progress is derived from required fields, not stored as a guess.

## 6. Security is middleware, applied to every request

The Express app mounts these before any route:

| Piece | Why it is there |
|-------|-----------------|
| `helmet` | Security headers (nosniff, frame deny, CSP, HSTS in production) |
| CORS allow-list | Only the configured frontend origins, with credentials |
| `express-rate-limit` | Caps requests per IP so login and the public form cannot be hammered |
| Input sanitize | Strips null bytes and trims body and query values |
| Optional CSRF | Double-submit cookie, turned on with `CSRF_ENABLED` |
| Body size limit | JSON capped at 2mb so a huge payload cannot pin the process |
| `trust proxy` | Rate limits and audit IPs use the real client IP behind a proxy |

Production startup refuses to boot if `JWT_SECRET` is missing or still the default, or if the database password and CORS origin are empty. Fail closed, not fail open.

Staff actions are written to `audit_logs`: form create/update/status, user create, activate/deactivate, password change, password reset. Each row has the actor, action, entity, metadata, and IP. That is the trail an interviewer will ask for when the question is "how do you know who changed this?"

Zod schemas validate request bodies on the API (`createUser`, form payload, submission, upload). The same library validates the settings form in the browser. Invalid input is rejected before it reaches a service.

## 7. Frontend structure matches the API

- `AuthContext` holds the token and the user, and restores the session from `localStorage` on load.
- One Axios client attaches the bearer token on every request. Pages never hand-build headers.
- Routes split public (`/login`, `/form/:slug`) from protected (`/`, `/forms`, `/submissions`, `/documents`, `/settings`, `/users`).
- Form builder, field editor, and a small UI kit (input, file drop, progress) keep the pages thin.
- Default onboarding fields live in one list and are shared by the builder and the API, so a new hire form starts complete: name, email, phone, PAN, Aadhaar, UAN, document uploads, previous-employer references, emergency contact.

CSV export of responses is generated on the server (`csv-stringify`), not assembled in the browser, so the export matches what is stored.

API docs are generated with swagger-jsdoc and served at `/api/docs`. Request logging uses Pino, which stays structured and is safe to ship, unlike `console.log` of full request bodies.

## 8. Config stays out of the code

Database, JWT, CORS, S3, upload limits, and the first admin account come from environment variables. `.env.example` documents them. `.env` is not committed.

The first admin is not a SQL `INSERT` of a plain password. `scripts/create-admin.js` looks up the `admin` role, bcrypt-hashes the password, and inserts the user. Roles themselves are seeded (`admin`, `hr`, `candidate`) before that script can run.

## What to say if they ask "why not X?"

- **Why not store files in the database?** Documents are large and private. S3 plus a metadata row keeps MySQL small and lets access expire with the signed URL.
- **Why JSON for answers instead of one row per answer?** The form is user-defined. A JSON document keyed by stable field ids avoids a migration every time HR adds a question, and the field id is what keeps history correct.
- **Why replace all fields inside a transaction?** A half-saved form (new title, old questions) is worse than a failed save. Commit both or neither, and keep old field ids so past submissions still resolve.
- **Why JWT instead of server sessions?** The API is stateless. The role travels in the token, and expiry is enforced on every request without a session table.
- **Why check roles in React and in Express?** The router hides pages. The API is what stops a forged request. Both are required; only the API is sufficient.

## Short map of where this lives

| Concern | Where |
|---------|--------|
| Schema | `backend/db/migrations/001_init.sql` |
| Safe UAN backfill | `backend/db/migrations/002_add_uan_field.sql` |
| Roles | `backend/db/seeds/001_roles.sql` |
| Transactions and pool | `backend/src/config/db.js` |
| Auth and roles | `backend/src/middleware/auth.js`, `role.js` |
| Uploads and signed URLs | `backend/src/services/uploads.service.js` |
| Form save | `backend/src/services/forms.service.js` |
| Public candidate form | `frontend/src/pages/PublicForm.jsx` |
| Manual SQL checks | `sql/HR_hiring_form.sql` |
