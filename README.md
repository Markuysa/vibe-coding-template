# Vibe-Coding Template

A reusable Claude Code project template: five specialist agent roles, four workflow
commands, and a permission baseline that keeps secrets and build noise out of the model's
context.

Built around Claude Code's **native** primitives — git worktrees, subagents, Agent View —
rather than third-party orchestration. Nothing here requires an extra tool to be installed.

## Structure

```
CLAUDE.md                      # project memory, loaded every session ({{...}} placeholders)
.gitignore                     # excludes .claude/worktrees/ and settings.local.json
.worktreeinclude               # gitignored files (.env) to copy into each new worktree
.claude/
  settings.json                # permissions deny/ask/allow + worktree.baseRef
  agents/                      # lead, dev, validator, reviewer, researcher
  skills/                      # /spec /plan /ship /retro
docs/
  PRD-template.md
  ARCHITECTURE-template.md
  QUICK_REF.md                 # cheat sheet, keep under 50 lines
  decisions/ADR-template.md
```

## Quickstart

```bash
npx degit Markuysa/vibe-coding-template my-project
cd my-project && git init          # worktrees require a git repository
claude
```

Then, in the first session:

1. *"Read CLAUDE.md and fill in the placeholders. Stack: `<your stack>`"*
2. `/spec <idea>` — one question at a time, produces `docs/PRD.md` with acceptance criteria
3. `/plan` — architecture sketch plus independent tickets, each sized for one agent
4. `claude --worktree <ticket>` per ticket — isolated checkout, isolated branch
5. `/ship` — validator, then reviewer, then a summary for you to decide on
6. `/retro` — turn what you explained twice into config, and commit it back **here**

The template is meant to compound. Step 6 is what makes the next project start smarter.

## The roles

Each role is a subagent in `.claude/agents/`. Tool access is enforced by the `tools` field,
not by asking nicely.

| Role | Model | Tools | maxTurns |
|---|---|---|---|
| `lead` | opus | all (inherits) | 60 |
| `dev` | sonnet | Read, Write, Edit, Bash, Grep, Glob, Skill — plus `isolation: worktree` | 80 |
| `validator` | sonnet | Read, Grep, Glob, Bash | 40 |
| `reviewer` | sonnet | Read, Grep, Glob, Bash | 40 |
| `researcher` | haiku | Read, Grep, Glob, WebSearch, WebFetch | 30 |

Sonnet is the default because cost scales with parallelism; opus is reserved for
decomposition and synthesis, where it actually pays off.

**An honest caveat on "read-only".** `validator` and `reviewer` have no `Write` or `Edit`,
but they do have `Bash` — and a shell can write files. That is a deliberate trade: without
Bash the validator cannot run tests and the reviewer cannot run `git diff`. The boundary
rests on the role prompt, not on a sandbox. For a real guarantee, enable
[sandboxing](https://code.claude.com/docs/en/sandboxing).

## Parallel work

Claude Code isolates parallel sessions natively. No board or orchestrator required:

```bash
claude --worktree feature-auth      # .claude/worktrees/feature-auth, branch worktree-feature-auth
claude --worktree "#1234"           # worktree from a pull request (quote the #)
claude agents                       # dashboard of every session — who is working, who is waiting
```

Both prerequisites ship with the template: `.claude/worktrees/` is already in `.gitignore`,
and `.worktreeinclude` already lists `.env` and `.env.local`. Extend the latter if your
project needs other gitignored files present — a worktree is a fresh checkout, so without
it every agent hits a missing `.env` on its first run.

Subagents can be isolated too: `isolation: worktree` in the frontmatter, already set on
`dev`. To drive many sessions from one process and steer them from a browser or phone:

```bash
claude remote-control --spawn worktree --capacity 5
```

## Permission rules: the syntax that actually matters

`.claude/settings.json` ships with a deny baseline covering secrets, build output, lock
files, binary assets, and destructive commands. There is **no `.claudeignore` file** in
Claude Code — that is a persistent myth; `permissions.deny` is the real mechanism.

| Form | Resolves to |
|---|---|
| `Read(.env)` | bare name uses gitignore semantics — matches **at any depth** (same as `Read(**/.env)`) |
| `Read(/src/**)` | anchored at the settings source; for `.claude/settings.json` that is the project root |
| `Read(~/.ssh/**)` | home directory |
| `Read(//tmp/x)` | absolute path — a **single** leading slash is *not* absolute |
| `Bash(npm run test *)` | prefix match; the space before `*` enforces a word boundary, so `ls *` will not match `lsof` |

Three ways to lose an hour:

- **`Write(...)` and `Glob(...)` are never matched** by file permission checks. They are
  accepted, silently do nothing, and warn at startup. Use `Edit(...)` and `Read(...)`.
- A `Read` deny rule also blocks `Edit` on the same path, but **not** `Write` or
  `NotebookEdit`. For "no tool may change this file", write both rules.
- Rules cover built-in tools and file commands in Bash (`cat`, `head`, `sed`), but **not
  arbitrary subprocesses** — a Python script will read `.env` straight past every rule.

A useful side effect: a `Read` deny rule also stops the IDE plugin from shipping an open
`.env` into context with every prompt.

## Cost and context

Parallel agents multiply token usage; subscription plans meter on a rolling 5-hour window
plus a weekly one, shared across Claude Code, chat, and Cowork and shared across models —
so switching with `/model` does not restore access.

- `/context` — where the current session's context is going, with suggestions
- `/usage` — plan usage bars, attributed to skills, subagents, plugins, and MCP servers.
  Press `d` / `w` to switch between 24 hours and 7 days
- Agent teams cost roughly **7x** a normal session when teammates run in plan mode: each
  teammate carries its own context window. Keep teams small, put teammates on Sonnet, and
  shut them down when their work is done
- Three to five parallel sessions is a realistic steady state. Beyond that the bottleneck
  is usually your own review throughput, not the model

## Notes

- All configuration is written in English on purpose: each role's `description` sits in
  context every session, and English tokenizes roughly twice as efficiently as Cyrillic.
- Agent frontmatter and settings schema evolve — check
  [code.claude.com/docs](https://code.claude.com/docs) if something is rejected at startup.
- Agent Teams (`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`) is experimental; `/resume` and
  `/rewind` do not restore teammates.
- The four workflow commands set `disable-model-invocation: true`, so Claude never fires
  them on its own and their descriptions stay out of context until you invoke them.

## License

MIT
