# Security findings register

Owner for all rows: bienvenudev. Date: 2026-11-09.

| # | Source | Finding | Severity | Decision | Justification / action | Revisit |
|---|--------|---------|----------|----------|------------------------|---------|
| 1 | Bandit | B105 hardcoded_password_string, `SECRET_KEY` in `campus_notifier/settings.py:27` | Low | Remediate | Key moved to `DJANGO_SECRET_KEY` env var (`.env`, gitignored; `.env.example` committed). The old key is **still in git history**, so removing it from the file is not rotating it. It is a `django-insecure-` dev placeholder, but any deployed environment must use a freshly generated key and the old one must be treated as burned. | Rotate before first real deploy |
| 2 | Trivy (python-pkg) | Django 5.2.7: 1 CRITICAL (CVE-2025-64459, SQL injection) + 7 HIGH | Critical/High | Remediate | Bumped to `Django==5.2.17`, which contains the fixes for every listed CVE. | Re-scan each PR |
| 3 | Trivy (OS, debian 13.7) | 49 HIGH in Debian packages (util-linux, libacl1, ...) | High | Accept risk | No fixed package version is published yet (status `affected`), so there is nothing to upgrade to. Gate uses `ignore-unfixed: true` so it fails as soon as a fix exists. Image is a dev demo, not internet-exposed. | 2026-11-23, rebuild on a newer base image |
| 4 | Checkov | CKV_AWS_136 ECR repository not KMS-encrypted (`terraform/main.tf`) | Medium | Remediate | Added `encryption_configuration { encryption_type = "KMS" }`. Checkov now: 4 passed, 0 failed. | n/a |
| 5 | gitleaks | `django-insecure-...` key not flagged | Info | Accept (by design) | gitleaks treats the prefix as a dev placeholder, and the commit hook only sees staged changes, not history. Bandit covers this case. | n/a |

## Gate policy

- **Secrets (gitleaks):** blocks on any real secret.
- **Image (Trivy):** blocks on CRITICAL with a published fix. HIGH is reported but does not block, since 49 unfixed OS findings would make the gate permanently red.
- **IaC (Checkov):** blocks (`soft_fail: false`). Nothing is suppressed.
- **Code (Bandit):** blocks. The repo is at zero findings after row 1, so blocking costs nothing now and stops regressions. Even a Low finding such as a hardcoded secret is cheap to fix on the day it appears.

## Discussion

**Why does CI run gitleaks when the pre-commit hook exists?** Hooks are local and optional: they can be skipped with `--no-verify`, are not installed on a fresh clone until `pre-commit install`, and only scan staged changes. CI is the enforced backstop that scans the whole PR history on a clean machine.

**Why did Bandit flag a key gitleaks passed?** They look for different things. gitleaks matches secret patterns and deliberately ignores the `django-insecure-` placeholder. Bandit reads the code and flags any string literal assigned to a name like `SECRET_KEY` or `password`, whatever the value.
