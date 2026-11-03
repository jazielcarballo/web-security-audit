---
name: web-security-audit
description: Audit the security of any web platform, API, or mobile app and its backend (any stack), fix what is safe to fix, and explain the risks in plain language to someone with no cybersecurity background. Use when the user asks for a security audit, security review, pentest-style review, or threat assessment of their web app, API, or mobile app.
disable-model-invocation: true
---

# Web and Mobile Security Audit for Non-Specialists

This skill runs a full security review of a web platform — website, API, mobile app, or all three and their backend. It is based on the OWASP Top 10, OWASP API Security Top 10, OWASP MASVS/MASTG, OWASP ASVS, and advanced pentesting vectors, and adapts them to the project's actual technologies. The person using it probably has no security background, so the explanation matters as much as the finding.

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
11. Mobile testing only on builds you own or are authorized to test, with test accounts and a staging backend. Never install, repackage, or instrument a third-party app, and never test a store-distributed production build unless it is explicitly in scope. Do not bypass store protections, payment, or DRM.
12. Do not run disruptive protocol tests (request smuggling, cache poisoning, race-condition bursts, password or OTP brute force) against production or shared infrastructure. These run only on staging, with the written scope, a small request budget, and immediate stop if the service degrades. A WAF blocking a probe is not proof the vulnerability is absent.

## Phase 1: Understand the project
Inspect the repository (package.json, requirements, composer.json, config files, infrastructure folders, deployment files) and answer:
- **What the platform does**, in one sentence.
- **Technologies**: frontend, backend, database, authentication, file storage, payments, hosting/CDN, external services (maps, email, AI).
- **Interfaces**: website, REST/GraphQL/gRPC API, WebSocket/realtime, webhooks, admin panel, CLI, background jobs.
- **Mobile app, if any**: Android, iOS, or hybrid (React Native, Flutter, Capacitor); distribution channel (store, internal, MDM); which APIs it calls; deep links and Universal/App Links; local storage; third-party SDKs; push notifications; app attestation (Play Integrity, DeviceCheck/App Attest).
- **Who uses it**: anonymous visitors, registered users, paying customers, mobile users, administrators, support staff, API integrators.
- **What is worth protecting**: personal data, money, content, reputation, admin accounts, and — for mobile — the signing keys and any data cached on the device.

If something can't be determined from the code, ask the person simple closed questions, for example: "Does your site charge cards? With which service?" Do not guess.

Show this summary plus the proposed scope — environments, URLs, accounts to be used, and the intensity limits you will respect — and ask for confirmation before continuing. No confirmation, no testing.

## Phase 2: Decide what applies
Go through the control catalog below and classify each control:
- **Applies**: reviewed in code or configuration.
- **Handled by the provider**: for example, password hashing in Supabase, Firebase, or Auth0, or encryption at rest in managed services. Mention it, but don't audit it.
- **Not applicable**: the technology isn't used. For example, no XML, no self-managed containers, or no LLM.
- **Manual review**: depends on external dashboards the code can't show.

Also load the matching stack module (see "Technology modules"), including the mobile modules when an app exists and the API modules when the backend is not plain HTML. On managed platforms, those modules often contain the most serious risks.

## Phase 3: Review
For each applicable control, find concrete evidence: file and line, or the command and its output.

### Order of operations
1. **Inventory the attack surface.** List every route, endpoint, page, and background job from route files, API specs (OpenAPI/GraphQL/gRPC), controllers and views, the frontend's fetch/axios calls, and the mobile app's endpoints. Include GraphQL operations, gRPC services, WebSocket channels, webhooks, admin/debug routes, old API versions, and file upload/download paths. Mark which ones read or modify data.
2. **Build an authorization matrix.** For each sensitive endpoint, consider all five callers: anonymous, user A, user B, another tenant, and admin. The server — not the UI, the mobile client, or the API gateway alone — must deny the wrong role, and user A must never reach user B's records. Test every HTTP verb, not only the one the UI uses.
3. **Always test with two accounts of your own.** Create user A and user B and run the IDOR/BOLA, BFLA, and tenant-isolation checks against both, including the spots checklists miss: bulk/batch operations, exports and downloads, pagination and filter parameters (`?user_id=`, `?org=`), cursor/Relay IDs, nested resources, GraphQL mutations, gRPC calls, mobile deep links, and background jobs.
4. **Test the client, not only the server.** Review the web bundle and mobile app for secrets, hidden endpoints, feature flags, source maps, debug builds, and client-side authorization that the server does not repeat.
5. **Check the parser boundaries.** If a proxy, CDN, WAF, or API gateway sits in front of the app, verify that it and the backend parse requests the same way (headers, duplicated parameters, content types, body size). Disagreements are where request smuggling and parameter pollution live.
6. **Scan with tools, then prove by hand.** Run the toolchain below, but never report a scanner output you have not reproduced yourself with a captured request/response. Scanners produce false positives, and one false report costs you the owner's trust.
7. When the project already has tools available, use them; if it doesn't, propose installing them and ask for confirmation.

