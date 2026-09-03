# Data protection — backup drills and the DPIA-inputs register

New file. Source: two independent builds each reaching a "backups configured, restore untested"
gap and a "DPIA due, nothing written down as it happened" gap, at different points in their
lifecycle. Both are the same shape of mistake — a requirement stated as a promise about the future
("backups are restorable", "a DPIA will be produced") with nothing that makes the promise
checkable before the day it is needed.

## Backup and restore as a checkable drill, not a promise

A spec that requires "a documented and tested restore" is not satisfied by a parameterised backup
configuration, however correct. Build the restore path as an **operator script**, not a runbook
paragraph, with these properties:

- **Dry-run by default.** The script prints the exact restore command it would run and exits 0
  without calling the provider. A flag is required to actually create anything.
- **No silent "now".** The point-in-time to restore to is a required argument for a real run;
  omitting it is only allowed on a dry-run, which prints what it *would* use so the operator sees
  the default before committing to it.
- **Never overwrite the live resource.** A restore creates a *new* instance; the script refuses a
  target name equal to the source and refuses a real run if the target already exists. This is the
  one invariant worth hard-coding rather than trusting operator discipline for — the failure mode
  is destroying the thing you were protecting.
- **A drill tears down its own target afterwards.** A second live copy of production data is
  itself a new processing activity, not a freebie.
- **A drill-record table, committed, empty until a drill actually happens.** Not a checkbox in a
  gate file — an actual table with date, source, target, restore point and operator, in the same
  document as the procedure. Its emptiness is the honest status: "the backup is configured and the
  restore path is rehearsable; it is not yet a signed-off restore." A gate that credits "tested
  restore" against a Bicep parameter and no drill row is crediting a claim, not a fact.

## The DPIA-inputs register, opened at the start of the phase that needs it

A DPIA (or equivalent data-protection assessment) written as one late sprint has to reconstruct
the reasoning behind every data-handling decision from code and memory, after whoever made the
decision has moved to the next sprint. Open a plain input register instead, at the **start** of
whichever phase first builds a mechanism with a data-protection dimension, and have the sprint
that builds each mechanism write its own entry while the reasoning is fresh — one paragraph, not a
report.

The discipline that matters is stating a mitigation's **limits** in the same entry as its
capability, not just what it does. A self-service view that partially satisfies a data-subject's
right of access is easy to overclaim later once it exists — record explicitly what it is scoped
to, what assurance level it actually proves (control of a mailbox is not identity proofing), and
which categories of data it does *not* surface (rate-limiter state, IP addresses, payment records
— whatever the general access-request procedure still has to cover). The register becomes the data
protection officer's own working material, produced as a byproduct of normal sprint work rather
than a research task bolted onto the end.
