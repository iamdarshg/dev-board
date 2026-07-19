# Protecting the `main` branch

The `KiCad checks` workflow (`.github/workflows/kicad-checks.yml`) runs the KiCad 9
CLI on every pull request into `main` and fails if the production board
(`dev-real-production.kicad_pcb`) is not DRC-clean (0 violations, 0 unconnected).

CI only *reports* status. To actually **block merges** when it fails, `main` must
require that check. GitHub branch-protection settings are not part of the repo
files, so apply one of the following once (repo admin required).

## Required status check context
Use the job's display name exactly:

```
DRC — dev-real-production (required)
```

## Option A — GitHub UI
Settings → Branches → Add branch ruleset (or "Add rule") for `main`:
- Require a pull request before merging (≥ 1 approval recommended).
- Require status checks to pass before merging → add **DRC — dev-real-production (required)**.
- Require branches to be up to date before merging.
- Do not allow force pushes; do not allow deletions.

## Option B — `gh` CLI (REST)
```sh
gh api -X PUT repos/iamdarshg/dev-board/branches/main/protection \
  -H "Accept: application/vnd.github+json" \
  -f 'required_status_checks[strict]=true' \
  -f 'required_status_checks[contexts][]=DRC — dev-real-production (required)' \
  -f 'enforce_admins=true' \
  -f 'required_pull_request_reviews[required_approving_review_count]=1' \
  -f 'restrictions=' \
  -F 'allow_force_pushes=false' \
  -F 'allow_deletions=false'
```
(`restrictions` must be sent as null; with `gh api` an empty `-f 'restrictions='`
sends null. If your `gh` version rejects it, use the raw curl form below.)

## Option C — raw REST (curl)
```sh
curl -X PUT \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/iamdarshg/dev-board/branches/main/protection \
  -d '{
    "required_status_checks": {
      "strict": true,
      "contexts": ["DRC — dev-real-production (required)"]
    },
    "enforce_admins": true,
    "required_pull_request_reviews": { "required_approving_review_count": 1 },
    "restrictions": null,
    "allow_force_pushes": false,
    "allow_deletions": false
  }'
```

Once the check has run at least once on a PR, its context becomes selectable in the
UI. After enabling, a PR that breaks production DRC cannot be merged into `main`.
