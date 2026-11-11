# How the guard decides

This page explains the mechanism behind the [README](../README.md). You do not need it to use
the action. The script is [`guard-installed-skill-edits.sh`](../guard-installed-skill-edits.sh).

## Failing closed

UNKNOWN is reserved for what genuinely cannot be determined: front matter no YAML parser accepts,
a `github-repo` whose value is a mapping or a sequence rather than a scalar, an unreadable commit,
a diff that cannot be computed, or `yq` missing from `PATH`. The action never treats any of those
as "no provenance", because "no provenance" is what marks a skill local and editable — it is the
verdict that permits the edit. Missing CI context is UNKNOWN too, rather than a silent pass,
with one exception: when the event supplies no base or head commit and the skill root is also
absent from the checkout, the action exits 0 with `nothing to check`.

The action exits 0 in several ordinary situations — a clean checkout with nothing to refuse, a
wholly new or wholly retired skill, and the programmed-sync exemption — but the *missing-root
early exit* is reserved for the case where the installed-skill root genuinely does not exist in
either referenced commit or, when no commits are supplied, in the checkout.

## Checkout and commits

The checkout is part of the usage, not a precondition you can assume. With no checkout every
path is absent, so a root-existence test alone would find nothing and exit 0 — a required
guard reporting success having evaluated nothing. The action therefore checks that it is
inside a git work tree first and reports UNKNOWN when it is not.

Whether the root exists is decided from the two referenced commits, not from the working
directory: every other decision this action makes reads git objects, so a sparse checkout that
omits the root — or a step that deleted it — does not read as "nothing to check" while both
commits contain it.

A missing commit and a commit without the root fail a git lookup identically, so under the
depth-1 default of `actions/checkout` the base commit's absence would otherwise be read as an
absent root. Each commit is therefore proven readable first, and an absent one is UNKNOWN.
`fetch-depth: 0` makes both commits available, to this check and to the diff.

On `merge_group` events the base and head SHAs come from `github.event.merge_group` because
that event carries no `pull_request` object. It also carries no PR identity, which is why the
programmed-sync exemption cannot be evaluated there.

## Changed paths

Changed paths come from the **merge base** (a three-dot diff), not from the two commits
directly. A base branch that advances after a PR branches — most often the daily updater
editing a synced skill — would otherwise be attributed to the PR, and the guard would refuse a
pull request that never touched that skill.

Renames are checked on both the removed and added paths, so moving a file out of a retained
synced skill is still an edit. Adding a wholly new skill directory is allowed, and so is
retiring one wholesale: removing an installed skill is a local decision, and only editing a
synced one is not.

## The skill root

Equivalent dot components and repeated separators in `skill-root` are normalised before
checking Git trees. Absolute paths and parent traversal are refused as invalid context.

With `skill-root: .` the skill directories sit at the repository root, so the first path
component of a changed file is only a *candidate*. A top-level directory that has no
`SKILL.md` at the base commit is simply not a skill — editing `.github/workflows/ci.yaml`
does not make `.github` a malformed install. Under a dedicated root the opposite holds: every
subdirectory is meant to be a skill, so one missing its `SKILL.md` is reported UNKNOWN.

## Reading provenance

Provenance is read from `metadata.github-repo` in each skill's `SKILL.md` at the pull request
**base**, not on the current path, using a real YAML parser (mikefarah `yq` v4 with
`--front-matter=extract`) rather than string matching. Block and flow style, quoted and escaped
keys, anchors, aliases and merge keys, multi-line scalars and an indented root are all resolved
rather than guessed at, and a value is classified by its YAML tag rather than by how it renders.

Provenance must be a **direct child** of the `metadata` mapping: a top-level `github-repo`, or
one nested deeper such as `metadata.source.github-repo`, is not provenance and does not mark a
skill as synced — otherwise a stray key could make a local skill permanently uneditable. A
record that reads cleanly and simply names no source repository means the skill is local, and a
local skill stays editable.

The Python `yq` wrapper shares the command name and does not support `--front-matter=extract`,
so with it installed instead every provenance lookup returns UNKNOWN. Where `yq` is absent the
guard reports UNKNOWN rather than falling back to guessing — an unreadable `SKILL.md` must never
be classified local.
