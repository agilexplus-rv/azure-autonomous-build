# azure-autonomous-build

A portable skill for building and operating production software autonomously on Azure, with
Cloudflare DNS and Stripe payments — covering the orchestration model, the adversarial gate
system, the CI contract that makes "no stubs, no stales" mechanical, and the deployment, security
and integration traps that each cost a day to learn.

Distilled from a real build: a government compliance portal (Next.js 16, Azure Container Apps,
PostgreSQL, Blob + Defender malware scanning, Key Vault, Event Grid, ACS Email, Stripe,
Cloudflare) produced by parallel agents over ~30 hours against ~200 traceable requirements.

## Why a skill and not a plugin

A plugin is Claude-specific machinery — commands, agents, MCP wiring — that Cursor and Hermes
cannot consume. A skill is portable markdown, which is the only real common denominator across
harnesses.

So there is **one canonical body of content** (`SKILL.md` + `reference/`) and thin per-harness
entry points in `adapters/` that *reference* it rather than copy it. Two copies of the same rules
drifting apart is the exact failure this skill warns about in §3 — do not fork the content.

## Layout

```
SKILL.md                          the spine — read this first
reference/intake.md               the questions to ask the operator, before anything is built
reference/loop.md                 the remediation loop, termination, the continuation prompt
reference/retrofit.md             adopting the method onto work already under way
reference/orchestration.md        sprint script skeleton, slice prompts, the written record
reference/gates.md                verifier splits, synthesiser prompt, verdict schema
reference/azure.md                estate, parameterisation, preflight, deployment traps
reference/security.md             malware scanning, signed URLs, grants, CSP, secrets
reference/failure-signatures.md   symptom → real cause index
reference/harness-handover.md     passing a live build to another tool mid-flight
reference/handover.md             custody handover, off-boarding, restore drill, teardown
adapters/                         how to load it in each harness
```

## Installing

### Claude Code

Project-level (travels with the repo, which is usually what you want):

```bash
mkdir -p .claude/skills && cp -r azure-autonomous-build .claude/skills/
```

User-level (available in every project):

```bash
mkdir -p ~/.claude/skills && cp -r azure-autonomous-build ~/.claude/skills/
```

The YAML frontmatter in `SKILL.md` is what Claude Code reads. It loads on demand when the
`description` matches what you are doing; you can also invoke it explicitly.

### Cursor

```bash
mkdir -p .cursor/rules && cp adapters/cursor.mdc .cursor/rules/azure-autonomous-build.mdc
cp -r azure-autonomous-build .cursor/skills/     # so the @-references resolve
```

The `.mdc` file carries Cursor's own frontmatter (`description`, `globs`, `alwaysApply`) and
points at the canonical content. Set `alwaysApply: true` if you want it loaded for every request
rather than matched by description.

### Hermes / any other harness

```bash
cp adapters/AGENTS.md ./AGENTS.md          # or append to an existing one
cp -r azure-autonomous-build ./            # keep the reference/ tree reachable
```

`AGENTS.md` is the widely-supported convention for agent instructions. The adapter is a pointer
plus the ten rules that matter most, so it degrades usefully even in a harness that reads only
one file.

## Using it

The skill is written to be **applied, not recited**. The highest-value parts, in order:

1. **`SKILL.md` §1** — Sprint Goals as exit tests, and the integration-point rule. If you adopt
   nothing else, adopt these two.
2. **`SKILL.md` §2** — the gate, and specifically the instruction to *downgrade any claimed pass
   whose evidence could not have failed*.
3. **`reference/failure-signatures.md`** — keep it open. It turns a two-hour diagnosis into a
   two-minute one, repeatedly.

## Adapting it to a different stack

Most of this is not Azure-specific. The orchestration model, the gate system, the contract, and
roughly half the failure signatures apply to any cloud. What is Azure-specific is confined to
`reference/azure.md` and the malware-scanning half of `reference/security.md`.

If you port it: keep the *shape* of the security section even where the services differ —
provider-native malware scanning, content sniffing over extensions, short-lived permission-checked
signed URLs, column-scoped database grants, managed identity over secrets, and a strict CSP. Those
are properties, not products.
