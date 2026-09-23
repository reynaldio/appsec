# Scope is a network control, not a prompt

By [Reynaldi Oeoen](https://www.linkedin.com/in/reynaldio) · authorized application-security testing

A note on the single most important safety property when you let automation —
scanners, scripts, or an LLM-driven agent — send live traffic at a target: **the
boundary of what may be touched has to be enforced by the network, not by the
thing doing the sending.** No customer, target, or engagement is named.

## The failure mode

The convenient way to scope automated testing is to *tell* it the scope. A config
line, a system prompt, a `--target` flag: "only test `app.example.com`." This
works right up until the tool does something you didn't predict:

- a scanner follows a redirect to a third-party host,
- a crawler discovers an outbound link and queues it,
- an SSRF probe you fired resolves to an internal address,
- an LLM agent "reasons" that checking an adjacent host would be helpful,
- a typo or a templating bug sends the payload one octet off.

In every case the instruction was correct and the traffic still went somewhere it
shouldn't. Scoping-by-instruction fails *open*: when the tool misbehaves, the
packet leaves. For unauthenticated scanning that's an inconvenience. For active
exploitation against systems you don't own, it's the difference between an
authorized test and an unauthorized intrusion.

## The property you actually want

Scope should fail **closed**: if any component is wrong — buggy, confused, or
adversarially steered — the out-of-scope packet does not leave the machine. That
is only achievable if the enforcement lives *below* the tool that's sending, at a
layer the tool cannot talk its way past.

Concretely: every unit of testing runs behind an **egress proxy that allows only
the scope's destinations**, compiled from the engagement's scope rules into an
allowlist. A tool that decides to reach `8.8.8.8`, or an agent that decides to
scan the host next door, gets a **connection refused** — not a strongly worded
reminder. The scope rule is data that becomes a firewall, not text that becomes a
suggestion.

## Why this matters more with an LLM in the loop

A deterministic scanner misbehaves in bounded, familiar ways. An LLM-driven agent
is different in a way that's directly relevant here: it generates its own next
action, and it can be *steered* by the very target it's testing — a reflected
string in a response, a crafted error message, an injected instruction in a page
it fetched. If scope enforcement lives in the agent's prompt, then prompt
injection is scope escalation. If scope lives in the egress allowlist, a
successful injection changes what the agent *tries* and changes nothing about
what the network *permits*. The control is outside the blast radius of the thing
being controlled.

This is the general principle behind "the LLM is never a security boundary." The
model can be part of *deciding* what to test; it can never be the thing that
*enforces* where traffic may go.

## Layers, not a single wall

Enforcing scope at the network layer pairs with two other structural controls:

- **Sandboxed execution.** The tool that sends traffic runs in an isolated
  container — its own process, filesystem, and network namespace — so the egress
  proxy is the *only* route out. Enforcement you can bypass by opening a second
  socket isn't enforcement.
- **A floor that outranks scope.** Some destinations are off limits to *every*
  engagement no matter what an allowlist says — reserved address space, cloud
  metadata endpoints, public resolvers, infrastructure that isn't the target.
  That floor is compiled in and can only ever be added to, never weakened at
  runtime. (Its own writeup:
  [The floor no authorization can unlock](platform-floor-off-limits.md).)

Authorization decides *whether* testing may start; scope rules decide *what* is
in bounds; the network decides *what actually leaves*. Only the last one is a
control, and it's the one that has to be structural.

## The rule

> If a mistake in your tooling can send a packet out of scope, you don't have
> scope enforcement — you have a scope *preference*. Put the boundary where a
> bug, a confused model, or a hostile response cannot move it: in the network,
> failing closed.

## References

- OWASP — SSRF (why "the tool resolved somewhere unexpected" is a whole bug class)
- Prompt injection as a boundary-crossing threat for tool-using agents
- Principle of least privilege applied to egress, not just identity
