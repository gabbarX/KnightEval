# Security Policy

## Supported versions

KnightEval is pre-1.0. Security fixes are applied to the latest released version
only. Once 1.0 ships, this policy will be updated with a support window.

## Reporting a vulnerability

**Please do not open a public issue for security problems.**

Report privately via GitHub's [private vulnerability reporting][gh-advisory]
("Report a vulnerability" on the repository's **Security** tab), or contact the
maintainer directly through their GitHub profile.

Please include:

- A description of the issue and its impact
- Steps to reproduce (a minimal config/dataset if relevant)
- Affected version(s)

We aim to acknowledge reports within a few days and will keep you updated on the
fix and disclosure timeline. Coordinated disclosure is appreciated.

## Scope & threat model notes

KnightEval is a local-first CLI. A few things are security-relevant by design:

- **Secrets are environment-only.** API keys are read from the environment and
  must never be written to config files or saved runs. A leaked key in
  `run.json` or logs is a security bug — please report it. (There is a test that
  asserts redaction; a regression there is in scope.)
- **The endpoint under test is user-supplied.** KnightEval sends your dataset to
  the HTTP endpoint you configure and, if a judge is configured, to that judge
  provider. Data leaves your machine only for endpoints/judges you configure.
- **HTML reports are self-contained** (no CDN, inlined assets). Report generation
  must not enable script injection from endpoint responses (e.g. answer text
  rendered into HTML). Escaping/sanitization issues are in scope.
- **Dependency vulnerabilities** in our declared dependencies are in scope;
  please report or open a Dependabot-style PR.

[gh-advisory]: https://docs.github.com/en/code-security/security-advisories/guidance-on-reporting-and-writing-information-about-vulnerabilities/privately-reporting-a-security-vulnerability
