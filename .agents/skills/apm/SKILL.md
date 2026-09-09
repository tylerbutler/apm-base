---
name: apm
description: >-
  Use Agent Package Manager (APM) in this repository. Load this skill before
  any APM configuration change, including edits to apm.yml, regeneration of
  apm.lock.yaml, or changes to dependencies, devDependencies, targets,
  includes, scripts, compilation settings, registries, policy, or MCP/LSP
  configuration. This applies to small edits and changes made through CLI
  commands, even when the user does not mention APM or this skill. Also use
  it for authoring or deploying .apm primitives, syncing agent files,
  dependency installs and updates, compilation, packaging, keeping content
  repository-only, and APM errors. Does not cover application performance
  monitoring or unrelated package managers.
---

# Use APM

APM manages agent instructions, skills, prompts, agents, hooks, and service
configuration. Read [the reference](references/apm.md) for command details,
manifest examples, troubleshooting, and links to the upstream documentation.

## Repository boundaries

- Edit `apm.yml` for package metadata, dependencies, and targets.
- Author content for consumers under `.apm/`. This repository uses
  `.apm/instructions/`, `.apm/prompts/`, and `.apm/skills/`.
- Treat `apm.lock.yaml` and `apm_modules/` as generated. Use APM to change
  the lockfile. Do not patch dependency files in `apm_modules/`.
- Most files under `.agents/skills/`, `.claude/`, and `.github/` are deployed
  copies. Check the lockfile's `deployed_files` and `local_deployed_files`
  before deciding who owns a file. Edit its source, then deploy it.
- **This skill is a repository-only exception.** Its source is
  `.agents/skills/apm/`; `.claude/skills/apm` links to that directory.
  Do not copy it into `.apm/`, add it to `includes`, or register it as a
  production dependency. APM does not manage these two paths.

Keep the manifest's `claude` and `copilot` targets unless the task calls for a
change. Do not use global installation or change user-wide APM settings for a
repository task.

## Work sequence

1. Read `git status --short`, `apm.yml`, and the relevant source files.
   Inspect the lockfile when dependencies or deployed files are involved.
2. Run `apm --version` and `apm <command> --help` for the operation you need.
   The reference describes APM 0.29.1; installed CLI help takes precedence
   over examples when flags differ.
3. Select the smallest operation from the table. Preview changes where
   supported, then apply the requested change. A preview is not a substitute
   for performing an authorized task.
4. Inspect the diff, including the lockfile and deployed files. Preserve
   unrelated work. Report errors rather than bypassing them.

| Task | Operation |
|---|---|
| Deploy edited skills, prompts, agents, hooks, or instructions | `apm install` |
| Add a dependency | `apm install 'owner/repo#ref'` |
| Reproduce the committed dependency resolution | `apm install --frozen` |
| Resolve dependencies without deployment | `apm lock` |
| Update a dependency | `apm update --dry-run <package>`, then `apm update --yes <package>` |
| Remove a dependency | `apm uninstall <package>` |
| Remove packages after deleting manifest entries | `apm prune` |
| Build aggregate instruction context | `apm compile --dry-run`, then `apm compile` |
| Validate instruction structure without generating output | `apm compile --validate` |
| Check installed content and policy | `apm audit` (`--ci` for a CI gate) |
| Inspect package contents without writing a bundle | `apm pack --dry-run --verbose` |
| Create a distributable archive when requested | `apm pack --archive -o ./dist` |

Use the manifest entry or direct locked key from `apm deps list` for package
selectors. Quote refs that contain shell metacharacters. Scope updates to the
requested package; omit the selector only for a requested full update.

## Distinctions that prevent mistakes

**Install and update:** `install` reuses locked resolutions where possible.
`update` refreshes dependencies and rewrites resolution data. Do not delete the
lockfile to obtain newer versions. `--force` does not mean update.

**Install and compile:** `install` deploys individual primitives. `compile`
builds instruction context such as `AGENTS.md` and `CLAUDE.md`; it does not
deploy changed skills or prompts. Do not run compile as a substitute for install.
Existing native rules can make aggregate output redundant.

**Frozen and audited:** `--frozen` checks manifest/lockfile consistency.
Use `audit` to check deployed bytes and policy. Do not assume frozen mode proves
the absence of local edits.

**Local and shipped:** with `.apm/` present, `includes: auto` selects that
source layout for local plugin content. It does not mean all repository files.
Dependency content comes from lockfile-attested deployed paths. After changing
publication boundaries, inspect the verbose pack preview and confirm that
intended content is present and repository-only content is absent.

Avoid `--force`, `--allow-insecure`, and `--trust-transitive-mcp` as repair
shortcuts. These flags relax collision or security protections. Inspect the
failure and obtain approval before accepting the associated risk. Read help
before treating any `--dry-run` as side-effect-free: uninstall previews still
run pre-uninstall scripts.
