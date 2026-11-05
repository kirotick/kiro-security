# KIRO enterprise security and compliance gap assessment — Stage 1 and 2

| Field | Value |
|---|---|
| Version | 1.0 (draft for review) |
| Owner | KIRO owner (@kirotick) — CISO role not yet staffed |
| Created | 2026-11-05 |
| Last review | 2026-11-05 |
| Approval status | **Awaiting owner review** |
| Applicable systems | All 10 KIRO repositories, kiro-pos.netlify.app, Neon project `kiro` |
| Prepared by | AI-assisted review (Claude). Not an independent audit, legal opinion or VAPT. |

## How this was done (evidence, not assumptions)
- Read the source of every repository in `KIRO-WORKSPACE` (POS payment flow, server auth, password hashing, payment state machine, migrations, AI data flow, logging).
- Read live configuration through APIs: GitHub rulesets and security settings, Netlify headers, Neon region and history retention.
- Ran the existing test suites (105 unit / integration tests incl. real-PostgreSQL tenant isolation over HTTP; 33 browser tests) — all pass.
- Checked public texts (website, POS app) for certification or regulatory claims: **none found**.
- Researched current Indian regulation from official and legal sources (see `COMPLIANCE_REGISTER.md`). No secret values are reproduced here.

## What KIRO is today (honest baseline)
| Area | Current state |
|---|---|
| Live | Website, KIRO LUX shop and the **POS web app** at kiro-pos.netlify.app. POS stores each shop's data **only in that device's browser** (IndexedDB). No server is involved in billing today. |
| Built, not deployed | Core API (kiro-backend), KIRO PAY (kiro-payments, disabled), mobile app. Database: Neon project `kiro`, region **Singapore** (aws-ap-southeast-1), branches production / qa / dev. |
| Not built | Customer display, QR table ordering, online ordering, KDS beyond the POS kitchen screen, procurement, recipes/food costing, attendance/shifts, multi-branch and central admin dashboards, merchant onboarding, refunds, reconciliation, device management, observability, AWS infrastructure. |
| Payments | POS records UPI, card, cash, split and credit **as entered by the cashier**. The PhonePe dynamic-QR integration exists in kiro-payments but is disabled and has never been run against a provider. |

The directive describes a 10,000-restaurant enterprise platform; today KIRO is an early product with a strong engineering base (tests, CI, governance) but most enterprise systems not yet built. The findings below separate **risks in what exists** from **gaps to what is planned**.

## Severity counts
| Critical | High | Medium | Low | Informational |
|---|---|---|---|---|
| 2 | 8 | 8 | 4 | 4 |

## Findings

### CRITICAL

**G-01 — UPI payments are marked paid on the cashier's word (unverified).**
- *Evidence:* kiro-pos `ACT.payUpi` records `recordPayment(o, 'upi', due, { ref })` when the cashier taps "Payment received"; the UPI QR is a static `upi://pay?pa=<shop VPA>&am=…` link. Nothing checks a bank or provider.
- *Attack scenario:* a customer shows a fake or edited payment screenshot, a payment to a different VPA, or a payment for a smaller amount; the cashier taps "Payment received"; the bill closes as paid UPI. Insider variant: staff records UPI for a cash bill and keeps the cash.
- *Business impact:* direct revenue loss for restaurants; disputes; reputational damage to KIRO if KIRO is later presented as "verified UPI".
- *Correction:* (1) now: label these payments in the POS and on receipts as **"UPI — confirmed by cashier, not verified"**, require the UTR/reference for UPI, add a day-end "unverified UPI" report to match against the bank statement; (2) Phase 3: verified dynamic QR via kiro-payments + provider webhooks + server-side status check; bills become "Paid (verified)" only from that path.
- *Effort:* (1) 1–2 days; (2) 4–6 weeks incl. provider onboarding. *Verify:* POS test that UPI without verification never shows "verified"; payment-lifecycle tests (see TEST_STRATEGY in kiro-qa-testing).

**G-02 — Production database is in Singapore; payment data must be stored only in India.**
- *Evidence:* Neon project `kiro` region `aws-ap-southeast-1`. RBI circular DPSS.CO.OD.No 2785/06.08.005/2017-18 (6 Apr 2018) and RBI FAQs: entire payment-system data stored only in India; responsibility on the authorised operator, applied to its service providers and vendors. CERT-In Directions (28 Apr 2022): logs kept within India for 180 days.
- *Attack / failure scenario:* not an attack — a contractual and regulatory blocker: an acquiring bank / payment aggregator will require India-only storage of transaction data (bills, transaction references, amounts) by its technology vendors.
- *Business impact:* cannot go live with KIRO PAY; possible termination by the provider if launched.
- *Correction:* move production (and staging) PostgreSQL to India **before any payment data exists** (today the payment tables are empty): AWS RDS PostgreSQL in Hyderabad (ap-south-2) with DR in Mumbai (ap-south-1), as in the directive; keep Neon only for dev/qa or confirm an India region. Logs to an India region.
- *Effort:* 1–2 weeks (Phase 2). *Verify:* infrastructure-as-code review; AWS Config rule on region; data-location section in the provider's vendor questionnaire.

