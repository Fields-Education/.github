# Review Commands

Pull request comment commands for controlling the automated review checks
(Warden and opencode) on a single PR. This is the downstream
setup guide; the workflow itself lives in this repository at
`./.github/workflows/review-commands.yml`.

## Commands

Post a comment on the pull request conversation with the command on its own
line. Matching is case-insensitive and surrounding whitespace is ignored; a
command embedded in a sentence (for example `/warden skip please`) does not
match. One comment can contain several commands on separate lines.

| Command | Effect |
| --- | --- |
| `/warden skip` | Disable Warden for this PR (removes `warden:on`) |
| `/warden run` \| `allow` \| `resume` | Enable Warden (adds `warden:on`) |
| `/opencode skip` | Disable opencode review (adds `opencode:off`) |
| `/opencode run` \| `allow` \| `resume` | Re-enable opencode review (removes `opencode:off`) |
| `/reviewer skip` | Disable both (removes `warden:on` and adds `reviewer:off`) |
| `/reviewer run` \| `allow` \| `resume` | Enable both (adds `warden:on` and removes `reviewer:off`) |

Only comments from authors with the `OWNER`, `MEMBER`, or `COLLABORATOR`
association count. Within one comment, the last command affecting a given
label wins.

## Labels

The commands change labels, which can also be changed directly (requires
triage permission or higher):

- `warden:on` — Warden is enabled for the PR; without it, Warden skips
- `opencode:off` — opencode review is disabled for the PR
- `reviewer:off` — both are disabled for the PR

Label state is authoritative. Warden requires `warden:on`, and `reviewer:off`
overrides that opt-in as the shared kill switch. A comment does not control
Warden by itself; the Review Commands workflow applies the corresponding label
change. Missing labels are created automatically the first time a command adds
them.

## What the workflow does

When an authorized command comment is created, the workflow:

1. adds or removes the matching label
2. reacts to the comment with an eyes emoji as acknowledgement
3. cancels and re-runs the latest affected review workflow run for the PR
   head SHA so the skip or resume takes effect immediately — required-workflow
   rulesets only re-evaluate on push, reopen, or re-run, so this step is what
   makes the command feel instant

A skipped check completes successfully, so required-check enforcement is
satisfied while the skip is active.

## Setup for a downstream repository

Add `.github/workflows/review-commands.yml` to the repository:

```yaml
name: Review Commands

on:
  issue_comment:
    types: [created]

permissions:
  contents: read
  issues: write
  pull-requests: read
  actions: write

jobs:
  commands:
    if: github.event.issue.pull_request
    uses: Fields-Education/.github/.github/workflows/review-commands.yml@main
    secrets: inherit
```

That is the whole setup. The Warden GitHub App credentials
(`WARDEN_APP_CLIENT_ID` and `WARDEN_PRIVATE_KEY`) are org-level and flow in
through `secrets: inherit`; no repository-level secrets or variables are
needed.

### Inputs

Pass these under `with:` on the `uses:` job if the defaults do not match your
repository:

| Input | Default | Purpose |
| --- | --- | --- |
| `warden-workflow-name` | `Warden` | Workflow whose runs are re-synced when `warden:on` or `reviewer:off` changes |
| `opencode-workflow-name` | `opencode` | Workflow whose runs are re-synced when `opencode:off` or `reviewer:off` changes |

A repository that does not run one of the workflows needs no configuration;
the sync step logs that no runs were found and moves on.

## Behavior without the stub

Repositories without the stub still honor labels: the Warden workflow reads
the live label state at the start of every run. Comment commands are not
processed without the Review Commands workflow. For org-required workflows,
manually changing a label takes effect on the next push or a manual re-run of
the check from the Checks UI.

## Notes and limitations

- Only newly created comments are processed; editing an old comment does not
  trigger the workflow.
- Only PR conversation comments count — review-thread comments and review
  bodies are not scanned.
- Label changes are made with the Warden GitHub App token. Without it the
  workflow falls back to `GITHUB_TOKEN` and logs a warning, because events
  created with `GITHUB_TOKEN` never trigger other workflows.
- The cancel/re-run step uses `GITHUB_TOKEN` with `actions: write`; explicit
  re-runs are not subject to the `GITHUB_TOKEN` event suppression.

## Verifying the setup

On a non-draft test pull request:

1. without `warden:on`, expect the Warden check to complete green without
   analysis
2. comment `/warden run` — expect an eyes reaction, the `warden:on` label,
   and a full Warden analysis to start
3. comment `/warden skip` — expect the label to disappear and the Warden check
   to re-run and complete green without analysis
4. optionally repeat with `/reviewer skip` to confirm both reviewers respond
