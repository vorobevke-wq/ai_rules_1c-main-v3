---
description: Unified "done" gate for any non-trivial 1C change — syntax, logic, style, impact, metadata
alwaysApply: false
category: quality
---

# Verification Checklist — the closing gate

**When to load this file:** before declaring **any** non-trivial change "done" — whether the change came from a direct edit, the subagent pipeline (`subagent-pipeline.md`), the OpenSpec apply phase, or a debug fix (`systematic-debugging.md`).

**Companion files:** `tooling-playbooks.md` (per-task tool sequences), `subagent-pipeline.md` (where this gate sits in the workflow), `systematic-debugging.md` (the bug-specific extension of phase 4).

The checklist is adapted from the `verification-before-completion` skill of [obra/superpowers](https://github.com/obra/superpowers) and binds the 1C-specific MCP toolchain into a single closing gate. It does not introduce new tools — it just defines the order and the stopping criteria.

## Hard gates — run on every full-cycle change

You MUST run all five gates in order. Each gate has an explicit pass / fail criterion and an explicit retry budget.

### Gate 1 — BSL Diagnostics (`diagnostics`)

- Run `diagnostics` on every touched `.bsl` module. No exceptions.
- Pass criterion: zero `error` items. `warning` items are reviewed in Gate 3.
- Retry budget: **1 call by default, up to 3 only if the previous run returned a substantive defect** (per `AGENTS.md → MCP Tool Calling → B. Limits and non-determinism`). Gates 2 and 3 share the same per-validator policy. A "cycle" is one logical edit of one module; every new edit starts a new cycle. Style / formatting / BSLLS noise alone does **not** justify a re-run.

If the budget is exhausted with substantive errors remaining: fix the substantive errors, document any remaining style warnings in the delivery summary, and stop. Do not loop infinitely on style noise.

### Gate 2 — Logic, performance, style & ITS (`check_1c_code`, `check_type="review"`)

- Run on every touched module. Always after Gate 1 passes — never before, otherwise the AI checker drowns in syntax noise.
- Pass criterion: no `critical` or `error` severity items.
- `warning` items: triage. Inside-scope warnings (introduced by your change) — fix. Pre-existing warnings outside your scope — leave alone (Surgical Changes).
- AI non-determinism rule: if `check_1c_code` returns inconsistent results across runs on the **same** code, do not loop on it. Take the strictest result, fix what is fixable, document the rest.

### Gate 3 — Style & ITS compliance (covered by Gate 2 review mode)

- If Gate 2 already ran with `check_type="review"`, this gate is satisfied.
- If you only ran a syntax-only probe earlier, follow it with one review pass using `check_type="review"` before delivery.
- Pass criterion: no `error` severity items.
- `warning` items: same triage rule as Gate 2.
- For specific warnings that are intentional and justified: add a `//BSLLS:<rule>` suppression with a 1-line explanation, per `dev-standards-core.md §2 → "Formatting"`. Blanket suppressions without justification are forbidden.

### Gate 4 — Impact analysis (only when public surface changed)

Skip this gate **only** when the change is fully internal:

- a private procedure of a non-export common module;
- a procedure of a form module that has no `Экспорт`;
- a comment / docstring / `//BSLLS:` suppression edit.

In every other case run impact analysis:

- **`search_metadata`** call-graph templates (`list_callers_of_routine`, `list_callees_of_routine`, `call_graph_subtree`) on every changed export procedure / function to find callers and downstream calls.
- Fallback to **`rlm-tools-bsl`** helpers (`find_references_to_object`, `find_code_usages`, `find_call_hierarchy`, `find_callers_context`) when Metacode is unavailable or non-actionable.
- For metadata changes (new attribute, renamed object, removed attribute): **`search_metadata`** with `find_objects_using_object` + `find_usages_of_object` template operations to list every metadata reference that needs to be reviewed.

Pass criterion: every caller / dependent listed by impact analysis was either not affected by the change, or explicitly handled in the plan, or explicitly noted as a follow-up risk in the delivery summary. Silent breakage of downstream code is a defect.

**Graceful degradation — when no impact-analysis tool is exposed.** If neither Metacode `search_metadata` impact templates nor `rlm-tools-bsl` impact helpers are available in the current session (none of the relevant MCPs are running), do **not** silently skip the gate. Instead:

1. State the fact explicitly in the Delivery summary under **Risks** as a fixed line: *"Impact analysis not run — no graph / rlm-tools-bsl MCP exposed in this session; downstream callers and metadata references were not enumerated."*
2. For metadata changes, perform a best-effort manual review based on what the agent already knows about the change (which forms / modules / queries touch the affected object) — list those callers as candidates that still need review, marked as such.
3. Do not promote a quick-fix to "verified" if a metadata or public-API change went through without impact analysis. If the change is risky and the user cannot accept the residual risk, hand off to a session that has the MCP exposed.

Skipping the gate without recording it under Risks is a defect.

### Gate 5 — Metadata XML validation (only when XML was edited)

Skip this gate **only** when no metadata XML was touched.

When XML was edited:

- Relevant `1c-metadata-manage` validator on every modified XML file (`form-validate`, `meta-validate`, `role-validate`, `skd-validate`, `mxl-validate`, etc.). Pass criterion: zero validation errors.
- For non-trivial metadata edits (new objects, attributes, tabular sections, forms): prefer the `1c-metadata-manage` skill over hand-edited XML. If hand edits were used, additionally cross-check `metadata-xml-workarounds.md` for the recurring traps (LineNumber, PagesGroupExtInfo, Page.enabled, UID uniqueness).
- For `Form.xml` edits: also confirm the form opens in Configurator without warnings — schema validity is necessary but not sufficient.

## Soft gates — run when applicable

These gates are not always required, but their absence in the listed scenarios is a defect.

### Soft gate A — Reproduction case (debug tasks only)

For any change that originated as a bug fix:

- The exact reproduction case from `systematic-debugging.md → Phase 1` was rerun **after** the fix and no longer triggers the symptom.
- The reproduction case is documented in the delivery summary so the user can verify it.
- All temporary `ЗаписьЖурналаРегистрации("Debug.*"`, `ПоказатьЗначение`, breakpoints, hard-coded test values introduced during debugging were removed.

### Soft gate B — Plan adherence (any change with a written plan)

Triggers — apply this gate when any of the following is true:

- The change went through `subagent-pipeline.md` (Stage 4a "spec-compliance review").
- The change is an OpenSpec **apply** of an active proposal — there is a `openspec/changes/<id>/tasks.md` (and optionally `design.md` / `proposal.md` / delta `specs/`).
- The user explicitly approved a written plan in chat before implementation (numbered steps, file paths, verification points).

Checklist (same shape regardless of plan source):

- Every task / step in the plan was executed; no task was silently skipped.
- The diff against the plan was summarized file by file in the delivery report (use `git diff --name-only` to verify).
- No file outside the plan was edited; if it was, the deviation is explicitly justified in the delivery summary.
- For OpenSpec specifically: `tasks.md` ticks reflect actual completion; spec deltas under `changes/<id>/specs/` are updated when behaviour observably changed (per `sdd-integrations.md`).
- For the subagent pipeline specifically: Stage 4a (spec-compliance review by the parent agent) was executed and passed before Stage 5.

### Soft gate C — User-explicit code review (only when user asks)

If — and only if — the user explicitly requested a code review:

- The `1c-code-reviewer` subagent was invoked.
- Critical / major issues were addressed before delivery; minor issues were summarized.
- For non-review-requested tasks, gates 2 and 3 already cover the routine quality bar through `check_1c_code` — do not auto-trigger the reviewer subagent (forbidden by `subagents.md`).

## Delivery summary — what the user sees

After all gates pass, the delivery report MUST contain:

1. **What was done** — 1–3 lines, no preamble.
2. **Files changed** — every path in backticks, one line per file describing the nature of the change.
3. **Context sources** — required for non-trivial BSL / metadata changes (per `AGENTS.md → MCP Tool Calling → A.3`). List the sources actually used (templates, project code, metadata, platform / БСП / ITS docs, ITS standards) and briefly state why any normally relevant source was skipped. Skipping a relevant source silently counts as a defect. Omit this section only for docs-fix / quick-fix tasks where no BSL / metadata change was made.
4. **Risks / nuances** — only real ones. If Gate 4 fell back to graceful-degradation mode (no impact-analysis MCP exposed), record it here verbatim. If there are no real risks, omit the section.
5. **Follow-ups** — any defects observed but **not** fixed (out-of-scope dead code, pre-existing lints, downstream callers flagged by Gate 4 that need future review). Empty = omit.

Do not include in the delivery summary:

- a retelling of the user's request;
- a list of which tools you called (unless that list IS the **Context sources** section above);
- thanks, apologies, introductions, conclusions;
- markdown sections added "for structure" with no content.

## Anti-patterns

- **Skipping Gate 1** "because the edit was tiny" — `diagnostics` is the cheapest BSL gate; skipping it never saves time.
- **Running gates in the wrong order** — running `check_1c_code` before `diagnostics` wastes the AI checker on syntax-broken code.
- **Looping on AI non-determinism** — if `check_1c_code` returns different items each run on the **same** code, take the strictest set and stop. Do not burn the 3-call budget on noise.
- **Marking the task done** with un-removed `Debug.*` log entries or temporary `ПоказатьЗначение` calls — soft gate A failure.
- **Auto-running `1c-code-reviewer`** when the user did not ask — soft gate C failure, and a direct violation of `subagents.md`.
