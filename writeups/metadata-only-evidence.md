# Proving impact without exfiltration: metadata-only evidence

By [Reynaldi Oeoen](https://www.linkedin.com/in/reynaldio) · authorized application-security testing

A note on how I evidence data-exposure findings — SQL injection, exposed
databases, public cloud buckets, path traversal — **without pulling the
underlying data**. The goal is a finding a defender can act on and an auditor can
trust, that never puts me in possession of someone's records. No customer,
target, or engagement is named; all examples are generic.

## The problem with "proof by exfiltration"

The instinctive way to prove a data-exposure bug is to pull the data: dump the
`users` table, download the bucket, read `/etc/passwd`, screenshot a row of
PII. It's vivid and it convinces people. It's also the wrong thing to do on an
authorized engagement:

- **It creates a second breach.** Now a copy of the victim's data lives in your
  notes, your ticket system, your screenshots. You have become a place that data
  can leak from.
- **It's often out of scope.** "You may test for vulnerabilities" is not "you may
  read our customers' records." Many programs draw exactly this line.
- **It doesn't prove more than metadata does.** For triage, the defender needs to
  know *that* a boundary failed and *how far* it reaches — not the contents.

So the discipline I hold is: **prove the vulnerability with the smallest
observation that establishes impact, and stop there.** For data exposure, that
observation is almost always metadata, not data.

## What "enough proof" looks like, by class

The pattern is the same each time — capture the *shape and reach* of the
exposure, never a value.

| Class | Sufficient evidence | What I deliberately do **not** capture |
|---|---|---|
| SQL injection | schema/table/column **names**, row **counts**, DB version — proving the parameter is injectable and the data is reachable | any actual row value |
| Exposed database (no/default creds) | database and collection **names**, document **counts**, server version banner | any stored document's contents |
| Public cloud bucket | object **key names**, object **count**, total **bytes** | the body of any object |
| Path traversal / LFI | an **oracle signature** (the response difference that only a successful traversal produces) + the **depth** reached | the contents of the file read |

In each row, the left column is decisive: an injectable parameter that returns
the real table names and a row count *is* a proven SQLi — the defender does not
need the rows to know they must fix it. The right column is what turns a finding
into a liability.

## The path-traversal case, concretely

Path traversal is the sharpest example, because the tempting proof is literally
"here are the contents of `/etc/passwd`." You don't need them. A traversal is
proven by an **oracle**: craft two requests that differ only in the traversal
payload, and show that the vulnerable one produces a response the control one
cannot — a distinctive length, a status flip, a signature string that only
exists if the server resolved `../` outside the intended directory. Record:

- the exact payload and the control request,
- the response *signature* that distinguishes them (length / status / a
  structural marker), and
- the **depth** the payload reached.

That triple proves the parameter reads files outside its jail, at a known depth,
without ever storing the file. The finding is complete; no file content is in
evidence.

## Why metadata is *more* auditable, not less

There's a worry that metadata-only proof is weaker. In practice it's stronger for
the thing evidence is for — being replayed and believed later:

- **A recorded execution beats a narration.** The evidence is the exact command
  or request sent, the identity it was sent under, the response status, and a
  **hash of the response body** — not my prose describing what I saw. A reader
  months later can re-run it and get the same hash.
- **Reproduced, not asserted.** A finding isn't treated as real until it's
  reproduced independently from a clean state. Metadata reproduces cleanly; a
  one-time data dump does not.
- **It survives handoff.** An auditor or the customer's own team can verify a
  metadata-plus-hash finding without me handing them a spreadsheet of their own
  leaked data.

## The rule

> Capture the smallest observation that proves impact, prove it with a recorded
> and reproducible execution, and never take custody of data you were only meant
> to test the protection of.

This is testing done against systems I'm authorized to test, inside an agreed
scope, with the same rate-limiting and scope-allowlisting as any active work.
The metadata-only bar isn't a limitation I work around — it's the point.

## References

- OWASP Web Security Testing Guide — evidence handling and minimizing data touched
- OWASP API Security Top 10 — API3 (excessive data exposure), API1 (BOLA)
- CWE-22 (path traversal), CWE-89 (SQL injection), CWE-200 (exposure of sensitive
  information)
