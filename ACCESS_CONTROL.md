# Access control matrix

## GitHub (account kirotick, all KIRO repositories)

| Who | Role | Can push to main | Can merge PR | Can change settings |
|---|---|---|---|---|
| kirotick (owner) | Admin | No (ruleset: PR required) | Yes, after checks pass (D3) | Yes |
| Future developer | Write | No | With owner approval | No |
| GitHub Actions | Read-only token by default | No | No | No |

## KIRO application roles (kiro-backend `perms.js`)

| Permission | owner | manager | accounts | pos |
|---|---|---|---|---|
| Create sales | ✔ | ✔ | | ✔ |
| See all sales | ✔ | ✔ | ✔ | own only |
| Reports, insights, Ask KIRO | ✔ | ✔ | ✔ | |
| Items: write / see cost | ✔ / ✔ | ✔ / ✔ | – / ✔ | |
| Price override, any discount | ✔ | ✔ | | capped |
| Manage staff | ✔ | | | |
| Configure payments | ✔ | | | |

## Services

| Service | Production secret store | Who can read |
|---|---|---|
| Neon (database) | Neon console; `DATABASE_URL` in Render | Owner |
| Netlify (website) | Netlify settings | Owner |
| Render (API, payments) | Render environment | Owner |
| GitHub CI | Repository secrets `TEST_DATABASE_URL` (Neon `qa` branch only) | Workflows on `main` and PRs from this repo (never forks) |
