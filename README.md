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
  agents/                      # lead, designer, frontend, backend, qa, dev, validator, reviewer, researcher
  skills/                      # /spec /plan /ship /retro /autopilot /board + execute-ticket, next-ticket, unblock
  autopilot.json               # kill switch for unattended execution
  scripts/setup.sh             # dependency install for cloud sessions
.github/workflows/ci.yml       # the merge gate autopilot depends on
docs/
  tickets/                     # the queue: one file per ticket, statuses derived from git
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
3. `/plan` — architecture sketch plus independent tickets in `docs/tickets/`, one file each
4. `claude --worktree <ticket>` per ticket — isolated checkout, isolated branch
5. `/ship` — validator, then reviewer, then a summary for you to decide on
6. `/retro` — turn what you explained twice into config, and commit it back **here**

The template is meant to compound. Step 6 is what makes the next project start smarter.

## The team

Each role is a subagent in `.claude/agents/`. Tool access is enforced by the `tools`
field, not by asking nicely. Every ticket names the `role` that implements it;
`execute-ticket` delegates accordingly, and the lead routes by the same table.

| Role | Model | Takes | maxTurns |
|---|---|---|---|
| `lead` | opus | decomposition, routing, synthesis — never implements | 60 |
| `designer` | sonnet | tokens, primitives, design system, component states | 60 |
| `frontend` | sonnet | screens and client logic, on the design system + API contract | 80 |
| `backend` | sonnet | APIs, storage, pipelines — behind the contract | 80 |
| `qa` | sonnet | e2e/integration suites that prove acceptance criteria | 60 |
| `dev` | sonnet | generalist fallback for cross-cutting tickets | 80 |
| `validator` | sonnet | runs tests/lint/types, green-red verdict, no write tools | 40 |
| `reviewer` | sonnet | reviews the diff, no write tools | 40 |
| `researcher` | haiku | scouting code and docs, returns a summary | 30 |

All implementers carry `isolation: worktree`, so parallel tickets cannot touch each
other's files. Sonnet is the default because cost scales with parallelism; opus is
reserved for decomposition and synthesis, where it actually pays off.

**Handoffs are how work flows through the team.** Each ticket ends with the implementer
writing a `## Handoff` section into the ticket file — what exists now, where it lives,
what to import rather than rebuild. The next specialist starts by reading the handoffs of
its ticket's dependencies. Designer → frontend → qa is a chain of these sections, carried
by git: a handoff lands in main only when the ticket's branch merges.

