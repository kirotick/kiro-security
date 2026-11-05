# KIRO threat model (STRIDE, first version)

## Assets
Shop sales and invoices, staff logins, owner reports, customer phone numbers, payment merchant keys (encrypted), webhook events, the live website.

## Trust boundaries
Browser (website, POS app, LUX shop) · Mobile app · Core API (kiro-backend) · Payment service (kiro-payments) · Neon PostgreSQL · Payment provider webhooks · GitHub / CI · Netlify / Render dashboards.

| Threat | Where | Control in place | Evidence |
|---|---|---|---|
| Spoofing: forged or expired login token | Core API, payments | HS256 verified with timing-safe compare; `alg:none`, expired, tampered rejected | kiro-backend `auth-cors.test.js`, `http.test.js` |
| Spoofing: fake payment webhook | Payments | Signature check, merchant + exact amount match, unknown merchant rejected | kiro-payments `unit.test.js` |
| Tampering: one shop changes another shop's data | Core API | Every query filters by `business_id`; other tenants get 404 | `http.test.js` tenant isolation |
| Tampering: SQL injection | Core API | Parameterised queries | `http.test.js` injection cases |
| Repudiation: who changed what | All | Git history, PR records, POS audit log, webhook event log | — |
| Information disclosure: secrets in code | GitHub | Push protection, secret scanning, gitleaks, `.gitignore` for `.env*` | CI `secrets` job |
| Information disclosure: XSS in shop/POS inputs | Browser | Escaped rendering | QA `xss-and-headers.spec.ts` |
| Denial of service: login brute force | Core API | 30 logins / 15 min per IP; account lock after 5 wrong passwords | `http.test.js` rate limit |
| Elevation: cashier uses owner functions | Core API, POS | Role table enforced on the server | `core.test.js`, `http.test.js` roles |
| Elevation: unreviewed code to production | GitHub | Ruleset on `main`, required checks | Repository rulesets |
