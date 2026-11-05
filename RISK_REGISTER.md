# KIRO risk register

Severity: P0 critical · P1 high · P2 medium · P3 low. Status as of 2026-11-05. Source = how it was found.

| ID | Risk | Sev | Source | Mitigation / plan | Status |
|---|---|---|---|---|---|
| R-01 | Only one person can review changes; GitHub forbids self-approval, so "required review" is not enforced | P1 | Governance inspection | PR + passing checks required; owner self-merges (D3). Add a second reviewer before live payments | Accepted until a 2nd developer joins |
| R-02 | All repositories are public (D1): business logic and payment code are readable | P2 | Owner decision | No secrets in code (push protection, gitleaks, QA secret scan); keys only in Netlify/Render/Neon | Accepted |
| R-03 | Core API auth is marked "PLACEHOLDER AUTH" (HS256 JWT, shared secret) | P2 | Code review | Tested (forged, `alg:none`, expired, tampered tokens refused). Review / rotate design before production API launch | Open |
| R-04 | Bill/GST arithmetic exists in kiro-pos (offline) and kiro-backend | P2 | Code map | Shared test vectors run in both repos' CI; long term: POS syncs bills to the API | Open |
| R-05 | Token verification code is copied into kiro-payments (ADR-0002) | P3 | Repository split | Same test vectors in both repos; replace with a versioned package | Open |
| R-06 | No PostgreSQL row-level security; tenant isolation is in queries | P2 | Audit | HTTP tests prove Business B cannot read or change Business A; consider RLS before many tenants | Open |
| R-07 | No Content-Security-Policy on the website | P3 | QA security scan | Pages use inline scripts and embedded images; plan a nonce-based CSP | Open |
| R-08 | Payments could be switched on without approval | P1 | Directive | Disabled by default (ADR-0004); enabling needs approval record here | Mitigated |
| R-09 | Free Neon and Render tiers sleep / limit connections | P2 | Load test | Paid tiers before real shops; pool size documented | Open |
| R-10 | Firefox not verified on the owner's PC (missing Visual C++ runtime); weekly CI found the POS app blank in Firefox | P2 | QA | Investigate in CI with traces | Open |
| R-11 | POS marks UPI paid on the cashier's confirmation (fake screenshot / wrong amount / insider fraud) | P0 | Gap assessment G-01 | Interim labelling + UTR + day-end report (Phase 0); verified dynamic QR (Phase 5) | Open |
| R-12 | Production database region is Singapore; payment data must stay in India | P0 before payments | Gap assessment G-02 | Move production to AWS India before any payment data (Phase 2) | Open |
| R-13 | No central logs / alerts; CERT-In 6-hour reporting and 180-day India logs not yet possible | P1 | Gap assessment G-09 | Phase 2 observability | Open |

Full list of findings G-01…G-22 with evidence: [ENTERPRISE_GAP_ASSESSMENT.md](ENTERPRISE_GAP_ASSESSMENT.md). Regulatory mapping: [COMPLIANCE_REGISTER.md](COMPLIANCE_REGISTER.md).

Update this file in a pull request whenever a risk is found, changes or is closed.
