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
