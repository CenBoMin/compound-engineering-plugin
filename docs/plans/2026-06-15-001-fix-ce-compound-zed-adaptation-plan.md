---
status: active
date: 2026-06-15
type: fix
title: "fix: Repair and complete the Zed adaptation of ce-compound"
---

# Fix: Repair and Complete the Zed Adaptation of ce-compound

## Summary

The `.agents/skills/ce-compound/` skill was created with significant gaps vs the source `plugins/compound-engineering/skills/ce-compound/`. 71 gaps were identified (9 high, 20 medium, 42 low severity). The most critical problems are: missing auto-invoke triggers, no context-extraction phase before `spawn_agent` (sub-agents can't see conversation history), missing CONCEPTS.md bootstrap redirect, and anemic Phase 2.5 refresh-check logic and Phase 1 research instructions.

This plan repairs the single file `.agents/skills/ce-compound/SKILL.md` through four focused implementation units.

## Problem Frame

**Source `ce-compound` assumptions incompatible with Zed's `spawn_agent`:**
- Source assumes sub-agents share the parent agent's conversation context (Claude Code "background tasks")
- Zed's `spawn_agent` creates **isolated** sub-agents with no access to parent conversation history
- The current Zed adaptation tells the orchestrator to `spawn_agent` Context Analyzer and Solution Extractor without first extracting and packaging the conversation context — these agents will have no data to work with

**Gaps from incomplete porting:**
- Auto-invoke trigger phrases not ported
- CONCEPTS.md bootstrap redirect not ported (user asks to create CONCEPTS.md → gets full doc workflow instead)
- Preconditions format changed from structured XML to plain list
- Phase 2.4 vocabulary capture missing key constraints (coherence neighborhood bounds, conservative border at creation)
- Phase 2.5 refresh check missing 2 triggers and all branching logic (when to invoke vs. recommend vs. skip)
- Lightweight mode missing CONCEPTS.md discoverability tip
- Success output formats inconsistent with source (fewer options, missing alternate output for doc-update case)

## Scope Boundaries

- **In scope:** Edit `.agents/skills/ce-compound/SKILL.md` only. Reference files (`references/`, `assets/`, `scripts/`) are already correct.
- **Not in scope:** Installing `ce-sessions`, `ce-compound-refresh`, or any other missing skill. We note where Zed can't call them but keep the structural logic for future use.
- **Not in scope:** Creating sub-agent files (Context Analyzer, Solution Extractor, Related Docs Finder are inline roles, not standalone agent files).

## Key Technical Decisions

| Decision | Rationale |
|----------|-----------|
| **KTD1. Context extraction is a separate Phase 0.5** | Placed before Phase 1's `spawn_agent` calls. The orchestrator must extract and package conversation context before any sub-agent work begins. This mirrors the source's auto-memory scan timing (Phase 0.5) and creates a clean contract. |
| **KTD2. All sub-agent context is inlined in `spawn_agent` `message`** | Confirmed pattern from other Zed skills: no skill uses `{kind: "file", path: ...}`. Parent reads files (schema, templates) and conversation context, then inlines everything. Sub-agents are fully self-contained. |
| **KTD3. `ce-compound-refresh` references are kept structurally but noted as unavailable** | The source's Phase 2.5 decision logic and argument-construction guidance is valuable even when the target skill isn't installed. Keep the branching, but note that in Zed the user must be told to implement `ce-compound-refresh` first. |
| **KTD4. Auto-invoke uses a table format** | Source uses XML `<auto_invoke>` tags. Zed convention is markdown tables (see `ce-brainstorm`, `ce-plan`). A table of trigger phrases + examples is more readable and consistent. |
| **KTD5. Phase 3 uses generic review prompts, not agent names** | The specialized agents (`ce-performance-oracle`, etc.) don't exist as Zed skills. If they're added later, the `spawn_agent` call with a domain-specific prompt is trivially upgradeable to name a known agent. |

## Implementation Units

---

### U1. Head Section: Frontmatter, Auto-Invoke, Mode Detection, CONCEPTS.md Bootstrap, Preconditions

**Goal:** Restore the complete head section with all missing structural elements. This anchors the rest of the skill in a discoverable entry point.

**Dependencies:** None (first unit).

**Files:** `.agents/skills/ce-compound/SKILL.md` (lines 1-55, current head section)

**Approach:**
- Frontmatter stays as-is (`target: zed` is correct)
- Add Auto-Invoke table (trigger phrases: "that worked", "it's fixed", "working now", "problem solved" + manual override note)
- Fix Mode Detection to mention `$ARGUMENTS` parsing explicitly and strip `mode:` tokens
- Add Mode Detection note: in headless mode, strip `mode:headless` and use remainder as context hint
- Add CONCEPTS.md bootstrap redirect section: if user asks specifically to create/bootstrap CONCEPTS.md, redirect to `ce-compound-refresh` (noting it's not yet installed), then exit
- Restore `<preconditions>` XML format with three `<check>` elements
- Keep Support Files section as-is (correct)
- Keep Pre-resolved Context section as-is (correct)

**Patterns to follow:**
- Auto-invoke format: match `ce-plan`'s table style (source uses `<auto_invoke>` XML, but Zed skills use markdown)
- Preconditions: restore source's XML-like `<preconditions enforcement="advisory">` with `<check condition="...">` elements

**Test scenarios:**
- Happy path: user runs `/ce-compound`, sees auto-invoke triggers listed in the skill description, mode detection correctly parses `mode:headless`, preconditions are checked before proceeding
- Edge case: user invokes specifically to create CONCEPTS.md → redirected correctly (not running full doc workflow)
- Error/failure: headless mode with unsolved problem → structured error output

**Verification:** Read SKILL.md lines 1-60; confirm auto-invoke table present, CONCEPTS.md redirect present, `<preconditions>` block present, mode detection mentions `$ARGUMENTS` parsing.

---

### U2. Execution Strategy + Phase 0.5 Context Extraction

**Goal:** Add the critical context pre-extraction phase that enables `spawn_agent` sub-agents to function correctly. Also fix the Full/Lightweight mode selection flow.

**Dependencies:** U1 (head section must define mode detection first).

**Files:** `.agents/skills/ce-compound/SKILL.md` (Execution Strategy section, Full Mode section)

**Approach:**
- Execution Strategy: add headless-mode shortcut ("skip Full vs Lightweight, go to Full")
- Add Full vs Lightweight chat options (keep existing, they're correct)
- Add Phase 0.5 Context Extraction before Phase 1 with:
  - Explanation: "Sub-agents spawned via `spawn_agent` start with a blank conversation — they cannot read the parent agent's chat history"
  - Structured extraction: orchestrator extracts problem description, error messages/symptoms, investigation steps, working solution, root cause, module/area, conversation context hint from `$ARGUMENTS`
  - Package as `## Conversation context for compound documentation` block
  - In headless mode: if context too thin, emit structured failure and `Documentation skipped`

**Patterns to follow:**
- `ce-simplify-code` pattern: parent extracts context, inlines into spawn message

**Test scenarios:**
- Happy path: orchestrator extracts conversation context → all sub-agents receive it in their spawn message → correct doc is produced
- Edge case: conversation is sparse (user just said "mode:headless [topic]") → structured failure emitted
- Error/failure: headless mode, no solved problem detected → `Documentation skipped` output

**Verification:** Phase 0.5 appears between Execution Strategy and Phase 1. It instructs the orchestrator to extract conversation context and explains why. Headless mode thin-context fallback is specified.

---

### U3. Phase 1 Research: spawn_agent Patterns + Search Strategy + GitHub Issues

**Goal:** Fix the three sub-agent definitions to use the correct Zed pattern (inlined context + reference content in `spawn_agent` message). Add the missing search strategy for Related Docs Finder. Add GitHub issue search.

**Dependencies:** U2 (Phase 0.5 context extraction provides the conversation context block that Phase 1 sub-agents receive).

**Files:** `.agents/skills/ce-compound/SKILL.md` (Phase 1 section)

**Approach:**

**Context Analyzer:**
- Add note: "receives the conversation context block from Phase 0.5" as first step
- Inline schema.yaml and yaml-schema.md content into spawn message (not "reads file" — parent passes content)
- Keep track classification logic (bug/knowledge)
- Add: "Does not force bug-track fields onto knowledge-track learnings or vice versa"

**Solution Extractor:**
- Same context-block note
- Inline schema.yaml content
- Add concrete-code-examples detail to Prevention section: "Include concrete code examples where applicable (e.g., gem configurations, test assertions, linting rules)"

**Related Docs Finder:**
- Add full 7-step search strategy:
  1. Extract keywords (module names, technical terms, error messages)
  2. Narrow to matching `docs/solutions/<category>/` directory if category is clear
  3. Use grep to pre-filter candidates by frontmatter fields
  4. If >25 candidates, re-run with specific patterns; if <3, broaden
  5. Read only frontmatter (first 30 lines) to score relevance
  6. Read only strong/moderate matches fully
  7. Return distilled links
- Add GitHub issue search: prefer `gh issue list --search` CLI, fall back to GitHub MCP tools if `gh` not available, skip if neither available
- Add overlap assessment with 5 dimensions (keep existing — already correct)

**Spawn agent dispatch:**
- Show concrete example: each spawn_agent message MUST include the conversation context block from Phase 0.5
- Show that reference file contents are inlined, not referenced by path

**Patterns to follow:**
- `ce-plan`'s use of `spawn_agent` with inlined `references/researchers.md` prompt content
- `ce-code-review`'s self-contained prompt pattern

**Test scenarios:**
- Happy path: all three sub-agents receive correct context + schema → return structured text → orchestrator assembles correct doc
- Edge case: Related Docs Finder finds >25 candidates → narrows search, returns focused results
- Edge case: `gh` CLI not available → falls back to grep only, notes gh was skipped
- Error: sub-agent returns unusable data → orchestrator handles gracefully (extract what's useful, note gaps)

**Verification:** Phase 1 shows `spawn_agent` calls with inlined context. Related Docs Finder has 7-step search strategy. GitHub issue search mentioned with fallback chain.

---

### U4. Phase 2 + Phase 2.4 + Phase 2.5 + Discoverability + Phase 3 + Lightweight + Output

**Goal:** Fix all remaining downstream sections — assembly, vocabulary capture, refresh check, discoverability, Phase 3, lightweight mode, and success output formats.

**Dependencies:** U1, U3 (definitions from earlier sections must be stable).

**Files:** `.agents/skills/ce-compound/SKILL.md` (Phase 2 through end of file)

**Approach:**

**Phase 2 Assembly:**
- Add overlap rationale paragraph: "The reason to update rather than create: two docs describing the same problem and solution will inevitably drift apart. The newer context is fresher and more trustworthy, so fold it into the existing doc."
- Add `last_updated: YYYY-MM-DD` frontmatter field when updating existing doc due to high overlap
- Add `<sequential_tasks>` wrapper hint (markdown note, not XML)

**Phase 2.4 Vocabulary Capture:**
- Add: "Do not pre-judge from memory that nothing qualifies — the reference's criteria are non-obvious"
- Add conservative border guidance for new file creation: "At creation, hold the qualifying bar conservatively for borderline terms. A borderline term defers to a later run."
- Add coherence neighborhood bounds: "neighborhood only, never a full-file audit; refresh only on evidence already in hand; if judging a neighbor would require investigation this learning did not do, flag it for a future refresh"
- Add: "If no terms qualified, record that outcome explicitly — the visible scan-and-no-result record is the audit signal"

**Phase 2.5 Refresh Check:**
- Add missing trigger: "A pattern doc now looks overly broad, outdated, or no longer supported by the refreshed reality"
- Add missing non-trigger: "Refresh would require a broad historical review with weak evidence"
- Add branching decision logic:
  - One obvious stale candidate → recommend with narrow scope hint
  - Multiple candidates in same area → ask user (interactive) / note in report (headless)
  - Context already tight or lightweight mode → don't expand; recommend as next step
  - Headless mode → never invoke, surface recommendation in report
- Add argument construction: guide for building scope hints (file path, module name, category)
- Add: "Do not invoke without an argument unless user explicitly wants a broad sweep"
- Note: `ce-compound-refresh` is not yet installed as a Zed skill. Recommendations go to the user, not to auto-invoke

**Discoverability Check:**
- Add: "This runs every time — the knowledge store only compounds value when agents can find it"
- Add semantic assessment note: "not a string match — could be a line in an architecture section, a bullet in a gotchas section, spread across multiple places"
- Add CONCEPTS.md example: `CONCEPTS.md  # shared domain vocabulary (entities, named processes, status concepts) — relevant when orienting to the codebase or discussing domain concepts`

**Phase 3:**
- Note: skip entirely in headless mode
- In interactive mode, ask user if they want specialized reviews
- Use generic domain-specific prompts (not agent names since the specialized agents don't exist)
- Add: "If the specialized agents are implemented as Zed skills in the future, replace the generic prompts with named `spawn_agent` calls to the corresponding agent"

**Lightweight Mode:**
- Add CONCEPTS.md discoverability tip to output format
- Add overlap-is-acceptable note: "lightweight mode skips overlap check — that's acceptable; a refresh run can catch it later"
- Add: verify discoverability tip shows in output when instruction files don't surface the store

**Success Output — Headless:**
- Fix overlap line to include path on moderate: `<none | low | moderate — see <path> | high — existing doc updated>`
- Fix CONCEPTS.md line to include seeded count: `created with N entries (M seeded from the learning's area)`
- Add clarification header text: "Emit a structured terminal report and end the turn"
- Add "Documentation complete" / "Documentation skipped" terminal signals

**Success Output — Interactive:**
- Add alternate output block for existing doc update scenario
- Add "What's next?" with 5 options: Continue workflow, Link related docs, Update other references, View docs, Other
- Keep the "Files written" format consistent

**Patterns to follow:**
- Source ce-compound's output format for consistency
- Zed markdown conventions from `references/markdown-rendering.md`

**Test scenarios:**
- Phase 2.4: vocabulary capture finds qualifying terms → adds/refines silently. No qualifying terms → records "scanned, no qualifying terms"
- Phase 2.5: high overlap with existing doc → update existing + add `last_updated`. Moderate overlap → create new doc + flag consolidation
- Phase 2.5: one obvious stale candidate → recommend targeted refresh. Multiple candidates → ask user
- Discoverability: instruction file doesn't surface store → propose edit, get consent (interactive) or apply silently (headless). Already sufficient → no action
- Lightweight output: shows file path, track, category, discoverability tips, vocabulary outcome
- Headless output: structured terminal report with all fields filled correctly
- Interactive output: "What's next?" menu presented, user selection routed

**Verification:** Read SKILL.md from Phase 2 through end. Confirm all sections have correct content per the approach above. Validate headless output format matches source's terminal-report contract. Validate interactive "What's next?" has 5 options.

---

### Deferred to Follow-Up Work

- **`ce-compound-refresh` as a Zed skill**: Phase 2.5 references a skill that doesn't exist in Zed. Creating it is a separate task.
- **Specialized reviewer agents**: `ce-performance-oracle`, `ce-security-sentinel`, etc. If these become Zed skills, Phase 3 should be upgraded to use named `spawn_agent` calls instead of generic prompts.
- **Session history integration**: The source `ce-sessions` skill isn't installed. The Zed version omits this entirely, which means cross-session context is lost. If `ce-sessions` is ported later, re-integrate the Phase 1 step 4 from the source.

## Assumptions

1. The orchestrating agent can read the reference files (`references/schema.yaml`, etc.) and inline their content into `spawn_agent` messages.
2. Python 3 is available on the system for `validate-frontmatter.py`.
3. The `gh` CLI is not assumed to be available — GitHub issue search falls back gracefully.
4. `spawn_agent` in Zed accepts a `message` string that can include structured text blocks and file contents.

## Open Questions

None resolved — all technical decisions are answerable from existing skill patterns and the source document.

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Context extraction misses critical details | Medium | High — sub-agents produce poor output | Explicit structured extraction format in Phase 0.5; orchestrator is the main agent that participated in the conversation |
| `spawn_agent` message size limits with large reference content | Low | Medium | Reference files are small (<5KB each); conversation context is typically <2KB |
| User expects `ce-compound-refresh` to exist after reading Phase 2.5 | Medium | Low — user disappointment | Note in Phase 2.5 that the skill isn't installed yet; recommend as future work |
