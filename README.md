# Pagination-aware GraphQL collection probing for BOLA

A technique writeup on finding Broken Object-Level Authorization (BOLA / IDOR)
in GraphQL APIs whose collection fields require pagination arguments. This is a
methodology note from my own authorized application-security testing — no
customer, target, or engagement is named, and all examples are generic.

## Background

Broken Object-Level Authorization is consistently the top item in the OWASP API
Security Top 10. The classic form is an endpoint like `GET /api/orders/{id}`
that returns any order when it should return only the caller's. GraphQL has the
same class of flaw, but the shape is different: instead of one object id in a
URL path, authorization has to hold across every field that resolves an object
or a list of objects.

Automated BOLA probing for GraphQL usually walks the introspection schema,
finds query fields that return object types, and issues each one with a
low-privilege token to see whether it leaks another tenant's data. That works
for scalar-argument lookups (`order(id: ...)`). It quietly **misses** a large
class of fields: **collection fields that reject a query unless you supply the
pagination arguments they require.**

## The gap

Many GraphQL servers model list access as a connection:

```graphql
type Query {
  orders(first: Int!, after: String): OrderConnection!
}
```

`first` is non-null. A naive prober that emits `{ orders { edges { node { id } } } }`
gets a **schema validation error** back — the request never reaches the
resolver, so the authorization behavior of `orders` is never actually tested.
The field looks "covered" in the report because it was attempted, but the
attempt bounced off validation before any object-level check ran. That is a
false negative in the worst place: a list endpoint is exactly where a BOLA flaw
leaks the most rows at once.

## The technique

Make the probe pagination-aware. For each collection field:

1. **Read the argument types from introspection.** Identify which arguments are
   non-null (`Int!`, `String!`, etc.) and must be filled for the query to
   validate.
2. **Fill required pagination arguments with valid minimal values.** `first: 1`
   (or the smallest the schema allows) is enough to get past validation and
   reach the resolver — you are testing authorization, not exfiltrating volume.
3. **Select a minimal leaf set.** Ask only for an `id` (and, where present, the
   connection's `pageInfo`) so the query is cheap and the response is easy to
   diff. Selecting heavy nested objects just makes the request more likely to
   error for unrelated reasons.
4. **Drop empty argument lists.** A zero-argument collection op must be emitted
   as `orders` with no `()` — an empty `orders()` is itself a validation error
   and reintroduces the false negative you were trying to remove.
5. **Issue the query under the low-privilege identity and compare.** If tenant A's
   token returns tenant B's object ids, that is a BOLA finding.

Concretely, the generated probe for the schema above becomes:

```graphql
query {
  orders(first: 1) {
    edges { node { id } }
    pageInfo { hasNextPage endCursor }
  }
}
```

## Why "reach the resolver" is the whole point

Authorization checks live in resolvers. Schema validation runs before any
resolver executes. So any probe that fails validation is testing the schema, not
the authorization logic — it can never observe a BOLA flaw, because the code that
would (or wouldn't) enforce the check never ran. Making the probe satisfy
validation is the difference between "we attempted the field" and "we tested the
field."

## Evidence discipline

A finding produced this way is only worth reporting if it can be reproduced and
shown, not asserted. The bar I hold myself to:

- **The claim is backed by a recorded execution** — the exact query sent, the
  identity it was sent under, the HTTP status, and a hash of the response body.
- **The finding is reproduced independently** before it is treated as real —
  run again, from a clean state, and confirm the same cross-tenant leak.
- **Deduplicate by fingerprint, not by title** — re-running an engagement must
  update the existing finding, not file a new one each pass.

The goal is that a reader can replay the finding from the record months later
without trusting my narration of it.

## Scope and authorization

Everything above is run only against systems I am authorized to test, inside an
explicitly agreed scope. Collection probing multiplies the number of requests
sent, so it belongs behind the same rate-limiting, scope-allowlisting, and
authorization checks as any other active testing — never pointed at a target you
do not have written permission to test.

## References

- OWASP API Security Top 10 — API1:2023 Broken Object Level Authorization
- GraphQL spec — validation runs prior to execution (resolvers)
- Relay Cursor Connections specification — the `first`/`after` connection model
