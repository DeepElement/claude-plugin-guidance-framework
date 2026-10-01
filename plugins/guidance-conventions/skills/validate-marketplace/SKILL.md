---
name: validate-marketplace
description: Validate a Claude Code plugin marketplace repository against DeepElement's guidance conventions (naming, required docs, attribution). Use when the user asks to check, lint, audit, or validate a plugin marketplace repo, or its plugins, for convention compliance.
---

# Validate Marketplace

Checks the marketplace under validation, a Claude Code plugin marketplace
repository (typically a published marketplace authored with the guidance
marketplace), against DeepElement's guidance conventions. This skill does
**not** validate the plugin/marketplace file schema itself (required
fields, allowed types, directory layout) — that standard is owned and
maintained by Anthropic and evolves independently of this plugin. For
schema validity, defer to the official docs:

- Plugins reference: https://code.claude.com/docs/en/plugins-reference
- Creating plugins: https://code.claude.com/docs/en/plugins
- Plugin marketplaces: https://code.claude.com/docs/en/plugin-marketplaces

If a file doesn't parse as valid JSON, or is missing fields the docs above
describe as required, report that as a **schema issue** and point the user
to the relevant doc page above rather than guessing at what's required —
the schema can change and this skill should not be a second source of truth
for it.

## What this skill checks

Everything below is a DeepElement convention on top of the standard, not
part of the standard itself.

1. **Marketplace naming.** `.claude-plugin/marketplace.json`'s `name` field
   should describe the marketplace's own identity (e.g. matches the repo
   purpose), and its `plugins` entries should each resolve to a real
   directory under `plugins/`.

2. **Plugin naming prefix.** Every plugin directory under `plugins/`, and
   the corresponding `name` in its `.claude-plugin/plugin.json`, should
   share one consistent short prefix derived from the marketplace's own
   namespace (e.g. `guidance-<name>` in the guidance marketplace). Flag
   any plugin whose name doesn't share the established prefix.

3. **Required top-level docs.** The repo root should have:
   - `README.md` describing the marketplace, how to add it
     (`/plugin marketplace add ...`), and its plugin naming convention.
   - `LICENSE` present, and referenced from the README.

4. **No restated standard.** Scan README/docs for content that duplicates
   or paraphrases the base plugin/marketplace schema (e.g. hand-written
   lists of `plugin.json` fields, directory layout diagrams that mirror the
   official reference). Flag these as drift risk and recommend replacing
   them with a link to the relevant docs.claude.com/code.claude.com page,
   since the base standard is a living convention that can change out from
   under a static description.

5. **Attribution note (if the guidance marketplace's license model is
   reused).** If the LICENSE of the marketplace under validation contains
   a public-attribution clause (as opposed to plain MIT/Apache/BSD),
   confirm the README states the attribution requirement in plain
   language, not just by reference to a LICENSE section number.

## How to run the check

1. Locate `.claude-plugin/marketplace.json` at the repo root. If missing,
   stop and report: this isn't a marketplace repo.
2. Parse it; list every entry in `plugins`.
3. For each plugin entry, resolve its path and read
   `.claude-plugin/plugin.json`.
4. Run checks 1–5 above.
5. Report findings grouped as **Schema issues** (point to official docs)
   vs. **Convention issues** (the guidance marketplace's own opinions),
   each with the file and a one-line fix suggestion. Do not silently
   auto-fix — report and let the user decide, unless they've explicitly
   asked you to fix issues found.
