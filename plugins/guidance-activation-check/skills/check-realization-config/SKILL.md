---
name: check-realization-config
description: Pre-flight check invoked by a concept-plugin realization before it does real work, or by a user asking to check/audit their marketplace-plugin-settings.yml. Validates that a workspace's selected realization and its configuration are present, current, and complete for a given concept. Use before executing any concept realization's actual behavior, and whenever the user asks whether their marketplace plugin configuration is valid or complete.
---

# Check Realization Config

This is the activation-check convention described in
[docs/architecture.md](../../../../docs/architecture.md) (§ "Activation
check: a shared convention, not a hook"). Any concept realization's
instructions should invoke this skill as their first step, before doing
real work, so a missing or stale workspace configuration produces one
clear, early, blocking message — not a confusing failure deep inside the
realization.

## When invoked for a specific concept (normal pre-flight use)

You will be told, or must determine from context, which **concept name**
is about to be used (e.g. `secrets`) and which **realization** the
calling skill believes it is (its own registered name, from its
`skills/realize-<provider>/SKILL.md`).

1. **Locate the settings file.** Look for `marketplace-plugin-settings.yml`
   at the consuming workspace's root. If it doesn't exist, halt and report:
   *"No marketplace-plugin-settings.yml found. This concept requires one
   with a `<concept>:` entry selecting a realization and its config."*
   Do not proceed to guess defaults.

2. **Find the concept's entry.** Look for a top-level key matching the
   concept name. If missing, halt and report which key is expected.

3. **Confirm the realization matches.** Compare the entry's `realization`
   value to the realization currently trying to run.
   - If they don't match, halt: the workspace has selected a different
     realization than the one being invoked. Report both values and
     which file/skill to check.
   - If the configured realization doesn't correspond to any installed
     realization (shipped by the defining marketplace or
     workspace-authored) — for example, after an update to the defining
     marketplace renamed or removed it — halt and report this as a
     **stale configuration**, naming the missing realization and pointing
     at `marketplace-plugin-settings.yml` as the file to fix.

4. **Validate the config block.** Read the matched realization's
   `skills/realize-<provider>/schema.json` (a JSON Schema document, per
   the [`schema.json` convention](../../../../docs/architecture.md#the-schemajson-convention-tier-2-and-tier-3),
   which must be a superset of the concept's `skills/realization-contract/schema.json`).
   Validate the entry's `config:` block against it:
   - Every field listed in the schema's `required` array is present.
   - Every field's value matches the schema's declared `type`.
   If anything is missing or invalid, halt and report each specific
   field (using the schema's `description` if present), what's expected,
   and that the fix is in `marketplace-plugin-settings.yml` under
   `<concept>.config`.

5. **Only if all checks pass**, report success in one line and allow the
   calling realization to proceed. Do not perform the realization's actual
   work yourself — this skill only gates it.

## When invoked directly by a user ("check my config", "audit my settings")

Run the same checks as above, but for every concept entry present in
`marketplace-plugin-settings.yml` (not just one), plus:

- Flag any concept entry in the settings file that doesn't correspond to
  any installed concept plugin (leftover config for a removed plugin).
- Flag any installed concept plugin (has `skills/concept/`) with no entry
  in the settings file at all — it will fall back to its default
  realization (per the architecture doc, §4) but may still need `config`
  filled in; check the default realization's `schema.json` and report
  what's needed if anything is required.

Report results grouped by concept, each as pass/fail with specifics —
never just "invalid," always the field and file to fix.