### HIGH

**G-03 — No written approval of the payment model; legacy provider API.**
- *Evidence:* RBI (Regulation of Payment Aggregators) Directions, 2025 (15 Sep 2025, effective immediately): non-bank PAs need RBI authorisation; a Payment Gateway is technology infrastructure that does not handle funds. KIRO's model (direct-to-merchant settlement, KIRO never holds funds) fits a technology service provider, **but** this depends on the bank/PA's written confirmation of the exact multi-merchant setup. kiro-payments implements PhonePe's salt-key `X-VERIFY` dynamic-QR flow; current provider APIs and onboarding must be confirmed with the provider.
- *Impact:* a model that accidentally involves KIRO in fund flows would require authorisation KIRO does not have.
- *Correction:* obtain written confirmation from the chosen bank/PA; legal opinion on classification; re-verify the provider's current API version before Phase 3. *Effort:* weeks (external). *Verify:* signed agreement + legal opinion filed as evidence.

**G-04 — Authentication is not enterprise-grade.**
- *Evidence:* kiro-backend issues 12-hour HS256 JWTs with one shared secret (also needed by kiro-payments, ADR-0002); no refresh-token rotation, no revocation list, no MFA, no session/device list, no admin roles; code marked "PLACEHOLDER AUTH". Positives: scrypt hashing, timing-safe compare, `alg:none`/expired/forged tokens rejected (tested), lockout after 5 wrong passwords, per-IP rate limits.
- *Attack:* a stolen token works for up to 12 h and cannot be revoked; a leaked shared secret lets anyone mint tokens for any tenant.
- *Correction:* short-lived access tokens (≤15 min) + rotating refresh tokens bound to a device/session record; asymmetric signing (EdDSA/ES256) with keys in AWS KMS so services only hold public keys; MFA (TOTP, then passkeys) for owners and all KIRO staff; session list with remote sign-out; admin plane separate from tenant plane. *Effort:* 3–4 weeks. *Verify:* auth test suite incl. revocation, refresh reuse detection, MFA bypass attempts.

**G-05 — Tenant isolation relies on every query being correct (no RLS).**
- *Evidence:* 25 `business_id` filters in SQL; HTTP tests prove Business B cannot read/change Business A today; no PostgreSQL Row-Level Security.
- *Attack:* one future query missing `WHERE business_id = $1` leaks or alters another restaurant's data (IDOR).
- *Correction:* enable RLS on every tenant table with `current_setting('kiro.tenant_id')` set per transaction by a non-superuser application role; keep app-level filters (defence in depth); cross-tenant tests in every release (already in CI; extend to every endpoint automatically). *Effort:* 1–2 weeks. *Verify:* a test that disables the app filter and proves RLS still blocks.

**G-06 — Payment lifecycle, ledger and reconciliation are missing.**
- *Evidence:* `bills.status` allows only `Pending | Paid | Review`; no append-only ledger, no refunds, no reversal handling, no provider status polling, no reconciliation, no dead-letter queue. Positives: verified webhook signature, exact amount and merchant match, idempotent `Pending→Paid` (8 concurrent callbacks → 1 winner, tested), QR expiry.
- *Correction:* full state machine (Created → Awaiting QR → Pending → Processing → Successful / Failed / Expired / Cancelled → Reversed / Refund initiated / Partially refunded / Refunded / Under investigation) enforced in the database (transition table + trigger) and code; append-only `payment_events` ledger; provider status checks for stale Pending; daily reconciliation with an exception queue. *Effort:* 4–6 weeks. *Verify:* lifecycle tests for every scenario in Part 13 of the directive.

**G-07 — POS devices: unencrypted local data, weak PIN storage, no device management.**
- *Evidence:* IndexedDB stores bills, customers (names, phone numbers in the credit book), staff and settings in the browser; staff PINs are hashed with a single SHA-256 (fast; 4–6 digit PINs can be brute-forced offline from a copied database); no device registration or remote revocation; any browser with the URL can start a new shop.
- *Attack:* a lost or stolen tablet exposes sales and customer data; a dismissed employee keeps a device.
- *Correction:* register devices against a tenant (server-issued device credential), remote revoke and wipe-on-next-sync, encrypt local data with a key released by the server after login, PIN verification server-side or via a slow KDF with attempt limits; managed devices (MDM) for restaurant hardware. *Effort:* 3–5 weeks after the API is live. *Verify:* lost-device drill; local storage inspection test.

**G-08 — Backup and disaster recovery do not meet the proposed RPO/RTO.**
- *Evidence:* Neon free tier, point-in-time history **6 hours**, single region, no restore ever tested; POS data has no backup at all except a manual JSON download.
- *Correction:* RDS Multi-AZ in Hyderabad, automated backups + PITR (≥7–35 days), cross-region snapshot copy to Mumbai, quarterly restore drills with measured RPO/RTO; POS sync to server with conflict-safe replay. *Effort:* part of Phase 2 (1–2 weeks) + POS sync (Phase 4). *Verify:* restore drill report with timings.

