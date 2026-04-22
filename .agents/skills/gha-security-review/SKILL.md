---
name: gha-security-review
description: 'GitHub Actions security review for workflow exploitation vulnerabilities. Use when asked to "review GitHub Actions", "audit workflows", "check CI security", "GHA security", "workflow security review", or review .github/workflows/ for pwn requests, expression injection, credential theft, and supply chain attacks. Exploitation-focused with concrete PoC scenarios.'
allowed-tools: Read Grep Glob Bash
---

<!--
Attack patterns and real-world examples sourced from the HackerBot Claw campaign analysis
by StepSecurity (2025): https://www.stepsecurity.io/blog/hackerbot-claw-github-actions-exploitation
-->

# GitHub Actions Security Review

Find exploitable vulnerabilities in GitHub Actions workflows. Every finding MUST include a concrete exploitation scenario — if you can't build the attack, don't report it.

This skill encodes attack patterns from real GitHub Actions exploits — not generic CI/CD theory.

## Scope

Review the workflows provided (file, diff, or repo). Research the codebase as needed to trace complete attack paths before reporting.

Reason about the full workflow graph, not just one file at a time. The dangerous trigger, effective permissions, and attacker-controlled execution step may be separated across:

- entry workflows in `.github/workflows/`
- local reusable workflows called via `workflow_call`
- local composite actions under `.github/actions/`
- scripts, config files, or local actions executed after checkout

### Files to Review

- `.github/workflows/*.yml` and `.github/workflows/*.yaml` — all workflow definitions, including reusable workflows
- `action.yml` / `action.yaml` — composite actions in the repo
- `.github/actions/*/action.yml` and `.github/actions/*/action.yaml` — local reusable actions
- Config files loaded by workflows: `CLAUDE.md`, `AGENTS.md`, `Makefile`, shell scripts under `.github/`

### Out of Scope

- Workflows in other repositories (only note the dependency)
- GitHub App installation permissions (note if relevant)

If a caller references an external reusable workflow you cannot inspect, reason from the caller-visible trigger, permissions, ref, and inputs only. Do not assert callee-only behavior you cannot verify.

## Threat Model

Only report vulnerabilities exploitable by an **external attacker** — someone **without** write access to the repository. The attacker can open PRs from forks, create issues, and post comments. They cannot push to branches, trigger `workflow_dispatch`, or trigger manual workflows.

The primary high-confidence signal is not a privileged trigger by itself. It is a privileged workflow context paired with downstream checkout, materialization, or execution of PR-controlled refs, artifacts, caches, scripts, or local actions.

**Do not flag** vulnerabilities that require write access to exploit:
- `workflow_dispatch` input injection — requires write access to trigger
- Expression injection in `push`-only workflows on protected branches
- `workflow_call` input injection where all callers are internal
- Secrets in `workflow_dispatch`/`schedule`-only workflows

## Confidence

Report only **HIGH** and **MEDIUM** confidence findings. Do not report theoretical issues.

| Confidence | Criteria | Action |
|---|---|---|
| **HIGH** | Traced the full attack path, confirmed exploitable | Report with exploitation scenario and fix |
| **MEDIUM** | Attack path partially confirmed, uncertain link | Report as needs verification |
| **LOW** | Theoretical or mitigated elsewhere | Do not report |

For each HIGH finding, provide all five elements:

1. **Entry point** — How does the attacker get in? (fork PR, issue comment, branch name, etc.)
2. **Payload** — What does the attacker send? (actual code/YAML/input)
3. **Execution mechanism** — How does the payload run? (expression expansion, checkout + script, etc.)
4. **Impact** — What does the attacker gain? (token theft, code execution, repo write access)
5. **PoC sketch** — Concrete steps an attacker would follow

If you cannot construct all five, report as MEDIUM (needs verification).

---

## Step 1: Classify Triggers and Load References

For each workflow, identify triggers and load the appropriate reference:

| Trigger / Pattern | Load Reference |
|---|---|
| `pull_request_target` | `references/pwn-request.md` |
| `issue_comment` with command parsing | `references/comment-triggered-commands.md` |
| `${{ }}` in `run:` blocks | `references/expression-injection.md` |
| PATs / deploy keys / elevated credentials | `references/credential-escalation.md` |
| Checkout PR code + config file loading | `references/ai-prompt-injection-via-ci.md` |
| Third-party actions (especially unpinned) | `references/supply-chain.md` |
| `permissions:` block or secrets usage | `references/permissions-and-secrets.md` |
| Self-hosted runners, cache/artifact usage | `references/runner-infrastructure.md` |
| Any confirmed finding | `references/real-world-attacks.md` |

Load references selectively — only what's relevant to the triggers found.

## Step 1.5: Build the Execution Graph

Before scoring findings, trace the execution chain across local workflow boundaries:

- identify entry workflows that an external attacker can influence
- follow local `uses: ./.github/workflows/...` and `uses: ./.github/actions/...` edges
- carry the effective trigger trust and token/secret scope from caller to callee
- look for PR-controlled refs, merge refs, artifacts, caches, scripts, or config being materialized anywhere downstream
- if the trigger is introduced in one file and unsafe checkout or execution happens in another, treat that as one attack path

