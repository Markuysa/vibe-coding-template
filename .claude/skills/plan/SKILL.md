---
name: plan
description: Turns docs/PRD.md into an architecture sketch and a set of independent tickets sized for one agent each.
argument-hint: [optional: which part of the PRD to plan]
disable-model-invocation: true
allowed-tools: Read, Write, Edit, Glob, Grep
---

Scope: $ARGUMENTS

Read `docs/PRD.md` first. If it does not exist, stop and say to run `/spec` — planning
without acceptance criteria produces tickets that nobody can verify.

**Write no implementation code in this command.** You are producing a plan; `dev` writes
the code. If you find yourself drafting a function body, you have gone too far.

## Step 1 — Architecture

Read enough of the existing code to know what is already there. Then sketch, in
`docs/ARCHITECTURE.md` (template at `docs/ARCHITECTURE-template.md` if present):

- The shape of the change: which modules, which boundaries, what talks to what
- Decisions that are hard to reverse later, each with the alternative you rejected and why.
  These belong in `docs/decisions/` as ADRs — one file per decision
- What you are deliberately *not* building yet

Where you are guessing about the existing system, say so rather than presenting a guess
as a reading.

## Step 2 — Tickets

Split the work so each ticket is:

- **Independent.** Two tickets touching the same file cannot run in parallel — that is the
  overwrite scenario the whole worktree setup exists to prevent. If two pieces are coupled,
  make them one ticket with an ordered plan inside, not two.
- **One agent's worth.** If a ticket needs three checkpoints and a day, split it.
- **Verifiable.** Carries its own acceptance criteria, traced back to the PRD.

For each ticket write:

```
### <short title>
Worktree:  <kebab-case-name>          → claude --worktree <name>
Files:     <likely paths>
Depends:   <other tickets, or "none">
Acceptance criteria:
  - ...
```

## Step 3 — Order

State explicitly which tickets can start at once and which must wait. Recommend how many
to run in parallel: three to five is the working range, not "all of them". Parallelism is
bounded by review throughput and by plan usage limits, not by how many tickets exist.

End with what you are least sure about in this plan. That is the part most worth a human
looking at before any agent starts.
