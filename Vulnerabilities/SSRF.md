# 🛡️ Server-Side Request Forgery (SSRF)

> A collection of SSRF vulnerability research notes, attack patterns, bypass techniques, and lessons learned from public bug bounty reports.

---

## 📚 Topics Covered

| Category | Description |
|---|---|
| 🌐 Basic SSRF | Understanding server-side requests controlled by attackers |
| 🔗 URL Manipulation | SSRF through user-controlled URLs and parameters |
| 🪝 Webhooks | SSRF through callback and webhook functionality |
| 📥 File Fetching | SSRF through image, document, and resource imports |
| 🔄 Redirect Abuse | Using redirects to bypass SSRF protections |
| 🕵️ Blind SSRF | Detecting SSRF without seeing the response |
| ☁️ Cloud Metadata | Accessing cloud metadata services through SSRF |
| 🔀 Filter Bypass | Bypassing SSRF protections and URL validation |
| 🌍 Internal Services | Accessing internal networks and services |
| 📡 Detection Methods | Burp Collaborator, Interactsh, DNS and HTTP callbacks |
| 🔐 Impact Analysis | Internal access, data exposure, and privilege escalation |

---

# 📝 Vulnerability Notes

---

<details>
<summary><b>🔴 SSRF Leading to Cloud Metadata Access and Remote Code Execution</b></summary>

<br>

**Source:** [HackerOne Report #341876](https://hackerone.com/reports/341876)

---

## 🐞 Vulnerability

A screenshot generation feature allowed users to provide content that was processed by the server.

The server generated screenshots by visiting URLs controlled by the user.

Because the backend could make HTTP requests internally, an attacker could force the server to request internal services.

This resulted in a Server-Side Request Forgery (SSRF) vulnerability.

---

## 🔍 Root Cause

The application trusted user-controlled URLs and allowed the backend to access arbitrary destinations.

The server could communicate with internal resources that should not have been accessible externally.

Normal flow:

```
User URL
  |
  v
Screenshot Service
  |
  v
Public Website
```

Attack flow:

```
Attacker URL
  |
  v
Screenshot Service
  |
  v
Internal Services
```

The application failed to restrict access to:

- Internal IP addresses
- Cloud metadata services
- Private infrastructure

---

## ⚔️ Exploitation

1. Find a feature where the server fetches or processes a URL.

Examples:

- Screenshot generators
- URL previews
- Image fetchers
- Webhooks
- Import features

2. Replace the URL with an internal destination.

Example:

```
http://metadata.google.internal/
```

3. The server sends the request internally.

4. Retrieve sensitive information returned by the internal service.

---

## 🔗 Attack Chain

The SSRF vulnerability allowed access to cloud metadata services.

```
SSRF
  |
  v
Cloud Metadata Service
  |
  v
Cloud Credentials / Secrets
  |
  v
Kubernetes Credentials
  |
  v
Container Access
  |
  v
Remote Code Execution
```

The attacker accessed cloud metadata, retrieved Kubernetes credentials, and eventually gained root access inside containers.

---

## 🎯 Impact

Possible impacts:

- Access internal services
- Read cloud metadata
- Steal temporary credentials
- Access private APIs
- Retrieve secrets
- Reach internal infrastructure
- Achieve Remote Code Execution

---

## 🎯 Hunting Strategy

Look for features that make the server send requests.

Common parameters:

```
url
uri
link
image
source
callback
webhook
redirect
proxy
fetch
import
```

Test destinations:

```
localhost
127.0.0.1
Internal hostnames
Private IP ranges
Cloud metadata endpoints
```

Cloud metadata examples:

AWS:
```
http://169.254.169.254/
```

Google Cloud:
```
http://metadata.google.internal/
```

Azure:
```
http://169.254.169.254/metadata/
```

Main question:

"Can I make the server request something that I cannot access directly?"

</details>
