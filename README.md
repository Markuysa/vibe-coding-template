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
  skills/                      # /spec /plan /ship /retro + execute-ticket
  scripts/setup.sh             # dependency install for cloud sessions
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
`dev`.

## Running it from your phone

None of this lives in the repository — it is machine- and account-level configuration, so
there is nothing here to commit. Set it up once per machine and it applies to every project
using this template.

| Feature | What it is for | How to start it |
|---|---|---|
| **Agent View** | See every session at once: who is working, who is blocked, who finished | `claude agents`, or `←` from any session. The same list appears in the Code tab on claude.ai and in the Claude mobile app |
| **Remote Control** | Steer a running session: live output, approve or reject changes, redirect it | `claude remote-control` (server mode, spacebar shows a QR code) · `claude --remote-control` (normal interactive session) · `/remote-control` from a session already running |
| **Dispatch** | File *new* work from your phone; it runs on your machine | Desktop app → **Cowork** tab → pair with the mobile app |

Mnemonic: **Agent View watches · Remote Control steers · Dispatch creates.**

The combination worth knowing about is server mode with worktrees:

```bash
claude remote-control --spawn worktree --capacity 5
```

One process, up to five sessions, each in its own isolated worktree, all steerable from a
browser or phone. This is the closest native equivalent to a task board.

### Push notifications

No third-party app needed. Run `/config` and enable **Push when Claude decides** and
**Push when actions required**. They fire while Remote Control is connected. You can also
ask in the prompt: *"notify me when the tests finish"*. Notifications are suppressed while
you are typing in the connected terminal. If `/config` shows **No mobile registered**, open
the Claude app once so it refreshes its push token.

The payoff for the workflow above: when an agent hits a permission prompt at the plan
checkpoint, you approve it from wherever you are instead of it sitting idle until you get
back to the desk.

### Requirements and limits

- **Plan**: Remote Control needs Pro, Max, Team, or Enterprise. Dispatch is **Pro or Max
  only** — not available on Team or Enterprise.
- **Auth**: claude.ai login via `/login`. **API keys do not work** — unset
  `ANTHROPIC_API_KEY` if it is in your environment. Tokens from `claude setup-token` do not
  work either.
- **Endpoint**: unavailable on Bedrock, Google Cloud's Agent Platform, and Microsoft
  Foundry, and whenever `ANTHROPIC_BASE_URL` points somewhere other than
  `api.anthropic.com`.
- **Workspace trust**: run `claude` in the project directory once to accept the trust dialog.
- Your machine must stay awake, and the local process must keep running — closing the
  terminal ends the session. An outage longer than roughly 10 minutes times the session out.
- Starting an ultraplan session disconnects Remote Control; both occupy the same interface.
- Some commands are local-only (`/plugin`, `/resume`). From the phone you get `/compact`,
  `/clear`, `/context`, `/usage`, and `/model` / `/effort` with the value passed as an
  argument instead of a picker.

### What actually leaves your machine

While Remote Control is connected, **the full session transcript** — your messages, the
model's replies, and tool activity — is stored on Anthropic's servers. That is what keeps
devices in sync and allows reconnecting after a drop. Code execution and filesystem access
stay local. Organizations with Zero Data Retention cannot use it, and the
`disableRemoteControl` setting turns it off entirely.

Practical consequence: do not run Remote Control on sessions where secrets or regulated
data are pulled into context.

## Running it in the cloud

Everything above runs on your machine. The same configuration also runs unattended on
Anthropic's infrastructure, because a cloud session is a fresh VM with your repository
cloned — so `CLAUDE.md`, `.claude/agents/`, `.claude/skills/` and `.claude/settings.json`
all come along. Anything living in `~/.claude/` does not, which is why this template keeps
everything in the repo.

The loop is: GitHub issues hold the queue, a routine picks one up, a pull request comes back.

### 1. Turn a plan into issues

`/plan` prints its tickets, asks you to confirm, then creates them with `gh issue create`.
Each issue carries its acceptance criteria verbatim — that text is the contract the
executor and validator work from. Issues are labeled `ready` or `blocked`.

### 2. Set up the cloud environment

At [claude.ai/code](https://claude.ai/code), point your environment's **Setup script** at
`.claude/scripts/setup.sh`. It installs project dependencies into the VM and the result is
cached between sessions. Extend the project-specific section at the bottom for codegen,
migrations, or seed data.

### 3. Create the routine

At [claude.ai/code/routines](https://claude.ai/code/routines), create a routine with this
prompt — short on purpose, because the logic lives in the repo where it is versioned:

```
Read the issue number from the routine-fire-payload block,
then run the execute-ticket skill for that issue.
```

Attach an **API trigger** and generate a token. Select the repository and the environment
from step 2.

### 4. Dispatch

```bash
curl -X POST https://api.anthropic.com/v1/claude_code/routines/<routine-id>/fire \
  -H "Authorization: Bearer <token>" \
  -H "anthropic-beta: experimental-cc-routine-2026-04-01" \
  -H "anthropic-version: 2023-06-01" \
  -H "Content-Type: application/json" \
  -d '{"text": "Work on issue #42"}'
```

One call per issue. Dispatching explicitly rather than letting the routine hunt for work
avoids two sessions claiming the same issue, and keeps parallelism under your control —
which matters, because cloud sessions draw down the same plan limits as everything else.

The response contains a session URL. Watch it from the browser or the Claude mobile app,
answer questions, redirect it.

### 5. Review

`execute-ticket` opens a pull request from a `claude/`-prefixed branch with `Closes #42`,
the acceptance criteria checked off, and the test output. You review and merge. It never
merges, and the deny rules make sure of that rather than trusting the prompt.

Issue labels track state: `ready` → `in-progress` → `in-review`, or `needs-attention` when
the validator came back red.

### What changes when nobody is watching

Routines run with **no approval prompts at all**. Three consequences worth internalizing:

- **`ask` rules do nothing.** They are an interactive guard only. Anything that must never
  happen belongs in `deny` — which is why `gh pr merge`, `git merge`, `git rebase`, and
  `gh release create` sit there rather than in `ask`.
- **`execute-ticket` must not set `disable-model-invocation`.** That flag would stop a
  routine from invoking it. The four `/`-commands set it; this one deliberately does not.
- **There is no secrets store yet.** Environment variables are visible to anyone who can
  edit the environment. `.worktreeinclude` does not apply in the cloud — it is a local
  worktree mechanism.

Interactive work stays interactive: `/spec` and `/plan` are conversations and are pointless
in an unattended run. Decompose with Claude, dispatch the result.

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

- Agent View and Remote Control are research previews; flags and behavior may change.
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
