# The loop — what drives sprints in sequence

A gate produces a verdict. **Something has to consume it.** On the first reference build that
something was the operator, re-prompting by hand at every boundary: the sequencing lived in their
messages, not in the method, and so it did not survive being handed to anyone else.

This file is that engine, written down. Without it you have gates and no loop.

## The loop, as code

```js
const MAX_ROUNDS = 3
let round = 0, verdict = await gate(sprint), previous = null

while (!verdict.goalMet || verdict.unverifiedAreas.length) {
  if (++round > MAX_ROUNDS) { escalate(verdict); break }
  if (previous && noNewFindings(previous, verdict)) { escalate(verdict); break }

  // Group by AREA OF CODE, never one PR per finding: three PRs in one area
  // each put the other two behind, costing a rebase and a full CI pass each.
  const areas = groupByArea(verdict.findings.filter(f => f.severity !== 'improvement'))
  await parallel(areas.map(a => () => agent(fixPrompt(a), { isolation: 'worktree' })))

  previous = verdict
  verdict = await gate(sprint, { regressionOn: areas })   // same areas, plus what was fixed
}
```

What the code encodes, and why each line is there:

- **Rounds are bounded.** An unbounded loop with a model on both sides will run all night and
  produce a verdict nobody reads.
- **A round fixes the findings of the round before it, and nothing else.** Scope creep inside a
  fix round is how a sprint becomes a fortnight.
- **The re-gate runs the same areas plus a regression check on what was fixed.** A fix that
  breaks a neighbouring area and is never re-checked is worse than the original finding.
- **The finder never fixes its own finding** — a shallow diagnosis must not be quietly patched
  over by whoever made it.
- **Two consecutive rounds with no new findings is a stop, not a fourth round.** Record what
  remains, honestly labelled, and tell the operator.

## Terminate loudly

The loop ends in exactly one of three states, and each is written into the gate file:

| End state | What it means | What you do |
|---|---|---|
| `goalMet`, nothing unverified | the sprint is done | mark its requirements, move on |
| rounds exhausted | the loop is not converging | **stop and escalate** — say which findings survived and what you tried |
| no new findings, some open | diminishing returns | record the remainder open, at severity, with what would settle each |

Never end by quietly lowering the bar. A finding renamed "known limitation" at 3am is the finding
that surfaces in the client's first incident.

## What may be marked DONE

A requirement is DONE when a gate **that could have failed it** says so, on a squash-merged PR,
with at least one test naming the requirement. Not when the code is written. Not when CI is green.

**Never mark requirements DONE out of an ungated sprint.** On the reference build the two longest
sprints had no gate, and the ungated sprints are exactly where the unreachable features hid.

## Phase and pass boundaries

- A **phase** ends with its adversarial gate returning `goalMet`, not with its last merge.
- A **pass** ends at a written handover (`harness-handover.md` between harnesses,
  `handover.md` for custody), never at a date and never at the point where patience ran out.
- Before starting any sprint: main is green, and the deployed estate serves the journey. Starting
  a sprint on a red main means every failure that follows is ambiguous.

## The orchestrator continuation prompt

Autonomous runs get interrupted — laptops close, context compacts, sessions end. What restarts
the loop is a prompt with a fixed shape. Keep this template filled in and current in the state
file, so that resuming is copy-paste rather than archaeology:

```
Continue the autonomous <project> Pass <n> run.

READ FIRST
  <state file>  — decision register, specification gaps, checkpoint
  <plan>        — phase gate protocol, execution model, where this pass stops

STATE (as of <date>)
  Built and gated: <phases/sprints, with their gate verdict files>
  Live at <url>: what serves, and what must correctly 404
  Settled with the operator: <decisions that must not be relitigated>
  In flight: <workflow/run id> — <what it is doing>

WHEN <in-flight work> FINISHES
  1. Act on its gate. If goalMet is false, launch a remediation round for the outstanding
     blockers ONLY, then re-gate. If true, mark those requirements and record the verdict file.
  2. <publish/deploy step> so the live estate never lags main.
  3. Append any decisions taken to the register, with their reasoning.
  4. Launch <next sprint>: <one-line goal as an exit test> · tier <t> · gate <weight>.

RULES THAT COST TIME RECENTLY
  <three to six traps most recently paid for, one line each>

STOP CONDITION
  Keep going until <phase or pass boundary>. Before any sprint, verify main is green and the
  estate serves the journey. Never leave zero estates alive.
```

The "rules that cost time recently" block is not padding. It is the cheapest place to put a
lesson so that the next round does not re-buy it.

## Put the model on the record

Name **tiers** in the plan so it does not go stale — but record the **model and harness that
actually ran** each sprint, in that sprint's gate verdict.

A second build named tiers everywhere and models nowhere, for good stated reasons. The result:
weeks later nobody could answer "what built the malware-scanning template?" from the repository,
and the honest answer had to be reconstructed from commit trailers. A tier is a policy. A record
of what ran is evidence, and only evidence survives a handover.
