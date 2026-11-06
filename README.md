# Neario ReviewFlow backend: logins, roles, agency isolation

Next.js (route handlers) + any Postgres. Plain JavaScript, no ORM, one runtime dependency besides Next (`pg`).

## Who can see what
| Login | Sees | Can do |
|---|---|---|
| Neario super admin | everything | create agencies, create direct clients |
| Agency user | only businesses where `agency_id` = their agency | add clients (always under their own agency), open client data (logged) |
| Client owner | only their own business | see their data and who viewed it |

One function, `businessScope()` in `lib/service.js`, decides visibility. Every query that touches client data goes through it.
A business you cannot see returns **404**, the same as one that does not exist.
Every time an agency or Neario opens a client's feedback, a row is written to `view_log`. Owners can read the rows about their own business.

## Deploy on Vercel
1. Create a Postgres database (Neon via the Vercel Marketplace works) and copy its connection string.
2. Vercel project env vars: `DATABASE_URL`, `SESSION_SECRET` (see `.env.example`).
3. Locally, with those two set: `npm install && npm run migrate && SUPER_ADMIN_EMAIL=you@example.com SUPER_ADMIN_PASSWORD='12+ characters' npm run seed`.
4. `vercel` to deploy.

## Endpoints (JSON; sign-in is a cookie)
- `POST /api/auth/login` `{email,password}` · `POST /api/auth/logout` · `GET /api/me`
- `POST /api/invites/accept` `{token,password}` (also signs the person in) · `POST /api/invites/resend` `{userId}`
- `GET|POST /api/agencies` (super admin) `{name,ownerEmail,ownerName}`
- `GET|POST /api/businesses` `{name,type,city,plan,ownerEmail,ownerName,ownerPermission:true}` (agency users and super admin; super admin may add `agencyId`, or omit for a direct client)
- `GET /api/businesses/:id` · `GET /api/businesses/:id/feedback?reason=...` (logged for agency/admin)
- `GET /api/view-log`
- `POST /api/feedback/:businessId` `{rating,text}`: public, for the customer QR page

Create calls return `inviteToken`. **You must email it** to the invited person as a link such as
`https://your-site/accept?token=...` (7 days, single use, stored only as a hash). There is no email sending in this code yet.

## Tests
`npm test` runs 25 tests (isolation between agencies, roles, invites, lockout, sessions, CSRF, error hiding).
They need an empty throwaway Postgres: `PSQL_ARGS="-h localhost -U postgres mydb" npm test` (uses the `psql` command, and wipes the `public` schema).

Verified here against PostgreSQL 16. **Not run here:** `pg` driver in `lib/db.js`, `next build`, and the migrate/seed scripts. The network blocked npm, so run `npm install && npm run build` once and tell me about any error.

## Not built yet (needed before real customers)
- Email sending for invites and password reset (no "forgot password" yet).
- Rate limiting on `/api/auth/login` (per IP) and `/api/feedback/*`. Per-account lockout exists. Use Vercel Firewall rate limits.
- Connecting the HTML prototypes to these endpoints (they still use example data).
- Payments, Google Business Profile, WhatsApp, AI replies.
- Row-level security in Postgres as a second safety net.
