# appsec — application-security methodology notes

Technique and methodology notes from my authorized application-security testing,
by [Reynaldi Oeoen](https://www.linkedin.com/in/reynaldio). I lead cybersecurity
work and am building an AI application-security platform; these are sanitized
writeups of techniques and testing discipline from that work.

**Ground rules for everything here:**

- No customer, target, or engagement is ever named. All examples are generic.
- Everything described is run only against systems I'm authorized to test,
  inside an agreed scope, with scope enforced structurally (an egress allowlist),
  not by intention.
- Findings are backed by **recorded, reproducible executions** — a command or
  request, the identity it ran under, the response status, and a hash of the
  response — not by narration.

## Writeups

| Writeup | What it covers |
|---|---|
| [Pagination-aware GraphQL collection probing for BOLA](writeups/graphql-bola-pagination.md) | Finding Broken Object-Level Authorization in GraphQL collection fields that require pagination arguments — a class naive probers silently skip because the query fails schema validation before reaching the resolver. |
| [Proving impact without exfiltration: metadata-only evidence](writeups/metadata-only-evidence.md) | How I evidence data-exposure findings (SQLi, exposed DBs, public buckets, path traversal) with metadata and oracles — proving impact without taking custody of the underlying data. |
| [Scope is a network control, not a prompt](writeups/scope-as-a-network-control.md) | Why the boundary of what automated testing may touch has to be enforced by the network (an egress allowlist that fails closed), not by a config line or a system prompt a bug or a steered agent can move. |
| [The floor no authorization can unlock](writeups/platform-floor-off-limits.md) | A control above authorization: destinations automated testing must never reach — cloud metadata, reserved space, resolvers, gov/mil — compiled in so no allowlist or agent can widen past them. |

## Themes across the notes

- **Reach the code that enforces the check.** A probe that fails validation, or a
  scan that never authenticates, tests the wrong thing. Coverage means the
  authorization logic actually ran.
- **Prove the smallest sufficient observation.** Impact is established by shape
  and reach — names, counts, oracles, depth — not by exfiltrating data.
- **Evidence is a recorded execution, reproduced independently.** A finding you
  can't replay from a clean state isn't finished.

## Contact

[LinkedIn — Reynaldi Oeoen](https://www.linkedin.com/in/reynaldio)
