---
name: validate-concept-plugin
description: Validate that a plugin claiming to be a "Concept" plugin correctly follows the concept/realization architecture (Tier 1 concept, Tier 2 realization contract, Tier 3 realizations, default realization, activation-check invocation), validate a workspace-authored realization skill against a concept's published contract, check that concepts reference each other only by name in the required backtick-quoted form, and detect when a concept plugin has drifted into needing more than one realization contract (or is bundling more than one capability into Tier 1) and should be split into sibling concepts. Use proactively whenever the user is authoring or editing a plugin under plugins/*/skills/concept/, plugins/*/skills/realization-contract/, or plugins/*/skills/realize-*/ — this always includes scanning for leaked cross-concept references (check C), not just structural checks — whenever a consuming workspace is authoring its own realization skill for a published marketplace's concept, or whenever the user explicitly asks to check/validate/audit a concept plugin, a workspace realization, cross-concept references, or whether a plugin should be split.
---

# Validate Concept Plugin

Checks the concept plugins of a published marketplace, and
workspace-authored realizations of them, against the concept/realization
architecture described in
[docs/architecture.md](../../../../docs/architecture.md). Read that
document first if you haven't already — this skill enforces it, it
doesn't restate it.

This skill only applies to plugins/skills that opt into the pattern. It
does not apply to plain, single-tier plugins, and should never demand that
a plugin adopt this structure — only that a plugin or realization which
*has* adopted it (see "How to detect a Concept plugin" below) does so
correctly.

This skill covers four distinct checks, run independently depending on
what's being authored or what the user asks for:

- **A. Concept plugin structure** — checks 1–9 below, for a concept
  plugin of a published marketplace, living in the `plugins/` tree of the
  marketplace under validation. Check
  C (below) runs automatically as part of set A whenever a Tier 1 or
  Tier 3 file is authored or edited — it is not gated behind a separate
  audit request.
