# Security precautions

## Malware scanning on uploads

Use the cloud provider's own on-upload scanning — **Microsoft Defender for Storage** — rather
than hand-rolling one or bolting on a container.

- **Enable it per storage account**, not subscription-wide, if the subscription holds unrelated
  resource groups. A subscription-scope plan is a side effect on resources your template never
  created, which breaks a "parameter changes only" install promise.
- **Cap the monthly scanned volume.** Pricing is per-account plus per-GB with no free tier once
  enabled, so a runaway upload loop runs away with the bill too.
- **Use the provider's built-in quarantine** (blob soft-delete) rather than a hand-rolled
  move/delete — it is a maintained feature with a recoverable retention window.
- **Not every region offers it.** Preflight must check, and the template must degrade to a
  documented fallback rather than failing halfway.

**Prove it with a real EICAR file** through the real pipeline. The exit test is: infected file
quarantined, never rendered to a reviewer, uploader notified. A unit test proves nothing here.

**The scan-result webhook must be reachable.** On the reference build the route existed, was
correct, was CI-green — and was never registered in the release gate, so it 404'd. The event
service completes a one-time validation handshake before it delivers anything; against a 404 that
handshake cannot complete, the subscription is never created, and **no verdict could ever arrive
however correctly scanning was configured**. Also: a `visibility` guard meant to hide unscanned
documents from reviewers was dead code, imported by nothing.

## Accepting files

- **Content sniffing, never extension or client `Content-Type`.** Use a maintained library. The
  decisive test is a PNG renamed `.pdf` being rejected, and the *sniffed* type being what gets
  persisted.
- **Enforce the size limit in the application, not the framework.** If uploads cross a middleware
  layer with its own body cap, the framework's generic 500 pre-empts your refusal path: no
  `file_too_large`, no log line, no audit row, and the user waits a full minute to be told
  nothing. Whatever the transport cap is, it must sit **above** the enforced limit plus multipart
  overhead — and the client should refuse oversize files by `File.size` before uploading a byte,
  as a courtesy. The server remains the security boundary.

## Serving files back

- Private container, keys namespaced per record so one record's files can never address another's.
- **Short-lived signed URLs issued only after a server-side permission check**, with
  `Content-Disposition: attachment` and a restrictive content type, so uploads are never served
  inline from the application origin.
- Use **user-delegation** signatures (managed identity), never an account key.
- Never log a signature or a full signed URL.
- Exit tests: bare URL 403s · expired signature 403s · tampered signature 403s · another record's
  prefix 404s · path traversal refused · issuing without the permission check refused **and
  audited** · a not-yet-clean file offers no working link.

## Secrets

- **Managed identity everywhere.** The environment carries endpoints and names, never credentials.
- Third-party secrets go into the vault **out of band** and are referenced by URI. Never template
  parameters, never outputs, never deployment arguments — the deployment history retains them and
  discloses them to anyone with reader access long afterwards. A `@secure()` marker suppresses the
  value in history but does not make routing a secret through a template safe.
- Fail the build on any live payment key reaching the repo **or a resolved environment**.

## Database privileges

Default privileges grant `SELECT` and nothing else. Every write privilege arrives with the
migration that introduces the write path, and `UPDATE` grants name **columns**.

Two failures worth internalising:

1. **Five write paths shipped with no grant at all.** A public enrolment form rendered fine and
   was refused at the insert — in the whole life of that release, not one submission was ever
   recorded, and health checks were green throughout.
2. **The guard then passed a broken tree.** It asked whether `UPDATE` was granted on the *table*;
   it was, on ten *other* columns. It was also line-oriented, so a multi-line `update X\n set …`
   never matched. Meanwhile the live test asserted a column *must not* be app-writable — a column
   written on every save — so the suite going green was evidence the app was broken.

**Test as a role holding only the app's group membership**, against a real database with every
migration applied. Running as the owner proves nothing, and is what hid the above for two cycles.

Make the audit log append-only **at the grant level** and prove the app role cannot `UPDATE` or
`DELETE` it.

## Content-Security-Policy

- `script-src 'nonce-…' 'strict-dynamic'`. Without `'strict-dynamic'`, modern framework chunk
  loaders inject non-nonced scripts and **every client-side navigation dies** with a
  `ChunkLoadError` — invisible in dev mode, fatal in production.
- Do **not** substitute `'self'` (trusts any same-origin script on a service accepting uploads) or
  add `'unsafe-eval'` (some validation libraries probe for `Function()` and degrade gracefully when
  refused — a harmless console entry is not worth weakening the policy).
- `style-src` usually needs `'unsafe-inline'` in practice: a nonce cannot authorise a `style=`
  attribute, and component libraries emit hundreds.
- `frame-ancestors 'none'`.
- Test CSP on a **production build**, and assert zero violations across a real journey.
