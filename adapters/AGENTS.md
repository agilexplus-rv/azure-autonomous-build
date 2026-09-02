# Agent instructions — autonomous builds on Azure

Canonical content lives in `azure-autonomous-build/SKILL.md` and its `reference/` tree.
**Read it before starting a sprint.** This file is a pointer plus the rules that matter most, so
it stays useful in a harness that reads only one file. Do not fork the content — two copies of the
same rules drifting apart is a failure this skill exists to prevent.

## Working method

- **Every sprint has ONE falsifiable Sprint Goal, written as an exit test.** It must be able to
  fail. "Payments are implemented with Stripe Checkout" cannot; "a replayed webhook produces
  exactly one transition and one payment row, a dropped webhook is recovered *and the customer
  emailed*, withdraw-then-pay lands in an error state that never enters the queue" can. The loop
  ends when that goal is green on a **squash-merged PR** — not when the code is written.
- **Split into 3–5 slices, each in its own worktree, each declaring the files it owns.** Pre-assign
  migration numbers per slice. Exactly one slice may edit the dependency manifest.
- **ONE slice owns the integration point** — route, mount, wiring — runs **last** against merged
  main, and its exit test walks the real journey with **no hand-constructed URL**. A component with
  no caller is the same class of defect as a stub: it passes every check and does nothing. This has
  happened four times.
- **Prefer a maintained library.** Cap a slice at ~1,500 lines. Build the smallest thing that
  satisfies the requirement — thoroughness belongs in the tests and the gates.
- **Model selection by the cost of a subtle mistake**, not difficulty. Strongest tier for schema,
  authz, money, publication, anything irreversible. Never give the cheapest tier a task whose exit
  condition is "make CI green" — it cannot persist through a diagnostic loop.

## Gates

A sprint is not done because its slices merged. Run **N parallel verifiers over disjoint areas,
then one synthesiser**. Verifiers find; they do not fix and do not open PRs.

The synthesiser must **downgrade any claimed PASS whose evidence could not have failed** — a
dev-mode check for a production-only defect, a test asserting the *absence* of an error string, a
permission check run as the database owner. A **missing report is UNVERIFIED, never a pass.**

Gate prompts must require: production builds; permission defects reproduced as the least-privileged
principal; **positive** assertions; and planted violations proving each guard *can* fail.

## Hard rules

- Never commit to main. Never merge on red or pending. Squash only.
- **Never rename the required aggregating CI check** — it blocks every PR in the repository.
- Don't merge while parallel agents hold open PRs — every merge puts the others behind.
- **Verify state, never output.** A pipeline's status is the last command's.
- Every route registers in the release/feature gate. Every DB write needs a **column-scoped** grant.
- Deploy only via the wrapper script, **always with the image parameter**, never raw cloud CLI.
  Once per phase, before each gate, never into an in-flight CI run.
- Uploads: provider-native malware scanning proven with a real EICAR file; **content sniffing, never
  extension**; short-lived permission-checked signed URLs with attachment disposition; managed
  identity, never account keys or secrets in environment variables.
- CSP: `'strict-dynamic'` with a nonce. Not `'self'`, not `'unsafe-eval'`.

## If you are continuing a build that started elsewhere

You have received this from another harness mid-flight. The rules above did not travel with the
code — they travel only because they are written here.

**Do not build these, even when asked:**

> authn/authz · money · the malware-scanning pipeline · backup and restore · database grants ·
> retention, erasure and anything the DPIA names · publication and irreversible transitions ·
> **and the infrastructure-as-code that provisions any of them**

The consequence of a subtle mistake in that list did not change when the harness did. If one must
be touched here, say so, make the change, and **send it back for a gate by the strongest model
before it merges**. Handing work back is normal in this method, not an escalation.

**Before any feature work:** take one trivial change through the whole loop — branch, CI, the
aggregating check green, squash merge, deploy, visible on the live estate. A harness that cannot
complete the loop on a one-line change will not complete it on a migration.

**Record what ran.** Every sprint's gate verdict names the harness and model that produced the
work, not only its tier. Otherwise nobody can answer "what built this?" in six weeks.

Full detail: `reference/harness-handover.md`. If the build you inherited was not run this way at
all, start with `reference/retrofit.md` — audit before you resume forward.

## When something breaks

Check `reference/failure-signatures.md` first. It maps symptoms to real causes — a CI job dying in
2 seconds with no steps, a deploy reporting success while skipping the apps, a route 404ing though
its file exists, a static check passing a file it cannot see. Each was diagnosed the slow way once
already.
