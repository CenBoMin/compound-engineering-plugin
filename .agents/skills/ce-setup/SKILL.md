---
name: ce-setup
target: zed
description: "Diagnose and configure compound-engineering environment. Checks CLI dependencies, plugin version, and repo-local config. Offers guided installation for missing tools. Use when troubleshooting missing tools, verifying setup, or before onboarding."
disable-model-invocation: true
argument-hint: ""
---

# Compound Engineering Setup (Zed)

Interactive setup for compound-engineering — diagnoses environment health, cleans obsolete repo-local CE config, and helps configure required tools. Review agent selection is handled automatically by `/ce-code-review`; project-specific review guidance belongs in `AGENTS.md` or `.zed/tasks.json`.

## Interaction Pattern

In Zed, interact with the user **conversationally** through the chat. Present findings, ask yes/no or choice questions, and wait for the user's response before proceeding. Do not attempt to use platform-specific question tools (`AskUserQuestion`, `multiSelect`, etc.) — plain chat interaction is the Zed pattern.

The workflow has two phases: **Diagnose** and **Fix**.

---

## Phase 1: Diagnose

### Step 1: Run the Health Check Script

Display: "Compound Engineering — checking your environment..."

Run the bundled check script. Do not perform manual dependency checks — the script handles all CLI tools, agent skills, repo-local CE file checks, and `.gitignore` guidance in one pass.

```bash
bash scripts/check-health
```

Display the script's output to the user.

### Step 2: Evaluate Results

After the diagnostic report, check whether:

- Any CLI tools are missing (reported as `🟡` in the Tools section)
- Any agent skills are missing (reported as `🟡` in the Skills section)
- `compound-engineering.local.md` is present at the repo root and needs cleanup
- `.compound-engineering/config.local.yaml` does not exist or is not safely gitignored
- `.compound-engineering/config.local.example.yaml` is missing or outdated

**If everything is green** — all tools installed, no repo-local cleanup needed, and config exists and is gitignored — display the summary and stop:

```
 ✅ Compound Engineering setup complete

    Tools:  🟢 agent-browser  🟢 gh  🟢 jq  🟢 vhs  🟢 silicon  🟢 ffmpeg  🟢 ast-grep
    Skills: 🟢 ast-grep
    Config: ✅

    Run /ce-setup anytime to re-check.
```

Stop here and do not proceed to Phase 2.

**Otherwise**, proceed to Phase 2. Handle repo-local cleanup first, then config bootstrapping, then missing dependencies.

---

## Phase 2: Fix

> **Note:** Steps 3-4 only apply inside a git repository. If the health check script reported `in_repo=no`, skip Steps 3-4 and go directly to Step 5.

### Step 3: Resolve Repo-Local CE Issues

Check whether the current directory is inside a git repository:

```bash
git rev-parse --is-inside-work-tree 2>/dev/null
```

If not, skip this step. Otherwise, resolve the repository root (`git rev-parse --show-toplevel`). Always wrap `$repo_root` in double quotes when constructing shell commands (`"$repo_root"`). Use `--` argument separator before paths when supported (e.g., `rm -- "$repo_root/..."`).

