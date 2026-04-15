# Observational Memory — Load Context

Before proceeding with any work, check if the observational memory system is installed and load the user's behavioral profile.

## Step 1: Check Installation

Check if `~/.observational-memory/memory.db` exists.

**If it exists**, skip to Step 2.

**If it does NOT exist**, run through setup:

1. Check if the `observational-memory` CLI is available: `which observational-memory`
2. If not installed, install it:
   ```
   pip install --break-system-packages git+https://github.com/jgatt493/jg-observational-memory.git
   ```
   If pip fails with a permissions error, try with `--user` flag instead of `--break-system-packages`.
   If the repo is not accessible, tell the user: "The observational memory repo may be private. Check https://github.com/jgatt493/jg-observational-memory for access."
3. Run `observational-memory install` — this creates `~/.observational-memory/`, initializes the SQLite database, and wires the Claude Code Stop hook.
4. Check if `ANTHROPIC_API_KEY` is set in the shell environment. **This is required, not optional.** The observer calls Claude Haiku to extract observations. If not set:
   - Tell the user: "ANTHROPIC_API_KEY is required. Add it to ~/.zshenv so it's available to all shells including the hook subprocess."
   - Stop setup here — the system will silently fail without it.
5. If the user has existing Claude Code sessions (check if `~/.claude/projects/` has `.jsonl` files), run backfill and reflect:
   ```
   observational-memory backfill
   observational-memory reflect --all
   ```
   This processes past sessions and generates the initial behavioral profile. It may take a few minutes.

Once setup is complete, continue to Step 2.

## Step 2: Load Behavioral Context

1. Read `~/.observational-memory/memory/global.md` — core behavioral rules. Apply as firm instructions.
2. Read `~/.observational-memory/memory/global_context.md` if it exists — contextual annotations and provenance. Apply as informational background (explains *why* rules exist, flags evolving patterns). Not directive.
3. Derive the project slug from the current working directory basename: lowercase, replace non-alphanumeric characters with `-`, strip leading/trailing `-`.
4. Check if `~/.observational-memory/memory/projects/{slug}.md` exists. If it does, read it — project-specific core rules. Apply as firm instructions.
5. Check if `~/.observational-memory/memory/projects/{slug}_context.md` exists. If it does, read it — project-specific contextual annotations. Apply as informational background.
6. When global and project rules conflict, project rules take precedence.

## Step 3: Confirm

After loading, print a single-line confirmation so the user knows the skill ran. Format:

`Observational memory loaded: global (Xk chars), {slug} (Xk chars)` — listing only the files that were found and loaded. If no files were found, print `Observational memory: no profiles generated yet`.

## Important

- If `global.md` does not exist yet, skip it — the memory system has not yet generated observations. This is normal on a fresh install before any sessions have been observed.
- Core files (`global.md`, `{slug}.md`) are rules — follow them strictly.
- Context files (`global_context.md`, `{slug}_context.md`) are informational — they explain the *why* behind rules and flag things that might be evolving. Use them to understand intent, not as directives.
