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
