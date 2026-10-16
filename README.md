# Web Security Audit

A [Claude](https://claude.ai) skill that runs a full security review of a web platform — any stack — fixes what is safe to fix, and explains the risks in plain language to someone with **no cybersecurity background**.

It adapts the review to the technologies it actually finds (Supabase, Firebase, Cloudflare, Vercel, Stripe, WordPress, Node, Laravel, Rails, Spring, .NET, Django, and more), so you get concrete findings with file and line references instead of a generic checklist.

## What it does

- **Understands** the platform: what it does, its stack, who uses it, and what is worth protecting.
- **Reviews** the applicable controls from the OWASP Top 10, OWASP ASVS, and real-world pentesting vectors: authentication and sessions, access control (IDOR, BOLA, multi-tenant isolation), injection, XSS, CSRF, SSRF, file uploads, business logic, payments, headers, infrastructure, dependencies, CI/CD, and AI/LLM usage.
- **Tests methodically**: inventories the attack surface, builds an authorization matrix (anonymous / user A / user B / admin), and reproduces each finding by hand before reporting it.
- **Fixes** what is safe to fix automatically, asks before behavioral changes, and writes manual steps for external dashboards.
- **Writes a report** (`security/report.md`) in plain words, with severity levels, business-impact examples, a manual action checklist, and a glossary.

## Safety rules

The skill is safety-first. It will not proceed without confirmation that you own the system or have written authorization, and it only runs **active** tests against **local or staging** environments. Against production it does passive reads only. Payments are tested in sandbox mode. It never exfiltrates real user data, throttles every test to avoid disruption, and redacts secrets in its output.

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
- The optional toolchain it may suggest (`gitleaks`, `semgrep`, `npm audit`/`osv-scanner`, `testssl.sh`, OWASP ZAP, `nuclei`) is used only with your confirmation.

## Repository structure

```
web-security-audit/
├── SKILL.md      # The skill (frontmatter + instructions)
├── README.md
└── LICENSE
```

## License

[MIT](LICENSE)
