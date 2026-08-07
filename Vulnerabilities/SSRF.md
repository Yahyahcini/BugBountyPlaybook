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
