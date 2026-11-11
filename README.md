# Guard Installed Skill Edits

Flag pull-request edits to agent skills installed from another repository.

Use this action when your repository checks in skills installed with `gh skill`.
Editing an installed copy can lose your changes the next time the skill is updated.
The action fails the pull-request check and names the skill's source repository,
so contributors know where to make the fix.

For example, if `.agents/skills/ways-of-working/SKILL.md` was installed from
`devantler-tech/agent-plugins`, a pull request editing that copy fails the check. Make the
change in `devantler-tech/agent-plugins`, then update the installed skill.

| Pull-request change | Result |
| --- | --- |
| Edit a skill installed from another repository | Check fails and names its source repository |
| Move or delete a file inside an installed skill | Check fails |
| Add a new skill | Check passes |
| Edit a skill maintained locally | Check passes |
| Remove an entire skill | Check passes |
| Update skills through the configured updater | Check passes on the pull request when its account and branch match the configured inputs |

A skill counts as installed when its `SKILL.md` on the pull request's base branch names a source
repository: a non-empty `github-repo` value directly under `metadata` in its front matter. A
skill without one is maintained locally.

## Usage

Create `.github/workflows/guard-skills.yaml`:

```yaml
name: Guard installed skill edits
on:
  pull_request:
  merge_group:
    types: [checks_requested]
permissions: {}

jobs:
  guard:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - name: Checkout
        uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        with:
          fetch-depth: 0
          persist-credentials: false

      - name: Guard installed skill edits
        uses: devantler-tech/guard-installed-skill-edits-action@0b57d72a07a7d2d1b4100b4a84985d03401e18ae # v1.0.2
```

The job needs no secrets. Before you adapt it:

- **Keep `fetch-depth: 0`.** The guard compares the pull request's base and head commits, and
  the default checkout fetches only one commit. Without both, the check fails as `UNKNOWN`.
  Fetching the base ref explicitly also works.
- **Run it where mikefarah [`yq`](https://github.com/mikefarah/yq) v4 is on `PATH`.**
  GitHub-hosted Ubuntu runners include it. The Python `yq` wrapper has the same command name
  but cannot read `SKILL.md` front matter, so with it every pull request that touches a skill
  fails as `UNKNOWN`. Check with `yq --version`.
- **Merge queues do not apply the updater exemption.** A merge-queue run has no pull-request
  account or branch to match, so the updater's change is checked like any other edit. If your
  updater's pull requests go through a merge queue, exempt them in your own workflow.

To block merging when the check fails, make `guard` a required status check in your branch
protection or ruleset. The action reports the result; the repository rule enforces it.

The example pins verified releases. Check the
[action releases](https://github.com/devantler-tech/guard-installed-skill-edits-action/releases)
when choosing a newer version.

## Inputs

| Input | Purpose | Required | Default |
| --- | --- | --- | --- |
| `skill-root` | Repository-relative directory that holds the installed skills. Use `.` when skills sit at the repository root. | no | `.agents/skills` |
| `sync-actor` | Exact GitHub login of the account that opens skill-update pull requests. A GitHub App's login ends in `[bot]`. | no | `botantler-1[bot]` |
| `sync-branch` | Exact head-branch name of the skill-update pull request | no | `deps/agent-skills-update` |

Set `sync-actor` and `sync-branch` to your own updater's account and branch. The default branch is
the one the [update-agent-skills](https://github.com/devantler-tech/update-agent-skills-action)
example uses, and the default account is this organisation's own bot. The full interface is in
[`action.yml`](action.yml).

## Results

The action has no outputs. It reports through the check result and the job log:

| Result | Log message | What to do |
| --- | --- | --- |
| Pass | none, `nothing to check`, or `programmed sync (...); exempt` | Nothing. |
| Fail | `refused: <path> edits synced skill <skill> (upstream: <repo>)` | Make the change in `<repo>`, then update the installed skill. |
| Fail | `UNKNOWN: commit <sha> is not present in this clone` or `could not determine changed paths` | Fetch full history in the checkout; see [Usage](#usage). |
| Fail | `UNKNOWN: not inside a git work tree` | Add a checkout step before the action. |
| Fail | `UNKNOWN: BASE_SHA and HEAD_SHA are required` | Run the workflow on `pull_request` and `merge_group` events only. |
| Fail | `UNKNOWN: <path> at base carries a metadata mapping this guard cannot parse` | Check the runner's `yq` (see [Usage](#usage)). If the front matter of the named `SKILL.md` is invalid, repair it on the base branch as described below. |
| Fail | `UNKNOWN: cannot read <path> at base` | Git could not read the file from the base commit. Check out full history without a partial-clone `filter`. |
| Fail | `UNKNOWN: <path> exists at base but SKILL.md is missing` | Restore the skill's `SKILL.md` on the base branch as described below. |
| Fail | `UNKNOWN: skill-root must be repository-relative` or `must not contain parent traversal` | Correct `skill-root`. |

`UNKNOWN` means the guard could not check the change. It fails then, and never passes a change
it could not check.

The guard reads `SKILL.md` from the base branch, so a pull request that repairs an invalid or
missing `SKILL.md` there fails the same way. Repair it with the updater's pull request, which is
exempt, or remove the whole skill directory in one pull request and add it again in the next.

`nothing to check` means the `skill-root` directory does not exist, so the check passes
without examining any skill. If your skills live somewhere else, set `skill-root` to that
directory, or the guard protects nothing.

How the guard reads front matter and decides which files changed is described in
[How the guard decides](docs/how-it-works.md).

## Maintainers

Release and release-recovery steps are in [Releasing](docs/releasing.md). Repository
conventions are in [`AGENTS.md`](AGENTS.md).
