# Security Guidelines

> Non-negotiable. Every PR is blocked until all checks below pass.

---

## 1. Input Validation

```
 HTTP / CLI / Queue
        │
        ▼
  ┌─────────────┐   reject   ┌─────────────┐
  │  Boundary   │──────────▶ │  400 / WARN │
  │  Validator  │            └─────────────┘
  └─────┬───────┘
        │ clean data only
        ▼
   Domain logic
```

- Validate at **every trust boundary** (HTTP handler, message consumer, CLI arg). Reject early, log at WARN.
- Never trust downstream callers. Validate again at each layer that touches external data.
- Use allowlists over denylists. Fail closed on unknown values.

---

## 2. No Secrets in Code

- **Never** commit secrets, tokens, or credentials — not even in tests or comments.
- Rotate first, then purge history (`git filter-repo` / BFG) if a secret leaks.
- Runtime config → Azure Key Vault. CI/CD → GitHub OIDC. Local dev → `az login` / `gh auth`.
- Reference by name only: `${{ secrets.AZURE_CLIENT_ID }}`. No inline values, ever.

---

## 3. AuthN / AuthZ

- Authenticate at the edge. Propagate identity via signed JWT/OIDC claims.
- Authorize **at the operation**, not just the route.
- **Default-deny**: new endpoints are inaccessible until explicitly permitted in policy.

---

## 4. Data Access

- Parameterize every query. No string concatenation into SQL. Ever.
  ```python
  # ✅  cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
  # ❌  cursor.execute(f"SELECT * FROM users WHERE id = {user_id}")
  ```
- Encode output for its destination context (HTML → escape, JSON → encode, Shell → quote).

---

## 5. Cryptography

- Use vetted libraries. **Never roll your own crypto.**
- Passwords: Argon2id (preferred) or bcrypt cost ≥ 12. Never MD5, SHA-1, or unsalted SHA-256.
- TLS 1.2 minimum (1.3 preferred). Disable RC4, 3DES, NULL ciphers.

---

## 6. Logging

- Never log secrets, tokens, full PII, or full request bodies.
- Mask pattern: `email=a***@d***.com`, `pan=****1234`.
- Structured JSON logs. Include: `correlation_id`, `user_id` (hashed), `operation`, `latency_ms`, `outcome`.
- Stack traces → logs only. API responses → generic message + `correlation_id`.

---

## 7. Dependencies

- Pin to exact versions in production. No `latest`, no unbounded `^`.
- Renovate / Dependabot enabled; security alerts auto-PR'd.
- New dependency = one-line justification in the PR: why this lib, why now, what was ruled out.

---

## PR Gate Checklist

- [ ] `gitleaks` scan clean (no secrets in diff)
- [ ] All new HTTP handlers have authN + authZ
- [ ] All new DB queries are parameterized
- [ ] New dependencies justified in PR description
- [ ] Logs masked for PII
