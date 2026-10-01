# Contributing

Thanks for considering a contribution to `claude-plugin-guidance`. This
repo is the guidance marketplace — an authoring toolkit for marketplace
authors building their own Claude Code plugin marketplaces. See the
[README](./README.md) for what it is, and
[docs/architecture.md](./docs/architecture.md) for the four role names
used here and the concept/realization pattern referenced throughout this
guide.

## Before you start

- For anything beyond a small fix (typo, broken link, small doc
  clarification), please open an issue first to discuss the change.
  This is especially true for anything touching the architecture in
  `docs/architecture.md` — it's a deliberate design, and changes to it
  affect every plugin built against it.
- This repo does not redefine the official Claude Code plugin/marketplace
  standard (directory layout, manifest schema, component types). If your
  change would restate or duplicate that standard rather than linking to
  it, it will need to be reworked before merge — see "No restated
  standard" below.

## Pull request workflow

All changes land through a pull request into `main` — direct pushes
aren't accepted (branch protection requires at least one approving
review). To contribute:

1. Fork the repo and create a branch for your change.
2. Make your change.
3. Run the relevant validation skill (see below) and fix anything it
   flags.
4. Open a PR describing what changed and why.
5. A maintainer reviews and merges.

Keep PRs focused — one plugin, one doc change, or one fix per PR, rather
than bundling unrelated changes together.

## Conventions for the guidance marketplace

Every plugin here follows a small set of conventions, checked by the
`guidance-conventions` plugin's `validate-marketplace` skill:

- Plugin names are prefixed `guidance-<name>`, matching the guidance
  marketplace's own name.
- Every plugin has a `.claude-plugin/plugin.json` per the
  [official plugin reference](https://code.claude.com/docs/en/plugins-reference).
- The repo root keeps `README.md` and `LICENSE` up to date and
  cross-referenced.
- Nothing here restates the official plugin/marketplace standard — link
  to the [official docs](https://code.claude.com/docs/en/plugins-reference)
  instead of describing schema or directory layout that Anthropic
  maintains and can change independently of this repo.

If you have `guidance-conventions` installed, ask Claude to run
`validate-marketplace` against your change before opening a PR.

## Naming the roles

Use the role names defined in
[docs/architecture.md](./docs/architecture.md#the-four-roles): guidance
marketplace, published marketplace, realization marketplace, and
consuming workspace. "Tier" is reserved for Tier 1/2/3 inside one concept
plugin; never use "tier", "level" or "layer" for a role or for a
relationship between roles.

## Adding a plugin

1. Create `plugins/guidance-<name>/.claude-plugin/plugin.json` following
   the [official plugin reference](https://code.claude.com/docs/en/plugins-reference).
2. Add an entry to `.claude-plugin/marketplace.json`'s `plugins` array
   pointing at `./plugins/guidance-<name>`.
3. Run `validate-marketplace` (from `guidance-conventions`) to check it
   against the conventions above.

### If your plugin represents a swappable capability

If what you're building has more than one reasonable implementation
(e.g. it wraps a provider, a service, or a technology that a consuming
workspace might want to swap out), it should follow the concept/realization
pattern described in full in
[docs/architecture.md](./docs/architecture.md). In short:

- One **Tier 1 concept** skill (`skills/concept/`) describing the
  capability abstractly, with no provider-specific detail. This alone
  is a complete, publishable plugin — see "Concept-only plugins" below.
- One **Tier 2 realization contract** (`skills/realization-contract/`),
  with a `schema.json` sibling file, defining what any realization must
  satisfy. Required once you're adding a first realization, not before.
- One or more **Tier 3 realizations** (`skills/realize-<provider>/`),
  each with its own `schema.json` superset, at least one marked as the
  plugin's default. Prefer binding a realization to an existing public
  MCP server or CLI for that provider over implementing a direct
  service integration from scratch — see architecture doc §7.
- Each realization invokes the activation check
  (`guidance-activation-check`) as its first instruction.

**One concept plugin, one contract.** If you find yourself needing a
second, incompatible `schema.json` for the same plugin, that's a sign
the plugin is actually two concepts — split it into sibling plugins
instead (see architecture doc §8). The same drift is often visible
straight from Tier 1's prose, before any second contract exists: if your
concept's operations don't share one coherent verb+object, or its
"when to use this" reads as more than one scenario, split before writing
Tier 2 — see architecture doc §7.

**Referencing another concept.** If your plugin needs another concept's
capability, reference it by that concept's bare name in backticks (e.g.
`` `secrets` ``) — never by a specific realization's name, and never
assume one is active (see architecture doc §5). This is the only form a
cross-concept reference takes; treat it like a defined term pointing at
that concept's own Tier 1 skill.

Run `validate-concept-plugin` (from `guidance-conventions`) against your
plugin before opening a PR; it checks structure, contract/schema
validity, naming, activation-check invocation, cross-concept reference
style, and flags (non-blocking) any sign the plugin should be split or
has drifted from these conventions.

#### Concept-only plugins

You don't need a realization in hand to propose a concept. Publishing
Tier 1 alone — no contract, no realization — is a valid, complete PR if
the capability is worth naming now, whether or not anyone (including
you) is ready to build a realization for it yet. `validate-concept-plugin`
recognizes this as a concept-only plugin and doesn't ask for a Tier 2 or
Tier 3 you haven't written. Add Tier 2 and a first Tier 3 realization in
a later PR whenever one is ready — see architecture doc §4.

## Reporting bugs and requesting features

Open a GitHub issue. Include:

- What you expected vs. what happened (for a bug), or what you're trying
  to accomplish (for a feature request).
- Which plugin/skill is involved, if applicable.

## Security issues

Please do not open a public issue for a security concern. See
[SECURITY.md](./SECURITY.md) for how to report one privately.

## Licensing of contributions

This repository is licensed under Apache 2.0 with an added public
attribution requirement — see [LICENSE](./LICENSE). By submitting a
pull request, you agree that your contribution is licensed under the
same terms.
