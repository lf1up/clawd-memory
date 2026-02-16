# Persona Dependencies Matrix

## Frontend (Aurelia)
- Skills:
  - `vercel-react-best-practices`
  - `react-best-practices`
  - `nextjs-expert`
  - `react-modernization`
  - `web-design-guidelines`
  - `vercel-composition-patterns`
- Expected project dependencies:
  - React/Next.js repo
  - Node.js + package manager

## Backend (Septimus)
- Skills:
  - `aws-infra`
  - `DevOps`
- Expected tooling:
  - backend runtime + migration tooling (repo-specific)
- Optional credentials:
  - AWS credentials/profile (only when cloud actions are requested)

## DevOps (Aquila)
- Skills:
  - `kubernetes`
  - `aws-infra`
  - `DevOps`
- Expected binaries:
  - `kubectl` (and `oc` for OpenShift)
  - `aws` (optionally `az` / `gcloud`)
- Optional setup:
  - kubeconfig/cluster context
  - CI provider access

## Testing (Cassia)
- Skills:
  - `testing-patterns`
  - `e2e-testing-patterns`
  - `web-design-guidelines`
- Expected tooling:
  - unit/integration framework in repo
  - CI test command(s)
  - Playwright/Cypress for E2E where applicable

## General Ops (Mercury)
- Skills:
  - `openclaw-mcp-plugin`
  - `openclaw-github-assistant`
  - `todoist`
  - `automation-workflows`
  - `playwright-browser-automation`
- Credentials/setup (optional by workflow):
  - GitHub: `GITHUB_TOKEN`, `GITHUB_USERNAME`
  - Todoist: `TODOIST_API_TOKEN`
  - MCP: configured MCP servers
  - Playwright: browser binaries installed

## Current Machine Readiness (last check)
- Playwright: installed and browser binaries present ✅
- Todoist CLI: installed ✅
- Todoist token: not configured yet ⚠️
- GitHub token/username: not configured yet ⚠️
- MCP server config: not configured yet ⚠️
