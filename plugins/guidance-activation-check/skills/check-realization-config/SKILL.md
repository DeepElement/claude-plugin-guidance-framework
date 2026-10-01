---
name: check-realization-config
description: Pre-flight check invoked by a concept-plugin realization before it does real work, or by a user asking to check/audit their marketplace-plugin-settings.yml. Resolves the concept and the selected realization across the marketplaces installed in the consuming workspace and the workspace's own realizations, confirms the caller is the selected realization and that its declared contractVersion range accepts the installed contract, and validates the workspace's config against the realization's schema.json. It checks config against a schema only, never calls a live API, and does not detect that an external service changed or verify credentials. Use before executing any concept realization's actual behavior, and whenever the user asks whether their marketplace plugin configuration is valid or complete.
allowed-tools: Bash(claude plugin list *), Bash(realpath *), Bash(git rev-parse --show-toplevel)
---

# Check Realization Config

This is the activation-check convention described in
[docs/architecture.md](../../../../docs/architecture.md#6-activation-check-a-shared-convention-not-a-hook).
Any concept realization's instructions invoke this skill as their first
step, before doing real work, so a missing, stale or ambiguous workspace
configuration produces one clear, early, blocking message instead of a
confusing failure deep inside the realization.

Speak to the consuming workspace's owner. This skill validates `config`
against a `schema.json` and nothing else: it never calls a live API, and
it cannot see an external service, CLI or MCP server change after the
realization was written, verify credentials or reachability, or
authenticate its caller. It shows consistency, not authenticity
([what it does not do](../../../../docs/architecture.md#what-the-activation-check-does-not-do)).

## Inputs

A realization invokes this skill with three things:

- the **concept**, as written in the realization's `realizes` block
  (`secrets@acme-concepts`), or a bare name (`secrets`);
- the realization's own **name** (`aws-secrets-manager`);
- the realization's own `CLAUDE_SKILL_DIR`, the directory holding its
  `SKILL.md` and `schema.json`. The caller writes this variable inline in
  its own `SKILL.md`, where Claude Code substitutes it; it cannot be read
  from here ([substitutions](https://code.claude.com/docs/en/skills#available-string-substitutions)).

A **legacy caller** passes names only. With no directory, this skill cannot
identify the caller by location, so it compares names literally, skips the
contract gate (step 5) and the no-entry fallback (step 4), and otherwise
runs the steps below.

## Where things are looked up

- **Workspace root:** `${CLAUDE_PROJECT_DIR}`. It is written inline in this
  `SKILL.md`; do not use `$CLAUDE_PROJECT_DIR` in a Bash command, which does
  not have it. Claude Code substitutes it from v2.1.196 onward
  ([substitutions](https://code.claude.com/docs/en/skills#available-string-substitutions));
  on an older version the placeholder stays literal, so say that the
  workspace root cannot be determined and halt, rather than guess one.
- **Settings file:** `${CLAUDE_PROJECT_DIR}/marketplace-plugin-settings.yml`.
- **Workspace-authored realizations:** the files matched by
  `${CLAUDE_PROJECT_DIR}/.claude/skills/realize-*/schema.json`, one level
  deep ([where a workspace-authored realization lives](../../../../docs/architecture.md#where-a-workspace-authored-realization-lives)).
- **Installed plugins:** step 0. Never work out another plugin's location
  from the caller's directory or from this skill's own; a sibling plugin
  has its own install path.

**Comparing directories.** Every comparison of two directories (a plugin's
`projectPath` against the workspace root, the caller's directory against a
candidate's) is made after resolving real paths with `realpath`, so a
symlink, a relative path or a trailing slash does not read as a mismatch.

**Worktrees.** A linked git worktree that has no `.claude/skills` of its
own loads the main checkout's project skills instead (Claude Code v2.1.277
or later; see
[skill discovery](https://code.claude.com/docs/en/skills#discovery-from-parent-and-nested-directories)).
A caller loaded that way has a directory outside the workspace root, the
glob above then finds none of the workspace's realizations, and a project
or local plugin recorded against the main checkout's path fails the
`projectPath` comparison, so the check could report a false "stale" or
"selected a different realization". Therefore, when the caller's directory
is under neither the workspace root nor an installed plugin's `installPath`,
or when `git rev-parse --show-toplevel` run in the workspace root does not
resolve to the workspace root, label the whole result **BEST-EFFORT**, and
add to any such halt that the session may be a worktree without its own
`.claude/skills`.

## When invoked for a specific concept (normal pre-flight use)

0. **List the installed plugins.** Run `claude plugin list --json` through
   Bash, calling the `claude` binary directly, and parse stdout only
   (ignore anything on stderr). See
   [`claude plugin list`](https://code.claude.com/docs/en/plugins/cli-reference#plugin-list)
   for the fields.
   - Keep entries whose `scope` is `user`, or whose `scope` is `project` or
     `local` and whose `projectPath` equals the workspace root. Drop the
     rest. When one `id` remains more than once, keep one entry.
   - Entries with `scope` `session` (plugins passed with `--plugin-dir` to
     this `claude plugin list` call, id `name@inline`) are dropped too: the
     list does not report the running session's own `--plugin-dir` plugins,
     and a session plugin is not installed for the workspace's other
     users. If a halt names a plugin that is only available this way, say
     that it is not visible to this check instead of calling it missing.
   - An entry's `installPath` is that plugin's root. `id` is
     `name@marketplace`.
   - A plugin is **loaded** only when `enabled` is `true` **and** it has no
     non-empty `errors`. Never treat `enabled` alone as loaded: a plugin
     whose dependency is unmet can still report `enabled: true`.
   - An unmet dependency is an `errors` entry whose `errorDetails` item has
     `type` `dependency-unsatisfied` or `dependency-version-unsatisfied`
     ([dependency errors](https://code.claude.com/docs/en/plugins/troubleshooting#dependency-errors);
     `errorDetails` needs Claude Code v2.1.268 or later, and on an older
     version read the `errors` text instead). The fields are documented
     under [JSON output](https://code.claude.com/docs/en/plugins/cli-reference#json-output).
   - **If the CLI is unavailable** (not found, a non-zero exit, or output
     that is not JSON), fall back and label every result **BEST-EFFORT**:
     read `~/.claude/plugins/installed_plugins.json` for the installed
     plugins and their `installPath`, `enabledPlugins` from the user,
     project and local settings files for which are enabled, and each
     plugin's `.claude-plugin/plugin.json` `dependencies` for which are
     installed and enabled. The fallback cannot confirm that a plugin loaded.
     Say so in the output.

1. **Locate the settings file.** Read `marketplace-plugin-settings.yml` at
   the workspace root. A missing file means there is no entry for any
   concept; carry on and let step 4 decide, and say in any halt that the
   file was not found.

2. **Resolve the concept.** A concept plugin is a loaded plugin with
   `skills/concept/SKILL.md` under its `installPath`. Its concept name is
   its plugin name without the first hyphen-delimited segment (`secrets`
   for `acme-secrets`), and its qualifier is the marketplace in its `id`.
   - **Zero matches: halt.** Say which of these applies:
     - *The defining plugin is found but not installed or enabled.* A
       plugin that would define this concept is installed but disabled or
       reporting errors, is named by another plugin's unmet-dependency
       error, or is offered by a registered marketplace without being
       installed. `claude plugin list --json --available` prints one object
       instead of an array: `installed` holds the array step 0 parses, and
       `available` holds one object per uninstalled plugin (`pluginId`,
       `name`, `marketplaceName`). Name the missing `plugin@marketplace` and give the
       remedy: `claude plugin install <plugin>@<marketplace> --scope project`,
       then `/reload-plugins`. Observed on Claude Code 2.1.286, enabling a
       relative-path plugin in settings did not by itself install its
       cross-marketplace dependencies; the docs promise neither behaviour
       ([architecture §9](../../../../docs/architecture.md#what-the-consuming-workspace-must-do)),
       so do not promise it either way.
     - *No such concept.* Name the concept, the marketplace to register
       when the name was qualified, and say the missing piece is the plugin
       that defines it.
   - **A bare name matching concepts from more than one marketplace: halt**
     and print the qualified forms, for example: "Concept `secrets` is
     defined by `acme-concepts` and `other-concepts`; pass `secrets@acme-concepts`
     or `secrets@other-concepts`."
   - **No `skills/realization-contract/` in the concept plugin: halt** with
     "no contract to realize". A concept-only plugin cannot be selected
     against.
   - Otherwise read the contract's `skills/realization-contract/schema.json`
     and keep its `contractVersion`.

3. **List the candidate realizations** for the resolved concept. A candidate
   has a name (its `realize-<name>` directory without the prefix), a source
   and a directory:
   - **The workspace's own:** each `.claude/skills/realize-*/` found above
     whose `SKILL.md` carries a `realizes` block naming this concept with
     the resolved qualifier. A workspace realization without that block is
     not a candidate; if a selection names one, say so in the halt.
   - **The defining plugin's:** its `skills/realize-*/` directories. Source
     is the defining marketplace.
   - **Realization plugins':** the `skills/realize-*/` directories of other
     loaded plugins whose `SKILL.md` `realizes` block names this concept
     with the resolved qualifier. Source is the marketplace in the plugin's
     `id`.

4. **Find the concept's entry and select.** Look for a top-level key equal
   to the bare concept name in the settings file.
   - **An entry with `realization`.** The value is a bare name or
     `realization@marketplace`.
     - A bare name that a workspace realization carries **resolves to the
       workspace's own**. The workspace wins; this is not a collision.
     - Otherwise a bare name matching installed candidates from more than
       one marketplace **halts**, printing the qualified forms to copy:
       "Realization `aws-secrets-manager` for concept `secrets@acme-concepts`
       is offered by `acme-aws-realizations` and `acme-other`; select
       `aws-secrets-manager@acme-aws-realizations` or
       `aws-secrets-manager@acme-other` in `marketplace-plugin-settings.yml`."
     - A qualified name matches installed candidates from that marketplace
       only; it is how the workspace chooses an installed one over its own.
     - **Zero matches is a stale configuration: halt.** Name the missing
       realization, the three places searched (the defining plugin, loaded
       realization plugins, `.claude/skills/realize-*/`), and
       `marketplace-plugin-settings.yml` as the file to fix. When the
       realization is offered by a plugin that is installed but not loaded,
       report that distinctly (name the `plugin@marketplace` and the remedy
       from step 2) instead of calling it stale.
   - **The caller must be the target.** Compare the selected candidate's
     directory with the caller's directory. If they differ, halt: the
     workspace selected a different realization than the one invoked.
     Report both and which file or skill to check. A legacy caller is
     compared by name.
   - **No entry** (or no file). Only a caller with a directory can take
     this path.
     - The caller is the defining plugin's default (its directory is in the
       defining plugin and it is marked the default, per
       [architecture §4](../../../../docs/architecture.md#4-a-concept-may-be-published-with-no-realization-at-all))
       **and** its `schema.json` has no required fields: run, and give the
       notice "no explicit selection; running the defining marketplace's
       default".
     - The caller is that default but its schema has required fields:
       **halt** with "no entry; this default needs config" and name the
       missing fields.
     - The caller is not the default: **halt** with "not selected", and
       name the entry to add.
     - The concept has no default: **halt** with "select one" and list the
       candidates in their qualified forms.
     [Why the default can run](../../../../docs/architecture.md#no-entry-for-the-concept).
   - An entry with no `realization` value halts, naming the key to add.

5. **Gate on the contract version.** Read the `realizes` block in the
   caller's `SKILL.md`. Its `concept` must name the resolved concept, or
   halt. Its `contractVersion` is a range over the Tier 2 `contractVersion`
   read in step 2. If the installed version does not satisfy the range,
   **halt**, giving the installed version and the range, and say that the
   realization's author needs to publish a compatible realization or the
   workspace needs to stay on a contract version it accepts. A range that
   is invalid, or an installed contract with no `contractVersion`, also
   halts.
   - A caller with **no range declared** (a legacy caller, or a defining
     plugin's own realization, where the block is only recommended) skips
     the gate; say so in a note.

6. **Validate the config block.** Read the `schema.json` beside the
   caller's `SKILL.md` (the caller's directory plus `schema.json`, a JSON
   Schema document per the
   [`schema.json` convention](../../../../docs/architecture.md#the-schemajson-convention-tier-2-and-tier-3),
   a superset of the concept's Tier 2 schema). A legacy caller has no
   directory; use the matched candidate's. Validate the entry's `config:`
   block against it, and no more than this:
   - Every field listed in the schema's `required` array is present.
   - Every field's value matches the schema's declared `type`.
   If anything is missing or invalid, halt and report each field (using the
   schema's `description` if present), what is expected, and that the fix
   is in `marketplace-plugin-settings.yml` under `<concept>.config`.

7. **Only if every step passes**, report success and let the calling
   realization proceed, with any notices from steps 0 to 5 (the
   running-the-default notice, a skipped gate, BEST-EFFORT results). Say
   what was not checked: "Config matches the realization's schema; no live
   call was made to the service." Do not perform the realization's actual
   work; this skill only gates it.

## When invoked directly by a user ("check my config", "audit my settings")

There is no caller, so the checks that compare against a caller (the
caller must be the target, the caller's contract range, the no-entry
fallback for a caller) do not apply. Run steps 0 to 4 and 6 for every
concept entry in `marketplace-plugin-settings.yml`, validating each entry
against the `schema.json` of the realization it selects, and add:

- **Leftover entries.** An entry no loaded concept plugin defines. Say
  whether a plugin that defines it is installed but not loaded (step 2)
  or nothing defines it.
- **Concepts with no entry.** For each loaded concept plugin that has a
  contract and no entry, report whether it would run on an implicit
  default (and what that default needs in `config`) or has no default, and
  say "select explicitly". A concept-only plugin has nothing to configure.
- **Ambiguous names.** Every concept name defined by more than one
  marketplace, and every realization name that more than one installed
  marketplace offers for one concept, with the qualified forms to copy. A
  name a workspace realization also carries is not ambiguous.
- **Workspace realizations.** Each `.claude/skills/realize-*/` found, the
  concept its `realizes` block names, and any with no block or a `schema.json`
  missing.

Report results grouped by concept, each as pass/fail with specifics.
Never just "invalid": always the field and the file to fix, and say that
nothing was called live.
