# Warden Rollout

This repository is the source of truth for the org-level Warden workflow.

## Shared Workflow

- Workflow file: `./.github/workflows/warden.yml`
- Org base Warden config: `./warden-base.toml`
- Intended host repository: `Fields-Education/.github`
- Intended ruleset target: organization repositories on their default branch
- Default Warden action parallelism: `parallel: 4`

The workflow follows the Warden org setup pattern:

- canonical workflow lives in the org `.github` repository
- organization ruleset requires that workflow for targeted repositories
- repositories without `warden.toml` load the org base config and complete with no matched triggers
- repositories can override file-analysis parallelism with `[runner] concurrency = 16` in `warden.toml`
- the target repository checkout uses the pull request head SHA so Warden reads the same code it annotates
- the workflow passes `base-config-path: .warden-org/warden-base.toml` so the org base config is merged with repository overlays
- the workflow installs Node 24 before running Warden because Warden's `v0.34.x` action bundle preloads the Pi runtime, whose dependencies require Node APIs not present in Node 20

## Split Analyze and Report

Since Warden `0.38.x` the workflow runs the action twice in the same job:

- `mode: analyze` runs the skills and writes a structured findings file. It does
  not create checks, post comments, or resolve stale comments.
- the GitHub App token is minted after analysis completes, so the 1-hour
  installation token is always fresh for the reporting phase (the old layout
  minted it before a potentially 60-minute analysis)
- `mode: report` re-reads the merged config from the checkout and receives the
  reporting inputs (`base-config-path`, `report-on`, `request-changes`). It
  creates completed check runs and posts or resolves review comments.

Because analyze mode creates no check runs and report mode only creates
completed ones, there is no `in_progress` window for Warden checks. The old
`cleanup-warden-checks` job that completed stale checks after failures and
cancellations was removed for that reason.

## Skipping Warden

Two gates run before analysis, both evaluated against live PR state when the
job starts:

- Draft pull requests are skipped automatically.
- An authorized comment containing the line `/warden skip` (or
  `/reviewer skip`) disables Warden for the pull request. `/warden run`,
  `/warden allow`, and the `/reviewer` equivalents re-enable it. The last
  command wins, in comment creation order, regardless of prefix. The
  `/reviewer` prefix is shared with the org's opencode-based review system so
  one comment can address both reviewers.

Command rules:

- only comments from authors with `OWNER`, `MEMBER`, or `COLLABORATOR`
  association count
- matching is case-insensitive and the command must be on its own line
- only pull request conversation comments are scanned, not review-thread
  comments or review bodies

A skipped run still completes successfully, so the required-workflow ruleset is
satisfied — `/warden skip` makes the check green without analysis.

Caveat: required workflows installed by org rulesets only trigger on default
`pull_request` activity (opened, synchronize, reopened); `issue_comment` events
never trigger them in target repositories. After commenting a skip or run
command, re-run the Warden check from the Checks UI (or push a new commit) for
the command to take effect. If an analysis is already running on a PR that is
too large to finish, cancel the run first, then re-run it after commenting.

## Preserved Behavior

- `WARDEN_API_KEY` stays externalized as a secret
- `WARDEN_SENTRY_DSN` can come from either an org variable or org secret
- `WARDEN_API_KEY` is exposed to Pi as `FIREWORKS_API_KEY`
- if `WARDEN_APP_CLIENT_ID` and `WARDEN_PRIVATE_KEY` are present, the workflow uses a GitHub App token
- if those app secrets are not present yet, the workflow falls back to `GITHUB_TOKEN` with the same write permissions the local workflow used

## Pi Fireworks Configuration

Warden `0.34.0` defaults to the Pi runtime. The shared org base config pins that explicitly:

```toml
[defaults]
runtime = "pi"
```

The workflow maps the existing `WARDEN_API_KEY` secret to Pi's Fireworks credentials:

```yaml
FIREWORKS_API_KEY: ${{ secrets.WARDEN_API_KEY }}
WARDEN_FIREWORKS_API_KEY: ${{ secrets.WARDEN_API_KEY }}
```

Do not pass the Fireworks key through `anthropic-api-key`, `ANTHROPIC_API_KEY`, or `WARDEN_ANTHROPIC_API_KEY`. Those names make Pi treat the key as Anthropic credentials.

Do not set `WARDEN_MODEL` or `[defaults.auxiliary].model` to `accounts/fireworks/models/...` with Warden `0.34.0`. Pi itself supports Fireworks model IDs like `accounts/fireworks/models/kimi-k2p6`, but Warden's `0.34.0` Pi selector validation only accepts one slash in configured model values. With only `FIREWORKS_API_KEY` present and no explicit model configured, Pi selects its Fireworks default model.

Repositories can still set a repo-local `[defaults.auxiliary]` stanza with a Warden-valid Pi selector when they need to override the org default or when they run Warden outside the shared workflow.

An org-installed GitHub App for Warden already exists, so a new app does not need to be created or installed. The remaining GitHub App setup is to store that app's credentials as org Actions secrets.

## Org Actions Configuration

Expected org-level configuration:

- Secret: `WARDEN_API_KEY` (required; Fireworks API key)
- Variable: `WARDEN_MODEL` (do not set for the Pi Fireworks workflow on Warden `0.34.0`)
- Secret: `WARDEN_SENTRY_DSN` (optional, recommended)
- Secret: `WARDEN_OTLP_ENDPOINT` (optional, recommended for Warden telemetry)
- Secret: `WARDEN_OTLP_HEADER` (optional, recommended with `WARDEN_OTLP_ENDPOINT`)
- Variable: `WARDEN_BASE_URL` (not used by the Pi Fireworks workflow)
- Variable: `WARDEN_APP_CLIENT_ID` (recommended)
- Secret: `WARDEN_PRIVATE_KEY` (recommended)

Notes:

- GitHub does not allow reading existing secret values back out, so moving repo-level secrets to org-level requires setting the org secrets with the same values manually.
- `WARDEN_BASE_URL` was required for the old Anthropic-compatible Fireworks shim. Pi uses its built-in Fireworks provider instead.
- `WARDEN_MODEL` must remain unset unless the value is a Warden-valid Pi selector with exactly one slash, such as `openai/gpt-5.5`. Fireworks model IDs currently include additional slashes, so they cannot be pinned through Warden `0.34.0` config.
- GitHub App client IDs are identifiers, not credentials. The private key is the sensitive part and must stay in `WARDEN_PRIVATE_KEY`.

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
