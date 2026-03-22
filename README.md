# Claude Code GitHub Actions — Setup Guide

End-to-end guide for setting up Claude Code as an autonomous GitHub Actions agent using AWS Bedrock, with shared skills and a reusable workflow.

## Architecture

```
┌─────────────────────────────────────┐
│  longwind48/claude-skills (central) │
│  ├── skills/fix-issue/SKILL.md      │
│  └── .github/workflows/             │
│      └── claude-reusable.yml        │──── Plugin Marketplaces (runtime)
└──────────────┬──────────────────────┘     ├── antonbabenko/terraform-skill
               │ workflow_call              ├── hashicorp/agent-skills
               │                            └── Sushegaad/Claude-Skills-GRC
    ┌──────────┴──────────┐
    │                     │
┌───┴───┐           ┌────┴────┐
│ Repo A│           │ Repo B  │   ... any repo with thin caller workflow
│claude │           │claude   │
│.yml   │           │.yml     │
└───┬───┘           └────┬────┘
    │                     │
    └────────┬────────────┘
             │ OIDC
    ┌────────┴────────┐
    │   AWS Bedrock   │
    │   (us-east-1)   │
    └─────────────────┘
```

---

## Step 1: Create a GitHub App

1. Go to **GitHub Settings → Developer Settings → GitHub Apps → New GitHub App**
2. Configure:
   - **Name**: `claude-code-bot` (or any name)
   - **Homepage URL**: your org URL
   - **Webhook**: Uncheck "Active" (not needed)
   - **Permissions**:
     - Repository: Contents (Read & Write), Issues (Read & Write), Pull requests (Read & Write), Metadata (Read)
   - **Where can this app be installed?**: "Only on this account" (or "Any account" for org-wide)
3. After creation, note the **App ID**
4. Generate a **Private Key** (downloads a `.pem` file)
5. **Install the app** on each repository that will use Claude Code

## Step 2: Add GitHub Secrets

On each repo (or at the org level), add these secrets:

| Secret | Value |
|--------|-------|
| `APP_ID` | The GitHub App ID from Step 1 |
| `APP_PRIVATE_KEY` | Contents of the `.pem` private key file |
| `AWS_ROLE_TO_ASSUME` | ARN of the IAM role (created in Step 3) |

## Step 3: Set Up AWS OIDC Authentication

This lets GitHub Actions authenticate to AWS without static credentials.

### 3a. Create the OIDC Identity Provider

```bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1
```

### 3b. Create the IAM Role

Create a role `GitHubActions-ClaudeCode-Bedrock` with this trust policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
      },
      "StringLike": {
        "token.actions.githubusercontent.com:sub": [
          "repo:<GITHUB_ORG>/<REPO_NAME>:*"
        ]
      }
    }
  }]
}
```

Add a `repo:` entry for each repository that will use Claude Code.

### 3c. Attach Policies

**Required** — Bedrock access:
```bash
aws iam attach-role-policy \
  --role-name GitHubActions-ClaudeCode-Bedrock \
  --policy-arn arn:aws:iam::aws:policy/AmazonBedrockFullAccess
```

**Optional** — Read-only access for infrastructure auditing (allows `aws` CLI commands):
```bash
aws iam attach-role-policy \
  --role-name GitHubActions-ClaudeCode-Bedrock \
  --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess
```

### 3d. Enable Bedrock Model Access

In the AWS Console, go to **Amazon Bedrock → Model access** and enable the models you want (e.g., Claude Sonnet 4.6). Use the `us.` prefix for cross-region inference (e.g., `us.anthropic.claude-sonnet-4-6`).

## Step 4: Create the Central Skills Repo

This repo holds custom skills and a reusable workflow that all other repos reference.

```bash
gh repo create <org>/claude-skills --public --description "Shared Claude Code skills and reusable workflow"
```

### Repo structure:
```
claude-skills/
├── skills/
│   └── fix-issue/SKILL.md      # Custom skill for autonomous issue resolution
└── .github/
    └── workflows/
        └── claude-reusable.yml  # Reusable workflow
