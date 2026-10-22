# Changelog

All notable changes to this skill are documented here. Dates are `YYYY-MM-DD`.

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