### Toolchain
- Secrets in code and git history: `gitleaks` or `trufflehog`
- Dependencies: `npm audit` or `osv-scanner` (JS), `pip-audit` (Python), `composer audit` (PHP), `cargo audit` (Rust)
- Runtime and base image: `php -v`, `node -v`, `python -V`, `java -version` executed in the running process, plus the Dockerfile `FROM` tag and the hosting panel version. Lockfile scanners do not see this layer.
- Static analysis: `semgrep` with the `p/security-audit` ruleset, plus the stack's own (Brakeman, Bandit, ESLint security plugins)
- Headers and transport: `curl -I` against staging; `testssl.sh` if certificates are in scope
- Dynamic scan, staging only: OWASP ZAP baseline scan and `nuclei` with default templates
- API discovery and testing, staging only: `arjun` and `paramspider` for hidden parameters, `jwt_tool` for token weaknesses, `grpcurl` for gRPC reflection, `websocat` for WebSocket origin checks, `graphql-cop`/`clairvoyance` for GraphQL
- Race conditions, staging only: Burp Repeater's single-packet attack or Turbo Intruder, with a small fixed burst and a stop condition
- Mobile: MobSF for a first static pass, `apktool`/`jadx` for Android, `plutil`/`codesign`/class-dump for iOS, `gitleaks` over the decompiled tree, and Frida/objection/drozer only on a rooted or jailbroken test device you control
- Cloud and containers: `trivy` for images and IaC, `checkov` for Terraform/Kubernetes, and `prowler`/`scoutsuite` if cloud credentials are in scope and the owner authorizes read-only access
- Supply chain: `osv-scanner`, `syft` for an SBOM, and `cosign` to verify signatures where the project signs artifacts
- Supabase: `supabase db lint` (Security Advisors)

Do not use WAF-bypass, credential-stuffing, or request-flood tooling in a defensive audit. Verify that the WAF exists and that the app does not depend on it for authorization, then stop.

Rate each finding:
- 🔴 **Urgent**: someone could steal data or money, or take over accounts, with little effort. Fix today.
- 🟠 **Important**: exploitable, but requires more effort or special conditions. Fix this week.
- 🟡 **Improvement**: strengthens defenses or limits damage if something fails. Fix this month.
- ⚪ **Informational**: good practice, no immediate risk.

Don't inflate severity. A missing header is not urgent; an exposed payment key is. But an unauthenticated remote code execution with public exploitation counts as 🔴 even when it needs special conditions to trigger: the person cannot wait for a week while scanners try it. If the vulnerable version was exposed while the advisory was public, patching alone is not enough — open the incident-response track in Phase 4. The same urgency applies to request smuggling or cache poisoning on a shared cache, a production secret embedded in a mobile binary, an exported Android component that exposes user data, and an OAuth redirect that steals authorization codes.

## Phase 4: Fix
- **Automatic fix** (no need to ask): escaping output, adding headers, parameterizing queries, updating dependencies to patch or minor versions, and adding files to `.gitignore`.
- **Fix with confirmation**: changes that alter visible behavior, such as CORS, caching, data-access policies, major version upgrades, or blocking routes. First explain what could stop working. **Upgrading the language runtime or base image belongs here**, even on a patch release: it can change behavior for the whole app and needs a restart.
- **Manual action**: anything that lives in external dashboards. Write it as numbered steps, for example: "Go to Stripe → Developers → API keys → Roll key".
- **Mobile fixes**: safe to do automatically — remove secrets and debug logs from the client, move tokens to Keychain/Keystore, disable cleartext traffic, restrict WebView file access and JS bridges, and set `android:allowBackup="false"` / `android:debuggable="false"`. With confirmation — certificate pinning, app attestation, root/jailbreak detection, and obfuscation, because they can break updates or old devices. Server-side authorization is always the real fix; client-side checks are only defense in depth.
- **Protocol fixes**: reject ambiguous `Content-Length`/`Transfer-Encoding`, duplicate parameters, and unexpected content types; make the proxy, gateway, and backend use the same parser; cap header, body, and query sizes. These can change behavior, so apply them with confirmation and a staging test.

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
One line per technology: the runtime, framework, and base image version found, the version it should be on, and where it was checked (container, CI, hosting panel), with the date. For mobile, add the app version/build, the target and minimum OS versions, the main SDKs, and whether the release is signed and obfuscated. This is the record that answers "were we exposed?" later.

