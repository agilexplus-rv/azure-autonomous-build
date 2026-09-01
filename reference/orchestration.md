# Orchestration — the sprint script

A sprint is a script, not a conversation. Whatever harness you run in, the shape is the same;
only the spawn primitive differs.

## Skeleton

```js
export const meta = { name, description, phases: [...] }

const COMMON = `...operational contract, scope discipline, house style...`

// 1. BUILD — parallel where the dependency graph allows.
const first = await parallel([
  () => agent(SLICES[0].prompt, { label, phase: 'Build', isolation: 'worktree' }),
  () => agent(SLICES[1].prompt, { label, phase: 'Build', isolation: 'worktree' }),
])
const third  = await agent(SLICES[2].prompt, {...})   // depends on one of the above
const fourth = await agent(SLICES[3].prompt, {...})   // OWNS THE INTEGRATION POINT — runs LAST

// 2. DEPLOY — once, before the gate, carrying the deployment traps in its prompt.
const deployed = await agent(deployPrompt, { phase: 'Deploy' })

// 3. GATE — N verifiers over DISJOINT areas, then a synthesiser.
const reports = await parallel(VERIFIERS.map(v => () =>
  agent(v.prompt, { phase: 'Gate', isolation: 'worktree' })))

const missing = VERIFIERS.filter((v, i) => !reports[i]).map(v => v.key)
if (missing.length) log(`WARNING: no report from ${missing.join(', ')} — UNVERIFIED, not passing`)

const verdict = await agent(synthesisPrompt(reports, missing), { schema: GATE_SCHEMA })
```

Four things in that skeleton are not decoration:

1. **Worktree isolation on every agent.** A shared tree corrupted in-flight work twice.
2. **The integration-point slice runs LAST**, against merged main, and walks the real journey.
3. **A missing report is UNVERIFIED.** Silence must not read as success.
4. **A structured verdict**, so "done" is a value rather than a vibe.

## What every slice prompt must carry

```
OPERATIONAL CONTRACT
  1. Branch off up-to-date main. Commit. Push. PR. Wait for CI. Merge ONLY when green.
     Squash only. Never commit to main.
  2. Never rename the required aggregating check — it blocks every PR in the repo.
  3. If the PR reads BEHIND, update the branch once and wait for CI. If it goes BEHIND again
     because a sibling merged, WAIT rather than rebasing into a moving target.
  4. Every new route registers in the feature/release gate, or it is a dead 404.
  5. Every new DB write needs its COLUMN-SCOPED grant in a migration.
  6. Do NOT deploy. A dedicated later step deploys once, before the gate.
  7. NEVER judge a command by a piped exit status. Verify the EFFECT.
  8. You are in your OWN worktree — install dependencies there rather than reaching into
     another checkout.

SCOPE DISCIPLINE (measured, not advisory)
  - PREFER A MAINTAINED LIBRARY. Hand-rolling anything security-sensitive (parsers, crypto,
    archive/image formats) needs a stated reason in the PR body.
  - Cap the slice at ~1,500 lines. Past that, split into parallel slices — which is FASTER,
    because they then run concurrently.
  - Build the SMALLEST thing that satisfies the requirement. The requirement is the bar, not
    the ceiling. Thoroughness belongs in the tests and the gates.

YOU OWN (touch nothing else): <explicit file list, including the pre-assigned migration number>
```

The scope rules exist because of measurement, not taste. One slice shipped **6,621 insertions
across 104 files**. Another hand-wrote an XLSX codec — 515 lines of writer, 635 of reader — because
it over-applied a "no external dependencies" requirement that was about the landing page not
fetching third-party *runtime* assets, and said nothing about build-time dependencies. Say that
distinction explicitly; agents get it wrong.

## The written record — three files, three jobs

| File | Written by | Holds |
|---|---|---|
| progress/state | **the gate only** | checkpoint for a cold start, decision register, specification gaps, findings log |
| traceability | each slice, **its own IDs only** | requirement → files → at least one test |
| efficiency log | whoever learns the lesson | measured fixes with before/after — not a diary |

The checkpoint is written assuming the reader has no memory of the session, and ends with a
literal "Resume by saying…" line. That is what makes an interrupted run recoverable rather than
archaeological.
