# Web Security Audit

> **Last updated: 2026-11-03** — added API and mobile coverage (OWASP API Top 10 and MASVS/MASTG), plus modern web vectors: request smuggling, HTTP parameter pollution, GraphQL/gRPC/WebSocket, JWT/OAuth hardening, XXE, deserialization, SSTI, prototype pollution, race conditions, cloud/container/CI-CD, and supply chain. See [CHANGELOG.md](CHANGELOG.md).

A [Claude](https://claude.ai) skill that runs a full security review of a web platform, API, or mobile app and its backend — any stack — fixes what is safe to fix, and explains the risks in plain language to someone with **no cybersecurity background**.

It adapts the review to the technologies it actually finds (Supabase, Firebase, Cloudflare, Vercel, Stripe, WordPress, Node, Laravel, Rails, Spring, .NET, Django, GraphQL, gRPC, WebSockets, Android, iOS, React Native, Flutter, and more), so you get concrete findings with file and line references instead of a generic checklist.

## What it does

- **Understands** the platform: what it does, its stack, who uses it, and what is worth protecting.
- **Reviews** the applicable controls from the OWASP Top 10, OWASP API Security Top 10, OWASP MASVS/MASTG, OWASP ASVS, and real-world pentesting vectors: authentication and sessions, JWT/OAuth, access control (IDOR, BOLA, BFLA, multi-tenant isolation), injection, XSS, CSRF, SSRF, file uploads, XXE, deserialization, SSTI, prototype pollution, HTTP parameter pollution, request smuggling, cache poisoning, business logic, race conditions, payments, headers, infrastructure, cloud/containers, dependencies, runtime and base-image advisories, CI/CD, supply chain, mobile apps, and AI/LLM usage.
- **Covers APIs and mobile**: REST, GraphQL, gRPC, WebSocket, webhooks, API versioning, pagination/cursors, and Android/iOS/hybrid app storage, exported components, deep links, WebView, token storage, pinning, attestation, and backend parity.
- **Tests methodically**: inventories the attack surface, builds an authorization matrix (anonymous / user A / user B / another tenant / admin), and reproduces each finding by hand before reporting it.
- **Fixes** what is safe to fix automatically, asks before behavioral changes, and writes manual steps for external dashboards.
- **Writes a report** (`security/report.md`) in plain words, with severity levels, business-impact examples, a manual action checklist, a mobile subsection when relevant, and a glossary.

## Safety rules

The skill is safety-first. It will not proceed without confirmation that you own the system or have written authorization, and it only runs **active** tests against **local or staging** environments. Against production it does passive reads only. Payments are tested in sandbox mode. Mobile testing is limited to builds and test devices you control. Request smuggling, cache poisoning, and race-condition bursts never run against production. It never exfiltrates real user data, throttles every test to avoid disruption, and redacts secrets in its output.

## Installation

Clone or copy this folder into your Claude skills directory.

**macOS / Linux**

```bash
git clone https://github.com/jazielcarballo/web-security-audit.git ~/.claude/skills/web-security-audit
```

**Windows (PowerShell)**

```powershell
git clone https://github.com/jazielcarballo/web-security-audit.git "$env:USERPROFILE\.claude\skills\web-security-audit"
```

Or download `SKILL.md` and place it at `<skills-dir>/web-security-audit/SKILL.md`.

## Usage

This skill is set to **explicit invocation only** (`disable-model-invocation: true`), so it will not trigger on its own. Invoke it by name:

```
/web-security-audit
```

or ask directly, for example:

> Run a security audit on this project and explain the risks in plain language.

## Requirements

- Claude with skill support (Claude Code, Claude desktop app, or Claude.ai).
- The optional toolchain it may suggest (`gitleaks`, `semgrep`, `npm audit`/`osv-scanner`, `testssl.sh`, OWASP ZAP, `nuclei`, `jwt_tool`, `grpcurl`, `websocat`, `trivy`, `syft`, MobSF, `apktool`/`jadx`, Frida/objection) is used only with your confirmation.

## Repository structure

```
web-security-audit/
├── SKILL.md      # The skill (frontmatter + instructions)
├── README.md
├── CHANGELOG.md  # Dated record of changes
└── LICENSE
```

## Author

Created by **Jaziel Carballo** ([@jazielcarballo](https://github.com/jazielcarballo)).

## License

[MIT](LICENSE)
