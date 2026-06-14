---
name: ce-compound
target: zed
description: "Document a recently solved problem to compound your team's knowledge or CONCEPTS.md, the project's shared domain vocabulary. Use when you've fixed something non-trivial and want to make sure the solution is findable later. Captures structured docs to docs/solutions/ with YAML frontmatter for searchability."
argument-hint: "[optional: brief context] [mode:headless] "
---

# ce-compound (Zed)

Coordinate parallel research and assembly to document a recently solved problem while context is fresh. Creates structured documentation in `docs/solutions/` with YAML frontmatter for searchability and future reference.

**Why "compound"?** Each documented solution compounds your team's knowledge. The first time you solve a problem takes research. Document it, and the next occurrence takes minutes. Knowledge compounds.

## Usage

```
/ce-compound                            # Document the most recent fix
/ce-compound [brief context]            # Provide additional context hint
/ce-compound mode:headless              # Non-interactive run for automations
/ce-compound mode:headless [context]    # Non-interactive run with context hint
```

## Auto-Invoke

When the user communicates that a problem has been solved, this skill can auto-trigger to capture the solution:

| Trigger Pattern | When |
|----------------|------|
| "that worked" | User confirms a fix is effective |
| "it's fixed" | User declares the problem resolved |
| "working now" | User verifies the solution works |
| "problem solved" | User explicitly states completion |

When a trigger phrase is detected, automatically run `/ce-compound mode:headless` to capture the solution without interrupting the user's flow. Manual override: run `/ce-compound [context]` directly at any time.

## Mode Detection

Parse the `$ARGUMENTS` string for mode flags. Tokens starting with `mode:` are flags, not context — strip all `mode:*` tokens before treating the remainder as the brief context hint.

| Mode | When | Behavior |
|------|------|----------|
| **Interactive** (default) | No `mode:` token present | Ask Full vs Lightweight. Prompt for Discoverability Check consent. End with "What's next?" menu. |
| **Headless** | `mode:headless` in arguments | No blocking questions. Run **Full mode**. Apply Discoverability Check edit silently if a gap exists. Skip Phase 3. End with structured terminal report. |

## Support Files

These files are the durable contract for the workflow. Read them on-demand at the step that needs them — do not bulk-load at skill start.

- `references/schema.yaml` — canonical frontmatter fields and enum values (read when validating YAML)
- `references/yaml-schema.md` — category mapping from problem_type to directory (read when classifying)
- `references/concepts-vocabulary.md` — CONCEPTS.md format and inclusion rules (read in Phase 2.4 when domain terms surface)
- `assets/resolution-template.md` — section structure for new docs (read when assembling)

When spawning sub-agents, pass the relevant file contents into the task prompt so they have the contract without needing cross-skill paths.

## CONCEPTS.md Bootstrap Requests

