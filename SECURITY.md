# Security Policy

## Supported versions

Security fixes are applied to the latest version on the default branch. Older
releases may not receive security updates.

| Version | Supported |
| --- | --- |
| Latest | Yes |
| Older releases | No |

## Reporting a vulnerability

Please do not report security vulnerabilities in public issues, pull requests,
or discussions.

Use GitHub's **Report a vulnerability** option on the repository's **Security**
tab to submit a private report. If that option is unavailable, contact the
repository maintainers privately through GitHub and include the repository name
and the subject **CETP security report**.

Please provide:

- a clear description of the vulnerability and its potential impact;
- the affected version, commit, file, or component;
- reproducible steps or a minimal proof of concept;
- any relevant logs, configuration, or environment details;
- suggested mitigation, if known.

Redact credentials, personal information, production data, and unredacted
PostgreSQL query plans from reports. CETP plans can contain query literals and
other sensitive values; use the project's plan-redaction command before sharing
them:

```bash
cetp redact plan.json -o plan.redacted.json
```

## Response process

Maintainers will acknowledge a report as soon as practical, investigate the
reported behavior, and provide updates when the assessment changes. Confirmed
issues will be fixed or mitigated in the latest supported version, with release
notes describing the impact and resolution when appropriate.

Please allow maintainers reasonable time to investigate and release a fix before
making vulnerability details public. Coordinated disclosure is preferred.

## Scope

Reports are especially valuable for:

- unintended disclosure of query or plan data;
- unsafe handling of files, paths, or configuration;
- command or code injection;
- dependency and model-artifact supply-chain risks;
- CI/CD gate behavior that could bypass an intended SLA decision.

This project is a research prototype. Running it against production systems,
databases, or data remains the operator's responsibility; do not include
production secrets or private data in issues, pull requests, test fixtures, or
repository artifacts.
