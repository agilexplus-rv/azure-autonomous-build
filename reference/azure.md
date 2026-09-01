# Azure — estate, parameterisation, deployment

## The governing rule

**Your own dev estate is provisioned by the same package the client will use.** Do not click
resources into existence or run CLI one-liners. Write the IaC, and let the IaC create the dev
estate. Every sprint then exercises the package daily, so it cannot drift from the thing you hand
over. From that point there is **no other way to create an environment**.

## A typical estate

```
2 × Container Apps (web + a separate worker — background jobs need a long-running process)
1 × Container Apps managed environment + managed certificate
1 × PostgreSQL Flexible Server            TLS enforced, Entra auth
1 × Storage account                       private containers, Defender malware scanning
1 × Key Vault
1 × Container Registry
1 × Communication Services + Email + Domain
1 × Event Grid topic + system topics       scan results, email delivery/bounce
1 × User-assigned managed identity
1 × Log Analytics workspace
```

## Parameterisation — what makes it portable

- **Nothing global is hardcoded.** Globally-unique names are `${prefix}${uniqueString(rg.id)}`.
- **Region is a parameter with an allowed-values list**, which enforces a data-residency
  requirement in the template rather than in a document.
- **Regional capability genuinely differs.** Verify rather than assume: on one subscription, one
  region offered zone-redundant HA but not geo-redundant backup, and another the reverse. So
  availability and backup resolve *differently depending on the region the client picks*. Take
  `haMode`/`backupMode` as parameters, default them from a per-region capability table, and have
  preflight **print what the chosen region can and cannot do before anything is created**.
- **Some services do not move with the region parameter.** Communication Services is a `global`
  resource with its own separate data-residency property. Model it as a distinct parameter from
  day one.
- **Develop in a different region from the one you recommend**, so portability is exercised by
  every deployment rather than tested once.
- **Migrations run as a pre-traffic step**, idempotently. The client never runs the ORM by hand.
- **Two seeds, only one ships.** Reference data always; demo data refuses to run under
  `NODE_ENV=production`. Demo records reaching a client's production database is a data-protection
  incident, not an embarrassment.
- **First-admin bootstrap.** If accounts are invitation-only with no self-registration, there is
  no way to create the first administrator on a clean install. Resolve it with a one-time,
  single-use token written to the vault at deploy time, consumed once, then burned and audited.

## Preflight, before anything is created

Check against *their* subscription and *their* region: provider registration, region and SKU
availability, the regional capability report, quota headroom, a `what-if`, and policy evaluation.
Refuse a non-compliant region outright. A government or enterprise landing zone very likely denies
public ingress, mandates private endpoints and enforces tagging — discovering that during a
handover call is the expensive way.

Make **network shape a parameter** (`public | private`) if you do not know theirs. Retrofitting
private endpoints is a rebuild, not a setting.

## Deployment traps

- **One supported deploy path.** A raw `az deployment group create` against a template whose app
  resources are gated on image parameters **silently skips them and reports `Succeeded`**. Wrap it
  in a script; make the script the only way; say so in the script's own `--help`, because the
  client will hit this the first time they try to apply a change by hand.
- **Always pass the image parameter.** Without it the app resources are never declared, so the
  deployment **outputs come back empty** for app names — which publish scripts and live test
  suites read.
- **An apps-only fast path must be guarded.** Compare a hash of the infrastructure files against a
  tag recorded by the last full deployment, and refuse if they differ. An image-only path that
  silently skipped an infrastructure change would be the same trap as the raw deployment above.
- **Event Grid validates against the RUNNING app.** A deployment that ships a new webhook route
  *and* creates the subscription consuming it cannot succeed in one pass — the handshake hits the
  old image and 404s. The apps update before that step fails, so **run the same deploy again**.
  On a clean estate this bites on the first install; document it.
- **Never leave zero estates.** Deploy the replacement alongside, verify, then tear down the old.
  Container Apps environment deletion takes ~2 hours — kick it off, poll, report; never block.
- **Never deploy into an in-flight CI run** if browser suites test the live estate.
- **Deploy once per phase, before each gate.** Deploys measured ~20% of run time and mostly
  re-proved what CI had — but never defer them *past* a gate, because a whole class of defect is
  invisible to CI and appears only against the live estate.
- **Verify against the control plane, never the exit code.** Query the deployment's
  `provisioningState`. A piped command reports the pipe's status, not the script's.

## Access, if you deploy into a client tenant

Ask for **workload identity federation** for your CI — never a long-lived client secret — scoped
to a **custom role** carrying only what deployment needs, on the resource group and nothing wider.
Every deployment is then attributable to a run ID and revocable without touching the running
system.

Say plainly that standing access to a production system holding personal data makes you a
processor: the DPIA and the processing agreement must cover it explicitly — your role, the access
scope, retention of deployment logs, sub-processor status.

## Proving portability

Three increasing strengths, and only the third is proof:

1. two estates from one template with different prefixes and regions;
2. a rebuild from empty that passes the acceptance suite;
3. **an install into a foreign tenant, from the written guide alone, by someone who has never seen
   the code.** Anything less is a claim. Every manual intervention, undocumented step or code
   change needed is a defect to fix and re-prove, not a caveat to write up.
