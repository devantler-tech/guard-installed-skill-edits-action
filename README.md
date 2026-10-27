# Guard Installed Skill Edits

Flag pull-request edits to agent skills installed from another repository.

Use this action when your repository checks in skills installed with `gh skill`.
Editing an installed copy can lose your changes the next time the skill is updated.
The action fails the pull-request check and names the skill's source repository,
so contributors know where to make the fix.

For example, if `.agents/skills/ways-of-working/SKILL.md` was installed from
`devantler-tech/agent-skills`, a pull request editing that copy fails the check. Make the
change in `devantler-tech/agent-skills`, then update the installed skill.

| Pull-request change | Result |
| --- | --- |
| Edit a skill installed from another repository | Check fails and names its source repository |
| Add a new skill | Check passes |
| Edit a skill maintained locally | Check passes |
| Remove an entire skill | Check passes |
| Update skills through the configured updater | Check passes when its account and branch match the configured inputs |

See [Usage](#usage) for the workflow example.

## Requirements

The guard needs **mikefarah [`yq`](https://github.com/mikefarah/yq) v4 on `PATH`** — the Go
implementation, which supports `--front-matter=extract`. The Python `yq` wrapper shares the
command name and does **not** support that flag, so with it installed instead every provenance
lookup returns UNKNOWN. Check with `yq --version`.

Provenance is resolved with a real YAML parser rather than string matching, so quoting, escapes,
anchors, merge keys, flow style, null forms and comment rules are all handled correctly, and the
value is classified by its YAML tag rather than by how it renders. `yq` is preinstalled on
GitHub-hosted Ubuntu runners; where it is absent the guard reports UNKNOWN and fails rather than
silently falling back to guessing — an unreadable `SKILL.md` must never be classified local,
because local is the verdict that permits the edit.

The guard needs **both the base and head commits present locally**. The default
`actions/checkout` fetches depth 1, so the base commit is absent and the diff
cannot be computed; give the checkout enough history (`fetch-depth: 0`, or fetch
the base ref explicitly). When the diff cannot be computed the guard reports
UNKNOWN and fails — it never treats an unreadable diff as "nothing changed", and it does not
treat a missing work tree as "nothing to check" either.

`merge_group` is supported: that event carries no `pull_request` object, so the
SHAs fall back to `github.event.merge_group.base_sha` / `head_sha`. Note that a
merge-queue run carries no PR identity, so the programmed-sync exemption cannot
be evaluated there — a consumer whose updater PR passes through a merge queue
should exempt that lane at the workflow level.

## Inputs

| Input | Description | Required | Default |
| --- | --- | --- | --- |
| `skill-root` | Repository-relative installed-skill root | no | `.agents/skills` |
| `sync-actor` | Exact GitHub login of the programmed updater App | no | `botantler-1[bot]` |
| `sync-branch` | Exact head-branch name of the programmed updater PR | no | `deps/agent-skills-update` |

Equivalent dot components and repeated separators in `skill-root` are normalized before checking
Git trees. Absolute paths and parent traversal are refused as invalid context. Renames check both
the removed and added paths, so moving a file out of a retained synced skill is still an edit.

## Outputs

This action has no outputs.

## Usage

Create `.github/workflows/guard-skills.yaml` with the following workflow. It checks
pull requests and merge-queue groups on a GitHub-hosted Ubuntu runner, which includes
the required `yq`.

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

The example pins verified releases and needs no additional secrets. Check the
[action releases](https://github.com/devantler-tech/guard-installed-skill-edits-action/releases)
when choosing a newer version.

To prevent merging edits that fail this check, make `guard` a required status check
in your repository's branch protection or ruleset. The workflow reports failures;
the repository rule enforces the merge requirement.

The configured updater exemption uses the pull request's account and branch.
`merge_group` events do not carry that pull-request identity, so merge queues need
their own workflow-level handling for updater changes. See [Requirements](#requirements).

## How it detects changes

The checkout is part of the usage, not a precondition you can assume. With no checkout every
path is absent, so a root-existence test alone would find nothing and exit 0 — a required
guard reporting success having evaluated nothing. The action therefore checks that it is
inside a git work tree first and reports UNKNOWN when it is not, so a missing checkout fails
loudly instead of passing silently. The action exits 0 in several ordinary situations — a clean
checkout with nothing to refuse, a wholly new or wholly retired skill, and the programmed-sync
exemption — but the *missing-root early exit* is reserved for the case where the installed-skill
root genuinely does not exist in either referenced commit.

Whether the root exists is decided from the two referenced commits, not from the working
directory: every other decision this action makes reads git objects, so a sparse checkout that
omits the root — or a step that deleted it — no longer reads as "nothing to check" while both
commits contain it.

Both commits must be present in the clone for that question to be answerable at all. A missing
commit and a commit without the root fail a git lookup identically, so under the depth-1 default
of `actions/checkout` the base commit is absent and its absence would otherwise be read as an
absent root — a required guard passing without evaluating anything. Each commit is therefore
proven readable first, and an absent one is UNKNOWN. `fetch-depth: 0` is what makes both commits
available, to this check and to the diff.

Pass a different `skill-root` when the consumer installs skills somewhere other than
`.agents/skills`. A leading `./`, a trailing `/`, or `.` for skills at the repository root are all accepted and normalised.

With `skill-root: .` the skill directories sit at the repository root, so the first path
component of a changed file is only a *candidate*. A top-level directory that has no
`SKILL.md` at the base commit is simply not a skill — editing `.github/workflows/ci.yaml`
does not make `.github` a malformed install. Under a dedicated root the opposite holds: every
subdirectory is meant to be a skill, so one missing its `SKILL.md` is reported UNKNOWN.

Provenance is read from `metadata.github-repo` in the base `SKILL.md`, using a real YAML parser.
Block and flow style, quoted and escaped keys, anchors, aliases and merge keys, multi-line
scalars and an indented root are all resolved rather than guessed at, and a value is classified
by its YAML tag. A nested `metadata.source.github-repo` is a deeper key, not provenance, so a
skill carrying one is local and editable.

UNKNOWN is reserved for what genuinely cannot be determined: front matter no YAML parser accepts,
a `github-repo` whose value is a mapping or a sequence rather than a scalar, an unreadable commit,
a diff that cannot be computed, or `yq` missing from `PATH`. The action never treats any of those
as "no provenance", because "no provenance" is what marks a skill local and editable — it is the
verdict that permits the edit.

The check keys on `metadata.github-repo` in each skill's `SKILL.md` at the pull
request **base**, not on the current path. Provenance must be a **direct child**
of the `metadata` mapping: a top-level `github-repo`, or one nested deeper such
as `metadata.source.github-repo`, is not provenance and does not mark a skill as
synced — otherwise a stray key could make a local skill permanently uneditable.
Adding a wholly new skill directory is allowed, and so is retiring one wholesale:
removing an installed skill is a local decision, and only editing a synced one is
not. The programmed `update-agent-skills` PR is exempt only when both the actor
and the head branch match. Missing CI context, an unreadable `SKILL.md`, or
provenance that cannot be read is UNKNOWN rather than a silent pass. A record that
reads cleanly and simply names no source repository means the skill is local, and
a local skill stays editable.

Changed paths come from the **merge base** (a three-dot diff), not from the two
commits directly. A base branch that advances after a PR branches — most often
the daily updater editing a synced skill — would otherwise be attributed to the
PR, and the guard would refuse a pull request that never touched that skill.

## Release recovery

Releases run from protected `main`. To preview the next release without writing
anything, dispatch the `Release` workflow on `main` with `dry-run` enabled:

```sh
gh workflow run release.yaml --ref main -f dry-run=true
```

A dry run previews semantic-release's next version or reports that no release is
needed. It does not recreate a missing GitHub release for an existing tag.
If a release fails before creating its tag, repair the failure and rerun the
workflow from `main`.

If the tag was pushed but its GitHub release is missing, first verify that the
existing tag resolves to the intended reviewed commit on protected `main` and
that its CI passed. In the repository's **Releases** page, choose **Draft a new
release**, select that **existing tag**, generate and review its release notes,
and publish. Select **Publish this Action to the GitHub Marketplace** if the
release also needs to be listed there. Confirm the published release's source
commit and Marketplace version afterward. Do not move or delete the tag, or
try to edit an immutable release; use a new patch release at the same reviewed
commit if a published immutable release needs a separate Marketplace listing.
