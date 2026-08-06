# 🛡️ GraphQL Security

> A collection of GraphQL vulnerability research notes, attack patterns, and lessons learned from public bug bounty reports.

---

## 📚 Topics Covered

| Category | Description |
|---|---|
| 🔐 Authorization | IDOR, BOLA, privilege escalation |
| 🔑 Authentication | Session and access control issues |
| 📂 Data Exposure | Sensitive information leaks |
| 🔎 Query Abuse | Query manipulation and unexpected behavior |
| ⚙️ Mutation Abuse | Unauthorized state changes |
| 🧬 GraphQL Internals | Introspection, schemas, resolvers |

---

# 📝 Vulnerability Notes

---

<details>
<summary><b>🔴 GraphQL Connection Field Bypass Leading to Information Disclosure</b></summary>

<br>

**Source:** [HackerOne Report #489146](https://hackerone.com/reports/489146)

---

## 🐞 Vulnerability

Sensitive user and program metadata was exposed through the GraphQL `nodes` field.

The same objects were protected when accessed through:

```graphql
edges {
  node {
    ...
  }
}
```
but exposed when accessed through:
```
nodes {
  ...
}
```
one path is filtered(protected) and one is not.

## 🔍 Root Cause

A GraphQL library migration introduced the `nodes` field automatically on connections.

The new resolver returned objects through a different execution path where attribute-level authorization was not applied.

## ⚔️ Exploitation

1. Identify GraphQL connection objects.
2. Compare `edges { node }` with direct `nodes` access.
3. Query sensitive fields through `nodes`.
4. Retrieve data that should have been filtered.

## 🎯 Hunting Strategy

Look for:

- GraphQL connection objects (`users`, `teams`, `reports`, etc.)
- Alternative ways to access objects:
  - `nodes`
  - `edges { node }`
- Auto-generated GraphQL fields
- Schema changes after migrations
- Sensitive fields exposed without field-level authorization

</details>

---

<details>
<summary><b>🟠 GraphQL Field-Level Authorization Bypass</b></summary>

<br>

**Source:** [HackerOne Report #792927](https://hackerone.com/reports/792927)

---

## 🐞 Vulnerability

GraphQL allowed users to access internal fields that were not intended to be visible to them.

The operation itself was authorized, but sensitive fields inside the returned object were missing proper access control.

## 🔍 Root Cause

The application checked permission for accessing the object/action but failed to restrict specific sensitive fields.

Commonly affected fields:

- Emails
- Phone numbers
- Internal metadata
- Tokens
- Private information

## ⚔️ Exploitation

Request additional fields in GraphQL queries or mutations and check if restricted data is returned.

Example:

```graphql
query {
  object {
    public_field
    internal_field
  }
}
```

## 🎯 Hunting Strategy

Look for:

- Objects containing sensitive fields
- Mutations returning full objects
- Fields that reveal internal/private data
- Differences between user roles and accessible fields


</details>

---

<details>
<summary><b>🔴 GraphQL IDOR Through Missing Object Authorization</b></summary>

<br>

**Source:** [HackerOne Report #2122671](https://hackerone.com/reports/2122671)

---

## 🐞 Vulnerability

GraphQL mutations allowed users to modify or delete other users' objects by changing object identifiers.

The application checked that the request was valid but did not verify that the object belonged to the current user.

## 🔍 Root Cause

Missing object-level authorization (BOLA/IDOR).

The server trusted user-controlled IDs without checking ownership.

Example:
Original:
certification_id = 123

Changed:
certification_id = another_user_object_id (like : = 124)


The server processed the action on another user's object.

## ⚔️ Exploitation

1. Perform an action on your own object.
2. Capture the GraphQL request.
3. Find object identifiers.
4. Replace the ID with another object's ID.
5. Check if you can modify or delete data.

## 🎯 Hunting Strategy

Look for:

- GraphQL mutations containing IDs:
  - `id`
  - `userId`
  - `objectId`
  - `nodeId`
- Actions:
  - update
  - delete
  - archive
  - restore
- User-owned objects:
  - profiles
  - documents
  - certifications
  - settings
  - reports

</details>

---


<details>
<summary><b>🟡 GraphQL SSRF Through User-Controlled URL Parameter</b></summary>

<br>

**Source:** [HackerOne Report #1864188](https://hackerone.com/reports/1864188)

---

## 🐞 Vulnerability

A GraphQL query accepted a user-controlled URL parameter (`source`) and used it to perform server-side HTTP requests.

An attacker could provide an arbitrary URL, causing the application server to send requests to external or internal destinations, resulting in Server-Side Request Forgery (SSRF).

Example vulnerable query:

```graphql
query {
  allTicks(
    symbol: "TSLA",
    source: "https://attacker.com"
  ) {
    symbol
    server
    source
    ask
    time
    bid
  }
}
```

---

## 🔍 Root Cause

The backend trusted a user-controlled GraphQL parameter as the destination of an HTTP request.

The application expected a trusted source value but failed to restrict or validate the input.

Example:

Original:

```
source = "trusted_source"
```

Changed:

```
source = "https://attacker.com"
```

The server processed the attacker-controlled URL and sent the request.

---

## ⚔️ Exploitation

1. Identify GraphQL queries or mutations with parameters that may trigger server-side requests.
2. Replace the value with a Burp Collaborator or Interactsh URL.
3. Monitor for DNS or HTTP interactions.
4. Test internal destinations such as:
   - localhost
   - private IP addresses
   - internal hostnames

---

## 🎯 Hunting Strategy

Look for GraphQL parameters that may control server-side requests:

- `url`
- `uri`
- `source`
- `endpoint`
- `callback`
- `webhook`
- `proxy`
- `redirect`
- `image`
- `import`

Focus on features that:

- Fetch external resources
- Import data from URLs
- Process remote files
- Communicate with external services

---

## 🛡️ Prevention

- Never allow arbitrary URLs from user input.
- Use allowlists for trusted destinations.
- Validate protocol, hostname, and IP ranges.
- Prefer predefined values (enums) instead of user-controlled URLs.

</details>

---

<details>
<summary><b>🟡 GraphQL IDOR Through Missing Invoice Authorization</b></summary>

<br>

**Source:** [HackerOne Report #2207248](https://hackerone.com/reports/2207248)

---

## 🐞 Vulnerability

GraphQL operations `BillDetails` and `BillingDocumentDownload` allowed users to access other shops' invoices by changing the `BillingInvoice` ID.

The application verified that the invoice existed but failed to verify that the invoice belonged to the current user/shop.

---

## 🔍 Root Cause

Missing object-level authorization (IDOR / BOLA).

The GraphQL resolver trusted a user-controlled object ID:

```graphql
node(id:$id)
```

without checking ownership.

Example:

Normal request:

```
invoice_id = 123
```

Attacker changes:

```
invoice_id = 124
```

The server returns another shop's invoice because ownership validation is missing.

---

## ⚔️ Exploitation

1. Access your own invoice.
2. Capture the GraphQL request.
3. Identify object identifiers:
   - id
   - invoiceId
   - node(id:)
4. Replace your invoice ID with another valid ID.
5. Check if another user's data is returned.

Affected operations:

```graphql
BillDetails
BillingDocumentDownload
```

---

## 🎯 Impact

Exposed sensitive billing information:

- Email addresses
- Full addresses
- Invoice details
- Payment information
  - Last 4 card digits
  - Card type
  - PayPal email

---

## 🎯 Hunting Strategy

Look for GraphQL objects containing IDs:

```graphql
node(id:)
```

```graphql
object(id:)
```

```graphql
mutation(input:{id})
```

Test:

- Incrementing IDs
- UUIDs
- Global IDs
- Base64 encoded IDs

Focus on sensitive objects:

- invoices
- payments
- documents
- reports
- accounts
- subscriptions

Always test:

"Can I access another user's object by changing only the identifier?"

</details>

---

<details>
<summary><b>🟡 GraphQL Field-Level Authorization Bypass Leading to Information Disclosure</b></summary>

<br>

**Source:** [HackerOne Report #707433](https://hackerone.com/reports/707433)

---

## 🐞 Vulnerability

A GraphQL query exposed the `payment_transactions` field of programs to unauthorized users.

The field contained information that should only be accessible by program team members, but external users could query it directly.

Example:

```graphql
query Team_mini_profile {
  team(handle:"program") {
    id
    name
    payment_transactions {
      total_count
    }
  }
}
```

Response:

```json
{
  "payment_transactions": {
    "total_count": 9
  }
}
```

---

## 🔍 Root Cause

Missing field-level authorization.

The application protected access to sensitive data incorrectly.

The `team` object was publicly accessible, but sensitive fields inside the object were not restricted.

Example:

```
Team
 |
 |-- name ✅ public
 |
 |-- about ✅ public
 |
 |-- payment_transactions ❌ should be restricted
```

The server checked:

```
Can user access Team?
```

but failed to check:

```
Can user access payment_transactions?
```

---

## ⚔️ Exploitation

1. Find public GraphQL objects.
2. Inspect available fields.
3. Add sensitive fields manually.
4. Check if restricted information is returned.

Example:

Normal query:

```graphql
team(handle:"program"){
    name
}
```

Modified query:

```graphql
team(handle:"program"){
    name
    payment_transactions{
        total_count
    }
}
```

---

## 🎯 Impact

Unauthorized users could obtain private program information:

- Payment transaction count
- Information about program activity

The exposed field could also reveal whether a program was involved in private activity.

---

## 🎯 Hunting Strategy

Look for sensitive GraphQL fields:

```
payment
billing
transactions
members
emails
private
internal
admin
permissions
```

Test:

- Public objects with hidden fields
- Fields requiring higher privileges
- GraphQL queries from frontend applications
- Fields accessible without authentication

Main question:

"Can I access this field even though I cannot access this type of information normally?"

</details>

---
<details>
<summary><b>🔴 SQL Injection Through GraphQL Endpoint Parameter</b></summary>

<br>

**Source:** [HackerOne Report #435066](https://hackerone.com/reports/435066)

---

## 🐞 Vulnerability

A SQL Injection vulnerability existed on the **GraphQL endpoint**, but **not inside the GraphQL query itself**.

The endpoint accepted a user-controlled HTTP parameter:

```http
POST /graphql?embedded_submission_form_uuid=<value>
```

Instead of safely handling this parameter, the backend inserted it directly into an SQL statement, allowing arbitrary SQL execution.

---

## 🔍 Root Cause

The backend trusted the HTTP parameter and concatenated it into SQL instead of using parameterized queries.

Vulnerable code:

```ruby
safe_query += "SET SESSION #{key} TO #{value};"
```

Because `value` came directly from user input, an attacker could inject SQL commands.

Example:

Normal value:

```
123
```

Generated SQL:

```sql
SET SESSION embedded_submission_form_uuid TO '123';
```

Injected value:

```
1'; SELECT pg_sleep(5);--
```

Generated SQL:

```sql
SET SESSION embedded_submission_form_uuid TO '1';
SELECT pg_sleep(5);
--';
```

---

## ⚔️ Exploitation

1. Find user-controlled parameters accepted by the GraphQL endpoint.
2. Inject SQL payloads.
3. Confirm execution using a time-based payload.

Example:

```http
POST /graphql?embedded_submission_form_uuid=1';SELECT pg_sleep(5);--
```

If the response is delayed by approximately 5 seconds, the SQL query executed successfully.

---

## 🎯 Impact

An attacker could execute arbitrary SQL statements on the database.

Possible impacts include:

- Reading sensitive database information
- Extracting data
- Enumerating the database
- Bypassing intended security boundaries

In this report, the injection executed against HackerOne's secure schema, reducing the impact, but SQL execution was still achieved.

---

## 🎯 Hunting Strategy

When testing GraphQL, don't focus only on GraphQL queries.

Inspect every user-controlled input associated with the endpoint:

- URL parameters
- POST parameters
- Headers
- Cookies

Then ask:

- Does this value reach SQL?
- Is it concatenated into a query?
- Can special characters (`'`, `"`, `;`, `--`) change the SQL statement?

**Key takeaway:**

A GraphQL endpoint can suffer from classic SQL Injection even when the GraphQL query itself is completely safe. Always test the entire HTTP request, not just the GraphQL operation.

</details>
