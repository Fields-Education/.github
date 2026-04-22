# Warden Rollout

This repository is the source of truth for the org-level Warden workflow.

## Shared Workflow

- Workflow file: `./.github/workflows/warden.yml`
- Intended host repository: `Fields-Education/.github`
- Intended ruleset target: organization repositories on their default branch

The workflow follows the Warden org setup pattern:

- canonical workflow lives in the org `.github` repository
- organization ruleset requires that workflow for targeted repositories
- repositories without `warden.toml` rely on Warden's native warn-and-skip behavior

No custom skip step is included on purpose.

## Preserved Behavior

- `WARDEN_ANTHROPIC_API_KEY` stays externalized as a secret
- `WARDEN_MODEL` can come from either an org variable or org secret
- `WARDEN_SENTRY_DSN` can come from either an org variable or org secret
- `ANTHROPIC_BASE_URL` stays externalized as an Actions variable
- if `WARDEN_APP_ID` and `WARDEN_PRIVATE_KEY` are present, the workflow uses a GitHub App token
- if those app secrets are not present yet, the workflow falls back to `GITHUB_TOKEN` with the same write permissions the local workflow used

An org-installed GitHub App for Warden already exists, so a new app does not need to be created or installed. The remaining GitHub App setup is to store that app's credentials as org Actions secrets.

## Org Actions Configuration

Expected org-level configuration:

- Secret: `WARDEN_ANTHROPIC_API_KEY` (required)
- Variable: `WARDEN_MODEL` (optional, recommended if you want a pinned org-wide model)
- Secret: `WARDEN_SENTRY_DSN` (optional, recommended)
- Variable: `ANTHROPIC_BASE_URL` (required for the custom gateway override)
- Secret: `WARDEN_APP_ID` (recommended)
- Secret: `WARDEN_PRIVATE_KEY` (recommended)

Notes:

- GitHub does not allow reading existing secret values back out, so moving repo-level secrets to org-level requires setting the org secrets with the same values manually.
- `ANTHROPIC_BASE_URL` should be copied from the current repo-level variable into the org-level variable with the same name.
- `WARDEN_MODEL` is optional from Warden's perspective. Keeping it set as an org variable is a good way to pin model behavior org-wide and avoid unexpected default-model changes.
- GitHub App IDs are identifiers, not credentials. The private key is the sensitive part and must stay in `WARDEN_PRIVATE_KEY`.

## Ruleset

Recommended rollout:

1. Set the org secrets and variables above.
2. Commit and push `./.github/workflows/warden.yml` to `Fields-Education/.github` default branch.
3. Create an org branch ruleset in `evaluate` mode first.
4. After verifying runs, change enforcement from `evaluate` to `active`.

### Ruleset Details

- Target: `branch`
- Enforcement: `evaluate` first, then `active`
- Target repositories: `~ALL`
- Target branches: `~DEFAULT_BRANCH`
- Required workflow path: `.github/workflows/warden.yml`
- Required workflow ref: `main`

### Example API Call

This requires a token with `admin:org` scope:

```bash
gh auth refresh -h github.com -s admin:org

GITHUB_REPO_ID="$(gh api repos/Fields-Education/.github --jq '.id')"

gh api \
  --method POST \
  -H "Accept: application/vnd.github+json" \
  -H "X-GitHub-Api-Version: 2026-03-10" \
  "/orgs/Fields-Education/rulesets" \
  --input - <<EOF
{
  "name": "required-warden",
  "target": "branch",
  "enforcement": "evaluate",
  "conditions": {
    "ref_name": {
      "include": ["~DEFAULT_BRANCH"],
      "exclude": []
    },
    "repository_name": {
      "include": ["~ALL"],
      "exclude": []
    }
  },
  "rules": [
    {
      "type": "workflows",
      "parameters": {
        "workflows": [
          {
            "repository_id": ${GITHUB_REPO_ID},
            "path": ".github/workflows/warden.yml",
            "ref": "main"
          }
        ]
      }
    }
  ]
}
EOF
```

To move to enforcement later:

```bash
gh api \
  --method PUT \
  -H "Accept: application/vnd.github+json" \
  -H "X-GitHub-Api-Version: 2026-03-10" \
  "/orgs/Fields-Education/rulesets/RULESET_ID" \
  --input - <<EOF
{
  "name": "required-warden",
  "target": "branch",
  "enforcement": "active",
  "conditions": {
    "ref_name": {
      "include": ["~DEFAULT_BRANCH"],
      "exclude": []
    },
    "repository_name": {
      "include": ["~ALL"],
      "exclude": []
    }
  },
  "rules": [
    {
      "type": "workflows",
      "parameters": {
        "workflows": [
          {
            "repository_id": ${GITHUB_REPO_ID},
            "path": ".github/workflows/warden.yml",
            "ref": "main"
          }
        ]
      }
    }
  ]
}
EOF
```

## Weekly Schedule

The shared org-level workflow intentionally only handles pull requests.

The previous weekly scheduled behavior should stay local to consuming repositories for now. Required workflows via org rulesets are a good fit for PR enforcement, but they do not provide the same org-wide scheduled fanout behavior as a repo-local scheduled workflow.

For repositories that currently run Warden weekly:

- remove the local `pull_request` trigger once the org ruleset is active
- keep the weekly `schedule` trigger in a local schedule-only workflow, or keep the existing workflow but with only `schedule`

That avoids duplicate PR runs while preserving scheduled analysis.

## Validation Targets

Validate on:

- one repository that already has a valid `warden.toml`
- one repository that does not yet have `warden.toml`

Repositories without `warden.toml` are expected to warn and skip analysis without failing, per the Warden org setup docs.

## Current Limits

The local CLI token in this workspace does not have `admin:org`, so org Actions secrets, org variables, and org rulesets could not be read or changed via API from this session.

The workflow file in this repository is ready. The remaining work depends on:

- pushing this workflow to the default branch of `Fields-Education/.github`
- setting org-level secrets and variables
- creating the org ruleset with an org-admin token
- updating consuming repositories to keep only local schedule-only Warden runs where needed
