# guard-installed-skill-edits

This repository publishes the guard-installed-skill-edits GitHub Action. The shared engineering contract is [devantler-tech/monorepo AGENTS.md](https://github.com/devantler-tech/monorepo/blob/main/AGENTS.md).

## Maintenance

- Keep the root `action.yml` interface backward compatible; document any breaking change and use a breaking Conventional Commit PR title.
- Run `zizmor --offline .` and every script in `.github/tests/` with Bash. Required CI exercises the actual action through GitHub's runner.
- Workflows use GitHub's [recommended self repository references](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax#example-using-an-action-in-the-same-repository-as-the-workflow-at-the-running-commit-recommended). actionlint 1.7.12 does not recognize `$/`. Restrict any compatibility filter to the exact documented root reference; GitHub's native compiler and action executions must pass before merge.
- Preserve credential boundaries, immutable source references, signed commits, refusal controls, and default-off optional behavior.
- Action code and its runtime scripts are authored here. Never fix the retired internal copy in the shared catalogue.
- Release through `.github/workflows/release.yaml` after protected merge. Marketplace publication is a separate observable result; a release alone does not prove a listing.