**G-09 — No logging, monitoring, alerting or incident capability.**
- *Evidence:* services log a few lines to stdout; no central logs, no metrics, no alerts, no audit trail for admin actions, no on-call. CERT-In requires reporting listed incidents within 6 hours and 180-day log retention in India.
- *Correction:* structured JSON logs with correlation IDs and secret redaction → CloudWatch (ap-south-2) with 180-day+ retention; CloudTrail, GuardDuty, Security Hub; alerting; incident severity matrix and CERT-In reporting decision tree; tabletop exercise. *Effort:* 2 weeks (Phase 2) + ongoing. *Verify:* alert fires in a game-day test; log retention policy in IaC.

**G-10 — One person holds every role (no separation of duties).**
- *Evidence:* one GitHub account administers all repos and approves its own pull requests (decision D3); same person holds Netlify, Neon, GitHub and future AWS root access.
- *Impact:* a single compromised account or mistake affects everything; separation of duties required by ISO 27001 and by payment partners is impossible.
- *Correction:* add at least one engineer with review rights before payments; hardware security keys + MFA on GitHub, Netlify, Neon, AWS; AWS root locked away; break-glass procedure. *Effort:* hiring + 1 day of configuration. *Verify:* access review record.

### MEDIUM
| ID | Finding | Evidence | Correction | Effort |
|---|---|---|---|---|
| G-11 | scrypt cost below current OWASP guidance | `N=16384, r=8, p=1` | Argon2id (m=19 MiB, t=2, p=1) or scrypt N=2^17; rehash on login | 1 day |
| G-12 | No Content-Security-Policy | Live headers lack CSP; pages use inline scripts | Nonce/hash CSP, report-only first | 3–5 days |
| G-13 | Rate limits are per process, in memory; lockout can be abused to lock out a victim | `express-rate-limit` memory store; 5-failure lock | Redis-backed limits; progressive delays + CAPTCHA instead of hard lock; alert on spikes | 2–3 days |
| G-14 | AI feature sends business aggregates to an external US provider | `AI_BASE_URL` default Groq; facts are aggregates (no customer personal data); question passed as quoted untrusted text | Provider DPA and data-location review; option of an India-region model (e.g. Bedrock in Mumbai if available); AI action log; output validation; keep AI read-only | 1 week |
| G-15 | DPDP Act readiness (core duties from 13 May 2027) | No privacy notice, consent records, data-principal request process, retention schedule or breach-notification process; POS stores customer names/phones | Data map, notices, consent where needed, retention/deletion jobs, request handling, breach process | 3–4 weeks |
| G-16 | Bill/GST arithmetic implemented twice | kiro-pos and kiro-backend | Shared test vectors in both CIs now; server becomes source of truth after POS sync | 2 days |
| G-17 | Supply chain: no SBOM, unsigned builds, CDN-loaded React on the website | SRI pinned; Dependabot on; actions SHA-pinned | SBOM (CycloneDX) per release, signed artifacts (Sigstore/cosign), self-host React | 3–4 days |
| G-18 | API lacks versioning, schema validation library, correlation IDs | Handlers validate manually | `/v1` prefix, JSON-schema validation, request IDs in logs and responses | 1 week |

### LOW
| ID | Finding | Correction |
|---|---|---|
| G-19 | Contact form shows "thank you" but sends nothing | Connect (Netlify Forms or API) or say clearly it is a demo |
| G-20 | KIRO LUX shows demo sellers and sample prices | Already labelled "Demo sellers… samples until confirmed"; keep until real |
| G-21 | POS blank in Firefox in CI (R-10) | Investigate with CI trace |
| G-22 | No `security.txt`; vulnerability reporting only via GitHub | Publish `/.well-known/security.txt` |

### INFORMATIONAL
- I-01 Repositories are public by owner decision D1 (no secrets found; push protection on).
- I-02 Strong existing controls: tenant-isolation HTTP tests, webhook signature/amount/merchant checks, idempotent payment state change, encrypted merchant keys (KEK, tenant-bound), CodeQL, gitleaks, rulesets, Dependabot, SHA-pinned actions, security headers.
- I-03 No public claims of certification, RBI/NPCI status or "100% secure" were found.
- I-04 The directive's "AI must not move money, change settlement accounts, refund, delete records" is already true technically: the AI feature has read-only access to aggregates; keep it so by design (see TARGET_ARCHITECTURE).

## What must not happen next (stop conditions)
1. **Do not enable KIRO PAY** until G-02, G-03, G-06, G-09 and G-10 are closed and VAPT critical/high findings are fixed and retested.
2. **Do not market UPI as "verified"** until G-01 phase 2 exists.
3. **Do not store production customer or payment data outside India.**

## Independent validation required (cannot be replaced by this document)
Legal opinion on payment-model classification and DPDP duties · written approval from the acquiring bank / PA · CERT-In-empanelled VAPT before production payments · ISO/IEC 27001 certification audit (if pursued) · SOC 2 Type II auditor (later) · chartered accountant on GST invoicing rules.
