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
<summary><b>🟠 GraphQL Missing Field-Level Authorization Leaking User Email</b></summary>

<br>

**Source:** [HackerOne Report #792927](https://hackerone.com/reports/792927)

---

## 🐞 Vulnerability

The `addReportParticipant` mutation exposed the invited user's email address when inviting by username.

Although creating the invitation was allowed, the response returned a sensitive field (`email`) that should have remained private.

## 🔍 Root Cause

The invitation feature was migrated from REST to GraphQL.

The REST implementation enforced an Access Control List (ACL) that hid users' email addresses, but the same field-level authorization was not implemented in the new GraphQL resolver.

## ⚔️ Exploitation

1. Find a mutation that returns sensitive objects.
2. Execute the mutation with a valid username.
3. Request sensitive fields in the response (e.g. `email`).
4. Receive data that should have been filtered.

Example:

```graphql
mutation {
  addReportParticipant(...) {
    invitation {
      email
    }
  }
}
```

## 🎯 Hunting Strategy

Look for:

- Mutations returning newly created or modified objects
- Sensitive response fields (`email`, `phone`, `token`, `role`, etc.)
- Features recently migrated from REST to GraphQL
- Missing field-level authorization on GraphQL responses

</details>

---

