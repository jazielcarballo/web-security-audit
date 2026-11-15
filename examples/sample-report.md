# Security Audit — Acme Store

> **Example report.** Acme Store is fictional and every value below is invented. This file shows the structure, tone, and level of detail the skill produces; a real report is written in the owner's language, with real file and line references, and is kept out of public repositories.

**Reviewed commit:** `a1b2c3d` · **Date:** 2026-11-02 · **Environment:** staging (`staging.acme-store.example`) · **Pass:** full (web app + API)

## 1. Owner summary

Acme Store is in reasonable shape for launch, but one confirmed issue must be fixed before paying customers arrive: any logged-in user can read another customer's orders by changing the order number in the address. Two other gaps are lower risk and already patched.

- 🔴 1 critical · 🟠 1 high · 🟡 1 medium · ⚪ 2 hardening notes
- Do these first:
  1. Add an ownership check to `/api/orders/{id}` (fixed in this review, pending your deploy).
  2. Make the session cookie `HttpOnly` and `Secure` (fixed in this review).
  3. Confirm the two Supabase settings listed in *Needs validation*.

## 2. Findings

| Severity | Title | Area | Observed result |
| --- | --- | --- | --- |
| 🔴 Critical | Any customer can read other customers' orders | API authorization | `GET /api/orders/1042` returned another account's order to a test user |
| 🟠 High | Session cookie readable by JavaScript | Sessions | `Set-Cookie` for `session` had no `HttpOnly` or `Secure` flag |
| 🟡 Medium | Stack traces exposed in production responses | Error handling | A malformed request returned a full Node.js stack trace |

### 🔴 Any customer can read other customers' orders — `verified`

- **What we found**: The order detail endpoint looks up the order by its number but never checks who owns it. A logged-in customer can request any order number and receive someone else's name, address, items, and total.
- **What could happen**: A customer changes the number in the browser and can read a competitor's or a neighbor's purchase history, including their delivery address.
- **Analogy**: It is like a hotel giving you a key that opens every room with the same number.
- **What we did or what you need to do**: Added an ownership check in `src/api/orders/[id].ts:24` so the query is scoped to the authenticated user. Deploy the branch and confirm the retest below passes.
- **Status**: `verified` — reproduced with two test accounts before the fix, and the same request returned `404` after it.

<details>
<summary>Technical detail</summary>

- `GET /api/orders/{id}` → `200 OK` with `{"userId":"user-b", ...}` when called as `user-a`.
- OWASP API Security Top 10: API1:2023 — Broken Object Level Authorization.
- Fix scopes the lookup: `where({ id, userId: session.user.id })`.

</details>

### 🟠 Session cookie readable by JavaScript — `fixed`

- **What we found**: The login response sets the `session` cookie without `HttpOnly` or `Secure`. Any script that runs on the page can read it and send it elsewhere.
- **What could happen**: If a single third-party script on the storefront is compromised, the attacker can copy active sessions and shop or change account details as those customers.
- **What we did or what you need to do**: Set `HttpOnly`, `Secure`, and `SameSite=Lax` in `src/lib/session.ts:18`. Logout and login were retested after the change.
- **Status**: `fixed` — patched and exercised, not yet re-proven with a stolen-cookie attempt.
- **Note**: Existing sessions stay valid until they expire; consider rotating the signing key after deploy.

### 🟡 Stack traces exposed in production responses — `fixed`

- **What we found**: Unhandled errors return the full Node.js stack trace, including file paths and dependency names.
- **What could happen**: An attacker learns the framework, versions, and internal file layout, which makes a targeted follow-up attack cheaper. This is information disclosure, not direct access.
- **What we did or what you need to do**: Added a production error handler in `src/server.ts:41` that logs the detail server-side and returns a generic message.
- **Status**: `fixed`.

## 3. Needs validation

No severity assigned; these depend on a dashboard setting or deployment fact that is not in the repository.

| Title | Reference | Missing fact | How to confirm safely |
| --- | --- | --- | --- |
| Row-level security on `orders` | `supabase/migrations/0004_orders.sql:12` | Whether RLS is enabled in the production project, not only in migrations | Supabase dashboard → Authentication → Policies; read-only check |
| Stripe webhook signature | `src/api/webhooks/stripe.ts:9` | Whether `STRIPE_WEBHOOK_SECRET` is set in production | Compare the environment variable list in the hosting panel; do not print the value |

## 4. Hardening notes and what already works

Hardening (real gaps, no reachable boundary violation):
- ⚪ Add `Strict-Transport-Security` with a one-year max-age once HTTPS is stable on all subdomains.
- ⚪ The password reset link stays valid for 7 days; shorten it to 1 hour.

What already works:
- Passwords are hashed with bcrypt at cost 12.
- Login and reset endpoints are rate-limited per IP and per account.
- The admin area requires a second factor and rejects non-admin sessions server-side.
- Dependency scan found no known critical advisories at the reviewed commit.

## 5. Manual action checklist

- [ ] Deploy the ownership-check fix and re-run the two-account retest.
- [ ] Rotate the session signing key after the cookie change reaches all users.
- [ ] Enable RLS on `orders` in the production Supabase project.
- [ ] Confirm the Stripe webhook secret is set and matches the dashboard.
- [ ] Add HSTS at the edge once every subdomain serves HTTPS.

## 6. What was and wasn't reviewed

- **Covered**: authentication, sessions, order and profile APIs, payment flow (test mode), file upload, headers, client-side storage, dependency and container scan.
- **Not applicable**: mobile app (none), GraphQL (none), WebSockets (none).
- **Not verified**: production WAF rules, email authentication (SPF/DKIM/DMARC), backup and restore procedure — manual steps above.
- This was a full pass on the reviewed commit. It does not prove the codebase is clean, and a later change can introduce a new issue. This review reduces risk; it does not replace a professional penetration test, especially once the store handles real payments at scale.

## 7. Verified versions

| Technology | Found | Should be | Checked where | Date |
| --- | --- | --- | --- | --- |
| Node.js | 20.11.0 | 20.18.x or later | `node -v` on staging host | 2026-11-02 |
| Next.js | 14.1.0 | 14.2.x | `package.json` + running build | 2026-11-02 |
| PostgreSQL | 15.4 | 15.8 or later | Supabase dashboard | 2026-11-02 |

## 8. Rejected candidates

- **CSRF on checkout**: tested with a cross-origin form; rejected because the session cookie is `SameSite=Lax` and the endpoint requires a custom header. Not a finding.

## Glossary

- **IDOR**: when a request uses an object number the user does not own and the server does not check ownership.
- **HttpOnly**: a cookie flag that hides the cookie from JavaScript, so a script cannot steal it.
- **RLS**: database rules that filter rows per user, enforced by the database instead of the application.

---

*Audit performed with Web & Mobile Security Audit — https://github.com/jazielcarballo/web-security-audit*