- **B. A workspace-authored realization** — see "Validating a
  workspace-authored realization" below, for a realization skill living
  outside the marketplace under validation (e.g. in a consuming
  workspace's own `.claude/skills/`) that claims to satisfy a concept
  plugin's contract.
- **C. Cross-concept reference discipline** — see "Cross-concept
  reference check" below, scanning any concept's Tier 1 or Tier 3 skill
  for leaked coupling to a specific realization, and confirming any
  correct reference uses the required backtick-quoted concept-name form
  (architecture.md §5).
- **D. Split-signal check** — check 10 below, detecting when a concept
  plugin has outgrown the one-contract-per-concept rule and should be
  split into sibling concepts instead of accommodating a second contract
  shape in place.

## How to detect a Concept plugin

A plugin is a Concept plugin if it has a
`skills/concept/SKILL.md` file. Presence of that file is the sole
signal — no `plugin.json` field is required to opt in, and neither is
having any realization-contract or realization yet (see check 4: a
Concept plugin may legitimately have `skills/concept/` and nothing
else). If a plugin has `skills/realization-contract/` or
`skills/realize-*/` but no `skills/concept/`, flag that on its own (see
check 1) rather than assuming intent.

If a plugin has none of `skills/concept/`, `skills/realization-contract/`,
or `skills/realize-*/`, it's a plain plugin — skip all checks below and
say so.

## Checks (A: concept plugin structure)

For each Concept plugin found (or the one specified by the user):

1. **Tier 1 present and abstract.** `skills/concept/SKILL.md` exists and
   its content describes the capability without naming a specific
   provider/vendor/technology in the body (a provider name appearing only
   as a passing example is fine; the operations/description should not
   assume one specific provider's API or behavior). Flag any Tier 1 file
   that reads like documentation for one particular provider. Also check,
   non-blocking:
   - the operations a realization must support are enumerated as a
     short list (verb, inputs, outputs) rather than buried in prose
   - failure modes are named in the abstract (e.g. "not found," "not
     authorized") rather than as one provider's actual error names/codes
   - **operation-group coherence (architecture.md §7).** This is a
     different axis from abstraction level above — a Tier 1 file can
     name no provider at all and still be too broad. Read the enumerated
     operations and ask whether they're all variations on one verb+object
     (e.g. "store a secret," "retrieve a secret," "delete a secret" — one
     capability) or whether they split into two or more groups a reader
     would naturally name differently (e.g. "post a chat message" and
     "send an email" — two capabilities). Also check whether the
     concept's own "when to use this" reads as one scenario or several
     unrelated ones. Either sign is evidence Tier 1 is bundling more than
     one concept — report it the same way as check 10's split-signal
     findings (non-blocking, "consider splitting," named sibling concepts
     suggested, quote the operations/scenarios that led to the finding).
     This check works from Tier 1's prose alone and applies even to a
     concept-only plugin with no Tier 2/Tier 3 yet — unlike check 10,
     it doesn't need any realization or contract to exist first. As a
     softer secondary signal, note (but don't treat as disqualifying on
     its own) an operations list that's unusually long relative to a
     typical concept's 3-6 verbs — high count alone isn't proof of a
     bundled concept, but it's worth a second look alongside the
     coherence question above.

2. **Tier 2 present and is a contract, not an implementation** (only
   required once the plugin has ≥1 realization — see check 4).
   `skills/realization-contract/SKILL.md` exists and defines the
   operations/interface a realization implements and the identifying
   name convention a realization registers under. Flag if this file
   contains actual working implementation logic rather than a
   schema/contract definition — that content belongs in a Tier 3
   realization.

3. **Tier 2 `schema.json` present and valid.**
   `skills/realization-contract/schema.json` exists, parses as valid
   JSON Schema, and includes a `contractVersion` field (blocking if
   missing entirely; non-blocking if present but missing
   `contractVersion`). Per the
   [`schema.json` convention](../../../../docs/architecture.md#the-schemajson-convention-tier-2-and-tier-3),
   this should stay minimal — flag (non-blocking) any field that looks
   provider-specific rather than something every realization would need.
   Also flag (non-blocking) if `skills/realization-contract/SKILL.md`
   contains no example `config` block satisfying the base schema — this
   is what a realization author writing a workspace's own realization
   copies as a starting point.

4. **Realization count determines which checks below apply.** Count
   `skills/realize-<provider>/SKILL.md` directories.
   - **Zero realizations, and no `skills/realization-contract/` either:**
     this is a concept-only plugin (architecture.md §4) — a legitimate,
     permanent-or-transitional end state, not a failure. Report it as
     such (see "Reporting" below) and skip checks 2, 5, 6, 7 and the
     realization-contract half of check 3 entirely; only check 1 (Tier 1)
     applies.
   - **Zero realizations, but `skills/realization-contract/` exists:**
     a contract with nothing yet implementing it. This is unusual —
     per §4 a contract is only expected to exist once a first
     realization is being added — so flag it (non-blocking) as worth
     checking whether the contract was published prematurely, but still
     run checks 2–3 against it. Skip checks 5–7 (nothing to check them
     against).
   - **One or more realizations exist:** run all checks 2–9 as normal;
     `skills/realization-contract/` is now required (blocking if
     missing, per check 2).

5. **Exactly one default realization (only if ≥1 realization exists).**
   Among all `skills/realize-*/` directories, exactly one is marked as
   the default (per whatever marker the contract specifies, e.g.
   `default: true` in frontmatter or an explicit statement in the
   realization's SKILL.md). Flag if zero are marked default (workspace
   has no out-of-the-box option) or if more than one claims to be
   default (ambiguous). Not applicable to a concept-only plugin — there
   is nothing to default to.

6. **Every realization publishes a valid, superset `schema.json`.** Each
   `skills/realize-<provider>/schema.json` exists, parses as valid JSON
   Schema, and is a superset of Tier 2's `schema.json` — every field
   `required` in Tier 2 is still `required` here (never dropped or
   loosened), and field types/meanings aren't changed. Missing entirely
   is blocking; present but not a valid superset (e.g. narrows or drops
   a required field) is also blocking, since it silently breaks the
   contract Tier 2 promises. Also check, non-blocking:
   - every property has a `description`
   - optional fields have a `default` where a sensible one exists
   - the realization's own default status (§4 of the architecture doc)
     is stated in its SKILL.md, not only in the concept skill
   - if the realization implements a direct service integration (e.g.
     calling a provider's REST API itself) rather than binding to an
     existing MCP server or CLI for that provider, its SKILL.md doesn't
     explain why (no adequate MCP/CLI exists, or the marketplace author
     chose it deliberately) — per the Tier 3 best practice in
     architecture.md §7,
     binding to a preexisting MCP/CLI is the recommended first move, so
     an unexplained direct integration is worth flagging for the user
     to confirm was intentional

7. **Realization naming matches provider, not concept.** Flag
   (non-blocking) any `skills/realize-<provider>/` whose name is generic
   or repeats the concept name (e.g. `realize-default`) rather than
   naming the actual provider/technology it implements.

8. **Every realization invokes the activation check first.** Each
   realization's SKILL.md instructs Claude to run the activation check
   (from `guidance-activation-check`, if installed) before doing any real
   work, so missing/stale workspace configuration is caught with a clear
   message rather than failing inside the realization. Flag (blocking)
   any realization missing this step entirely; flag (non-blocking) one
   where the invocation is present but not the literal first instruction
   (e.g. operational steps precede it). If `guidance-activation-check`
   isn't installed for the marketplace under validation yet, note that as
   a separate, marketplace-wide gap rather than failing every realization
   individually.

9. **Naming consistency.** The realization identifying names used in each
   `skills/realize-*/SKILL.md` are unique within the plugin and match
   what Tier 2 says the naming convention should be (e.g. matching the
   directory name minus the `realize-` prefix).

10. **Split-signal check (§8 of the architecture doc): does this plugin
    actually hold two concepts?** This is a judgment call, not a hard
    rule — always report findings here as "consider splitting," never as
    a failure, and always show the evidence so the user can decide. All
    three proxies below need a Tier 2 contract and/or Tier 3 realizations
    to exist so there's something to compare — for a concept-only plugin
    (check 4) or one with a single realization so far, check 1's
    operation-group coherence sub-check is what can catch this same drift
    earlier, directly from Tier 1's prose, before a second realization
    ever gets built against an already-too-broad concept.
    Look for these structural proxies of drift:
    - **Branching language in Tier 1.** `skills/concept/SKILL.md`
      contains conditional phrasing keyed on realization type or
      provider category (e.g. "if using a chat-style realization... if
      using an email-style realization...", "for X-type providers... for
      Y-type providers..."). This is the clearest tell — Tier 1 should
      describe one uniform set of operations, not branch on how a
      realization implements them.
    - **Low schema overlap across realizations.** Compare the
      `properties` keys across all `skills/realize-*/schema.json` files
      for this concept. If they share little beyond what Tier 2's base
      schema already requires — i.e. most of what makes each realization
      "work" is realization-specific, not concept-shared — that's
      evidence the concept's single contract is straining to cover two
      different shapes of capability.
    - **Tier 2 itself branches.** `skills/realization-contract/SKILL.md`
      or its `schema.json` describes alternate/optional groups of
      required fields depending on realization "type" (e.g. a
      discriminated union, an `if/then` in the JSON Schema, or prose like
      "chat realizations must also provide... email realizations must
      also provide..."). A single contract needing internal branching to
      describe its own realizations is the same drift as Tier 1
      branching, appearing in Tier 2 instead.
    When any of these appear, report it as: what was found (with the
    quoted text or field list), why it suggests two concepts, and the
    suggested split — a name for each resulting sibling concept and a
    note that the original plugin's Tier 1 should reference the new
    sibling(s) by name (per §5) rather than absorbing their contract.

## Validating a workspace-authored realization (B)

Per §2 of the architecture doc, a consuming workspace can author its own
realization for a published marketplace's concept (e.g. in
`.claude/skills/`, outside the marketplace under validation) instead of
using one shipped by the defining marketplace. Use this check when the
user is authoring such a skill, or asks to validate one.

1. **Identify the target concept.** The workspace realization's SKILL.md
   must declare which concept and contract version it satisfies (per §1
   Tier 3 requirements). If it doesn't say, ask the user which concept
   plugin it's meant to realize — do not guess.

2. **Locate that concept's contract.** Find the concept plugin's
   `skills/realization-contract/schema.json` and `SKILL.md` (in the
   marketplace under validation, or wherever the concept plugin is
   installed). If the concept plugin isn't available to inspect, report
   that this check can't complete and say what's needed (the concept
   plugin installed or its contract files provided).

3. **Run the same checks as Tier 3 realizations in set A**, applied to
   this workspace skill instead of a `skills/realize-<provider>/`
   directory:
   - it publishes its own `schema.json` (blocking if missing)
   - that `schema.json` is a valid superset of the concept's Tier 2
     `schema.json` (blocking if it drops/narrows a required field)
   - every property has a `description` (non-blocking)
   - it invokes the activation check as its first instruction
     (non-blocking if present-but-not-first, blocking if entirely absent)
   - its declared identifying name doesn't collide with the name of an
     existing realization shipped by the defining marketplace for the
     same concept unless it's intentionally meant to override/replace it
     (ask the user if unclear)

4. **It does not need**: a matching directory name convention, a
   `default: true` marker (workspace realizations are opted into via
   `marketplace-plugin-settings.yml`'s `realization:` field, never as an
   implicit default), or to live under any particular path — those A-set
   rules are about the concept plugins of the marketplace under
   validation specifically.

Report the same way as set A: blocking vs. non-blocking, file and fix
suggestion per finding.

## Cross-concept reference check (C)

Per §5 of the architecture doc, one concept's instructions may reference
another concept by name only — written as that concept's name in
backticks (e.g. `` `secrets` ``), the required reference style — never a
specific realization, and never assuming one is active. This check runs
**proactively as part of set A**, automatically, any time a Tier 1
(`skills/concept/SKILL.md`) or Tier 3 (`skills/realize-*/SKILL.md`) file
is authored or edited and mentions another concept — it is not gated
behind an explicit "audit cross-concept references" request, though a
user can still ask for it standalone (e.g. to sweep the whole
marketplace at once).

1. **Scan for other concepts' realization names.** For each concept
   plugin found, collect the full set of realization identifying names
   registered by every *other* installed concept plugin (from their
   `skills/realize-*/SKILL.md` files).

2. **Check for leaked references.** Search this concept's `skills/concept/`
   and `skills/realize-*/` files for any of those other concepts'
   realization names appearing in the instruction text. A reference to
   another concept by its **concept name** (e.g. "use the `secrets`
   concept") is correct and expected — flag only references to a
   specific *realization* of another concept (e.g. "use
   `aws-secrets-manager`" instead of "use the `secrets` concept").

3. **Check the reference is in the required style.** Even when a
   reference correctly names the concept and not a realization, flag
   (non-blocking) one that doesn't backtick-quote the bare concept name
   (e.g. prose like "use the secrets capability" or "use Secrets" instead
   of "use the `secrets` concept") — per architecture.md §5, the
   backtick-quoted concept name is the required, sole form a
   cross-concept reference takes, precisely so it reads unambiguously as
   a reference to a term defined elsewhere (that concept's own Tier 1)
   rather than as ordinary prose. This is a mechanical, low-risk check
   (unlike check 2's judgment call on leaked realization names) — a
   correct-but-unquoted reference is a style fix, not a coupling problem.

4. **Check for hardcoded assumptions.** Flag (non-blocking) any
   instruction phrased as though a particular realization is always
   active for another concept (e.g. "since this uses AWS, assume the
   region config is set") rather than resolving it generically through
   that concept's own activation check.

Checks 2 and 4 are inherently best-effort text matching — report those
findings as suggestions to review, not certainties, since
natural-language text can mention a provider name for reasons unrelated
to a hard dependency (e.g. a comparison in documentation). Always show
the matched text so the user can judge intent. Check 3 is closer to
mechanical (does the reference use backticks around the bare concept
name, yes or no) and can be reported more confidently.

## Reporting

State which check set(s) ran (A, B, C, D, or a combination) before
listing findings, so the user knows what was and wasn't covered.

If check 4 found zero realizations and no `skills/realization-contract/`,
say so plainly and stop there for set A: *"This is a concept-only plugin
(0 realizations) — a valid, permanent-or-transitional state per
architecture.md §4. Only Tier 1 applies; checks 2, 5, 6, 7 and 3's
contract half don't apply here."* This is not a finding to fix — do not
list it under Blocking or Non-blocking.

Otherwise, group findings within each set as:

- **Blocking** — missing Tier 1, missing Tier 2 (when the plugin has
  ≥1 realization), missing or invalid Tier 2 `schema.json`, a
  realization or workspace-authored skill missing `schema.json` or
  whose `schema.json` isn't a valid superset of Tier 2's, zero or
  multiple default realizations among plugins that have ≥1 realization
  (set A only — not applicable to a workspace-authored realization or
  to a concept-only plugin), a realization/workspace skill missing the
  activation-check invocation entirely. These mean the plugin/skill
  cannot be used as documented, or silently breaks the contract, and
  should be fixed before merging/publishing/using it.
- **Non-blocking** — a `skills/realization-contract/` published with
  zero realizations yet to satisfy it, missing `contractVersion`,
  missing example config block, missing field descriptions/defaults,
  undocumented default status, an unexplained direct service
  integration where a preexisting MCP/CLI binding was preferred and no
  rationale is given, generic realization naming, activation check
  present but not first, naming drift, un-enumerated operations,
  non-abstract failure modes, or a bundled/incoherent operations group
  (check 1's breadth heuristic) in Tier 1, any set-C cross-concept
  reference finding (leaked realization name, unquoted reference style,
  or hardcoded realization assumption), and any set-D split-signal
  finding (all always non-blocking — they're evidence-based judgment
  calls for the user to weigh, never certainties). These degrade the
  guarantees the architecture doc promises (clear config errors,
  swappability, self-explanatory config, loose coupling, one contract
  per concept) but don't make the plugin unusable outright.

For each finding, give the file and a one-line fix suggestion. Do not
silently modify files — report and let the user decide, unless they've
explicitly asked you to fix issues found.
