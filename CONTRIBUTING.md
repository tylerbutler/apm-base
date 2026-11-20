# Contributing

## Release changes

Use [Trellis 1.7.0](https://github.com/tylerbutler/trellis/releases/tag/v1.7.0).
`apm.yml` is the only package-version source. The native adapter in
`trellis.toml` lets Trellis update its `version` field without changing the
other manifest content. Do not bump the version by hand.

For a user-facing change, add a native changelog fragment:

```sh
trellis changelog new --package apm-base --kind Fixed --body "Describe the fix."
```

Choose the kind that matches the change. `Fixed` produces a patch bump;
`Added` produces a minor bump. Do not add fragments for development-only or
documentation-only changes. Commit the fragment under `.changes/unreleased/`
with the change.

Before pushing, check the configuration and preview the next version:

```sh
trellis doctor
trellis version plan --json
```

After changes reach `main`, the release workflow creates or updates
`release/pending` with the version bump and generated changelog. An empty
version plan creates no release PR. Review and merge the release PR to approve
the release. The workflow validates its merge commit, creates Git tags and a
GitHub Release, then checks for more pending changes. It does not publish to a
registry.

Keep `0.2.5` unchanged and untagged during setup. The first release starts with
the next user-facing change; do not bootstrap or backfill older tags.

Exact tags such as `v0.2.6` are immutable. Each stable release also updates its
major and minor aliases, such as `v0` and `v0.2`. A new minor series leaves
older minor aliases intact. Use an exact tag or commit SHA for a fixed APM
dependency; use an alias only when updates within that series are intended.

If a release fails, correct the cause and rerun its original workflow run
before preparing another release. Do not move exact tags to repair a release.
