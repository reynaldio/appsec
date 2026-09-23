# The floor no authorization can unlock

By [Reynaldi Oeoen](https://www.linkedin.com/in/reynaldio) · authorized application-security testing

A note on a control that sits *above* authorization: a set of destinations that
automated testing must never reach **even when the operator is authorized and the
target is in scope.** No customer, target, or engagement is named.

## Why authorization isn't the top of the stack

The usual model is a ladder: you're authorized to test a target, you define a
scope, the scope permits some destinations. It's tempting to treat the top of
that ladder — a signed authorization plus an allowlist — as the final word on
where traffic may go. It shouldn't be.

Some destinations are dangerous to send attack traffic at *regardless of who
authorized it*, because the harm isn't to the target — it's to shared
infrastructure, to third parties, or to the safety of the testing system itself:

- **Cloud metadata endpoints** (`169.254.169.254` and friends) — a request here
  from inside a target's environment is how SSRF turns into credential theft.
  Automated traffic should never generate that request on purpose.
- **Reserved and special-use address space** — loopback, link-local, and other
  ranges that don't mean what a scope rule thinks they mean.
- **Public DNS resolvers and other shared internet infrastructure** — pointing a
  scanner or a flood of probes at these harms bystanders, not the target.
- **Cloud control planes and your own testing infrastructure** — an agent must
  never be able to turn its tooling against the platform running it, or against
  the provider's control APIs.
- **Government and military domains** — off limits for automated attack traffic
  as a matter of policy, no matter what a customer's allowlist contains.

None of these should be reachable because *a customer put them in scope by
mistake, or a compromised/steered agent tried to add them.* They're a floor.

## What "a floor" means, mechanically

Three properties make it a real floor rather than another allowlist entry:

1. **It outranks every tenant allowlist and every authorization.** When the floor
   and a scope rule disagree, the floor wins. Authorization can *narrow* what's
   tested; it can never *widen* past the floor.
2. **It's compiled in, not configured.** The prohibited categories live in the
   binary. No runtime write — no API call, no database row, no prompt — can
   weaken them. The corresponding data table may only ever **add** destinations,
   never remove or override one.
3. **It matches on the right thing.** Off-limits status keys on *what a
   destination is* (reserved range, metadata IP, resolver, gov/mil domain) — not
   on the sector of whoever is testing. A bank testing its own domain must not
   trip a "financial-sector" rule, because there shouldn't be one; the floor is
   about dangerous *destinations*, not sensitive *industries*.

The one category that legitimately comes from configuration is *this deployment's
own hosts* — the platform's own API and admin hostnames, which aren't knowable at
compile time. That entry is registered once at boot and is deliberately the
weakest of the categories, precisely because it's the only one that isn't in
source.

## The SSRF example, because it's the sharpest

Metadata-endpoint blocking is where this earns its keep. An SSRF finding is proven
by showing the server *can be made to* issue a request to an address it
shouldn't — you demonstrate the redirect, the resolved internal target, the
oracle that says the request happened. You do **not** need to actually pull cloud
credentials from `169.254.169.254` to prove the bug, and doing so would mean your
tooling just fetched live credentials into your evidence store. The floor makes
that impossible by construction: the probe proves reachability; the credential
fetch is refused at the network. Impact proven, custody never taken — the same
discipline as [metadata-only evidence](metadata-only-evidence.md), enforced
structurally instead of by good intentions.

## Why this belongs in a security portfolio

Anyone can write an exploit. The harder, more mature engineering problem is
running offensive tooling *safely at scale* — making it structurally unable to
harm the wrong thing, even when an operator is careless or an agent is steered.
A compiled-in floor that no authorization can unlock is what separates
"automation that attacks" from "automation you can point at real systems
responsibly."

## The rule

> Authorization decides whether testing starts. Scope decides what's in bounds.
> But some destinations are off limits to everyone, always — and that list has to
> live where no customer, no operator, and no model can edit it.

## References

- OWASP — SSRF and cloud metadata credential theft (`169.254.169.254`)
- RFC 5735 / RFC 6890 — special-use IPv4 address registries
- Defense in depth: controls that outrank configuration
