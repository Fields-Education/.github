# .github

Shared organization workflows live here.

- `./.github/workflows/warden.yml`: canonical org-level Warden workflow for org rulesets
- `./renovate.json`: repo-local Renovate config for the hosted GitHub App; manages GitHub Actions and `mise.toml` tool pins
- `./warden-base.toml`: org-level Warden base config for shared defaults
- `./docs/warden-rollout.md`: rollout notes, org prerequisites, and ruleset instructions
- `./docs/warden-sentry-otel.md`: Warden OpenTelemetry environment setup for Sentry logs/traces
