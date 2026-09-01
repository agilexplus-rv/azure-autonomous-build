# Failure signatures — symptom to real cause

Each of these was diagnosed the slow way at least once. Recognise the signature and skip that.

| What you see | What it actually is |
|---|---|
| Every CI job fails in ~2s, **zero steps executed**, no error message | Actions quota / spending limit. **And it MASKS real failures** — two PRs looked "blocked by billing" and had genuine test failures underneath once the budget was raised. Never read a quota outage as "it would have passed" |
| **No CI runs created at all**, workflow reports `active`, permissions `enabled` | Different failure entirely: Actions not acknowledged on a new repository. Only the Actions tab in the UI can clear it — the API reports the workflow as active regardless. Events fired *before* enablement do not retroactively run; push again afterwards |
| A CI run says `cancelled` and nothing seems wrong | A concurrency group with `cancel-in-progress`. A newer event superseded it. Benign |
| A deploy "succeeded" but the app is unchanged | A raw cloud-CLI deployment. App resources gated on image parameters are skipped silently while the deployment reports success |
| The deploy script exits 0 but the deployment failed | **You piped it.** A pipeline's status is the LAST command's. Verify against the control plane |
| `Webhook endpoint validation failed … (404)` during a deploy | The event service validating against the **running** app, still the old image. Re-run the same deploy |
| Deployment outputs are empty for app names | Deployed without the image parameter, so the app resources were never declared. Publish scripts and live tests read those outputs |
| `main` goes red on a change that was green on its own PR | A deploy ran during CI, and the browser suites test the live estate |
| A route 404s though the page file exists | Not registered in the release/feature gate — deny-by-default rewrites it to a 404 |
| A route 404s, no page file exists, but links point at it | Nothing checks that a *referenced* route exists. Two guards can both pass: one asserts the link's href, the other walks routes on disk |
| Middleware changes have no effect, no error | Framework renamed the file/export (Next 16: `proxy.ts` exporting `proxy`). A `middleware.ts` is silently ignored |
| `permission denied for table X` at runtime, CI green | A missing or too-narrow **column** grant. A table-level check passes while the columns actually written are ungranted |
| A static check says OK on a file you know is broken | The file contains a `0x00` byte and the reader treats it as binary, returning null. Thirteen guards skip it silently |
| Client-side navigation dies with `ChunkLoadError` | CSP `script-src` lost `'strict-dynamic'`. Do NOT "fix" with `'unsafe-eval'` or `'self'` |
| A test is green while the feature is visibly broken | It asserts the ABSENCE of an error string, or runs in dev where the defect is production-only |
| `initdb: invalid locale settings` | The shell's `LANG`/`LC_ALL`, not the test |
| `Symlink [project]/node_modules is invalid, it points out of the filesystem root` | A symlinked dependency directory in a worktree outside the project root. Install properly inside the worktree instead |
| A PR won't merge, all checks green | `BEHIND` — branch protection requires up-to-date branches. Update the branch and wait for CI again |
| An agent returns `null` / no report | It died on a transient error. Treat as **UNVERIFIED**, never as a pass |
| Two branches reappear on a mirrored repo offering "Compare & pull request" | A sync push without `--prune`. Branches deleted on the source survive on the target |
| A merged branch looks unmerged to `git merge-base --is-ancestor` | It was **squash**-merged — a new commit, so the branch's own commits are never ancestors of main. Compare *content* instead |