Broad permissions are an impact amplifier, not the root issue. Increase severity when permissions are broad, but do not report a high-confidence exploit unless attacker-controlled execution or materialization is also present.

## Step 2: Check for Vulnerability Classes

### Check 1: Privileged PR Context + PR-Controlled Materialization

Does a privileged PR-triggered workflow context later consume PR-controlled refs, artifacts, caches, scripts, or local actions?

- Look for `pull_request_target` or similarly privileged PR entrypoints
- Look across local reusable workflow and local action boundaries, not just within one file
- Look for `actions/checkout` with `ref:` pointing to PR head, PR ref, or merge ref
- Look for local actions (`./.github/actions/`) or scripts that are executed from a PR-controlled checkout
- Look for artifact, cache, or generated-file handoffs from untrusted PR execution into a privileged workflow
- Check if any downstream `run:` step, local action, or config load executes attacker-controlled content after the privileged context is established

Do not treat `pull_request_target` alone as a HIGH finding. The higher-confidence issue is privileged execution paired with attacker-controlled materialization later in the graph.

### Check 2: Expression Injection

Are `${{ }}` expressions used inside `run:` blocks in externally-triggerable workflows?
- Map every `${{ }}` expression in every `run:` step
- Confirm the value is attacker-controlled (PR title, branch name, comment body — not numeric IDs, SHAs, or repository names)
- Confirm the expression is in a `run:` block, not `if:`, `with:`, or job-level `env:`

### Check 3: Unauthorized Command Execution

Does an `issue_comment`-triggered workflow execute commands without authorization?
- Is there an `author_association` check?
- Can any GitHub user trigger the command?
- Does the command handler also use injectable expressions?

### Check 4: Credential Escalation

Are elevated credentials (PATs, deploy keys) accessible to untrusted code?
- What's the blast radius of each secret?
- Could a compromised workflow steal long-lived tokens?

### Check 5: Config File Poisoning

Does the workflow load configuration from PR-supplied files?
- AI agent instructions: `CLAUDE.md`, `AGENTS.md`, `.cursorrules`
- Build configuration: `Makefile`, shell scripts

### Check 6: Supply Chain

Are third-party actions securely pinned?

### Check 7: Permissions and Secrets

Are workflow permissions minimal? Are secrets properly scoped?

- treat broad permissions as an exploit amplifier
- check whether broader-than-needed permissions are defined in the entry workflow while unsafe execution happens in a called workflow or local action
- unnecessary privileged trigger modes are suspicious, but are not by themselves a complete exploit path

### Check 8: Runner Infrastructure

Are self-hosted runners, caches, or artifacts used securely?

## Safe Patterns (Do Not Flag)

Before reporting, check if the pattern is actually safe:

| Pattern | Why Safe |
|---|---|
| `pull_request_target` WITHOUT downstream checkout or materialization of PR-controlled refs/artifacts | Never executes attacker code |
| `${{ github.event.pull_request.number }}` in `run:` | Numeric only — not injectable |
| `${{ github.repository }}` / `github.repository_owner` | Repo owner controls this |
| `${{ secrets.* }}` | Not an expression injection vector |
| `${{ }}` in `if:` conditions | Evaluated by Actions runtime, not shell |
| `${{ }}` in `with:` inputs | Passed as string parameters, not shell-evaluated |
| Actions pinned to full SHA | Immutable reference |
| `pull_request` trigger (not `_target`) | Runs in fork context with read-only token |
| Any expression in `workflow_dispatch`/`schedule`/`push` to protected branches | Requires write access — outside threat model |

**Key distinction:** `${{ }}` is dangerous in `run:` blocks (shell expansion) but safe in `if:`, `with:`, and `env:` at the job/step level (Actions runtime evaluation).

## Step 3: Validate Before Reporting

Before including any finding, read the actual workflow YAML and trace the complete attack path:

1. **Read the full workflow graph** — don't rely on grep output alone
2. **Trace the trigger and trust boundary** — confirm the event and check `if:` conditions that gate execution
3. **Trace checkout/materialization across files** — follow local `workflow_call` and local action edges to find where attacker-controlled content is actually consumed
4. **Confirm attacker control** — verify the ref, artifact, cache, comment body, title, branch name, or config is externally controllable
5. **Map permissions and secrets at each hop** — broad scope can raise severity, but should not substitute for a real execution path
6. **Check existing mitigations** — env var wrapping, author_association checks, restricted permissions, SHA pinning

If any link is broken, mark MEDIUM (needs verification) or drop the finding.

**If no checks produced a finding, report zero findings. Do not invent issues.**

## Step 4: Report Findings

````markdown
## GitHub Actions Security Review

### Findings

#### [GHA-001] [Title] (Severity: Critical/High/Medium)
- **Workflow**: `.github/workflows/release.yml:15`
- **Trigger**: `pull_request_target`
- **Confidence**: HIGH — confirmed through attack path tracing
- **Exploitation Scenario**:
  1. [Step-by-step attack]
- **Impact**: [What attacker gains]
- **Fix**: [Code that fixes the issue]

### Needs Verification
[MEDIUM confidence items with explanation of what to verify]

### Reviewed and Cleared
[Workflows reviewed and confirmed safe]
````

If no findings: "No exploitable vulnerabilities identified. All workflows reviewed and cleared."
