# Hud Agent Runner

A reusable GitHub Actions workflow that polls Hud Workflow Manager for pending tasks, executes them in parallel using claude agent sdk, and reports results back.

## Authentication

The runner authenticates to Hud via **GitHub OIDC**
The calling job must grant `permissions: id-token: write`

## Quick Start

Create a workflow file in your repository (e.g. `.github/workflows/hud-agent-runner.yaml`):

```yaml
name: Hud Agent Runner

on:
  schedule:
    - cron: '*/5 * * * *'
  workflow_dispatch:
    inputs:
      remaining_iterations:
        description: 'Remaining continuation iterations (decremented on each self-dispatch)'
        required: false
        default: '10'

concurrency:
  group: hud-agent-runner
  cancel-in-progress: false

jobs:
  run:
    permissions:
      id-token: write # required for GitHub OIDC → Hud token exchange
      contents: write
      pull-requests: write
    uses: code-hud/agent-runner/.github/workflows/agent-runner.yml@v1
    secrets:
      anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}
      github-token: ${{ secrets.GITHUB_TOKEN }}

  continue:
    permissions:
      actions: write
    needs: run
    if: needs.run.outputs.has_tasks == 'true'
    runs-on: ubuntu-latest
    steps:
      - name: Resolve remaining iterations
        id: resolve
        run: echo "remaining=${REMAINING_ITERATIONS:-10}" >> "$GITHUB_OUTPUT"
        env:
          REMAINING_ITERATIONS: ${{ inputs.remaining_iterations }}

      - name: Trigger next iteration
        if: steps.resolve.outputs.remaining > 0
        run: |
          REMAINING=$(( ${{ steps.resolve.outputs.remaining }} - 1 ))
          gh workflow run <your-workflow-filename>.yaml -f remaining_iterations="$REMAINING"
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          GH_REPO: ${{ github.repository }}
```
