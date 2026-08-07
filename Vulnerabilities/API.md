# 🛡️ API Vulnerabilities

> A collection of API security research notes, attack patterns, bypass techniques, and lessons learned from public bug bounty reports.

---

## 📚 Topics Covered

| Category | Description |
|---|---|
| 🔑 Broken Authentication | Weak, missing, or bypassable API authentication |
| 🚪 Broken Object Level Authorization (BOLA/IDOR) | Accessing objects belonging to other users |
| 🧩 Broken Function Level Authorization | Reaching privileged endpoints without proper role checks |
| 📦 Mass Assignment | Overwriting protected fields through unexpected parameters |
| 🌊 Rate Limiting & Resource Abuse | Missing throttling leading to brute force or DoS |
| 🕳️ Excessive Data Exposure | APIs returning more data than the client needs |
| 🔀 Improper Input Validation | Injection and logic flaws through unchecked input |
| 🧬 GraphQL-Specific Issues | Introspection, batching, nested query abuse |
| 🗝️ API Key & Token Leakage | Exposed keys, predictable tokens, weak JWTs |
| 🔄 Improper Asset Management | Shadow, zombie, and undocumented API endpoints |
| 📡 Detection Methods | Recon, fuzzing, and tooling for API-specific testing |
| 🔐 Impact Analysis | Data exposure, account takeover, privilege escalation |

---