If invoked specifically to create or bootstrap `CONCEPTS.md` from scratch rather than to document a solved problem, do not run the normal workflow. This skill populates `CONCEPTS.md` only as a side effect of documenting a real learning (it seeds the *learning's area*, not the whole repo). Repo-wide concept-map creation is `ce-compound-refresh`'s job.

**Redirect:** Tell the user that standalone `CONCEPTS.md` creation is handled by `/ce-compound-refresh` (not yet installed as a Zed skill). If they want to proceed with the full doc workflow anyway (because they have a solved problem to document), offer that as an alternative. If not, exit the workflow.

## Preconditions

<preconditions enforcement="advisory">
  <check condition="problem_solved">
    Problem has been solved (not in-progress)
  </check>
  <check condition="solution_verified">
    Solution has been verified working
  </check>
  <check condition="non_trivial">
    Non-trivial problem (not a simple typo or obvious error)
  </check>
</preconditions>

In headless mode, if preconditions aren't met, emit a structured failure report and end with `Documentation skipped`.

## Pre-resolved Context

**Git branch (pre-resolved):** !`git rev-parse --abbrev-ref HEAD 2>/dev/null || true`

If the line above resolved to a plain branch name, include it when dispatching sub-agents. If it still contains a backtick command string or is empty, omit it.

## Execution Strategy

**In headless mode**, skip the Full vs Lightweight question below and go directly to **Full Mode**. Proceed straight to research.

**In interactive mode**, present the user with two options:

```
1. Full (recommended) — the complete compound workflow. Researches,
   cross-references, and reviews your solution to produce documentation
   that compounds your team's knowledge.

2. Lightweight — same documentation, single pass. Faster and uses
   fewer tokens, but won't detect duplicates or cross-reference
   existing docs. Best for simple fixes or long sessions nearing
   context limits.
```

Wait for the user's choice before proceeding. Do not pre-select.

---

### Full Mode

<critical_requirement>
**The primary deliverable is ONE file — the final documentation.**

Phase 1 sub-agents return TEXT DATA to the orchestrator. They must NOT use Write, Edit, or create any files. Only the orchestrator writes files. Beyond the Phase 2 solution doc, its other writes are maintenance side effects:
- **`CONCEPTS.md`** — create or update in Phase 2.4 when a qualifying domain term surfaces.
- **A project instruction file** (AGENTS.md or CLAUDE.md) — a small edit when the Discoverability Check finds a gap.

Both ensure future agents can discover and ground in the knowledge store; neither makes the documentation any less the single deliverable — not additional deliverables, and creating CONCEPTS.md when absent is expected, not a violation of this rule.
</critical_requirement>

### Phase 0.5: Context Extraction (Orchestrator)

**Run this BEFORE dispatching sub-agents.** This step is critical because `spawn_agent` creates isolated sub-agents with no access to the parent agent's conversation history.

The orchestrator must extract the following from the current conversation and package it into a structured block labeled `## Conversation context for compound documentation`.

**Extraction checklist:**
1. **Problem description** — what was the issue being solved?
2. **Error messages / symptoms** — exact error text, stack traces, observable behavior
3. **Investigation steps** — what was tried, what didn't work, why
4. **The working solution** — what fixed it, with code snippets if available
5. **Root cause** — why the solution works
6. **Module/area** — which part of the codebase was affected
7. **Conversation context hint** — use the user's `$ARGUMENTS` brief context hint if provided

This structured context block is passed verbatim to each sub-agent in their `spawn_agent` message.

**In headless mode**, if the conversation is too thin to extract a coherent problem-solution pair, emit a structured failure and end:

```
✗ Documentation skipped (headless mode)

Reason: Could not extract a solved problem from conversation history

Documentation skipped
```

### Phase 1: Research

Launch research sub-agents in parallel via `spawn_agent`. Each sub-agent receives:
- The `## Conversation context for compound documentation` block from Phase 0.5
- Inlined content of the relevant reference file(s) (parent reads files, inlines content)
- The pre-resolved git branch if available

**Dispatch all three in a single turn for maximal parallelism.**

```text
spawn_agent label: "Context Analyzer"
  message: "You are a Context Analyzer. Classify the problem... [inline conversation context] [inline schema.yaml content] [inline yaml-schema.md content]"

spawn_agent label: "Solution Extractor"
  message: "You are a Solution Extractor. Extract the solution... [inline conversation context] [inline schema.yaml content]"

spawn_agent label: "Related Docs Finder"
  message: "You are a Related Docs Finder. Search for related docs... [inline schema.yaml content]"
```

**Important:** Read the reference files (`references/schema.yaml`, etc.) and inline their content into each `spawn_agent` message. Do not reference them by file path — isolated sub-agents may not resolve them correctly.

#### 1. Context Analyzer
   - Receives the conversation context block from Phase 0.5 as its primary input
   - Receives inlined `schema.yaml` and `yaml-schema.md` content for classification
   - Determines the track (bug or knowledge) from the problem_type
   - Identifies problem type, component, and track-appropriate fields:
     - **Bug track**: symptoms, root_cause, resolution_type
     - **Knowledge track**: applies_when (symptoms/root_cause/resolution_type optional)
   - Maps problem_type to category using the inlined category mapping
   - Suggests a filename using the pattern `[sanitized-problem-slug].md` — no date suffix
   - Returns: YAML frontmatter skeleton (must include `category:` field mapped from problem_type), category directory path, suggested filename, and which track applies
   - Does not invent enum values, categories, or frontmatter fields from memory; uses the inlined schema and mapping
   - Does not force bug-track fields onto knowledge-track learnings or vice versa

#### 2. Solution Extractor
   - Receives the conversation context block from Phase 0.5 as its primary input
   - Receives inlined `schema.yaml` content for track classification
   - Adapts output structure based on the problem_type track

   **Bug track output sections:**
   - **Problem**: 1-2 sentence description of the issue
   - **Symptoms**: Observable symptoms (error messages, behavior)
   - **What Didn't Work**: Failed investigation attempts and why they failed
   - **Solution**: The actual fix with code examples (before/after when applicable)
   - **Why This Works**: Root cause explanation and why the solution addresses it
   - **Prevention**: Strategies to avoid recurrence, best practices, and test cases. Include concrete code examples where applicable (e.g., gem configurations, test assertions, linting rules).

   **Knowledge track output sections:**
   - **Context**: What situation, gap, or friction prompted this guidance
   - **Guidance**: The practice, pattern, or recommendation with code examples when useful
   - **Why This Matters**: Rationale and impact of following or not following this guidance
   - **When to Apply**: Conditions or situations where this applies
   - **Examples**: Concrete before/after or usage examples showing the practice in action

#### 3. Related Docs Finder
   - Searches `docs/solutions/` for related documentation
   - Flags any related learning or pattern docs that may now be stale, contradicted, or overly broad

   **Search strategy (grep-first filtering for efficiency):**
   1. Extract keywords from the problem context: module names, technical terms, error messages
   2. If the problem category is clear, narrow search to the matching `docs/solutions/<category>/` directory
   3. Use grep to pre-filter candidate files BEFORE reading any content. Run multiple searches in parallel, case-insensitive, targeting frontmatter fields:
      - `title:.*<keyword>`
      - `tags:.*(<keyword1>|<keyword2>)`
      - `module:.*<module name>`
      - `component:.*<component>`
   4. If >25 candidates, re-run with more specific patterns. If <3, broaden to full content search
   5. Read only frontmatter (first 30 lines) of candidate files to score relevance
   6. Fully read only strong/moderate matches
   7. Return distilled links and relationships, not raw file contents

   **GitHub issue search:**
   Prefer the `gh` CLI: `gh issue list --search "<keywords>" --state all --limit 5`. If `gh` is not installed, fall back to GitHub MCP tools if available. If neither is available, skip GitHub issue search and note it was skipped in the output.

   **Overlap assessment** with the new doc being created across five dimensions: problem statement, root cause, solution approach, referenced files, and prevention rules. Score as:
     - **High**: 4-5 dimensions match — essentially the same problem solved again
     - **Moderate**: 2-3 dimensions match — same area but different angle or solution
     - **Low**: 0-1 dimensions match — related but distinct
   - Returns: Links, relationships, refresh candidates, and overlap assessment (score + which dimensions matched)

### Phase 2: Assembly & Write

**WAIT for all Phase 1 inputs to complete before proceeding.**

The orchestrating agent performs these steps sequentially:

1. Collect all text results from Phase 1 sub-agents
2. **Check the overlap assessment** from the Related Docs Finder:

   | Overlap | Action |
   |---------|--------|
   | **High** — existing doc covers the same problem, root cause, and solution | **Update the existing doc** with fresher context (new code examples, updated references, additional prevention tips) rather than creating a duplicate. The existing doc's path and structure stay the same. Add `last_updated: YYYY-MM-DD` to frontmatter. |
   | **Moderate** — same problem area but different angle, root cause, or solution | **Create the new doc** normally. Flag overlap for potential consolidation. |
   | **Low or none** | **Create the new doc** normally. |

   **Why update rather than create:** Two docs describing the same problem and solution will inevitably drift apart. The newer context is fresher and more trustworthy, so fold it into the existing doc rather than creating a second one that immediately needs consolidation.

3. Assemble complete markdown file from the collected pieces, reading `assets/resolution-template.md` for the section structure of new docs
4. Validate YAML frontmatter against `references/schema.yaml`, including the YAML-safety quoting rule for array items (see `references/yaml-schema.md` > YAML Safety Rules)
5. Create directory if needed: `mkdir -p docs/solutions/[category]/`
6. Write the file
7. **Run `python3 scripts/validate-frontmatter.py <output-path>`** to catch silent-corruption parser-safety issues. The script detects malformed `---` delimiter lines, unquoted ` #` in scalar values (silent comment truncation), and unquoted `: ` in scalar values (silent mapping confusion). Exit 0 means the doc is parser-safe; exit 1 means fix and retry until exit 0. Uses Python 3 stdlib only.

When creating a new doc, preserve the section order from `assets/resolution-template.md` unless the user explicitly asks for a different structure.

### Phase 2.4: Vocabulary Capture

**First, read `references/concepts-vocabulary.md`.** This is unconditional. Do not pre-judge from memory that nothing qualifies — the reference's criteria are non-obvious and qualifying terms often live in the surrounding conversation rather than the new doc itself.

Then, scan the new doc and the surrounding conversation for qualifying domain terms. If `CONCEPTS.md` exists at repo root, add missing qualifying terms and refine existing entries when new precision surfaced. If it does not exist and at least one qualifying term surfaced, create it.

**Seed the learning's area at creation — don't write a lone term.** When `CONCEPTS.md` does not yet exist, alongside the surfaced term also seed the core domain nouns of the area this learning touched, following the **Seed goal** and **Scope of a seed** rules in `references/concepts-vocabulary.md`.

**At creation, hold the qualifying bar conservatively for borderline terms.** A borderline term defers to a later run — clear core nouns are seeded, borderline ones wait. The conservatism is about quality, not count.

When bootstrapping the file, start with this preamble underneath the `# Concepts` heading:

> Shared domain vocabulary for this project — entities, named processes, and status concepts with project-specific meaning. Seeded with core domain vocabulary, then accretes as ce-compound processes learnings; direct edits are fine. Glossary only, not a spec or catch-all.

**Refresh the coherence neighborhood of any entry you touch.** When adding or editing an entry, also inspect its cluster siblings and the terms it cross-references. Bounds: neighborhood only, never a full-file audit; refresh only on evidence already in hand; if judging a neighbor would require investigation this learning did not do, flag it for a future refresh rather than editing on a guess.

If no terms qualified, record that outcome explicitly (e.g., "Vocabulary capture: scanned, no qualifying terms"). The visible scan-and-no-result record is the audit signal that the reference was consulted.

**Apply edits silently in every mode.**

### Phase 2.5: Selective Refresh Check

After writing the new learning, decide whether this new solution is evidence that older docs should be refreshed.

`ce-compound-refresh` is **not** a default follow-up. Use it selectively when the new learning suggests an older learning or pattern doc may now be inaccurate.

Consider refresh when:
1. A related learning recommends an approach that the new fix now contradicts
2. The new fix clearly supersedes an older documented solution
3. The current work involved a refactor, migration, rename, or dependency upgrade that likely invalidated older docs
4. A pattern doc now looks overly broad, outdated, or no longer supported by the refreshed reality
5. The Related Docs Finder surfaced high-confidence refresh candidates
6. The Related Docs Finder reported **moderate overlap** — possible consolidation opportunity

Don't refresh when:
1. No related docs were found
2. Related docs still appear consistent with the new learning
3. Overlap is superficial and does not change prior guidance
4. Refresh would require a broad historical review with weak evidence

**Decision logic:**
- **One obvious stale candidate** — recommend with narrow scope hint (file path, module, or category)
- **Multiple candidates in same area** — ask the user (interactive) or note in report (headless)
- **Context already tight or lightweight mode** — don't expand; recommend as next step with scope hint
- **Headless mode** — never invoke refresh; surface recommendation in terminal report only

**Argument construction for refresh scope hints:**
- Specific file: `ce-compound-refresh <filename>` or `ce-compound-refresh <module-name>`
- Module/component: `ce-compound-refresh <module>`
- Category: `ce-compound-refresh <category-name>`
- Do not invoke `ce-compound-refresh` without an argument unless the user explicitly wants a broad sweep

**Note:** `ce-compound-refresh` is not yet installed as a Zed skill. Recommendations go to the user, not to auto-invoke. Always capture the new learning first — refresh is a targeted maintenance follow-up, not a prerequisite for documentation.

### Discoverability Check

This runs every time — the knowledge store only compounds value when agents can find it. After the learning is written and the refresh decision is made, check whether the project's instruction files would lead an agent to discover and search `docs/solutions/` before starting work in a documented area.

1. Identify which root-level instruction files exist (AGENTS.md, CLAUDE.md, or both). Read the file(s) and determine which holds the substantive content. Ignore shims. If neither file exists, skip this check entirely.
2. Assess whether an agent reading the instruction files would learn three things:
   - That a searchable knowledge store of documented solutions exists
   - Enough about its structure to search effectively (category organization, YAML frontmatter fields like `module`, `tags`, `problem_type`)
   - When to search it (before implementing features, debugging issues, or making decisions in documented areas)

   This is a semantic assessment, not a string match. The information could be a line in an architecture section, a bullet in a gotchas section, or spread across multiple places. Use judgment — if an agent would reasonably discover and use the knowledge store, the check passes.

3. If the spirit is already met, no action needed.
4. If not, draft the smallest addition that communicates the three things. Match the file's existing style and density. A line added to an existing section is almost always better than a new headed section.

   Examples (not templates — adapt to the file):
   - Line in an existing directory listing: `docs/solutions/  # documented solutions to past problems (bugs, best practices, workflow patterns), organized by category with YAML frontmatter (module, tags, problem_type)`
   - Small headed section when nothing fits naturally:
     ```
     ## Documented Solutions
     `docs/solutions/` — documented solutions to past problems, organized by category with YAML frontmatter (`module`, `tags`, `problem_type`). Relevant when implementing or debugging in documented areas.
     ```

5. **If `CONCEPTS.md` exists at repo root**, run a parallel discoverability check for it. Same workflow as above. Example addition:
   `CONCEPTS.md  # shared domain vocabulary (entities, named processes, status concepts) — relevant when orienting to the codebase or discussing domain concepts`
   Skip this step entirely if `CONCEPTS.md` does not exist.

**In interactive mode**, explain why this matters and show the proposed change. Ask for consent via chat before making the edit.

**In headless mode**, apply the edit directly without prompting and surface it in the terminal report.

### Phase 3: Optional Enhancement (Interactive Mode Only)

**Skip Phase 3 entirely in headless mode.**

In interactive mode, based on problem type, optionally ask the user if they want specialized reviews. If they agree, use `spawn_agent` with domain-focused prompts:

- **performance_issue** → Focus review on query optimization, caching, N+1 patterns, memory usage
- **security_issue** → Focus review on vulnerability patterns, input validation, auth gaps, injection
- **database_issue** → Focus review on migration safety, data integrity, rollback strategy
- Any code-heavy fix → Focus review on code clarity, minimal diff, before/after correctness

This phase is optional. Ask once, do not repeat. If the specialized agents are implemented as Zed skills in the future, replace these generic prompts with named `spawn_agent` calls to the corresponding agent.

---

### Lightweight Mode

<critical_requirement>
**Single-pass alternative — same documentation, fewer tokens.**

This mode skips parallel sub-agents entirely. The orchestrator performs all work in a single pass, producing the same solution document without cross-referencing or duplicate detection.

Headless mode forces Full and does not enter Lightweight.
</critical_requirement>

The orchestrator performs ALL of the following in one sequential pass:

1. **Extract from conversation** (same as Phase 0.5 — identify problem, symptoms, solution, root cause from chat history)
2. **Classify**: Read `references/schema.yaml` and `references/yaml-schema.md`, then determine track (bug vs knowledge), category, and filename
3. **Write minimal doc**: Create `docs/solutions/[category]/[filename].md` using the appropriate track template from `assets/resolution-template.md`, with:
   - YAML frontmatter with track-appropriate fields, applying the YAML-safety quoting rule for array items
   - Bug track: Problem, root cause, solution with key code snippets, one prevention tip
   - Knowledge track: Context, guidance with key examples, one applicability note
   - Run `python3 scripts/validate-frontmatter.py <output-path>` and fix until exit 0
4. **Vocabulary capture (update-only)**: if `CONCEPTS.md` exists at repo root, read `references/concepts-vocabulary.md`, then scan for qualifying terms and add/refine entries silently. Do **not** bootstrap in lightweight mode. Record outcome in output.
5. **Skip specialized agent reviews** to conserve context

**Lightweight output:**
```
✓ Documentation complete (lightweight mode)

File:
- docs/solutions/[category]/[filename].md (created)
Track: <bug | knowledge>
Category: <category>

[If discoverability check found instruction files don't surface the knowledge store:]
Tip: Your AGENTS.md/CLAUDE.md doesn't surface docs/solutions/ to agents —
a one-line mention helps all agents discover these learnings.

[If CONCEPTS.md was refined this run and isn't surfaced in the instruction files:]
Tip: Your AGENTS.md/CLAUDE.md doesn't surface CONCEPTS.md —
a one-line mention helps agents find the shared vocabulary.

Vocabulary: <scanned, no qualifying terms | updated — N added, N refined>

Note: Created in lightweight mode. The overlap check is skipped in lightweight mode — that's acceptable; a refresh run can catch it later. For richer documentation (cross-references, detailed prevention strategies, specialized reviews), re-run /ce-compound in a fresh session.
```

No sub-agents are launched. No parallel tasks.

---

## What It Captures

- **Problem symptom**: Exact error messages, observable behavior
- **Investigation steps tried**: What didn't work and why
- **Root cause analysis**: Technical explanation
- **Working solution**: Step-by-step fix with code examples
- **Prevention strategies**: How to avoid in future
- **Cross-references**: Links to related issues and docs

## What It Creates

**Organized documentation:**
- File: `docs/solutions/[category]/[filename].md`

**Categories auto-detected from problem:**

Bug track:
- build-errors/
- test-failures/
- runtime-errors/
- performance-issues/
- database-issues/
- security-issues/
- ui-bugs/
- integration-issues/
- logic-errors/

Knowledge track:
- architecture-patterns/ — architectural or structural patterns (agent/skill/pipeline/workflow shape decisions)
- design-patterns/ — reusable non-architectural design approaches
- tooling-decisions/ — language, library, or tool choices with durable rationale
- conventions/ — team-agreed way of doing something
- workflow-issues/
- developer-experience/
- documentation-gaps/
- best-practices/ (fallback only, use when no narrower knowledge-track value applies)

## Common Mistakes to Avoid

| ❌ Wrong | ✅ Correct |
|----------|-----------|
| Sub-agents write files like `context-analysis.md`, `solution-draft.md` | Sub-agents return text data; orchestrator writes one final file |
| Research and assembly run in parallel | Research completes → then assembly runs |
| Multiple files created during workflow | One solution doc (plus optional CONCEPTS.md and instruction-file edit) |
| Creating a new doc when an existing doc covers the same problem | Check overlap assessment; update the existing doc when overlap is high |

## Success Output

### Headless mode

Emit a structured terminal report and end the turn. No "What's next?" question, no blocking prompt. End with `Documentation complete` as the terminal signal so callers can detect completion.

```
✓ Documentation complete (headless mode)

File: docs/solutions/<category>/<filename>.md  (created | updated)
Track: <bug | knowledge>
Category: <category>
Overlap: <none | low | moderate — see <path> | high — existing doc updated>
Instruction-file edit: <none needed | applied to <path> | gap noted, not applied>
CONCEPTS.md: <scanned, no qualifying terms | created with N entries (M seeded from the learning's area) | updated — N added, N refined>
Refresh recommendation: <none | scope hint>

Documentation complete
```

When no doc was written (e.g., headless invoked on a session where the problem is not yet solved):

```
✗ Documentation skipped (headless mode)

Reason: <one-sentence explanation — e.g., "no solved problem detected in conversation history" or "solution not yet verified">

Documentation skipped
```

### Interactive mode

```
✓ Documentation complete

Subagent Results:
  ✓ Context Analyzer: Identified <problem_type> in <module>, category: <category>
  ✓ Solution Extractor: <N> code fixes, prevention strategies
  ✓ Related Docs Finder: <N> related issues

Files written:
- docs/solutions/<category>/<filename>.md (created)
- CONCEPTS.md (created/updated with N entries)

This documentation will be searchable for future reference when similar issues occur in the <module> module.

What's next?
1. Continue workflow
2. Link related documentation
3. Update other references
4. View documentation
5. Other
```

**Alternate interactive output (when updating an existing doc due to high overlap):**

```
✓ Documentation updated (existing doc refreshed with current context)

Overlap detected: docs/solutions/<category>/<existing-filename>.md
  Matched dimensions: problem statement, root cause, solution, referenced files
  Action: Updated existing doc with fresher code examples and prevention tips

File updated:
- docs/solutions/<category>/<existing-filename>.md (added last_updated: YYYY-MM-DD)
```

Present "What's next?" options as numbered options in chat. Wait for the user's selection before proceeding or ending.

## The Compounding Philosophy

1. First time you solve a problem → Research (30 min)
2. Document the solution → docs/solutions/ (5 min)
3. Next time similar issue occurs → Quick lookup (2 min)
4. Knowledge compounds → Team gets smarter

**Each unit of engineering work should make subsequent units of work easier — not harder.**

## Related Commands

- `/ce-plan` — Planning workflow (references documented solutions)
- `/ce-brainstorm` — Explore requirements through dialogue
- `/research [topic]` — Deep investigation (searches docs/solutions/ for patterns)
