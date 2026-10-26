# guard-installed-skill-edits

This repository publishes the guard-installed-skill-edits GitHub Action. The shared engineering contract is [devantler-tech/monorepo AGENTS.md](https://github.com/devantler-tech/monorepo/blob/main/AGENTS.md).

## Maintenance

- Keep the root `action.yml` interface backward compatible; document any breaking change and use a breaking Conventional Commit PR title.
- Validate workflows with `actionlint` and run every script in `.github/tests/` with Bash. Required CI also exercises the actual action through GitHub's runner.
- Preserve credential boundaries, immutable source references, signed commits, refusal controls, and default-off optional behavior.
- Action code and its runtime scripts are authored here. Never fix the retired internal copy in the shared catalogue.
- Release through `.github/workflows/release.yaml` after protected merge. Marketplace publication is a separate observable result; a release alone does not prove a listing.
