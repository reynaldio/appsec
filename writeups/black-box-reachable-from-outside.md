# Black-box on purpose: what "reachable from outside" has to mean

By [Reynaldi Oeoen](https://www.linkedin.com/in/reynaldio) · authorized application-security testing

A note on a deliberate constraint: keeping the attacking side of a test
**black-box** — no source, no white-box toggle — so that every finding it
produces means *exploitable from outside* with no asterisk. No customer, target,
or engagement is named.

## The asterisk problem

Give an offensive tester the source code and their findings get faster and more
numerous — and also more ambiguous. "This function concatenates user input into a
query" is a true statement about the code that may or may not correspond to a
*reachable* vulnerability. Is the function called on a path an outsider can
trigger? Is the input actually attacker-controlled by the time it arrives? Is
there a filter three layers up that the code reading didn't account for?

A finding derived from reading code carries an implicit asterisk: *reachable, if
the surrounding conditions hold.* Sometimes they do; often the reachability
analysis is the hard part and gets hand-waved. The result is a report that mixes
"an attacker can do this" with "the code contains a pattern that could be a
problem," and the reader can't tell which is which without redoing the work.

## The constraint that removes the asterisk

Keep the attacking side **black-box**: its only surfaces are what an outside
attacker actually has — the web app, the mobile app's API, an API spec. No source
access, and crucially **no white-box toggle** that a busy operator would flip
"just to go faster." The constraint is structural, not a setting.

The payoff is a guarantee about what a finding *means*. If the black-box side
reports a vulnerability, it reports it because it **reached and triggered the
behavior from the outside** — it sent the request, it observed the effect. There
is no "in theory" path. Every finding is, by construction, reachable-from-outside,
because reaching it from outside is the only way the finding could have been
produced. The asterisk is gone.

## Where code analysis still belongs — the other side

None of this says code analysis is worthless — it says it belongs to a *different*
role with a *different* claim. Reading source is how you find the things black-box
testing structurally can't reach: a secret committed to git history and removed at
HEAD, a vulnerable dependency whose reachable symbol is never actually called, a
route registered only under a config flag, a weak crypto choice. Those are real
and worth finding.

But they're a distinct tier of claim — "present in the code" — and they shouldn't
be laundered into "exploitable." The clean design is a division of labor:

- The **black-box side** owns "reachable from outside," and its findings never
  carry an asterisk.
- The **code-reading side** owns "present in the code," and it can *propose* that
  one of its findings is exploitable — but promoting it to *proven exploitable*
  requires the black-box side to actually reach it from outside.

That promotion gate is what keeps the two claim types from blurring. A code
finding stays "present" until something that only has the outside view manages to
trigger it. Then, and only then, it's "exploited."

## Why the separation is worth the cost

Black-box is slower and finds fewer things than white-box. You accept that on
purpose, because the thing you're buying is *interpretability of the output*: a
reader can trust the reachability tier without redoing it. A report where
"exploitable" always means exploitable is worth more per finding than a longer
report where the reader has to re-derive which findings are real. Same principle
as [confident noise](confident-noise-independent-reproduction.md) — the value is
in what the tiers *guarantee*, not in the raw count.

## The rule

> If your tester can read the code, some of its "vulnerabilities" are really
> "patterns that might be reachable." Keep the attacking side black-box and every
> finding it files means *reachable from outside*, no asterisk. Let code analysis
> own a separate, honestly-labeled tier — and make it earn the word "exploitable"
> by being reached from the outside.

## References

- Black-box vs. white-box testing — the reachability gap
- OWASP — distinguishing reachable exploitability from code-level presence
- Reachability analysis in SAST/SCA (why "vulnerable package present" ≠
  "vulnerable symbol called")
