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
# 📝 Vulnerability Notes
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

---

<details>
<summary><b>🟠 Open Redirect via Firebase Dynamic Links API Key Exposure</b></summary>

<br>

**Source:** [HackerOne Report #1066410](https://hackerone.com/reports/1066410)

---

## 🐞 Vulnerability

Clario's frontend JavaScript exposed a Google Firebase Dynamic Links API key.

The exposed key allowed interaction with the URL-shortening service used by:

```
https://lnk.clario.co/
```

Due to improper URL validation on the redirect target, an attacker could generate a Clario-branded shortened link that redirected to any external website.

Attack flow:

```
Attacker-controlled URL
      |
      v
Clario Dynamic Link API
      |
      v
Malicious website
```

Because the link lived on a trusted `clario.co` domain, victims were far more likely to trust and click it.

---

## 🔍 Root Cause

The API accepted any destination URL for the dynamic link without checking it against an allow-list.

The application expected:

```
link=https://clario.co/page
```

but accepted:

```
link=https://attacker.com
```

Leaking the API key was what made this exploitable externally — anyone with the key could call the link-generation API directly, bypassing any client-side restrictions in the app's own UI.

---

## ⚔️ Exploitation

1. Search frontend JS for exposed keys:

```
AIza
firebase
googleapis
apiKey
```

2. Confirm the key works against the Dynamic Links API.

3. Generate a link pointing at an attacker-controlled domain:

```
https://attacker.com
```

4. Send the resulting `lnk.clario.co` link to a victim — it looks trusted, but redirects off-domain.

---

## 🎯 Impact

- Phishing using a trusted brand domain
- Credential theft
- Malware distribution
- Brand reputation abuse

---

## 🎯 Hunting Strategy

Look for exposed secrets in JS bundles:

```
apiKey
token
firebase
googleapis
AIza
```

And redirect-capable parameters:

```
url
redirect
next
return
callback
target
link
```

Test with an external domain and see if the redirect goes through unvalidated:

```
https://attacker.com
```

Main question:

"Can I make a trusted domain redirect users to a site I control?"

Trusted redirect + weak URL validation (+ leaked API key) = Open Redirect.

</details>
---

<details>
<summary><b>🔴 Unrestricted Access to Application State API Leads to Denial of Service</b></summary>

<br>

**Source:** [HackerOne Report #993722](https://hackerone.com/reports/993722)

---

## 🐞 Vulnerability

A missing authorization check existed on the PlayStation REST API endpoint:

```
PUT /api/application/state
```

This endpoint let the caller modify the application's state via a user-controlled JSON body:

```json
{
  "appState": "quiesce"
}
```

The `quiesce` state placed the application into an unavailable condition. Since the endpoint didn't verify the requester's privileges, any unauthenticated attacker could flip the app into this state and take the service down.

Attack flow:

```
Unauthenticated Attacker
      |
      v
PUT /api/application/state
      |
      v
Application Disabled
```

---

## 🔍 Root Cause

A sensitive, state-changing function was exposed with no authorization check — the API assumed only admins would know about or call it, rather than enforcing that at the server.

```
Any User
   |
   v
State Management API
   |
   v
Application Configuration Changed
```

---

## ⚔️ Exploitation

1. Find an API endpoint responsible for application state:

```
/api/application/state
```

2. Change the method from `GET` to `PUT`.

3. Send:

```
PUT /api/application/state HTTP/1.1
Content-Type: application/json

{
  "appState": "quiesce"
}
```

4. No privilege check is performed — the state changes immediately.

5. The application starts returning:

```
502 Bad Gateway
```

---

## 🔗 Attack Chain

```
Missing Authorization
      |
      v
Access Sensitive API Function
      |
      v
Change Application State
      |
      v
Denial of Service
```

---

## 🎯 Impact

- Disable application availability
- Trigger downtime with zero authentication
- Disrupt service for all legitimate users, from a single request

---

## 🎯 Hunting Strategy

Look for endpoints that perform sensitive actions:

```
/state
/status
/config
/settings
/admin
/maintenance
/shutdown
/disable
```

Test:

- Hidden/undocumented endpoints
- Different HTTP methods on the same path (`GET`, `POST`, `PUT`, `DELETE`)
- Admin-sounding functionality without an auth header
- Whether role/permission is actually enforced server-side, not just hidden client-side

Common parameters:

```
state
status
action
role
permission
enabled
config
```

Main question:

"Can I call an administrative function, or change something that should only be controlled by privileged users, without having the required permissions?"

---

Sensitive function + missing authorization = **Broken Function Level Authorization (BFLA)**.

</details>
