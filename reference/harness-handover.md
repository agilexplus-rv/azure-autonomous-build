# Harness handover — passing a live build to another tool mid-flight

Three handovers exist in this method, and conflating them is how enforcement gets lost:

| Handover | Reader | Where |
|---|---|---|
| **Cold start** | the next session of the same setup | `orchestration.md` — state file, `Resume by saying…` |
| **Harness** | a different tool or model continuing the build | this file |
| **Custody** | the team who owns the system after you leave | `handover.md` |

A harness handover is the moment the method stops being enforced by the thing that wrote it. If
the rules are not carried across explicitly, they do not cross.

## What travels

- **The adapter**, installed in the receiving harness — `adapters/` exists for exactly this.
  Content is canonical in `SKILL.md` and `reference/`; the adapter points at it rather than
  copying it, because two copies drift and drifting guards is the failure this method is about.
- **The operational contract**: the PR loop, the exact name of the required aggregating check
  (renaming it blocks every PR in the repository), the deploy command that must be used instead
  of the raw cloud CLI, route registration, column-scoped grants, never committing to main.
- **The connections**, each with its access model: source control, DNS, cloud tenant, payments,
  email. Say which are federated, which are tokens, and which the receiving harness must not
  hold at all.
- **The record**: state file, decision register, traceability, and every gate verdict so far.
- **The never-delegate list, in writing.**

## The never-delegate list

These categories do not move to a cheaper model or a less-trusted harness just because the build
did:

> authentication and authorisation · money · the malware-scanning pipeline · backup and restore ·
> database grants · retention, erasure and anything the DPIA names · publication and irreversible
> transitions · **and the infrastructure-as-code that provisions any of them**

**A harness switch does not lower the consequence of a mistake, so it does not lower the tier.**
If the receiving harness must touch one of these, the work comes back for a gate by the strongest
model before merge. Building it there is not the same as gating it there — and a gate is cheaper
than the incident, every time.

## The receiving harness proves the contract before it builds

Its first PR is a trivial change taken through the whole loop: branch, CI, aggregating check
green, squash merge, deploy, and the change visible on the live estate. Feature work starts after
that PR lands, not before. A harness that cannot complete the loop on a one-line change will not
complete it on a migration.

## Handing back

Work returns for a stronger-model gate when it touches the never-delegate list, when a gate
returns `goalMet: false` twice in the receiving harness, or at a phase boundary whose gate weight
is full. Handing back is a normal event in this method, not an escalation — say so, or it will be
avoided.

## Checklist at the boundary

Neither side signs this off alone:

- [ ] `main` is green and the deployed estate serves the real journey
- [ ] no blocker-severity findings open; every UNVERIFIED area named with what would settle it
- [ ] state file current, ending with the continuation prompt (`loop.md`)
- [ ] tier map recorded **with model and harness named**, not just tier names
- [ ] never-delegate list written into the receiving project's own rules file
- [ ] adapter installed and the canonical `reference/` tree reachable from it
- [ ] connections established and proved by one round trip each
- [ ] the receiving harness's contract PR merged and deployed
- [ ] what Pass 1 did **not** finish, listed as work the receiving harness may not do alone
