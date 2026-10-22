---
name: web-security-audit
description: Audit the security of any web platform (any stack), fix what is safe to fix, and explain the risks in plain language to someone with no cybersecurity background. Use when the user asks for a security audit, security review, pentest-style review, or threat assessment of their web app.
disable-model-invocation: true
---

# Web Security Audit for Non-Specialists

This skill runs a full security review of a web platform. It is based on the OWASP Top 10, OWASP ASVS, and advanced pentesting vectors, and adapts them to the project's actual technologies. The person using it probably has no security background, so the explanation matters as much as the finding.

## Safety rules (non-negotiable)
1. Before starting, get explicit confirmation that the person owns the system or has written authorization to test it. Without that confirmation, stop.
2. Active testing (sending requests, XSS payloads, request bursts) runs only against local or staging environments. Never run active tests against production with real users or real money. Against production, only passive reads are allowed: headers, DNS, certificates, and publicly accessible files.
3. Payments are tested only in sandbox or test mode.
4. Fixes go on a new branch `security/audit-YYYY-MM-DD`, with one commit per finding.
5. Do not delete data, rotate keys, or change settings in external dashboards. Give the person those steps to do themselves.
6. If you find an exposed key or password, stop and report it immediately, before continuing with anything else.
7. Agree on a written scope before testing: which environments, URLs, and accounts are in and out. Stay inside it; if something out of scope looks vulnerable, report it without testing it.
8. Throttle every active test. Even staging can be shared with other people's work: cap concurrent requests, prefer a few targeted probes over floods, and never run fuzzing or request bursts that could slow the service down.
9. Attack only this person's system. Do not test third-party services (Stripe, Cloudflare, and similar provider APIs or dashboards), do not phish or social-engineer anyone, and do not bring findings from other companies into this audit.
10. Evidence hygiene: prove impact only with your own test accounts (create user A and user B). Never read, copy, or paste real users' data into the report. Redact secrets in writing (show only the type and last 4 characters, for example `sk_live_••••7f2a`), and keep the report out of public repositories if it contains sensitive findings.

## Phase 1: Understand the project
Inspect the repository (package.json, requirements, composer.json, config files, infrastructure folders, deployment files) and answer:
- **What the platform does**, in one sentence.
- **Technologies**: frontend, backend, database, authentication, file storage, payments, hosting/CDN, external services (maps, email, AI).
- **Who uses it**: anonymous visitors, registered users, paying customers, administrators.
- **What is worth protecting**: personal data, money, content, reputation, admin accounts.

If something can't be determined from the code, ask the person simple closed questions, for example: "Does your site charge cards? With which service?" Do not guess.

Show this summary plus the proposed scope — environments, URLs, accounts to be used, and the intensity limits you will respect — and ask for confirmation before continuing. No confirmation, no testing.

## Phase 2: Decide what applies
Go through the control catalog below and classify each control:
- **Applies**: reviewed in code or configuration.
- **Handled by the provider**: for example, password hashing in Supabase, Firebase, or Auth0, or encryption at rest in managed services. Mention it, but don't audit it.
- **Not applicable**: the technology isn't used. For example, no XML, no self-managed containers, or no LLM.
- **Manual review**: depends on external dashboards the code can't show.

Also load the matching stack module (see "Technology modules"). On managed platforms, those modules often contain the most serious risks.

## Phase 3: Review
For each applicable control, find concrete evidence: file and line, or the command and its output.

### Order of operations
1. **Inventory the attack surface.** List every route, endpoint, page, and background job from route files, API specs (OpenAPI/GraphQL), controllers and views, and the frontend's fetch/axios calls. Mark which ones read or modify data.
2. **Build an authorization matrix.** For each sensitive endpoint, consider all four callers: anonymous, user A, user B, admin. The server — not the UI — must deny the wrong role, and user A must never reach user B's records.
3. **Always test with two accounts of your own.** Create user A and user B and run the IDOR/BOLA and tenant-isolation checks against both, including the spots checklists miss: bulk/batch operations, exports and downloads, pagination and filter parameters (`?user_id=`, `?org=`), nested resources, and background jobs.
4. **Scan with tools, then prove by hand.** Run the toolchain below, but never report a scanner output you have not reproduced yourself with a captured request/response. Scanners produce false positives, and one false report costs you the owner's trust.
5. When the project already has tools available, use them; if it doesn't, propose installing them and ask for confirmation.

