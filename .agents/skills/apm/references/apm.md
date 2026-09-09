# APM reference

Use this reference for details beyond the skill's command table. Command
examples were checked against APM 0.29.1. Upstream documentation can describe
newer behavior; check installed CLI help before using unfamiliar flags.

## Contents

- [Source and deployment](#source-and-deployment)
- [Dependencies and the lockfile](#dependencies-and-the-lockfile)
- [Targets and compilation](#targets-and-compilation)
- [Package boundaries](#package-boundaries)
- [Scripts and services](#scripts-and-services)
- [Troubleshooting](#troubleshooting)
- [Upstream documentation](#upstream-documentation)

## Source and deployment

An APM-native package has `apm.yml` and a `.apm/` source tree. The manifest
requires `name` and `version`. This repository also declares dependencies,
`targets: [claude, copilot]`, and `includes: auto`.

| Primitive | Source convention | Deployment in this repository |
|---|---|---|
| Instructions | `.apm/instructions/*.instructions.md` | `.github/instructions/` and `.claude/rules/` |
| Prompts | `.apm/prompts/*.prompt.md` | `.github/prompts/` and `.claude/commands/` |
| Skills | `.apm/skills/<name>/SKILL.md` with sibling resources | `.agents/skills/` and `.claude/skills/` |
| Agents | `.apm/agents/*.agent.md` | Target-specific agent directories |
| Hooks | `.apm/hooks/*.json` with referenced scripts | Target-specific hook files and settings |

Author commands as prompts; there is no `.apm/commands/` source directory.
Use skill frontmatter with `name` and a `description` that explains when to
load the skill. For instruction frontmatter, provide `description` and use
`applyTo` to scope the rule to file globs.

After editing a publishable skill or prompt:

```bash
apm install --dry-run
apm install
git diff --stat
git diff -- apm.yml apm.lock.yaml
```

Inspect the changed deployed files as well. The lockfile records dependency
ownership in `deployed_files` and hashes in `deployed_file_hashes`. Its
`local_deployed_files` and `local_deployed_file_hashes` describe the package's
own deployed content. Do not rewrite these records by hand.

Some target directories also contain hand-authored files, such as this
repository's APM skill link. A directory name alone does not prove APM ownership.

## Dependencies and the lockfile

Use a mapping under `dependencies`, with an `apm` list for agent packages:

```yaml
dependencies:
  apm:
    - owner/package#v1.2.3
    - owner/collection/skills/review#v2.0.0
    - git: https://github.com/owner/collection.git
      path: skills/format
      ref: v1.0.0
      alias: format-helper
  mcp: []
```

These are syntax examples, not dependencies to add to this repository.
`owner/repo/path` selects a subdirectory or supported primitive file.
For Git object entries use `ref`, not `version`. Use object-form `alias`;
do not append an `@alias` suffix to a Git shorthand reference. Marketplace
references have a separate `plugin@marketplace` syntax.

A branch or tag selects a Git ref. The lockfile records the exact resolved
commit. Tags can move upstream; a full commit SHA is the strongest Git pin.
Check the current dependency documentation before introducing semver ranges,
registries, or non-GitHub sources.

### Add or synchronize

```bash
apm install 'owner/package#v1.2.3'
```

This adds the dependency to the manifest and installs it. For object-form
dependencies, edit `apm.yml`, then run `apm install`.

Use `apm install --frozen` in CI when the lockfile must match the manifest.
It fails for a missing or out-of-sync lockfile. Do not remove frozen mode to
hide a CI failure; resolve the intended manifest change and commit the
APM-generated lockfile.

`apm lock` resolves and writes the lockfile without deploying or removing
files. Run install later when the workspace needs those resolved primitives.

### Inspect and update

```bash
apm deps list
apm outdated
apm update --dry-run owner/package
apm update --yes owner/package
```

Use the actual manifest entry or locked key where a package selector is
required. Preview changes to pinned refs before applying them. `--yes` skips
the interactive confirmation; use it after establishing the requested scope.
Review manifest, lockfile, and deployment changes together.

Prefer `apm update` over the deprecated `apm install --update`.
To upgrade the APM executable, use its installation method, such as
`brew upgrade apm` for Homebrew. `apm self-update` serves standalone installs.
Neither operation updates project dependencies.

### Remove

Use `apm uninstall <package>` to remove a declared dependency. If you have
already removed its manifest entry, use `apm prune` to remove unused packages.
Read each command's help before selecting flags. In this CLI version,
`apm uninstall --dry-run` still runs pre-uninstall lifecycle scripts.

## Targets and compilation

Keep the repository's explicit targets for repeatable output:

```yaml
targets:
  - claude
  - copilot
```

Declare `targets` or the legacy `target` field, not both. To override targets
for one install, use a comma-separated argument:

```bash
apm install --target claude,copilot
```

Repeated `--target` flags keep only the last value. `apm targets` shows target
information. The install CLI's resolution order is an explicit target flag,
the manifest, saved APM target configuration, then filesystem detection.
Check command help for compile-specific behavior.

Use compile for instructions, not for deploying skills or prompts:

```bash
apm compile --validate
apm compile --dry-run
apm compile
```

Compile does not fetch dependencies or update the lockfile. Default
distributed placement can create context files in subdirectories based on
instruction globs. Use `--single-agents` when one aggregate file is intended.
Preview `--clean` before removing obsolete generated context files.

Copilot reads `.github/instructions/`; Claude reads `.claude/rules/`.
APM can omit duplicate instruction content from aggregate files when native
rules already exist. Zero aggregate output does not by itself mean failure.
Inspect source discovery, target selection, and native rules before forcing
duplicate output.

## Package boundaries

This repository has two distinct classes of skills:

| Class | Location and ownership | Distribution |
|---|---|---|
| Package skill | `.apm/skills/`, deployed by APM | Ships to consumers |
| Maintainer APM skill | `.agents/skills/apm/`, with `.claude/skills/apm` as a relative link | Repository-only; not APM-managed |

Keep the maintainer skill out of `.apm/`, production dependencies, and explicit
`includes` entries. Do not add it to the lockfile's deployed paths.

With `.apm/` present, APM uses that layout for local content under
`includes: auto`. An explicit `includes` list is an exhaustive allow-list for
plugin packing; there is no `exclude` field. During install, the list records
publication consent but does not filter integrator discovery. Thus, excluding
a `.apm/` skill from `includes` is not a sufficient repository-only boundary.

For other maintainer tools that need APM-managed deployment, use a local-path
`devDependencies.apm` entry pointing outside `.apm/`, then `apm install --dev`.
APM excludes dev dependencies from pack output. This skill needs no such entry
because agents read its repository files without an APM deployment step.

Before shipping:

```bash
apm pack --dry-run --verbose
```

Check that the intended package skills and prompts appear, and
`skills/apm/` does not. For dependencies, pack reads lockfile-attested deployed
files and verifies recorded hashes; it does not treat `apm_modules/` as an
unrestricted source. A hand-authored skill beside deployed skills is not
automatically a dependency export.

When the user requests an artifact:

```bash
apm pack --archive -o ./dist
```

The default format in 0.29.1 is a Claude plugin bundle; the default archive is
ZIP. Consumers can install the resulting directory or archive with
`apm install <path>`. Do not pack or publish merely to deploy local edits.
The pack command can also generate plugin manifests for configured targets;
inspect the diff after writing artifacts.

Avoid deprecated `apm pack --target`: consumers choose deployment targets.
Avoid `--format apm` for installable bundles; that legacy format lacks the
plugin manifest expected by install. `--format agent-plugin` is a separate,
stricter format that cannot represent this repository's instructions, prompts,
and hooks. Consult its documentation before choosing it.

This boundary controls APM distribution, not Git visibility. Anyone who
clones the repository can read its repository-only skill.

## Scripts and services

`apm list` lists named commands in the manifest's `scripts` mapping.
This repository currently has `scripts: {}`; a `.prompt.md` file does not
automatically create an `apm run` script.

For an existing script, inspect its command before using `apm preview <name>`
or `apm run <name>`. Run and runtime support is experimental; check CLI help
before introducing it. Native prompt or slash-command use does not require
an APM script entry.

Declare MCP servers under `dependencies.mcp`, not `dependencies.apm`.
Read the MCP guide and `apm mcp --help` for discovery and configuration.
Do not invent server identifiers, put secrets in the manifest, or grant
blanket trust to transitive services to bypass an installation failure.

## Troubleshooting

| Symptom | Next action |
|---|---|
| Changed prompt or skill is absent from the agent | Check its source path and targets, then use install rather than compile. |
| Dependency did not advance | Inspect `apm outdated` and an update preview; install can reuse the lockfile. |
| Frozen install fails | Compare manifest and lockfile; resolve the intended change with APM. |
| Pack reports a hash mismatch | Inspect edits to the named deployed file, preserve intended work, then restore through install. |
| Pack reports missing provenance | Inspect the dependency's lock entry; install must record deployed files before packing. |
| Install reports a collision | Identify the local file's owner and preserve it; do not apply `--force` by default. |
| Auth, Git, or network failure | Use `apm doctor` and the authentication guide; do not expose tokens in logs. |
| Audit or policy blocks an operation | Read the finding and relevant policy; fix the cause rather than disable the check. |
| Compile produces no aggregate file | Check native-rule deduplication and instruction discovery. |
| Unknown flag or changed output format | Read installed command help and the matching upstream reference. |

## Upstream documentation

Fetch the relevant page when this reference does not cover the task. Prefer
official documentation to recalled syntax.

| Topic | Link |
|---|---|
| Documentation home | https://microsoft.github.io/apm/ |
| Package layout | https://microsoft.github.io/apm/concepts/package-anatomy/ |
| Manifest fields, includes, and dependency forms | https://microsoft.github.io/apm/reference/manifest-schema/ |
| Dependency workflows | https://microsoft.github.io/apm/consumer/manage-dependencies/ |
| Primitive and target support | https://microsoft.github.io/apm/concepts/primitives-and-targets/ |
| Skill authoring | https://microsoft.github.io/apm/producer/author-primitives/skills/ |
| Dev-only primitives | https://microsoft.github.io/apm/producer/author-primitives/ |
| Compile behavior | https://microsoft.github.io/apm/producer/compile/ |
| Bundle source selection and limitations | https://microsoft.github.io/apm/producer/pack-a-bundle/ |
| CLI reference | https://microsoft.github.io/apm/reference/ |
| Authentication | https://microsoft.github.io/apm/getting-started/authentication/ |
| MCP configuration | https://microsoft.github.io/apm/consumer/install-mcp-servers/ |
| Upstream source and documentation history | https://github.com/microsoft/apm |