If the platform has a mobile app, add a short "Mobile app" subsection to the findings with its own plain-language items: data stored on the phone, exported screens and deep links, WebView, token storage, pinning/attestation, and whether the server repeats every client-side check.

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
- Session cookies with `HttpOnly`, `Secure`, and `SameSite=Lax` (Strict can break returns from payment gateways or OAuth). Prefer the `__Host-` prefix. If the token lives in localStorage, XSS becomes more critical.
- **JWTs**: reject `alg: none` and any algorithm change; pin the expected algorithm; verify the signature with the correct key type; validate `exp`, `nbf`, `aud`, `iss`, and `jti`; reject unknown `crit` headers; never load keys from `jku`, `x5u`, or `jwk` supplied by the token; resolve `kid` only from a fixed server-side map, never as a file path or SQL value; short-lived access tokens with refresh-token rotation and reuse detection; a server-side deny-list by `jti`; tokens never in URLs, logs, or localStorage.
- **OAuth/SSO**: exact `redirect_uri` match (no wildcard, prefix, or path traversal), single-use `state` bound to the session, `nonce` for OIDC, PKCE required, no implicit or password grants, strict `aud`/`azp`/`iss` checks, ID tokens never accepted as access tokens, refresh-token rotation, device-code polling limits, issuer and JWKS pinned over TLS, no client secrets in mobile or SPA code, verified Android App Links and iOS Universal Links, account-linking CSRF protection, no token leakage through `Referer`, and scope checks on every grant.
- **OTP, magic links, and invites**: single use, short expiry, bound to the user and purpose, rate-limited, not returned in the API response, not placed in a URL that leaks through `Referer`, and compared in constant time.
- **MFA bypass**: test OTP replay, backup codes, "remember this device" tokens, and response manipulation. Enforce MFA server-side, never only in the UI.
- Rate limiting and CAPTCHA (for example, Cloudflare Turnstile) on login, signup, and password reset.
- Sessions are invalidated server-side on logout and on password change.
- Reset tokens are single-use and expire.
- Reset email links are built from a configured domain, never from the request's `Host` header.
- User enumeration: login, signup, password reset, and invites answer identically whether the account exists or not (same message, same status code, roughly the same response size and timing).
- Password change and email change require the current password or a fresh confirmation link; the session ID is regenerated at login (session fixation), and sessions end after an inactivity timeout.
- Default or well-known credentials (`admin`/`admin`, sample passwords in seed files) are rejected; failed logins are logged with IP and user agent.
- **Mobile tokens**: store them in Keychain (iOS) or Keystore/EncryptedSharedPreferences (Android) with device-only accessibility, never in `UserDefaults`, `SharedPreferences`, `AsyncStorage`, or a plist. Sensitive keys should require biometric/device authentication. Logout must clear them and revoke them server-side.

## B. Access control
- Least privilege per role.
- **BOLA/IDOR on every verb and representation**: GET, PUT, PATCH, DELETE, nested resources, bulk/batch operations, exports and downloads, pagination cursors, filter and sort parameters (`?user_id=`, `?org=`), GraphQL Relay global IDs, gRPC IDs, signed URLs, and cloud object URLs. User A must never reach user B's or another tenant's record.
- **BFLA**: admin and internal endpoints are blocked at the API for every role and every HTTP method, including method-override headers (`X-HTTP-Method-Override`) and content-type switching.
- **Object property level authorization**: responses return only the fields the caller may see; no internal IDs, password hashes, tokens, counters, or other users' PII in the payload.
- Mass assignment: an explicit allowlist per endpoint; users cannot modify fields like `role`, `plan`, `is_admin`, `owner_id`, `price`, `payment_status`, `credit_balance`, or `permissions`.
- Excessive data exposure: the API returns only the fields needed.
- Multi-tenant isolation: nothing a tenant supplies (org/team/tenant IDs in body, headers, filters, cursors, invitations, exports, background jobs) can point the request at another tenant's data.
- **Indirect privilege**: sharing, collaboration, invites, API keys, and background jobs must not grant more access than the person who created them has.

## C. Input and output
- Injection (SQL, NoSQL, OS commands, LDAP, XPath): parameterized queries, never concatenation. Watch NoSQL operators (`$ne`, `$gt`, `$where`) and raw ORM fragments.
- **XSS**: context-aware escaping for HTML, attribute, JavaScript, URL, and CSS contexts. Check `innerHTML`, `dangerouslySetInnerHTML`, `v-html`, `{@html}`, `|safe`, `mark_safe`, `{!! !!}`, and unescaped templates. Test stored, reflected, DOM-based, blind, mutation, and polyglot XSS, every supported language, LLM-generated content, and mobile WebViews. Add CSP with nonces or hashes and consider Trusted Types.
- CSRF: anti-CSRF tokens if the session uses cookies; `SameSite` is defense in depth, not a replacement for tokens on state-changing requests.
- **File uploads**: validate the real type (magic bytes), limit size, rename the file, store it outside the web server, disallow SVG and HTML, strip location metadata, and re-encode images. Test polyglots, ZIP bombs, filename path traversal and command injection, `phar://` on PHP, and uploads that later pass through a CDN or image pipeline.
- **SSRF**: if the server fetches user-supplied URLs, use an allowlist and block internal networks, link-local addresses, and every cloud metadata endpoint (AWS `169.254.169.254` with IMDSv2, GCP `metadata.google.internal` with the `Metadata-Flavor` header, Azure with `Metadata: true`, and equivalents). Validate after every redirect and after DNS resolution (rebinding), and cover webhooks, import/export, avatar URLs, PDF generators, and screenshot services.
- Open redirects: allowlist destinations, prefer relative paths, and never redirect to a user-supplied absolute URL. Test `//evil`, `@`, backslashes, encoded variants, subdomain tricks, and `javascript:`.
- Path traversal / arbitrary file read: paths coming from user input are normalized and validated server-side (`../`, encoded `%2e2e`, absolute paths, null bytes); files are served by ID or from a fixed base directory, never by concatenated path.
- **XXE**: disable DTDs and external entities in every XML, SAML, SVG, DOCX, and XLSX parser; test content-type switching, parameter entities, error-based, and out-of-band variants.
- **Insecure deserialization**: `ObjectInputStream`, XStream, Jackson default typing, .NET `BinaryFormatter` and `TypeNameHandling`, PHP `unserialize`/`phar://`, Python `pickle`/`yaml.load`, Node `node-serialize`, Ruby `Marshal.load`/`YAML.load`. Never deserialize untrusted data; use signed, allowlisted formats.
- **SSTI**: user input must never reach a template engine as template source (`render_template_string`, Twig, Freemarker, Velocity, Pebble, Thymeleaf, Smarty, Mako, ERB, Handlebars, EJS). Use logic-less templates or an allowlist of variables.
- **Prototype pollution**: recursive merge of user JSON, `__proto__`, `constructor.prototype`, query parsers, and client-side sinks. Use null-prototype objects, schema validation, and avoid recursive merge.
- **HTTP parameter pollution and cloaking**: duplicate parameters, array notation, encoding differences, and gateway-versus-backend precedence can make the WAF check one value while the app uses another. Test authentication and authorization with duplicate keys.
- **Request smuggling/desync**: `Content-Length`/`Transfer-Encoding` disagreements, HTTP/2 downgrade (`H2.CL`, `H2.TE`), and client-side desync. Staging only, low volume. Fix by using one parser end to end, rejecting ambiguous requests, and speaking HTTP/2 to the backend.
- **CSV/formula injection**: exported cells starting with `=`, `+`, `-`, or `@` must be escaped so they cannot run in Excel or Sheets.
- **ReDoS and algorithmic complexity**: user-supplied regex, deeply nested JSON/XML, huge pagination, and zip bombs. Set input size, depth, and time limits.
- **Dangerous execution sinks**: `eval`, `Function`, `child_process`/`exec`, `system`/`popen`, `Runtime.exec`, dynamic `include`/`require`, and template compilation from user input. Review only if the stack makes them reachable.

