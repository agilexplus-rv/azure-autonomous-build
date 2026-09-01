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
