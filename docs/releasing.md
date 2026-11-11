# Releasing

For maintainers of this repository. Releases are created by
[`.github/workflows/release.yaml`](../.github/workflows/release.yaml) with semantic-release.

## Preview a release

Releases run from protected `main`. To preview the next release without writing
anything, dispatch the `Release` workflow on `main` with `dry-run` enabled:

```sh
gh workflow run release.yaml --ref main -f dry-run=true
```

A dry run previews semantic-release's next version or reports that no release is
needed. It does not recreate a missing GitHub release for an existing tag.

## Recover a failed release

If a release fails before creating its tag, repair the failure and rerun the
workflow from `main`.

If the tag was pushed but its GitHub release is missing, first verify that the
existing tag resolves to the intended reviewed commit on protected `main` and
that its CI passed. In the repository's **Releases** page, choose **Draft a new
release**, select that **existing tag**, generate and review its release notes,
and publish. Select **Publish this Action to the GitHub Marketplace** if the
release also needs to be listed there. Confirm the published release's source
commit and Marketplace version afterward.

Do not move or delete the tag, or try to edit an immutable release; use a new patch
release at the same reviewed commit if a published immutable release needs a separate
Marketplace listing.
