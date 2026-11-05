# Incident response

1. **Contain**: revoke or rotate the affected secret, switch off the affected service or feature (e.g. leave payments disabled), protect evidence.
2. **Assess**: what data, which shops, since when (Neon history, Netlify/Render logs, GitHub audit log).
3. **Fix**: patch in a `security/*` branch with a pull request; add a test that would have caught it.
4. **Recover**: redeploy the fixed tag; restore data from a Neon branch if needed.
5. **Record** below: date, what happened, impact, fix, follow-ups. Inform affected shops when their data was involved.

## Incident log
| Date | Summary | Impact | Fix | Follow-up |
|---|---|---|---|---|
| — | No incidents recorded | — | — | — |