```

### Custom Skills

`skills/fix-issue/SKILL.md` — Makes Claude autonomously explore the codebase, plan, implement, and create a PR when triggered by a labeled issue. See [SKILL.md](skills/fix-issue/SKILL.md).

### Third-Party Skills (Plugin Marketplace)

Third-party skills are installed at runtime via the action's `plugins` and `plugin_marketplaces` inputs — no need to copy files:

| Marketplace | Plugin ID | What it does |
|-------------|-----------|--------------|
| [antonbabenko/terraform-skill](https://github.com/antonbabenko/terraform-skill) | `terraform-skill@antonbabenko` | Terraform best practices |
| [hashicorp/agent-skills](https://github.com/hashicorp/agent-skills) | `terraform-code-generation@hashicorp` | Terraform code generation + style |
| [Sushegaad/Claude-Skills-GRC](https://github.com/Sushegaad/Claude-Skills-Governance-Risk-and-Compliance) | `iso27001@grc-skills` | ISO 27001:2022 compliance auditing |

### Reusable Workflow

See [claude-reusable.yml](.github/workflows/claude-reusable.yml). Key features:
- AWS Bedrock via OIDC (no static credentials)
- Bot filter to prevent re-trigger loops
- `--allowedTools` for AWS CLI and `gh` CLI access
- AWS credentials + `GH_TOKEN` passed into Claude's sandbox via `settings.env`
- Plugin marketplace integration for third-party skills
- Custom skills cloned from this repo at runtime

## Step 5: Add Caller Workflow to Each Repo

Each repo gets a thin `.github/workflows/claude.yml`:

```yaml
name: Claude Code

on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]
  issues:
    types: [opened, assigned, labeled]
  pull_request_review:
    types: [submitted]

jobs:
  claude:
    uses: <org>/claude-skills/.github/workflows/claude-reusable.yml@main
    secrets:
      APP_ID: ${{ secrets.APP_ID }}
      APP_PRIVATE_KEY: ${{ secrets.APP_PRIVATE_KEY }}
      AWS_ROLE_TO_ASSUME: ${{ secrets.AWS_ROLE_TO_ASSUME }}
```

**Important**: This file must be on the repo's **default branch** (`main`) for `issue_comment` and `issues` events to trigger.

## Step 6: Usage

### Trigger via @claude mention
Comment `@claude <instruction>` on any issue or PR.

### Trigger via label (autonomous mode)
1. Create a GitHub issue describing the task
2. Add the `claude` label
3. Claude reads the issue, explores the codebase, implements changes, and creates a PR

### Trigger on new issues
Include `@claude` in the issue body when creating it.

---

## Troubleshooting

### Workflow not triggering
- The `claude.yml` must be on the **default branch** (`main`). Workflows on feature branches won't respond to `issue_comment` or `issues` events.
- For forks: GitHub Actions are **disabled by default**. Go to the repo's Actions tab and enable them.
- For forks: Issues are **disabled by default**. Enable via repo Settings → General → Features → Issues.

### Skipped workflow runs
- Claude's own comments re-trigger the workflow. The reusable workflow includes a `github.event.sender.type != 'Bot'` filter. If you're not using the reusable workflow, add this filter to your `if` condition.

### Claude runs out of turns
- Default is 200 turns. For large tasks, override in the caller workflow:
  ```yaml
  jobs:
    claude:
      uses: <org>/claude-skills/.github/workflows/claude-reusable.yml@main
      with:
        max_turns: 300
      secrets: ...
  ```

### Claude can't run AWS CLI commands
- AWS CLI requires explicit permission via `--allowedTools "Bash(aws *)"` in `claude_args`
- AWS credentials must be passed into Claude's sandbox via `settings.env`
- The reusable workflow handles both of these already

### Claude can't create GitHub issues or PRs via `gh` CLI
- `gh` CLI requires the `GH_TOKEN` environment variable in Claude's sandbox
- Must also be allowlisted: `--allowedTools "Bash(gh issue *),Bash(gh pr *)"`
- The reusable workflow handles both of these already

### Invalid Bedrock model ID
- Use the `us.` prefix for cross-region inference: `us.anthropic.claude-sonnet-4-6`
- List available models: `aws bedrock list-foundation-models --region us-east-1 --query "modelSummaries[?contains(modelId, 'claude')]"`

### GitHub App not installed
- The GitHub App must be **installed** on each repo (not just created). Go to the app's settings → Install App → select the repo.
- If you can't install on an org (e.g., `aws-samples`), fork the repo to your own account and install there.