**Each role's skill pool is editable.** The `skills:` frontmatter line preloads skills
into that role — the builder roles ship with [ponytail](https://github.com/DietrichGebert/ponytail)
attached this way. Add a skill to a role by adding its name to the line; remove it by
deleting it; write new skills as `.claude/skills/<name>/SKILL.md`. Roles are markdown —
editing the team is editing files.

The ponytail ruleset (MIT, vendored into `.claude/skills/ponytail/`) is scoped to the
builder roles on purpose: `validator` and `reviewer` keep their own priorities, and a
second opinion about code volume would only muddy their reports. Precedence is stated in
each builder's prompt — acceptance criteria, `CLAUDE.md`, and the test rule all outrank
it. Minimalism shapes the solution; it never trims the scope.

It is vendored rather than installed as a plugin so it travels into cloud sessions with the
repository, adding no marketplace fetch at session start and no third-party hooks. The
tradeoff is that upstream fixes are not pulled automatically.

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

The loop is: `docs/tickets/` holds the queue, a routine picks one up, a pull request comes
back. The queue is files in the repository — no issue tracker, no API, no hosting lock-in.
Statuses are **derived from git**, never stored: `done` can only land in main by merging
the ticket's branch, `ready`/`blocked` are computed from dependencies, and a claimed
ticket is simply one whose `claude/NNN-*` branch exists. The board cannot disagree with
reality because the board *is* reality. Full model: `docs/tickets/README.md`. Render it
any time with `/board`.

### 1. Turn a plan into ticket files

`/plan` prints its tickets, asks you to confirm, then writes one file per ticket to
`docs/tickets/NNN-slug.md` and commits them. Each file carries `role`, `depends`, and the
acceptance criteria verbatim — that text is the contract the executor and validator work
from.

### 2. Set up the cloud environment

At [claude.ai/code](https://claude.ai/code), point your environment's **Setup script** at
`.claude/scripts/setup.sh`. It installs project dependencies into the VM and the result is
cached between sessions. Extend the project-specific section at the bottom for codegen,
migrations, or seed data.

### 3. Create the routine

At [claude.ai/code/routines](https://claude.ai/code/routines), create a routine with this
prompt — short on purpose, because the logic lives in the repo where it is versioned:

```
Read the ticket id from the routine-fire-payload block,
then run the execute-ticket skill for that ticket.
```

Attach an **API trigger** and generate a token. Select the repository and the environment
from step 2.

### 4. Dispatch

Name the ticket explicitly:

```bash
curl -X POST https://api.anthropic.com/v1/claude_code/routines/<routine-id>/fire \
  -H "Authorization: Bearer <token>" \
  -H "anthropic-beta: experimental-cc-routine-2026-04-01" \
  -H "anthropic-version: 2023-06-01" \
  -H "Content-Type: application/json" \
  -d '{"text": "Work on ticket 3"}'
```

Or let the queue decide — see below.

### Working the queue without naming tickets

- **`next-ticket`** picks the lowest-id ready ticket and runs `execute-ticket` on it. It
  claims by **pushing the ticket branch** — a second run trying the same ticket fails the
  push and moves on, so the lock is git itself. It counts in-flight branches first and
  stops at the limit (default **1**). One ticket per run — it never loops.
- **`unblock`** is now a read-only report: with derived statuses a merge unblocks
  dependants by itself, so the skill just tells you what the latest merge freed. It is
  kept because routine prompts call it before `next-ticket`.

Unblocking keys off tickets being **merged into main**, not off pull requests being
opened, and that distinction is load-bearing: worktrees and cloud sessions branch from
the default branch, so a ticket freed while its dependency is still an open PR would
build against a tree that does not contain it. The derived model makes this automatic —
`done` cannot exist in main without the merge.

Wire them into one routine with a **GitHub trigger** on `pull_request.closed` filtered to
merged, and this prompt:

```
Run the unblock skill, then run the next-ticket skill.
```

Now merging a pull request unblocks whatever depended on it and starts the next ticket. The
chain advances on your merges, so a human review sits between every ticket by construction.
Add a daily schedule trigger to the same routine as a heartbeat, in case a run dies without
merging anything.

To run the queue locally instead, just ask for it in a session: *"run next-ticket"*.

The response contains a session URL. Watch it from the browser or the Claude mobile app,
answer questions, redirect it.

### 5. Review

`execute-ticket` opens a pull request from a `claude/`-prefixed branch: acceptance
criteria checked off, test output, and the ticket file flipped to `status: done` as its
last commit — merging is what lands `done` in main. You review and merge. It never merges
itself, and the deny rules make sure of that rather than trusting the prompt.

On GitLab it opens an MR via `glab`; with no remote at all it stops after the `done`
commit and names the branch for you to review — the queue is hosting-agnostic, only the
merge step differs. A red validator run parks the ticket at `needs-attention` with the
failing output written into the ticket file itself.

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

### Autopilot: running the queue without merging by hand

By default the chain advances on your merges. If you want it to run unattended end to end,
the merge gate does not disappear — it moves from you to CI.

**Do not enable this without required status checks.** `gh pr merge --auto` on a repository
with no required checks merges immediately, which is not autopilot, it is merging whatever
the agent wrote. `execute-ticket` refuses to queue an auto-merge in that situation.

Set it up in this order:

1. **CI that runs what the validator runs.** `.github/workflows/ci.yml` ships with the
   template: Go and Node jobs that no-op until the project has that stack, plus a secrets
   check. Add your own commands; keep the secrets job even if you drop the rest, because
   under autopilot nobody reads the diff before it merges.
2. **Branch protection** on the default branch: require those checks, and require the branch
   to be up to date before merging (`strict`). Leave required reviews off — that is the
   human gate you are deliberately replacing.
3. **Enable auto-merge** on the repository (`gh repo edit --enable-auto-merge`).
4. **Permissions.** `deny` beats `allow`, so a blanket `Bash(gh pr merge *)` deny blocks
   `--auto` too. Narrow it: allow `Bash(gh pr merge --auto --squash *)`, send every other
   merge form to `ask`, and keep `Bash(git merge *)` denied. Be honest with yourself that
   the permission rule is no longer the guarantee — branch protection is.
5. **Say autopilot in the routine prompt**, which is the only thing that turns step 8 of
   `execute-ticket` on:

   ```
   Run the unblock skill, then run the next-ticket skill. Autopilot mode.
   ```

Then start the whole queue with one command:

```
/autopilot on
```

It refuses unless all four preflight checks pass — a CI workflow exists, branch protection
lists required status checks, auto-merge is enabled, and something is actually `ready`. The
second of those is the one that matters: with no required checks, `--auto` merges on the
spot and autopilot means the opposite of what it looks like.

`/autopilot off` stops the next ticket from starting. It does not kill a run already in
flight, and it says so rather than implying everything stopped. The flag lives in
`.claude/autopilot.json` and is committed, because cloud sessions read the clone — an
uncommitted edit changes nothing — and because every flip should be in the history.
`/autopilot status` reports the queue without changing anything.

Keep `next-ticket` at one ticket in flight. Paired with `strict`, that is coherent: each
merge invalidates every other open branch, so a wider queue would just thrash on rebases.

What you keep: every change still arrives as a reviewable pull request, CI still has to be
green, and the history is still linear and revertible. What you give up: nobody sees a wrong
approach until it is already on the default branch — and with a dependency chain, whatever
merges first is what every later ticket builds on.

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

The `allow` list follows one principle: **inside its own worktree an agent may do anything
reversible; the gate stands at the boundary.** Building, testing, linting, formatting,
staging, and committing are mechanical and pre-approved. Crossing out of the worktree is
not: `git push` asks, `git merge`, `gh pr merge`, and `gh release create` are denied
outright. Adding a dependency (`go get`, `npm install <pkg>`) also asks, because that is a
decision rather than mechanics, and `git worktree remove --force` asks because it destroys
uncommitted work.

The shipped rules are stack-agnostic git and `gh`. **Add your project's build and test
commands** — the ones you put in `CLAUDE.md` — or the validator will prompt on every run.

Anything a workflow needs repeatedly belongs in `allow`, not in a skill's `allowed-tools`.
A skill's grant covers only the turn that invoked it and clears on your next message — so
in a multi-turn command like `/plan`, which confirms with you before writing tickets, the
grant has already expired by the time the follow-up commands run. The `gh` rules in
`allow` exist for exactly that reason.

Two lessons from running this for real, both of which cost an hour:

- **Bash rules match a prefix, so argument order matters.** `Bash(gh pr merge --auto
  --squash *)` does not match `gh pr merge 22 --auto --squash`, because the flags do not
  come first. Put flags immediately after the subcommand, or write the rule to match how
  you actually invoke it. This is why the real guarantee is branch protection, not the
  permission rule.
- **Permissions come from the project you opened the session in.** Running a session in
  one repository while operating on another silently uses the wrong `settings.json`, and
  every rule you carefully added appears not to work.

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
