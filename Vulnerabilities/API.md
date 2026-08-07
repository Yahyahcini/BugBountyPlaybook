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
<details>
<summary><b>🔴 Email Addresses Exposed in `getPersonBySlug` API</b></summary>

<br>

**Source:** [HackerOne Report #502753](https://hackerone.com/reports/502753)

---

## 🐞 Vulnerability

The `getPersonBySlug` API endpoint used by LGTM's frontend exposed the Google account email addresses of users, without any authorization check on who could request that data.

The endpoint was meant to return public profile info, but leaked account data the frontend never used.

Expected response:

```json
{
  "username": "user",
  "avatar": "image.png"
}
```

Actual response:

```json
{
  "username": "user",
  "avatar": "image.png",
  "google_email": "user@gmail.com"
}
```

---

## 🔍 Root Cause

The API trusted the frontend to control what got displayed, instead of restricting what data it returned. Since attackers can call the API directly (Burp, scripts), any field returned is exposed — regardless of what the UI shows.

```
User Request
      |
      v
getPersonBySlug API
      |
      v
Full user data object
      |
      v
Sensitive fields returned unfiltered
```

The API returned public **and** private fields (email, internal identifiers) with no distinction based on requester identity.

---

## ⚔️ Exploitation

1. Identify the endpoint:

```
getPersonBySlug
```

2. Find any username/slug belonging to another user:

```
getPersonBySlug("victim")
```

3. Call the API directly and inspect the raw JSON response:

```json
{
  "username": "victim",
  "email": "victim@gmail.com"
}
```

No authentication required — knowing the public slug is enough to pull the private email.

---

## 🔗 Attack Chain

```
Excessive Data Exposure
      |
      v
Bulk-harvest user emails via slug enumeration
      |
      v
Targeted phishing / social engineering
      |
      v
Possible account compromise
```

---

## 🎯 Impact

- Mass exposure of user email addresses
- User enumeration
- Targeted phishing potential
- Privacy violation with no auth required

---

## 🎯 Hunting Strategy

Look for endpoints returning user data:

```
/api/users/{id}
/api/profile/{username}
/api/person/{slug}
/api/account/{id}
```

Inspect the **raw response**, not just what the UI renders — sensitive fields often ship regardless:

```
email
phone
address
role / permissions
internal_id
tokens
metadata
```

Ask:

- Does this endpoint work unauthenticated?
- Does the response contain more than the frontend displays?
- Can I swap the id/slug and pull another user's data?

Main question:

"Does this API return information the client doesn't actually need?"

Excessive response data + missing authorization = information disclosure.

</details>
