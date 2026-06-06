---
description: Safe-refactoring checklist and sequencing — top-down analysis, bottom-up edits, mandatory pre-refactor impact analysis. Load whenever the task is a refactoring.
alwaysApply: false
category: development
---

# Safe Refactoring — Methodology

Refactoring is high-risk because the user-visible behaviour must stay identical while the code shape changes. Use the hybrid approach below.

## Sequencing

1. **Top-down analysis first.** Map the entire chain of calls and data flow for the area you intend to change. Do not start editing until you can answer: what are the entry points, who calls what, what registers / metadata are touched, what is the observable behaviour.
2. **Bottom-up edits.** Start with the lowest-level utility functions and work upward. Higher-level callers integrate the refactored helpers only after the helpers themselves are clean and verified.
3. **No "while we're here" edits.** Out-of-scope cleanup belongs to a separate, explicit task — see `AGENTS.md → Surgical Changes`.

## Pre-refactor impact analysis (mandatory)

Run the full sequence from `tooling-playbooks.md → Refactoring` before touching the first line:

- `search_metadata` (`object_structure` plus focused facets) — passport of the object being refactored.
- `search_metadata` usage / movement / call-graph templates → fallback `rlm-tools-bsl` helpers (`find_references_to_object`, `find_code_usages`, `find_register_movements`) — what breaks on change.
- `search_metadata` (`list_callers_of_routine` / `call_graph_subtree`) → fallback `rlm-tools-bsl` helpers (`find_call_hierarchy`, `find_callers_context`) — every caller of the routine.
- `search_metadata` (`find_objects_using_object` / `find_usages_of_object`) — every type reference before renaming / removing / changing structure.
- For registers — `search_metadata` (`find_documents_making_movements_into_register`) — every document that posts movements there.

If the impact-analysis MCPs are not exposed in the session, follow the graceful-degradation procedure from `verification-checklist.md → Gate 4` — do not refactor blind.

## Post-refactor verification

- `search_code` → fallback `rlm-tools-bsl` helpers (`git_search`, `find_code_usages`) — confirm no remaining references to the old names / patterns.
- Full closing gate — `verification-checklist.md` (every gate, no skipping).
- If the refactor is large enough to enter the subagent pipeline — `subagent-pipeline.md → Stage 3` (delegate to `1c-refactoring`).

## What this file is **not**

- Not a tool catalogue — that lives in `tooling-playbooks.md → Refactoring`.
- Not a list of anti-patterns to fix — that lives in `anti-patterns.md`.
- Not an architectural guide — that lives in `dev-standards-architecture.md` and the `1c-architect` subagent.