### Toolchain
- Secrets in code and git history: `gitleaks` or `trufflehog`
- Dependencies: `npm audit` or `osv-scanner` (JS), `pip-audit` (Python), `composer audit` (PHP), `cargo audit` (Rust)
- Runtime and base image: `php -v`, `node -v`, `python -V`, `java -version` executed in the running process, plus the Dockerfile `FROM` tag and the hosting panel version. Lockfile scanners do not see this layer.
- Static analysis: `semgrep` with the `p/security-audit` ruleset, plus the stack's own (Brakeman, Bandit, ESLint security plugins)
- Headers and transport: `curl -I` against staging; `testssl.sh` if certificates are in scope
- Dynamic scan, staging only: OWASP ZAP baseline scan and `nuclei` with default templates
- Supabase: `supabase db lint` (Security Advisors)

Rate each finding:
- 🔴 **Urgent**: someone could steal data or money, or take over accounts, with little effort. Fix today.
- 🟠 **Important**: exploitable, but requires more effort or special conditions. Fix this week.
- 🟡 **Improvement**: strengthens defenses or limits damage if something fails. Fix this month.
- ⚪ **Informational**: good practice, no immediate risk.

Don't inflate severity. A missing header is not urgent; an exposed payment key is. But an unauthenticated remote code execution with public exploitation counts as 🔴 even when it needs special conditions to trigger: the person cannot wait for a week while scanners try it. If the vulnerable version was exposed while the advisory was public, patching alone is not enough — open the incident-response track in Phase 4.

## Phase 4: Fix
- **Automatic fix** (no need to ask): escaping output, adding headers, parameterizing queries, updating dependencies to patch or minor versions, and adding files to `.gitignore`.
- **Fix with confirmation**: changes that alter visible behavior, such as CORS, caching, data-access policies, major version upgrades, or blocking routes. First explain what could stop working. **Upgrading the language runtime or base image belongs here**, even on a patch release: it can change behavior for the whole app and needs a restart.
- **Manual action**: anything that lives in external dashboards. Write it as numbered steps, for example: "Go to Stripe → Developers → API keys → Roll key".

**Verifying a runtime upgrade**: confirm the patched version in the running process (`php -v`, `node -v` inside the container or hosting panel), not only in the manifest or image tag, and restart the service. A patched binary that was never restarted is still the old one in memory.

**If a critical advisory was public while the system was exposed**, an update is not the end. Run an incident-response pass and tell the person plainly that the machine may have been compromised:
- Look for webshells and files modified during the exposure window, unknown admin or database users, new cron jobs or scheduled tasks, and outbound connections to unknown hosts.
- Read the logs around the window for unusual requests, error bursts, or requests to files that should not exist.
- Rotate every secret the process could read: `.env`, database credentials, API keys, and session or signing keys.
- Record the exposure dates and exactly what was checked, so the person has a trace if something surfaces later.

After each fix, verify that the app builds and existing tests pass. When reasonable, add a test proving the problem no longer occurs.

**Retest**: re-run the exact check that proved the vulnerability — same request, same payload, same account pair — and record each finding as `open`, `fixed`, or `verified`. A green build is not proof the security bug is gone; only the repeated proof is.

## Phase 5: Explain (report)
Generate `security/report.md` in the person's language, in plain words, with this structure:

### 1. Owner summary (10 lines max)
- How the platform is doing, in one honest sentence.
- How many findings per level (🔴🟠🟡⚪).
- The 3 most important things to do, in order.

### 2. Findings
Use the same format for each one:
- **What we found**: one or two sentences, no jargon.
- **What could happen**: a concrete example using the person's own business. For example: "A customer could change the ID in the address bar and edit a competitor's listing."
- **Analogy** (optional, only if it helps): "It's like your house key also opening your neighbor's door."
- **What we did or what you need to do**: fixed, pending your confirmation, or manual steps.
- **Status**: `verified` (reproduced, fixed, and reproduced again), `fixed` (patched, not yet re-proven), or `open`.
- **Technical detail**: file and line, OWASP category, and reference. Put it in a collapsible block or at the end, for whoever needs it.

### 3. Manual action checklist
Ordered by urgency, with `[ ]` checkboxes.

### 4. What was and wasn't reviewed
List the controls marked "not applicable", "handled by the provider", and "not verified". Close with an honest note: this review reduces risk, but it doesn't replace a professional penetration test, especially if the platform handles money or sensitive data at scale.

### 5. Verified versions
One line per technology: the runtime, framework, and base image version found, the version it should be on, and where it was checked (container, CI, hosting panel), with the date. This is the record that answers "were we exposed?" later.

### Glossary
Include only terms that appear in the report, each defined in one line. For example: "XSS: when someone gets your site to run malicious code in other visitors' browsers."

