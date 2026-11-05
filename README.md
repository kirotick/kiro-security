# kiro-security

Security architecture and evidence: threat model, risk register, access control, incident response, remediation tracking. **Never** store passwords, keys or customer data here.

Part of KIRO (Kitchen Intelligence & Restaurant Operations). Owner: @kirotick. Migrated with full history from `kirotick/KIRO_POS` (commit `252de09`), see kiro-documentation/governance/REPOSITORY_MIGRATION_PLAN.md.

## What this repository owns
- `THREAT_MODEL.md`, `RISK_REGISTER.md`, `ACCESS_CONTROL.md`, `INCIDENT_RESPONSE.md`

Nothing here may be copied into another KIRO repository; other repositories use it by version or through its API.

## Depends on
- Findings from kiro-qa-testing security scans and reviews of every repo

## Build
Documents only.

## Test
CI checks for committed secrets.

## Branches, reviews and releases
`main` is production and protected: changes arrive only by pull request with passing checks (see CONTRIBUTING.md). Versions follow semantic versioning; every release is a git tag `vMAJOR.MINOR.PATCH` with notes in CHANGELOG.md, so each deployment maps to one exact commit. Rollback = redeploy the previous tag.

## Security
See SECURITY.md. Never commit secrets: `.env` files are ignored, and secret scanning with push protection plus gitleaks in CI block them.
