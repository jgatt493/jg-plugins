---
name: decompose-to-queue
description: Use when the user asks to offload work to the local agent queue — triggered by "queue this", "have qwen do this", "send to pi", "decompose for qwen", "break this down and offload", "let qwen handle this". Decomposes high-level intent into atomic task files in ~/.aq/inbox/ for the `aq` runner to execute via local pi/Qwen3. Refuses to decompose judgment-heavy work and escalates back to the user.
---

# decompose-to-queue — Plan Once, Queue Atomic Tasks

Goal: take a high-level intent from the user and emit one or more atomic task files into `~/.aq/inbox/` that the `aq` CLI will execute via local pi+Qwen3 (and, in Phase 2, a local thinking model).

**Spend tokens here so they don't have to be spent in the local loop.** Your decomposition is the planning. Each task file should be small, prescriptive, and self-contained.

## Worth-offloading heuristic — RUN THIS FIRST

Before decomposing, check whether the work is actually a fit for local Qwen. **Refuse to decompose** (and explain why) if any of these apply:

| Sign | Why not Qwen |
|---|---|
| Requires choosing between two non-obvious options | needs judgment Qwen lacks |
| Debugging unfamiliar code/errors | needs reasoning across context |
| Code review or design feedback | needs taste |
| Anything where you'd want to ask the user a clarifying question | tasks must be unambiguous |
| Open-ended "investigate X" | not atomic, no clear done state |
| Touches code in repos you haven't read | Qwen will hallucinate |

If the request fails the heuristic, say so explicitly to the user: *"This one's mine — Qwen will struggle with [reason]. I'll do it directly."* Do not silently decompose-and-fail.

If the request passes (install commands, run-X-in-each-dir, scaffolding, file moves, repetitive setup, dependency installation, well-defined builds), proceed.

## Output: task files in ~/.aq/inbox/

Each task is a markdown file with YAML frontmatter. **Naming convention:** `NNN-<short-slug>.md` where `NNN` is a zero-padded sequence number scoped to the user's intent (e.g., `001-install-nix.md`, `002-install-arbor-via-nix.md`).

**Frontmatter schema (Phase 1):**

```yaml
---
id: 001-install-nix          # required, must match filename stem
type: execute                # 'execute' for Phase 1; 'plan' is Phase 2
executor: pi-qwen            # only pi-qwen is implemented in Phase 1
depends_on: []               # task ids that must complete first
timeout: 10m                 # 30s | 5m | 1h | raw seconds
retries: 1                   # how many times aq will retry on failure
verify: nix --version        # SHELL command run after; must exit 0 for success
---
```

**Body** is the prompt itself. Be terse, prescriptive, no hedging. Qwen3-Coder is literal — the more specific you are, the better.

### Verify is mandatory unless the task is unverifiable

The `verify` field is the queue's truth check. After the agent declares done, `aq` runs the verify command; if it exits non-zero the task is marked failed regardless of what the agent said. **Always include a verify** unless you genuinely cannot phrase one. Examples:

- Install something → `which <bin>` or `<bin> --version`
- Clone a repo → `test -d <path>/.git`
- Create a file → `test -f <path>`
- Modify a config → `grep <expected-string> <file>`
- Service starts → `curl -sf <healthcheck-url>`

## Workflow

1. **Run the heuristic.** Refuse if necessary. Otherwise proceed.
2. **Plan the DAG out loud (briefly).** One paragraph: what's the chain, where are the parallelizable branches.
3. **Verify `~/.aq/` exists.** If `~/.aq/inbox/` does not exist, run `mkdir -p ~/.aq/inbox` first.
4. **Write each task file** via the Write tool to `~/.aq/inbox/NNN-slug.md`. Use `depends_on` to express ordering.
5. **Report back to the user** with: count, IDs, the literal command to run.

### Reporting template

```
Queued N task(s):
  001-install-nix              ← root
  002-install-arbor-via-nix    ← depends on 001

Run with:  aq run
Watch:     aq tail 001-install-nix -f
Status:    aq status
```

## Decomposition principles

- **Atomic.** Each task does ONE thing. "Install nix" is atomic. "Install nix and configure flakes" is two tasks.
- **Idempotent where possible.** Prefer commands that don't fail on re-run.
- **Explicit cwd.** If a command depends on directory, include `cd <path> &&` in the prompt.
- **No model-of-the-user assumptions.** Don't write prompts that say "you know what I mean" — Qwen literally doesn't.
- **Failure-aware.** When a step might fail, include the recovery hint inline (e.g., *"if the install script asks for confirmation, pass `--yes`"*).
- **Use wsh for TUI.** Long prompts should remind Qwen to use `wsh` skills (already installed at `~/.pi/agent/skills/`) for any interactive prompt-driven installer.

## Worked example

User says: *"Install nix and then install the arbor package via nix"*

**Plan:** two tasks. 002 depends on 001.

**File 1: `~/.aq/inbox/001-install-nix.md`**

```markdown
---
id: 001-install-nix
type: execute
executor: pi-qwen
depends_on: []
timeout: 10m
retries: 1
verify: nix --version
---

Install Nix using the Determinate Systems installer (it handles macOS quirks).

Use a wsh session because the installer is interactive — load the `core` and
`drive-process` skills first.

Command to run:
  curl --proto '=https' --tlsv1.2 -sSf -L https://install.determinate.systems/nix | sh -s -- install --determinate

Pass `--yes` or auto-confirm if prompted.

End with `RESULT:` line stating the installed version.
```

**File 2: `~/.aq/inbox/002-install-arbor-via-nix.md`**

```markdown
---
id: 002-install-arbor-via-nix
type: execute
executor: pi-qwen
depends_on: [001-install-nix]
timeout: 5m
retries: 1
verify: which arbor
---

Install the `arbor` package via nix.

Source the nix profile if needed:
  . /nix/var/nix/profiles/default/etc/profile.d/nix-daemon.sh

Then:
  nix profile install nixpkgs#arbor

End with `RESULT:` line stating the installed binary path.
```

**Then tell the user:**

```
Queued 2 tasks:
  001-install-nix              ← root
  002-install-arbor-via-nix    ← depends on 001

Run with:  aq run
```

## When the queue runs back to you

If `aq` reports a failure on an offloaded task and the user asks you about it, **read the result file first**:

```
~/.aq/failed/<task-id>.result.md   # status + error reason
~/.aq/logs/<task-id>.jsonl         # full event stream
```

Then decide: rewrite the task with better instructions and `aq retry <id>`, or take it over directly.