## Communication style
- Write in the person's language.
- Define every technical term the first time it appears.
- Prefer "someone could..." over attack names.
- Avoid both alarmist and overly reassuring language. Never say "your site is secure"; say what was reviewed and what was found.
- If the person asks "what is that?", explain it without repeating the technical paragraph.

---

# Control catalog

## A. Identity, authentication, and sessions
- MFA required on your own admin panels.
- **MFA on provider dashboards**: hosting, database, payments, DNS/CDN, GitHub, and the domain registrar. These are the business's real admin panels. This is a manual action that is always included.
- Modern password hashing (Argon2id, bcrypt, PBKDF2) if authentication is self-built.
- Session cookies with `HttpOnly`, `Secure`, and `SameSite=Lax` (Strict can break returns from payment gateways or OAuth). If the token lives in localStorage, XSS becomes more critical.
- JWTs with a fixed algorithm, short expiry, and real signature verification. Decoding alone is never enough.
- Rate limiting and CAPTCHA (for example, Cloudflare Turnstile) on login, signup, and password reset.
- Sessions are invalidated server-side on logout and on password change.
- Reset tokens are single-use and expire.
- Reset email links are built from a configured domain, never from the request's `Host` header.
- OAuth/SSO: exact `redirect_uri` match, PKCE, and `state` validation.
- User enumeration: login, signup, password reset, and invites answer identically whether the account exists or not (same message, same status code, roughly the same response size and timing).
- Password change and email change require the current password or a fresh confirmation link; the session ID is regenerated at login (session fixation), and sessions end after an inactivity timeout.
- Default or well-known credentials (`admin`/`admin`, sample passwords in seed files) are rejected; failed logins are logged with IP and user agent.

## B. Access control
- Least privilege per role.
- IDOR/BOLA: every read or write of a resource verifies server-side that the user owns it.
- Admin routes and endpoints are blocked at the API, not just hidden in the UI.
- Mass assignment: users cannot modify fields like `role`, `plan`, `is_admin`, `owner_id`, `price`, or `payment_status`.
- Excessive data exposure: the API returns only the fields needed.
- Multi-tenant isolation: nothing a tenant supplies (org/team/tenant IDs in body, headers, filters, invitations, exports, background jobs) can point the request at another tenant's data.

## C. Input and output
- Injection (SQL, NoSQL, OS commands): parameterized queries, never concatenation.
- XSS: context-aware escaping. Check `innerHTML`, `dangerouslySetInnerHTML`, `v-html`, `|safe`, `{!! !!}`, and unescaped templates. Check every language the site supports.
- CSRF: anti-CSRF tokens if the session uses cookies.
- File uploads: validate the real type (magic bytes), limit size, rename the file, store it outside the web server, disallow SVG and HTML, and strip location metadata.
- SSRF: if the server fetches user-supplied URLs, use an allowlist and block internal networks and `169.254.169.254`.
- Open redirects and links built from unencoded parameters.
- Path traversal / arbitrary file read: paths coming from user input are normalized and validated server-side (`../`, encoded `%2e%2e`, absolute paths, null bytes); files are served by ID or from a fixed base directory, never by concatenated path.
- XXE, insecure deserialization, SSTI, and prototype pollution: review only if the stack makes them possible.

## D. APIs
- No `*` in CORS on routes that carry credentials.
- GraphQL: introspection disabled in production and query depth limited. Check for a GraphQL endpoint enabled by default even if nobody uses it.
- WebSockets or realtime: validate `Origin` and authenticate in the handshake or with per-channel policies.
- Rate limiting on public endpoints.

## E. Headers and transport
- HTTPS enforced, TLS 1.2 or higher.
- HSTS, CSP, `frame-ancestors` or X-Frame-Options, X-Content-Type-Options, and Referrer-Policy.
- Hide the `Server` and `X-Powered-By` headers, and stack traces in production.

## F. Business logic
- Workflow bypass: for example, reaching "order confirmed" or activating a plan without paying.
- Race conditions: simultaneous requests that redeem a coupon twice or duplicate an effect. Mitigation: transactions, locks, and unique constraints.
- Tampered values: negative quantities, extreme decimals, unexpected types (an array where a string is expected), and prices sent by the client.
- Abuse of free trials, coupons, or referrals.