**Path validation:** Reject paths containing shell metacharacters (`;`, `|`, `$`, `` ` ``, `(`, `)`, `{`, `}`, `<`, `>`, `&`, `!`). If the resolved repo root contains any of these characters, stop and warn the user.

If `compound-engineering.local.md` exists at the repo root, explain that it is obsolete (review-agent selection is automatic, and CE now uses `.compound-engineering/config.local.yaml` for any surviving machine-local state). Ask the user whether to delete it now.

### Step 4: Bootstrap Project Config

Check whether the current directory is inside a git repository (same check as Step 3). If not, skip this step. Otherwise, resolve the repository root (`git rev-parse --show-toplevel`). Always wrap `$repo_root` in double quotes when constructing shell commands (`"$repo_root"`). Use `--` argument separator before paths when supported (e.g., `rm -- "$repo_root/..."`). All paths below are relative to the repo root.

**Example file (always refresh):** Copy `references/config-template.yaml` to `<repo-root>/.compound-engineering/config.local.example.yaml`, creating the directory if needed. This file is committed to the repo and always overwritten with the latest template so teammates can see available settings.

**Local config (create once):** If `.compound-engineering/config.local.yaml` does not exist, ask the user:

> Set up a local config file for this project?
> This saves your Compound Engineering preferences (like which tools to use and how workflows behave). Everything starts commented out — you only enable what you need.

If the user approves, copy `references/config-template.yaml` to `<repo-root>/.compound-engineering/config.local.yaml`. If `.compound-engineering/config.local.yaml` is not already covered by `.gitignore`, offer to add the entry:

```
.compound-engineering/*.local.yaml
```

If the local config already exists, check whether it is safely gitignored. If not, offer to add the `.gitignore` entry as above.

### Step 5: Offer Installation

Present the missing tools and skills as a clear list in chat. Use the install commands and URLs from the script's diagnostic output. Group items under `Tools:` and `Skills:` so the user can see which runtime each item targets. Omit a group whose items are all installed.

Ask the user to tell you which items they want to install — either by naming specific ones or saying "all".

Example:

```
The following items are missing:

Tools:
  [ ] agent-browser - Browser automation for testing and screenshots
  [ ] gh - GitHub CLI for issues and PRs
  [ ] jq - JSON processor
  [ ] vhs (charmbracelet/vhs) - Create GIFs from CLI output
  [ ] silicon (Aloxaf/silicon) - Generate code screenshots
  [ ] ffmpeg - Video processing for feature demos
  [ ] ast-grep - Structural code search using AST patterns

Skills:
  [ ] ast-grep - Agent skill for structural code search with ast-grep

Which would you like to install? (say "all" for everything, or name specific items)
```

### Step 6: Install Selected Dependencies

For each selected dependency, in order:

1. **Show the install command** and ask for approval in chat:

   ```
   Install agent-browser?
   Command: CI=true npm install -g agent-browser --no-audit --no-fund --loglevel=error && agent-browser install && npx skills add https://github.com/vercel-labs/agent-browser --skill agent-browser -g -y
   ```

   This is a multi-part command — it installs the npm package, runs setup, and adds the agent skill in sequence. Show the full command so the user sees everything it will do before approving.

   Ask: "Run this command? (yes / skip)"

2. **If approved:** Run the install command using a shell execution tool. After the command completes, verify installation:
   - For a CLI tool: `command -v <tool-name>`
   - For an agent skill: check any of the known skill roots: `~/.claude/skills/<skill-name>`, `~/.agents/skills/<skill-name>`, or `~/.codex/skills/<skill-name>` (file, directory, or symlink)

   > **Note for `target: zed`:** On Zed, skills are loaded from `~/.agents/skills/`. If the `npx skills add` command installs the skill to a different location, manually copy or symlink it to `~/.agents/skills/<skill-name>/` (or the `.agents/skills/` directory inside this project). The health check script above will detect the skill in any of the three roots, but Zed only reads from `~/.agents/skills/`.

3. **If verification succeeds:** Report success.

4. **If verification fails or install errors:** Display the project URL as fallback and continue to the next dependency. Do not block on a single failure.

### Step 7: Summary

Display a brief summary:

```
 ✅ Compound Engineering setup complete

    Installed: agent-browser, gh, jq
    Skipped:   vhs

    Run /ce-setup anytime to re-check.
```

---

## Reference

| Phase       | Step                                                                                                                                                                     |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1. Diagnose | Run health check script, evaluate results                                                                                                                                |
| 2. Fix      | Resolve repo-local issues (delete obsolete `compound-engineering.local.md`), bootstrap `.compound-engineering/config.local.yaml`, offer install for missing dependencies |
| Final       | Summary report                                                                                                                                                           |

Required CLI tools (defaults): `agent-browser`, `gh`, `jq`, `vhs`, `silicon`, `ffmpeg`, `ast-grep`. Required skills: `ast-grep`.