## D. APIs
- **CORS**: no `*` on routes that carry credentials; exact origin allowlist; reject `null`; do not treat `Origin` as authentication; set a short preflight cache.
- **REST**: test verb tampering, `X-HTTP-Method-Override`, content-type switching (JSON, form, XML), mass assignment, excessive data exposure, and inconsistent authorization between versions.
- **GraphQL**: introspection disabled in production (but never rely on that alone), query depth/complexity/alias limits, batching and brute-force amplification, Relay IDOR, nested object authorization, mutation authorization, subscription authorization, CSWSH, field-suggestion schema leakage, file upload, GET versus POST, and CSRF.
- **gRPC**: reflection disabled in production, per-method authentication and authorization, metadata injection, BOLA, deadlines and message-size limits, TLS, and protobuf fuzzing.
- **WebSockets/realtime**: validate `Origin`, authenticate in the handshake and per message or channel, authorize subscriptions, test for cross-site WebSocket hijacking (CSWSH), message injection, maximum message size, rate limits, and token expiry on long-lived connections.
- **API inventory**: old versions (`/api/v1` still alive after `/api/v2`), internal and debug endpoints (`/swagger`, `/openapi.json`, `/graphiql`, `/actuator`, `/_profiler`, `/debug`), exposed specs, source maps, mobile-only endpoints, and deprecated routes still served with weaker controls.
- **Rate limiting**: per user, IP, API key, and tenant; on login, OTP, password reset, search, export, webhook registration, and expensive operations; return `429` with `Retry-After`.
- **Webhooks**: verify signatures over the raw body, make handlers idempotent, protect against replay, filter events, rotate secrets, and validate outbound URLs against SSRF.
- **API keys**: never in client bundles or mobile binaries; scope them, rotate them, restrict by domain or IP, and keep separate keys per environment.
- **Pagination and cursors**: cap page size, do not leak total counts to unauthorized callers, sign or validate cursors, and never trust client-provided sort or filter fields.
- **Transport and protocol**: HTTP/2 and HTTP/3 implementations checked against rapid-reset and smuggling advisories, header and body size caps, and a proxy/backend parser that agrees.

## E. Headers and transport
- HTTPS enforced, TLS 1.2 or higher (prefer 1.3).
- HSTS (with `includeSubDomains` and preload when ready), CSP with `frame-ancestors`, X-Content-Type-Options, and Referrer-Policy.
- **CSP**: nonces or hashes instead of `unsafe-inline`/`unsafe-eval`; consider Trusted Types; `object-src 'none'`; `base-uri 'self'`. A weak CSP is not a fix for XSS.
- **Isolation headers**: `Cross-Origin-Opener-Policy`, `Cross-Origin-Embedder-Policy`, and `Cross-Origin-Resource-Policy` where compatible; `Permissions-Policy` to disable unused browser features.
- **Caching**: personalized responses carry `Cache-Control: private, no-store`; sensitive endpoints send `Vary` correctly so a shared cache cannot mix users.
- Hide the `Server` and `X-Powered-By` headers, and stack traces in production.
- **HTTP/2 and HTTP/3**: check the server against rapid-reset and request-smuggling advisories, cap header and body sizes, and keep the edge and backend parsers aligned.
- Certificates: monitor expiry, enable CAA, and consider certificate transparency monitoring.