## G. Payments
- Card data never passes through your servers (use hosted checkout or the provider's elements).
- Webhooks verify signatures, are idempotent, and tolerate out-of-order events.
- Price or plan is selected from a server-side list; the client's amount is never accepted.
- Paid access is activated from the webhook, not from the success page.
- Cancellations, failed payments, and refunds are handled.

## H. Infrastructure and perimeter
- Secrets are out of the code and out of git history. Frontend keys are only those that are public by design.
- `.env`, `.git`, and backups are not web-accessible. Test passively with `/.env`, `/.git/config`, and `/backup.zip`.
- Source maps (`.js.map`) are not published in production.
- WAF and anti-DDoS, if the host offers them.
- The database is not reachable without controls (network restrictions, SSL, strong password).
- Self-managed servers or containers: minimal open ports, non-root user, updated system. Only if they exist.
- Cache and CDN: personalized responses carry `Cache-Control: private, no-store`. Check for cache poisoning via `X-Forwarded-Host` and cache deception via URL extension.
- DNS: no orphaned subdomains pointing to decommissioned services (subdomain takeover).
- Cost abuse ("denial of wallet"): restrict paid API keys (maps, AI, SMS) by domain and set quotas and billing alerts.
- Scraping: if the content is a business asset, apply rate limits and anti-bot rules.
- Object storage (S3, GCS, Azure Blob, Supabase/Firebase Storage): no public buckets or objects by default; bucket policies and ACLs reviewed; signed URLs expire quickly and cannot be reused for a different object.
- Email authentication: SPF, DKIM, and DMARC records published on the domain, so nobody can send mail as the business or phish its customers.

## I. Secure development and dependencies
- Dependency audit with the package manager. Recommend Dependabot or an equivalent.
- **Runtime and base image**: the language interpreter or server (PHP, Node, Python, Ruby, JRE, nginx, OpenSSL, glibc) is not a package the lockfile shows, so `npm audit` or `composer audit` will never flag its critical vulnerabilities. Get the exact version from the running process, compare it against the patched release of every open critical advisory, and check whether the branch is past end-of-life (no patches exist, so "up to date" is impossible).
- **Critical advisory sweep**: check the runtime and framework against advisories published since the last update, using the project's advisory page, GitHub Advisories, OSV, and CISA KEV for actively exploited issues. A patched dependency tree on top of a vulnerable runtime is still remotely exploitable.
- Static analysis (for example, Semgrep) and, if staging exists, a basic dynamic scan with OWASP ZAP.
- Lockfile committed.
- CI/CD: secrets live in the CI's secret store, never in workflow files or Dockerfiles; pull requests from forks cannot read secrets; nothing from `github.event.*` (PR titles, branch names) is interpolated into `run:` blocks; deploy tokens, PATs, and cloud roles carry the minimum scope; repositories that contain secrets stay private.

## J. Monitoring, backups, and compliance
- Logs of failed logins, permission changes, and errors, with no passwords or card data.
- Alerts on spikes in 401/403/429 errors or in billing.
- Automated backups and, above all, a restore that has been tested at least once.
- Privacy notice and compliance with the applicable personal data law (in Mexico, the LFPDPPP; in Europe, the GDPR). Recommend confirming the current version with a specialist.
- PCI-DSS: with hosted checkout, the burden is minimal. Explain this so it doesn't cause unnecessary alarm.

## K. AI and LLMs (only if the platform uses them)
- Treat model output as untrusted before executing tools.
- Require user confirmation for actions with side effects.
- In RAG, filter by permissions so the model only retrieves what the user is allowed to see.
- Model API keys stay server-side and have spending limits.

---

# Technology modules
Load only the modules for the technologies detected. If the stack isn't listed here, apply the same reasoning: what does the service expose by default, and what is configured in its dashboard?

### Supabase
- PostgREST exposes tables directly through the anon key, which is public. **RLS is the number one control**: it must be enabled on every table in `public`, with policies reviewed one by one (`using` and `with check`).
- Sensitive columns (plan, role, payment status) must not be editable by their owner. Use separate tables, column privileges, or triggers.
- `SECURITY DEFINER` functions with a fixed `search_path`; views with `security_invoker = true`.
- Storage: per-owner folder policies, size and MIME limits per bucket.
- Auth: email confirmation, Redirect URLs allowlist, CAPTCHA, and MFA for admins.
- The `service_role` key never goes in the frontend.
- Disable `pg_graphql` if unused; check Realtime authorization if used.
- Run the Security Advisors (`supabase db lint`).
- Test RLS with several clients (anonymous, user A, user B) against the API, not through the UI.

### Firebase
- Firestore, Realtime Database, and Storage: review security rules. Look for `allow read, write: if true` and rules that don't compare `request.auth.uid`.
- App Check enabled.
- Cloud Functions validate authentication.
- Restrict the browser API key by domain in Google Cloud.

### Cloudflare (Pages, Workers, DNS)
- Secrets via `wrangler secret`, never in `[vars]` in `wrangler.toml`.
- Headers in `_headers` or in the Worker.
- Cache Rules don't cache authenticated responses.
- SSL in Full (strict) mode.
- Rate Limiting Rules and Turnstile.
- Origin protected so nobody can bypass Cloudflare.
- Check for orphaned DNS records.

### Vercel / Netlify
- Environment variables: only public ones carry the `NEXT_PUBLIC_` or `VITE_` prefix.
- Protect deployment previews if they contain real data.
- Headers in the project configuration.
- Serverless functions validate authentication.

### Stripe
- Verify the webhook against the raw body (`constructEvent`, or `constructEventAsync` in Deno or Workers).
- Idempotency by `event.id`.
- `price_id` allowlist on the server.
- `client_reference_id` or `metadata` set server-side.
- Customer Portal sessions created server-side.
- Restricted keys with minimal permissions.
- Test with `stripe listen` and `stripe trigger`.

### Mercado Pago / PayPal / Conekta / others
- Validate notification authenticity (signature, or query the payment from the provider's API).
- Ensure idempotency.
- Take the amount from the server.
- Keep production credentials separate from test credentials.

### WordPress
- Core, plugins, and themes updated; remove inactive plugins.
- `xmlrpc.php` disabled if unused.
- User enumeration via `/?author=1` and `/wp-json/wp/v2/users`.
- Table prefix, `wp-config.php` permissions, and `DISALLOW_FILE_EDIT`.
- Run WPScan against staging.

### Node.js / Express / Next.js
- `helmet` or equivalent headers.
- Input validation with a schema (zod, joi).
- Avoid `eval` and `child_process` with input data.
- Guard against prototype pollution in merges.
- Next.js: Server Actions and API routes validate the session and resource ownership.

### PHP / Laravel
- **Runtime version**: `php -v` and `php -r 'echo PHP_VERSION;'` inside the running process (container, FPM pool, hosting panel), not only the image tag or the composer constraint. Cross-check `FROM php:x.y-*` in the Dockerfile, `require.php` and `config.platform` in `composer.json`, the CI workflows, `.platform.app.yaml`, and Forge/Envoyer or cPanel/Plesk settings. These drift: an updated image can still be running an old container.
- **Advisories**: compare the exact version against the patched releases of open critical advisories and against the support timeline. Branches past end-of-life (8.1 and older) receive no patches. `expose_php=Off` reveals less, but hiding a version is not fixing it.
- **Upgrade mechanics**: after updating, restart PHP-FPM or the web server and re-run `php -v`; the patched binary does nothing until the service restarts. If the version was exposed while an advisory was public, follow the incident-response pass in Phase 4.
- `APP_DEBUG=false` in production.
- `display_errors=Off` and `log_errors=On`; delete stray `phpinfo()` files and default sample `index.php` (FrankenPHP images before 1.12.5 shipped one that executes `phpinfo()`).
- `$fillable` or `$guarded` on models (mass assignment).
- `{!! !!}` only with trusted content.
- Policies and Gates in controllers.

### Python / Django / Flask
- `DEBUG=False` in production.
- `ALLOWED_HOSTS` defined.
- Review `|safe` and `mark_safe`.
- Never `pickle` external data.
- `SECRET_KEY` out of the repository.

### Ruby on Rails
- Authorization on every controller action (Pundit, CanCanCan), not only on the happy paths.
- Strong parameters; watch for models that still accept mass assignment.
- `config.force_ssl = true`, `protect_from_forgery with: :exception`, encrypted or signed session cookies.

### Java / Spring Boot
- Actuator endpoints disabled or authenticated in production; `/actuator/env`, `/heapdump`, and `/trace` must never answer the internet.
- No SpEL or template expression built from user input; no deserialization of untrusted data.

### .NET
- Authorization on every endpoint (a missing `[Authorize]` is the classic hole), with policy-based checks for admin actions.
- Mass binding limited with `[Bind(Include = ...)]`; machine keys not shared or predictable; debug off and custom errors enabled in production.

### Databases reachable from the internet
- MongoDB, Elasticsearch, Redis, Memcached, and open MySQL/Postgres without auth get found and ransomed within hours. Confirm from the outside that only the application's network can connect; managed databases must reject public connections.

### Nginx / Apache
- Directory listing off; dotfiles and backup files denied; no alias or path-traversal misconfiguration; server tokens minimal; the static config must not serve `.git` or `.env`.

### Angular
- `bypassSecurityTrustHtml` / `bypassSecurityTrustUrl` treated as dangerous; `innerHTML` bindings reviewed; framework version current (old versions had sandbox escapes).

### Static sites / plain HTML
- Review third-party scripts (SRI integrity and origin).
- Forms posting to external services: spam and key exposure.
- API keys in public JS.
