# Tracely Docs Gate

Blocks merging a GitHub PR unless its description carries a valid **Tracely footprint** —
a public Tracely share link that resolves to a real, non-empty Tracely document or page.

Repo-agnostic: checks out no code, builds nothing, ~5s per run.

## Install in a repo

```bash
mkdir -p .github/workflows
cp tracely-docs-gate.yml .github/workflows/tracely-docs-gate.yml
cp pull_request_template.md .github/pull_request_template.md # optional but helps
git add .github && git commit -m "ci: require Tracely docs footprint on PRs"
```

## Configure (only if the host changes)

Settings → Secrets and variables → Actions → **Variables** tab (Variables, *not* Secrets —
these are public URLs and `secrets` are unavailable to fork PRs anyway). Set them on the
**organisation**, then every repo inherits:

| Variable | Value | Required |
|---|---|---|
| `TRACELY_API_BASE` | `https://app.tracely.uk` | no - baked in as the default |
| `TRACELY_APP_URL` | `https://app.tracely.uk` | no - baked in as the default |

Unset `TRACELY_API_BASE` → the gate fails closed with an explanatory message.

## Make it actually block the merge

The workflow only reports a check. GitHub does the blocking:

Repo (or org ruleset) → Branch protection for the default branch →
**Require status checks to pass** → add **`Tracely footprint`**.

The check name only appears in that picker after the workflow has run once, so open one
throwaway PR first. Tick **Do not allow bypassing the above settings** if admins should be
gated too.

Using a merge queue? Add the same check to the queue's required list — otherwise the queue
merges around it.

## Per-repo knobs

All in the `env:` block of the workflow:

| Env | Default | Effect |
|---|---|---|
| `SKIP_LABEL` | `no-tracely-docs` | Label that skips the gate |
| `REQUIRE_PR_BACKLINK` | `false` | `true` → the Tracely doc must reference this PR (source URL contains the PR URL, or title contains `#<number>`) |
| `CHECK_DRAFTS` | `false` | `true` → gate drafts too |
| `PATHS_REQUIRING_DOCS` | empty (gate every PR) | Comma-separated path prefixes / extensions, e.g. `src/,api/,.sql` — gate only fires when a matching file changed |

Recommended rollout: ship with defaults, then flip `REQUIRE_PR_BACKLINK` to `true` once the
team is in the habit.

## What counts as a valid footprint

Anywhere in the PR description, case-insensitive:

```
https://app.tracely.uk/public/documents/<token>
https://app.tracely.uk/api/public/pages/<token>
Tracely: /public/documents/<token>
```

Each candidate is fetched from `GET {TRACELY_API_BASE}/api/public/{documents|pages}/{token}`.
It passes only when the response is 2xx **and** the doc has content. Fails on:

- no link in the description
- `404` — deleted, or sharing switched off in Tracely
- non-2xx / unreachable API
- doc exists but is **empty** (a blank shared doc must not buy a merge)
- `REQUIRE_PR_BACKLINK=true` and the doc does not reference the PR

Result is posted as one sticky PR comment (updated in place, never spammed) with the failure
reason and the fix instructions.

## Known limits

- Fork PRs hand the workflow a read-only token, so the comment step warns instead of posting.
The failing check still blocks the merge.
- Re-runs on `edited`, so fixing the description re-triggers automatically.
- Repo admins bypass unless bypassing is disallowed in the protection rule.
