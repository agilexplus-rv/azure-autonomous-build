# Handover — leaving a system someone else can run

## Two handovers. Do not conflate them

| | Reader | Where |
|---|---|---|
| **Cold start** | the next agent or session, mid-build | `orchestration.md` — the state file, the decision register, the `Resume by saying…` line |
| **Custody** | the team who will own this after you leave | this file |

The first is about not losing context. The second is about not being needed. A build can be
excellent at the first and fail the second completely.

## The test that defines "handed over"

`azure.md` §"Proving portability" gives the install proof: a foreign tenant, from the written
guide alone, by someone who has never seen the code. Custody handover extends it to *operating*:

> The receiving team performs the day-one list below, on their own tenant, from the written
> material alone, **while you stay silent in the room**.

Anything you have to say out loud is a missing document. Write it down, then re-run the list.
Every manual intervention is a defect to fix and re-prove, not a caveat to write up.

## Day one — the list they complete unaided

1. Ship a trivial change end to end: branch, PR, green gate, merge, deployed, visible.
2. Roll that deployment back.
3. Rotate one secret in the vault and prove the app picks it up without a redeploy.
4. Restore the database into a scratch resource group and state the measured RPO and RTO.
5. Take a failing request from a user report to a root cause in the logs and traces.
6. Handle a malware quarantine event: find it, see the uploader notification, release or destroy.
7. Create the first administrator on a clean install, then create an ordinary user.
8. Answer, from the decision register alone, why one non-obvious design decision was taken.

A team that cannot do 3, 4 and 5 does not own the system yet, whatever the contract says.

## Access off-boarding — the step everyone skips

Handover is not complete while you can still reach production. Inventory every identity the build
created or touched, and **revoke rather than stop using**:

| System | What to revoke or rotate |
|---|---|
| Azure | federated credentials for CI, the custom deployment role, any standing role assignment on the resource group, your accounts in the tenant |
| Key Vault | every access policy or RBAC grant naming you, plus every secret whose value you have seen |
| Database | administrator logins used during the build; the least-privilege application roles stay |
| Application | the first-admin bootstrap token if unburned, and any account you created for yourself |
| GitHub | deploy keys, PATs, Actions secrets, repository access, and the app-level installation |
| Cloudflare | API tokens scoped to the zone, and your membership of the account |
| Stripe | restricted keys, webhook signing secrets, dashboard membership |
| Email / messaging | connection strings and sender identities issued to you |

Two rules make this real:

- **Prove the revocation.** Run the deployment pipeline with your own credentials and require it
  to fail. Verify state, never output — an access removal you have only *asserted* is worth
  nothing.
- **Anything you have seen is compromised for handover purposes**, whether or not you kept it.
  Rotate it. This is cheaper than the conversation about why you did not.

Your processor status under the DPIA ends when access ends, not when the invoice does. If any
standing access survives handover by agreement, it must be named in the processing agreement with
its scope and its retention — see `azure.md` §"Access, if you deploy into a client tenant".

Leave exactly one break-glass path, owned and tested by **them**, documented with the conditions
under which it may be used and the audit it generates.

## Ownership that must move before go-live, not after

These are not credentials, and they are the ones that turn into a hostage situation:

- **The domain and its DNS zone** live in their account. A production system pointed at a zone you
  control is not theirs.
- **The Stripe account belongs to their legal entity.** Payouts, tax identity and liability follow
  the entity; an account you own cannot be handed over, only migrated, and migration means new
  keys, new webhooks and a live-traffic cutover. Get this right at the start.
- **The repository** transfers to their organisation. Transferring breaks Actions secrets and
  environment protection rules — re-create them on the far side and re-run the gate before you
  call it done.
- **The subscription and the billing owner**, with budget alerts set to their finance contact.
  Whoever pays decides, and a build whose owner cannot see its cost gets switched off by surprise.

## The restore drill

A backup configuration is a claim. A restore is the proof, and the two are not the same evidence.

Before handover, restore into a scratch resource group, **timed**, and record the measured RPO and
RTO rather than the vendor's quoted ones. Then have the receiving team do it themselves — the
drill is theirs to pass, not yours to demonstrate.

Restore the *dependencies* too: a database restored beside stale blob storage, or beside a vault
whose secrets have since rotated, is a system that comes back inconsistent.

## Handing over the record

The written record is part of the deliverable, not an artefact of how it was built:

- the state file, current as of the final gate;
- the decision register **with the reasoning intact** — a decision whose *why* is lost gets
  relitigated or silently reversed by people who were not there;
- traceability: requirement → files → at least one test;
- the open findings, labelled honestly.

**Do not launder open findings into "known limitations" at handover.** Anything a gate downgraded
to UNVERIFIED is handed over as UNVERIFIED, with what would settle it. A finding renamed at the
last minute is the one that surfaces in their first incident.

## Teardown and decommission

The same discipline in reverse, and it belongs in the guide before it is needed:

- Delete by **resource group**, and know what survives it — soft-deleted vaults and blobs,
  purge-protected keys, backups in a vault, diagnostic settings and domain verification records
  outside the group.
- **Purge protection means deletion is not deletion.** Plan the purge window, or the next install
  fails on a name that appears to be free and is not.
- Export what regulation requires you to retain **before** the delete, and record where it went.
- Release the domain records and the payment webhooks explicitly; an orphaned webhook retrying
  into nothing is someone's alert at 3am.

## Traps

| What you see | What it actually is |
|---|---|
| Handover call runs long on "just one more thing" | the written guide is incomplete; each verbal answer is a defect, log it and fix it |
| Their first deployment fails, yours never did | machine-specific state — a tool version, a local login, a cached credential — that the guide never named |
| Restore succeeds, application misbehaves | dependencies restored at inconsistent points in time |
| Access "removed" but a pipeline still deploys | revocation asserted, never tested against the running system |
| A defect appears months later that nobody can explain | the decision register was handed over without the reasoning |
