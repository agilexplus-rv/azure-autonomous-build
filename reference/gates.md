# Gates — verifier splits, synthesiser, schema

## Splitting verifiers into disjoint areas

The split is by **failure mode**, not by file. Four is a good number.

**For a payments-shaped sprint:**

| Verifier | Hunts |
|---|---|
| idempotency & recovery | replayed webhook (test *concurrent* duplicates, not just sequential — idempotency in application code loses that race), lost webhook recovered **and the customer emailed**, forged signature rejected |
| state divergence | withdraw-then-pay under both interleavings, refund against a **published** record leaving it byte-identical, expiry deleting blobs |
| money values | fee from configuration not a literal, zero-fee writing a waived row and making **no** provider call, no card field in our DOM, authorisation refused **at the API** not merely hidden |
| journey, a11y, grants | the real journey with no hand-constructed URL, every new route through the gate **and** by loading it, accessibility at the smallest viewport, both locales |

**For a rendering/publication-shaped sprint:** reachability · the safety constraints (what must
never appear) · outage and failure behaviour · accessibility and locales.

## The synthesiser prompt — the parts that matter

```
Four independent verifiers ran in parallel over disjoint areas. Merge into ONE verdict.

AREAS WITH NO REPORT — treat as NOT VERIFIED, never as passing, and list them: {missing}

YOUR JOB:
1. De-duplicate; keep the strongest evidence.
2. SPOT-CHECK any blocker whose evidence looks thin — reproduce it yourself before it ships.
   A wrong blocker costs a whole fix cycle.
3. DOWNGRADE to not-verified any claimed PASS whose evidence COULD NOT HAVE FAILED:
     - a dev-mode check for a production-only defect
     - a test asserting the ABSENCE of an error string
     - a permission check run as the database owner
   This class of false pass cost Sprint X two entire fix cycles.
4. Append merged findings to the state file and ship ONE PR through the normal loop.
5. Return the structured verdict.

goalMet is TRUE only if the journey is genuinely walkable, no blocker remains,
AND no area is unverified.
```

## Verdict schema

```json
{
  "goalMet": "boolean",
  "ranOn": {"harness": "string", "model": "string", "tier": "strongest | middle | cheapest"},
  "strongestModelReview": {"required": "boolean", "ran": "boolean", "evidence": "string"},
  "journeyReachable": "boolean",
  "unverifiedAreas": ["string"],
  "largestSliceInsertions": "integer",
  "handRolledWhereLibraryExisted": ["string"],
  "findings": [{
    "requirementId": "string",
    "severity": "blocker | bug | improvement",
    "summary": "string",
    "evidence": "the exact command and its output, or the reproduction"
  }],
  "summary": "string"
}
```

`strongestModelReview` is how a back-gate stops being a habit. `required` is true whenever the
sprint touched a never-delegate category — authn/authz, money, the malware pipeline, backup and
restore, grants, retention and anything the DPIA names, publication and irreversible transitions,
or the infrastructure that provisions them. **`required` and not `ran` is a blocker-severity
finding**, not a note.

This exists because of how the discipline actually failed. On one build a second harness did the
building and the strongest model reviewed it, phase after phase, with `Fable security review`
committed each time — and then, between two phases, it simply stopped. Nothing failed and nothing
flagged; the habit lapsed. The phase that followed carried subject-access export, erasure,
retention, database restore and the audit log, and went through a product gate only. A discipline
nobody records is a discipline nobody can notice the end of.

`ranOn` is how the repository answers "what built this?" after everyone has forgotten. The plan
names tiers so it does not go stale; the verdict names the model and harness that actually ran, so
a tier-map violation is visible in the record rather than reconstructed from commit trailers.

`largestSliceInsertions` and `handRolledWhereLibraryExisted` exist so scope discipline is
**measured each sprint**, not assumed. They are how you find out a slice shipped 6,600 insertions
or hand-wrote a codec a library already provides.

## Verifiers find; they do not fix

A separate agent writes the fix prompts from the findings. The finder does not fix its own
findings, so a shallow diagnosis is not quietly patched over by whoever made it.

**Split fix rounds by area of code, not by finding.** Three findings in one area should be ONE
PR. As three PRs they each invalidate the other two — every merge puts the others behind,
costing a rebase and a full CI pass each.

## What a gate reliably catches

From the reference build, gates found: a portability claim that had only ever been a dry run; an
estate torn down leaving nothing running; a framework rename that silently killed security headers
and locale handling; a QR code being fetched from a third-party service; a logo component never
mounted; a live container running an image three merges old; five missing database grants; an
unreachable webhook; a signed-URL issuer with no caller; and a primary call-to-action pointing at
a 404.

Every one of those passed CI.

## Prove each guard can fail — with committed fixtures

"Plant a violation" is not a one-off exercise at the moment the guard is written. Commit the
planted violation: a **clean tree and a violation tree per guard**, so the proof is repeatable by
anyone and reviewable in a diff. A second build did this from its first sprint — its Sprint 0.1
goal was literally *the guards exist and each one fails on a planted violation* — and the fixtures
are what keep it true afterwards.

A guard whose violation fixture stops failing has silently stopped guarding, and that is a finding
about the guard rather than about the tree it just passed.

## Commit the verdict

The structured verdict is written into the repository, one file per sprint and one per phase, not
left in a workflow log. "Done" then has a value someone can read six weeks later, the cold-start
file can point at it, and a claim of DONE has something to be checked against. A verdict that
exists only in a chat transcript or a CI run is an opinion with a timestamp.

## The CI that carries the gate

The static contract belongs in one fast job; everything slower runs beside it rather than behind
it. A shape that survived a long build:

| Job | Carries |
|---|---|
| **fast** | lint, typecheck, build, unit and integration tests, the whole `verify` contract |
| **browser** | the journey and accessibility suites, **sharded** across a matrix — the wall-clock lever that keeps the gate affordable |
| **fidelity** | the design-fidelity check, **uploading its diffs as artifacts on failure** |
| **acceptance** | the acceptance criteria, as their own job rather than mixed into unit tests |
| **aggregator** | the required status check, `needs:` all of the above |

Two traps live in that table. The aggregator's name is the one branch protection requires, so
renaming it blocks every PR in the repository. And **a skipped job counts as passing inside it** —
so any conditional fast path is a hole the moment a PR touches something it did not expect.

Upload the evidence a failing job produces. A visual diff or a trace that only exists inside a
finished run costs whoever picks it up a full re-run to see what you already saw.