## F. Business logic
- **State machine**: skip a required step, replay a one-time action, downgrade a state, or call a hidden endpoint directly. Example flows: verify email → set password → enable 2FA → dashboard; cart → address → shipping → payment → confirm.
- **Price, quantity, and currency**: negative, zero, or fractional quantities; hidden price or discount fields; currency confusion; coupon stacking, reuse, expired coupons, and coupons applied to the wrong product; cart SKU tampering after the price is calculated.
- **Refunds, chargebacks, and payouts**: refund more than was paid, refund to a different method, return less than claimed, payout-account races, and duplicate credits.
- **Trials, referrals, and quotas**: trial abuse through email aliases, disposable numbers, or device-ID rotation; referral loops; tier downgrade that keeps premium features; mid-cycle quota reset; add-on stacking.
- **Time-based logic**: promotion windows, `X-Forwarded-For`/`X-Real-IP`/date headers, token expiry, and scheduled jobs that trust client time.
- **Anti-automation**: CAPTCHA, rate limits, device fingerprint, and velocity checks; never rely on CAPTCHA alone.
- **Tampered types**: an array where a string is expected, an object where a number is expected, extra fields, and unexpected nulls.
- For concurrency bugs, see section M.

## G. Payments
- Card data never passes through your servers (use hosted checkout or the provider's elements).
- Webhooks verify signatures over the raw body, are idempotent, tolerate out-of-order events, and reject replays.
- Price or plan is selected from a server-side list; the client's amount is never accepted.
- Paid access is activated from the webhook, not from the success page.
- Cancellations, failed payments, refunds, chargebacks, and payouts are handled.
- 3-D Secure/SCA is implemented where required, and the return URL is validated.
- Production and test credentials are separate; restricted API keys are used; no secret key reaches the browser or the mobile app.

## H. Infrastructure and perimeter
- Secrets are out of the code and out of git history. Frontend keys are only those that are public by design.
- `.env`, `.git`, `.svn`, `.DS_Store`, `WEB-INF`, `phpinfo()`, and backups are not web-accessible. Test passively with `/.env`, `/.git/config`, `/backup.zip`, and `/backup.sql`.
- Source maps (`.js.map`) are not published in production; stack traces and framework debug pages are disabled.
- WAF and anti-DDoS, if the host offers them. Treat them as mitigation, never as authorization.
- The database is not reachable without controls (network restrictions, SSL, strong password).
- Self-managed servers or containers: minimal open ports, non-root user, updated system, no privileged mode, read-only filesystem where possible, no Docker socket mounted, and seccomp/AppArmor enabled.
- **Cloud**: least-privilege IAM, no wildcard policies, no long-lived root keys, MFA on the root account, no public buckets, snapshots, or disks, metadata service hardened (IMDSv2 on AWS), audit logs enabled (CloudTrail or equivalent), and secrets in a managed secret store.
- **Kubernetes**: RBAC least privilege, network policies, no automounted service-account tokens where not needed, admission control, resource limits, and no secrets in plain manifests.
- **Exposed services**: Kibana, Grafana, Prometheus, Jenkins, ArgoCD, etcd, Consul, Vault, Redis, MongoDB, Elasticsearch, Docker API, and the Kubernetes API must not answer the internet without authentication.
- **Cache and CDN**: personalized responses are not cached; test cache poisoning through unkeyed headers (`X-Forwarded-Host`, `X-Forwarded-Scheme`) and cache deception through URL extensions. A shared cache poisoning bug is urgent.
- **DNS**: no orphaned subdomains pointing to decommissioned services (subdomain takeover); CAA records configured.
- Cost abuse ("denial of wallet"): restrict paid API keys (maps, AI, SMS) by domain and set quotas and billing alerts.
- Scraping: if the content is a business asset, apply rate limits and anti-bot rules.
- Object storage (S3, GCS, Azure Blob, Supabase/Firebase Storage): no public buckets or objects by default; bucket policies and ACLs reviewed; signed URLs expire quickly and cannot be reused for a different object.
- Email authentication: SPF, DKIM, and DMARC records published on the domain, so nobody can send mail as the business or phish its customers.
- **Egress**: outbound traffic restricted to an allowlist or proxy, so SSRF and a compromised process cannot reach internal services or the internet freely.

## I. Secure development and dependencies
- Dependency audit with the package manager. Recommend Dependabot or an equivalent, plus an SBOM (`syft`).
- **Runtime and base image**: the language interpreter or server (PHP, Node, Python, Ruby, JRE, nginx, OpenSSL, glibc) is not a package the lockfile shows, so `npm audit` or `composer audit` will never flag its critical vulnerabilities. Get the exact version from the running process, compare it against the patched release of every open critical advisory, and check whether the branch is past end-of-life (no patches exist, so "up to date" is impossible).
- **Critical advisory sweep**: check the runtime and framework against advisories published since the last update, using the project's advisory page, GitHub Advisories, OSV, and CISA KEV for actively exploited issues. A patched dependency tree on top of a vulnerable runtime is still remotely exploitable.
- Static analysis (for example, Semgrep) and, if staging exists, a basic dynamic scan with OWASP ZAP.
- Lockfile committed and installed with the lockfile (`npm ci`, `composer install` from the lock, `pip install -r` with hashes).
- **Supply chain**: private packages scoped to a registry, dependency-confusion protection, typosquatting awareness, install scripts reviewed (`--ignore-scripts` where possible), artifact signing/provenance (`cosign`, SLSA), and no unreviewed packages added during an incident.
- **CI/CD**: secrets live in the CI's secret store, never in workflow files or Dockerfiles; pull requests from forks cannot read secrets; `pull_request_target` is used with extreme care; nothing from `github.event.*` (PR titles, branch names, issue bodies) is interpolated into `run:` blocks; deploy tokens, PATs, and cloud roles carry the minimum scope; OIDC replaces long-lived cloud keys; environments require approval; self-hosted runners are isolated; caches and artifacts cannot be poisoned across trust boundaries; repositories that contain secrets stay private.

## J. Monitoring, backups, and compliance
- Logs of failed logins, permission changes, data exports, admin actions, and errors, with no passwords, tokens, or card data.
- Alerts on spikes in 401/403/429 errors, WAF blocks, new admin users, mass data access, unusual exports, and billing.
- Automated backups and, above all, a restore that has been tested at least once.
- Privacy notice and compliance with the applicable personal data law (in Mexico, the LFPDPPP; in Europe, the GDPR). Recommend confirming the current version with a specialist.
- PCI-DSS: with hosted checkout, the burden is minimal. Explain this so it doesn't cause unnecessary alarm.
- Mobile: crash and tamper telemetry, no PII in logs, and a way to force an insecure app version out of service.

## K. AI and LLMs (only if the platform uses them)
- Treat model output as untrusted before executing tools; never pass it to a shell, database, or browser without validation.
- Require user confirmation for actions with side effects, and use a tool allowlist with least privilege.
- **Prompt injection**: test direct injection and indirect injection through RAG documents, web pages, emails, and tool output. The model must not be able to override system policy or exfiltrate data.
- In RAG, filter by permissions so the model only retrieves what the user is allowed to see; enforce it in the retrieval layer, not in the prompt.
- Encode model output before rendering it as HTML or Markdown, and sandbox any code or links it produces.
- Model API keys stay server-side, have spending limits, and are separate per environment.
- Log prompts and outputs with PII redaction, and pin model/version so behavior changes are deliberate.

## L. Mobile apps (Android and iOS)
The app is a client. The server enforces authorization and business rules; client-side checks are only defense in depth.
- **Static review**: decompile the APK/IPA and look for hardcoded secrets, API keys, endpoints, debug logs, and test credentials. Review `AndroidManifest.xml` / `Info.plist`, permissions, `android:exported`, `android:allowBackup`, `android:debuggable`, network security config, ATS exceptions, URL schemes, associated domains, and third-party SDKs.
- **Data at rest**: tokens and secrets in Keychain (iOS) or Keystore/EncryptedSharedPreferences (Android), never in `UserDefaults`, `SharedPreferences`, `AsyncStorage`, plists, SQLite, logs, backups, clipboard, or screenshots. Sensitive keys should require biometric/device authentication.
- **Authentication**: OAuth Authorization Code with PKCE, verified App Links/Universal Links, no client secrets in the binary, short sessions, logout that clears and revokes tokens, and server-side token validation.
- **Network**: TLS 1.2+, no cleartext, ATS strict, no trust-all `TrustManager`, certificate pinning and app attestation (Play Integrity, DeviceCheck/App Attest) verified server-side. Pinning is defense in depth, not a secret.
- **Platform interaction**: exported activities, services, receivers, and content providers; intent redirection and mutable PendingIntents; deep-link and Universal Link parameter validation; iOS URL schemes, XPC, app groups, and shared containers.
- **WebView**: disable JavaScript bridges unless required, validate every URL, block `file://` access from file URLs, avoid `addJavascriptInterface` with untrusted content, allowlist navigation, disable mixed content, and configure WKWebView message handlers safely.
- **Backend parity**: every web/API control applies. Device-bound headers and request-signing algorithms are extractable from the app, so they are not authentication by themselves; mobile-only endpoints need the same authorization and rate limits; retired API versions must be shut down; push-notification topics must not leak other users' messages.
- **Resilience and privacy**: root/jailbreak detection, anti-tampering, obfuscation, and attestation as layered defenses; minimal permissions; review tracking SDKs; iOS privacy manifest and Android data-safety declarations; signed release builds with no debug or test keys; a secure update path.
- **Testing safety**: use your own builds, test accounts, and a rooted/jailbroken test device you control. Do not repackage, instrument, or bypass protections on a third-party app.

## M. Race conditions and concurrency
- Test simultaneous requests against coupons, balances, stock, votes, follows, invites, password reset, OTP, refunds, payouts, quotas, and rate limiters. The classic result is a single-use benefit applied twice.
- Include **multi-endpoint races** (two different endpoints sharing state), connection-pool exhaustion, cache stampede, queue consumers, serverless concurrency, and database isolation levels.
- Use the single-packet/multi-request technique or a small bounded parallel burst on staging. Never flood production.
- Fix with database transactions, row locks or `SELECT ... FOR UPDATE`, unique constraints, atomic counters, idempotency keys, queue serialization, and monitoring for duplicate effects. An application-level "check then write" without a lock is not enough.

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

### GraphQL servers (Apollo, Hasura, Graphene, Yoga, graphql-php)
- Introspection and the playground/GraphiQL disabled in production unless deliberately public.
- Depth, complexity, alias, and operation-count limits; batching disabled or capped; persisted queries where possible.
- Authorization inside every resolver and every nested field, not only at the top-level query. Relay global IDs are object references and need the same ownership check.
- Subscriptions authenticate and authorize each channel; `Origin` is validated; CSWSH is tested.
- Verbose errors and field suggestions disabled in production (they leak the schema).
- File uploads validated like any other upload; GET requests must not mutate state.

### gRPC
- Server reflection disabled in production; keep `.proto` files private if they reveal internal services.
- Authentication and authorization on every method, not only on the gateway; metadata is not a trusted identity source.
- Deadlines, maximum message size, and rate limits configured; TLS required.
- Test BOLA with user A/user B tokens and metadata injection (`x-user-role`, `x-internal-service`).
- Protobuf input validated; no unbounded recursion or huge repeated fields.

### WebSockets / realtime services
- Authenticate the handshake and every message or subscription; tokens must expire on long-lived connections.
- Validate `Origin` and reject cross-site WebSocket hijacking; do not rely on cookies alone.
- Authorize each channel/topic per user and tenant; test predictable channel names and subscription IDs.
- Cap message size and rate; validate JSON/schema; protect against message injection and fan-out abuse.
- Use a dedicated realtime service with its own rate limits and audit logs.

### API gateways (Kong, Apigee, AWS API Gateway, Traefik, Nginx)
- Authentication and authorization at the service as well as the gateway; a gateway rule is not a substitute for a server check.
- Parser consistency: headers, duplicate parameters, content types, and body size limits must match the backend (smuggling and HPP live in the gaps).
- XML threat protection for SOAP/XML routes, JSON schema validation, request size limits, and timeouts.
- Rate limits per consumer, API key, and route; do not expose admin or metrics endpoints.
- Route shadowing and version confusion: `/api/v1` and `/api/v2` must not differ in authorization.

### AWS / GCP / Azure
- Root/owner account with MFA, no long-lived access keys, least-privilege IAM, and no wildcard `*` actions or resources.
- No public buckets, snapshots, disks, or database endpoints; block public access at the account level.
- Metadata service hardened (IMDSv2 on AWS); workloads use roles, not embedded keys.
- Audit logging enabled (CloudTrail, Cloud Audit Logs, Azure Activity Log) and shipped off-account.
- Secrets in a managed secret store with rotation; no secrets in user data, environment dumps, or CI logs.
- Security groups and firewall rules reviewed for `0.0.0.0/0` on management ports.

### Docker / Kubernetes
- Containers run as non-root, read-only filesystem where possible, no `--privileged`, no host network/PID, and no Docker socket mounted.
- Images from a trusted registry, pinned by digest, scanned (`trivy`), and signed (`cosign`); no `latest` in production.
- No secrets in image layers, build args, or environment dumps; use a secret manager or sealed secrets.
- Kubernetes: RBAC least privilege, network policies, resource limits, admission control, no automounted service-account tokens where unnecessary, and no dashboard/API exposed publicly.
- CI runners and build caches isolated; a compromised build cannot poison production artifacts.

### GitHub Actions / GitLab CI
- Secrets in the CI secret store, masked, and scoped to protected branches/environments; fork PRs never receive them.
- `pull_request_target` and `workflow_run` reviewed carefully; never check out and execute untrusted PR code with secrets.
- No `${{ github.event.* }}` interpolation into `run:` or shell scripts; pass values through environment variables and quote them.
- Least-privilege `GITHUB_TOKEN`/CI job tokens; OIDC to the cloud instead of long-lived secrets.
- Third-party actions pinned by commit SHA, not a mutable tag; self-hosted runners isolated and never used for public repos.
- Build artifacts and caches signed or scoped; no secret leakage in logs; dependency updates reviewed.

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
- `helmet` or equivalent headers; body limits (`express.json({ limit })`), `qs` depth/parameter limits, and a correct `trust proxy` setting.
- Input validation with a schema (zod, joi); reject unknown fields instead of stripping them silently.
- Avoid `eval`, `new Function`, and `child_process` with input data; never pass user input to a shell.
- Guard against prototype pollution in recursive merges and `JSON.parse`-then-merge patterns.
- Do not serve dotfiles or source maps with `express.static`; keep `NEXT_PUBLIC_`/`VITE_` variables public only.
- Next.js: Server Actions and API routes validate the session and resource ownership; middleware is not the only authorization layer.

### PHP / Laravel
- **Runtime version**: `php -v` and `php -r 'echo PHP_VERSION;'` inside the running process (container, FPM pool, hosting panel), not only the image tag or the composer constraint. Cross-check `FROM php:x.y-*` in the Dockerfile, `require.php` and `config.platform` in `composer.json`, the CI workflows, `.platform.app.yaml`, and Forge/Envoyer or cPanel/Plesk settings. These drift: an updated image can still be running an old container.
- **Advisories**: compare the exact version against the patched releases of open critical advisories and against the support timeline. Branches past end-of-life (8.1 and older) receive no patches. `expose_php=Off` reveals less, but hiding a version is not fixing it.
- **Upgrade mechanics**: after updating, restart PHP-FPM or the web server and re-run `php -v`; the patched binary does nothing until the service restarts. If the version was exposed while an advisory was public, follow the incident-response pass in Phase 4.
- `APP_DEBUG=false` in production; `APP_KEY` set and rotated if leaked.
- `display_errors=Off` and `log_errors=On`; delete stray `phpinfo()` files and default sample `index.php` (FrankenPHP images before 1.12.5 shipped one that executes `phpinfo()`).
- `$fillable` or `$guarded` on models (mass assignment); Policies and Gates in controllers; signed URLs for temporary access.
- `{!! !!}` only with trusted content; `unserialize` never on user input; `phar://` and `allow_url_include` disabled; `open_basedir` and `disable_functions` where practical.
- Sessions: `session.cookie_httponly`, `secure`, `samesite`; `password_hash` with Argon2id/bcrypt.
- XXE: use `libxml_disable_entity_loader(true)` (or the modern equivalent) and reject DTDs.

### Python / Django / Flask
- `DEBUG=False` in production; `ALLOWED_HOSTS` defined; real `SECRET_KEY` out of the repository.
- Review `|safe` and `mark_safe`; never `pickle`/`marshal` external data; use `yaml.safe_load`, not `yaml.load`.
- No `eval`/`exec` on user input; `subprocess` with `shell=False` and an argument list.
- Django: `SECURE_SSL_REDIRECT`, `SESSION_COOKIE_SECURE`, `CSRF_COOKIE_SECURE`, `SECURE_HSTS_SECONDS`, and explicit DRF permissions; `CORS_ALLOW_ALL_ORIGINS` off unless the API is public.
- Flask: `SESSION_COOKIE_HTTPONLY`, `SESSION_COOKIE_SECURE`, `SESSION_COOKIE_SAMESITE`, CSRF protection, and a strict CORS allowlist.

### Ruby on Rails
- Authorization on every controller action (Pundit, CanCanCan), not only on the happy paths; use scopes, not `find` plus a manual check.
- Strong parameters; watch for models that still accept mass assignment.
- `config.force_ssl = true`, `protect_from_forgery with: :exception`, encrypted or signed session cookies, `config.hosts` set.
- Never `Marshal.load` or `YAML.load` untrusted data; use `YAML.safe_load`; avoid `send`/`constantize` on user input.
- Credentials encrypted and out of git; CSP initializer configured; no secrets in `ENV` dumps or error pages.

### Java / Spring Boot
- Actuator endpoints disabled or authenticated in production; `/actuator/env`, `/heapdump`, and `/trace` must never answer the internet.
- No SpEL or template expression built from user input; no deserialization of untrusted data.
- Jackson default typing off; XStream and `ObjectInputStream` avoided or allowlisted; Spring Security `authorizeHttpRequests` on every route with `@PreAuthorize` for admin actions.
- CSRF enabled for cookie-based sessions; CORS explicit; no `permitAll` on internal routes.

### .NET
- Authorization on every endpoint (a missing `[Authorize]` is the classic hole), with policy-based checks for admin actions.
- Mass binding limited with `[Bind(Include = ...)]`; machine keys not shared or predictable; debug off and custom errors enabled in production.
- `BinaryFormatter` disabled; `TypeNameHandling.None` in Json.NET; ViewState MAC enabled and machine keys protected; anti-forgery on cookie-based POSTs; data-protection keys persisted and protected.

### Android (Kotlin/Java)
- Manifest: review `android:exported`, intent filters, permissions, `allowBackup`, `debuggable`, `usesCleartextTraffic`, and network security config. Exported activities/services/providers/receivers that touch user data are urgent.
- Storage: Keystore and EncryptedSharedPreferences for tokens; no secrets in `SharedPreferences`, SQLite, external storage, logs, or backups.
- WebView: no `addJavascriptInterface` with untrusted pages, no `setAllowFileAccessFromFileURLs(true)`, validate `shouldOverrideUrlLoading`, and disable mixed content.
- Deep links/App Links: `autoVerify`, path validation, and server authorization for every action the link triggers; beware intent redirection and mutable `PendingIntent`.
- Release: signed, minified, no debug logs, no test keys, Play Integrity verified server-side, and no secrets in `BuildConfig` or resources.
- Third-party SDKs: review permissions, trackers, and network endpoints; remove unused SDKs.

### iOS (Swift/Objective-C)
- `Info.plist`: ATS exceptions (`NSAllowsArbitraryLoads`), URL schemes, associated domains, `UIFileSharingEnabled`, and `NSAllowsArbitraryLoadsInWebContent`.
- Keychain: `kSecAttrAccessibleWhenUnlockedThisDeviceOnly` for secrets; nothing sensitive in `UserDefaults` or plists; biometric-bound items for high-value actions.
- Universal Links: validate `apple-app-site-association`, associated domains, and the paths/parameters the app acts on.
- WebView: safe `WKWebView` configuration, no broad `loadFileURL` read access, validate message handlers, and block arbitrary navigation.
- Entitlements: review keychain access groups, app groups, and associated domains; shared containers must not leak secrets across apps.
- Release: no debug builds, no test keys, DeviceCheck/App Attest verified server-side, privacy manifest complete.

### React Native / Flutter / Capacitor
- `AsyncStorage`, `SharedPreferences`, and `UserDefaults` are not secure storage; use Keychain/Keystore or an audited secure-storage plugin.
- JavaScript bundles and Dart snapshots are readable; no secrets, admin endpoints, or signing keys in the client.
- Deep links and WebView bridges follow the same rules as native; validate the URL and the caller before acting.
- OTA update mechanisms (CodePush, EAS Update, custom bundles) must be signed and verified; never load arbitrary remote code.
- Review native modules and third-party plugins; they can bypass the framework's security defaults.

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
