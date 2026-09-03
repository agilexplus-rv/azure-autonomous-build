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

### What the ceiling counts

State it, or the ceiling is decoration. The figure counts **code and configuration** — application
code, migrations, infrastructure templates, scripts, workflow files, catalogues. It excludes tests,
guard fixtures, generated documentation and gate transcripts, which are where thoroughness is
*supposed* to go. Every slice reports **both** numbers, the counted figure and the total, so the
gate can see the split rather than guess at it.

This is not pedantry. On one build, six of eleven slices across two sprints breached the ceiling as
originally written, and every one of them breached it on tests and transcripts. **A ceiling
breached every sprint constrains nothing** — it only trains everyone to ignore the number.

### Reserve migration numbers for the fix rounds

Pre-assigning a migration number per slice stops two agents colliding during the build. Then the
first fix round needs a schema change, and every number has already been claimed by a slice.
Reserve two per sprint, named in the sprint script beside the slice assignments. It costs nothing
and removes a collision that always arrives at the worst moment.

## Branching for parallel slices

Two levels are the obvious arrangement and the expensive one: slices merge to main, and with
up-to-date-branch protection **every merge puts every other open PR behind**, costing each a rebase
and a full CI pass. On a five-slice sprint that is paid four times over.

Three levels remove it:

| Level | Branch | Merges into | When |
|---|---|---|---|
| Slice | `slice/<phase>.<sprint>-<key>` | the phase branch | squash, on green CI, inside the sprint |
| Phase | `phase/<n>-<name>` | `main` | squash, once the **phase gate** returns `goalMet` |
| — | `main` | — | never committed to directly |

Every slice PR still runs the full contract and still waits for the aggregating check, so "a PR and
a gate for every sprint" stays literally true — but main moves once per phase instead of once per
slice, and the integration-owning slice runs last against the merged *phase* branch rather than
against a main shifting underneath it.

## The written record — three files, three jobs

| File | Written by | Holds |
|---|---|---|
| progress/state | **the gate only** | checkpoint for a cold start, decision register, specification gaps, findings log |
| traceability | each slice, **its own IDs only** | requirement → files → at least one test |
| efficiency log | whoever learns the lesson | measured fixes with before/after — not a diary |
| amendments | whoever agrees the change with the operator | **specification changes, kept apart from progress** — what changed, who agreed it, when. A spec edited in place loses the fact that it was ever different, and with it every argument already settled |

The checkpoint is written assuming the reader has no memory of the session, and ends with a
literal "Resume by saying…" line. That is what makes an interrupted run recoverable rather than
archaeological.
