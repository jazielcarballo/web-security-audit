# Changelog

All notable changes to this skill are documented here. Dates are `YYYY-MM-DD`.

## 2026-11-03

### Added

- Mobile coverage (Android, iOS, React Native, Flutter, Capacitor) based on OWASP MASVS/MASTG: static review, storage, Keychain/Keystore, exported components, deep links/Universal Links, WebView, intent redirection, token storage, pinning, app attestation, resilience, privacy, and backend parity. New section L and new technology modules.
- API coverage based on the OWASP API Security Top 10: BOLA/BFLA on every verb, object-property authorization, API inventory and shadow versions, pagination/cursor abuse, webhook security, API key leakage, and rate limiting per user/IP/key/tenant.
- GraphQL, gRPC, WebSocket, and API gateway modules: introspection, depth/complexity/alias limits, batching, Relay IDOR, nested authorization, subscriptions, CSWSH, reflection, metadata injection, and parser consistency.
- JWT and OAuth hardening: `alg: none`, algorithm confusion, `kid`/`jku`/`x5u`/`jwk` injection, claim validation, refresh rotation, `jti` deny-list, exact `redirect_uri`, `state`/`nonce`, PKCE, ID-token confusion, account-linking CSRF, and token leakage.
- Modern web vectors: request smuggling/desync, HTTP parameter pollution and cloaking, cache poisoning and deception, XXE, insecure deserialization, SSTI, prototype pollution, CSV/formula injection, ReDoS, and dangerous execution sinks.
- Race-condition section M: multi-endpoint races, single-packet requests, connection pools, serverless concurrency, idempotency, and locking.
- Cloud, container, and CI/CD coverage: IAM, IMDSv2, public storage, Kubernetes RBAC, image signing, `pull_request_target`, OIDC, runner isolation, dependency confusion, SBOM, and artifact provenance.
- Expanded AI/LLM checks: direct and indirect prompt injection, tool abuse, output encoding, RAG permission filtering, and spending limits.
- Mobile and API tools in the toolchain: MobSF, `apktool`/`jadx`, Frida/objection/drozer, `jwt_tool`, `grpcurl`, `websocat`, `graphql-cop`/`clairvoyance`, `trivy`, `checkov`, `syft`, and `cosign`.
- Mobile-specific safety rules and a mobile subsection in the report, including verified app/SDK versions.

### Documentation

- Expanded the README with the full audit and fix coverage, a project overview, principles, scope and limitations, a coverage matrix, the five-phase workflow, and a detailed safety section.
- Added a cybersecurity-themed cover image (`assets/cover.png`) to the repository and README.
- Expanded the GitHub About description and topics.
- Expanded the Author section with background, research interests, and contact links.
- Added `WORKLOG.md` for dated decisions and verification steps, linked from the project status section.

### Changed

- The skill now covers web, API, and mobile in the description, Phase 1, authorization matrix, severity guidance, and report.
- Protocol-level tests (smuggling, cache poisoning, race-condition bursts, credential brute force) are explicitly limited to staging with a bounded request budget.
- Severity guidance now treats shared-cache poisoning, request smuggling, mobile binary secrets, and exported-component data exposure as urgent.
- Technology modules for Node, PHP/Laravel, Python/Django/Flask, Rails, Java/Spring, and .NET were expanded with the deserialization, template, parser, and authorization pitfalls relevant to each stack.

## 2026-10-22

### Added

- Runtime and base-image advisory checks in control I. The language interpreter, web server, and container base (PHP, Node, Python, Ruby, JRE, nginx, OpenSSL, glibc) are invisible to lockfile scanners, so they now get their own version check against patched releases, end-of-life status, and active exploitation.
- Runtime version verification in the toolchain and in the PHP/Laravel module: the version is read from the running process, then cross-checked against the Dockerfile base tag, `composer.json`, CI workflows, and hosting panels.
- Incident-response pass for critical advisories that were public while the system was exposed: webshells and modified files, unknown users and cron jobs, outbound connections, secret rotation, and a record of the exposure window.
- "Verified versions" section in the report, so the audit leaves a dated record for later.
- Post-restart verification after a runtime upgrade: a patched binary that was never restarted is still the old one in memory.

### Changed

- Runtime and base-image upgrades are now classified as "fix with confirmation", since they can change behavior for the whole application.
- Severity rule: an unauthenticated remote code execution with public exploitation is rated urgent even when it needs special conditions to trigger.

## 2026-10-16

### Added

- Initial release: full security review based on the OWASP Top 10 and OWASP ASVS, stack-specific modules, safety rules, and a plain-language report.
