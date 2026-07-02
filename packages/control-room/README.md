# Control Room v2: Panel Cockpit + Trust Guard

Control Room is a Letta Code mod that gives long-running agent work a small, visible cockpit.

```text
CR [goal] Ship demo | [mode] edit | [next] Run smoke checks | [approval] ask | [verified] claimed | [risk] low | workspace
```

It keeps the session anchored around three things that otherwise blur together in chat:

- **Human intent**: what the user actually asked for and accepted.
- **Agent progress**: what the agent says it is doing next.
- **Harness reality**: what the runtime observed through tools, file changes, tests, and events.

The result is a lightweight trust surface: the user can glance at the cockpit and see the goal, mode, next step, approval posture, verification state, drift risk, and workspace.

## The tiny golden path

If you only remember four commands, use these:

```text
/cr goal Ship the demo
/cr next Run the smoke checks
/cr safe
/cr
```

That gives you:

1. a human-owned goal,
2. a visible next step,
3. approval required before the agent mutates Control Room progress state,
4. a compact cockpit view.

Then let the agent update its operational state with `control_room_update` as work progresses.

A minimal demo:

```text
/cr goal Polish Control Room for submission
/cr next Verify the cockpit and trust guard
/cr safe
/cr
```

Agent call:

```json
{
  "mode": "verify",
  "next": "Run live smoke checks",
  "verificationState": "claimed",
  "checkpoint": "Ready to test runtime behavior"
}
```

The important part: the agent can claim verification, but it cannot turn that claim into human acceptance.

## Why agents need a cockpit

Long-running coding sessions drift. The user sets a goal, the agent explores, tools run, files change, tests pass or fail, context compacts, and eventually nobody has a crisp answer to:

- What are we actually trying to do?
- Who set that goal?
- What is the next step?
- Has the result been verified, claimed, or merely hoped for?
- Did something change after verification?
- Can the agent silently rewrite its own progress state?

Control Room makes those answers explicit.

It is not a project manager. It is a trust layer for agentic work.

## The three layers of Control Room

Control Room works because it does not treat every piece of state as equally trustworthy. It keeps three layers separate.

### 1. Human layer: intent and acceptance

The human layer records things only the user should own:

- the real goal
- explicit acceptance
- human verification
- approval mode

Commands that write human-owned state:

```text
/cr goal <text>
/cr verified [note]
/cr safe
/cr lock
/cr unlock
```

Example:

```text
/cr goal Ship the contest demo
/cr verified Smoke test passed and UI looks right
```

This layer matters because the user should not need to wonder whether the agent quietly changed the target or marked its own work accepted.

### 2. Agent layer: progress and claims

The agent layer records what the agent believes is happening now:

- current working mode
- next step
- checkpoint notes
- evidence strings
- verification claims

Agent-facing tools:

```text
control_room_status
control_room_update
control_room_propose_goal
```

The key invariant:

```text
agent says verified -> Control Room records claimed
human says verified -> Control Room records verified
```

Agents can keep the cockpit current, but they cannot promote their own claim into human acceptance.

### 3. Harness layer: observed runtime facts

The harness layer records what Letta Code observed:

- tool starts and ends
- file-changing or shell-like activity
- verification-looking commands
- turn boundaries
- compaction and LLM events when supported
- recent tool signals

This layer is not semantic proof. It is evidence that something happened.

For example:

- if a test command is observed, Control Room can mark verification as `checking`;
- if file-changing activity happens after a claim, Control Room can mark verification as `stale`;
- if state may need attention at turn end, Control Room can remind the agent to update or continue normally.

Together, the three layers make the cockpit honest: human intent, agent narration, and runtime evidence stay visible and separate.

## The cockpit line

Control Room renders a compact panel line:

```text
CR [goal] <human goal> | [mode] <mode> | [next] <next step> | [approval] <auto|ask|locked> | [verified] <state> | [risk] <level> | <workspace>
```

Field meanings:

| Field | Meaning |
| --- | --- |
| `goal` | Human-owned goal for this workspace |
| `mode` | Current agent working mode: explore, plan, edit, verify, stuck, handoff |
| `next` | The next concrete step |
| `approval` | Whether agent progress updates are auto, ask, or locked |
| `verified` | Verification state: unknown, checking, claimed, verified, stale |
| `risk` | Lightweight drift heuristic |
| `workspace` | Current workspace key |

Color is used when supported:

```text
static labels use distinct soft/pastel ANSI colors
[verified] label uses pastel coral
[verified] value uses semantic colors: green verified, sunshine yellow checking/claimed/unknown, red stale
[approval] value stays plain/default text: auto, ask, or locked
[risk] value uses semantic colors: green low, sunshine yellow medium, red high
workspace is dim
```

The mod intentionally avoids fragile glyph-heavy UI after testing showed some symbols render as tofu boxes in Desktop terminal fonts.

## Verification words, painfully clarified

These commands sound similar, but they mean different things.

| Command/state | Who owns it? | Meaning |
| --- | --- | --- |
| `/cr verified [note]` | Human | The user confirms the current state is verified. This is the strongest signal. |
| `/cr verify <what>` | Human | This still needs checking. Sets mode toward verification work. |
| `/cr needs <what>` | Human | Alias for `/cr verify <what>`. Use it when something needs verification. |
| `/cr claim [note]` | Agent-grade/provisional | A claim that verification happened, but not human acceptance. |
| `/cr checkpoint [note]` | Human or workflow note | Records where the session is, without claiming verification. |
| `claimed` | Agent state | The agent says it checked something. Useful, but not final. |
| `verified` | Human state | The user confirmed it. |
| `stale` | Harness-derived state | Something changed after checking/claimed/verified. Re-check before trusting. |

A useful mental model:

```text
verify / needs  = please check this
claim           = agent says it checked this
verified        = human accepts this as checked
checkpoint      = breadcrumb, not proof
stale           = proof got old after a change
```

## Commands

Most users can start with the golden path above. The full command surface is here for power users.

```text
/cr                         show compact status
/cr detail                  show provenance and harness facts
/cr on|off                  enable/disable cockpit reminders
/cr goal <text>             set the human-owned goal
/cr goal clear              clear the goal
/cr mode <mode>             explore | plan | edit | verify | stuck | handoff
/cr next <step>             set the next step
/cr next clear              clear the next step
/cr verified [note]         human confirms current state is verified
/cr verify <what>           mark what still needs verification
/cr needs <what>            same as /cr verify
/cr claim [note]            provisional/agent-grade verification claim
/cr checkpoint [note]       record a checkpoint
/cr lock                    deny agent progress updates
/cr safe                    require approval for agent progress updates
/cr unlock                  allow agent progress updates
/cr expand|collapse         toggle expanded panel
/cr reset                   reset this workspace state
```

## Agent tools

Control Room exposes three agent-callable tools.

### `control_room_status`

Read-only and auto-approved.

Returns the current goal, mode, next step, verification state, approval state, drift heuristic, recent tool signal, changed file count when git is available, and state path.

### `control_room_update`

Agent progress update tool, governed by the Control Room approval mode.

The agent can update:

- mode
- next step
- checkpoint
- verification claim
- evidence string

The agent **cannot** use this tool to set the human-owned goal. If it tries to set `verificationState=verified`, Control Room downgrades that to `claimed`.

### `control_room_propose_goal`

Always asks for approval.

This is the native HITL path for goal changes proposed by the agent. If approved, the goal is recorded as human-owned with provenance:

```text
source: human
via: approved-agent-proposal
```

## Trust guard

Control Room registers a permission overlay for `control_room_update`.

```text
/cr unlock  -> approval auto: agent Control Room updates allowed
/cr safe    -> approval ask: agent Control Room updates require approval
/cr lock    -> approval locked: agent Control Room updates denied
```

In ask mode, the permission handler distinguishes approval and execution phases:

```text
approval phase  -> ask
execution phase -> allow after approval
```

This keeps the user in the loop without causing an approved tool call to be blocked a second time during execution.

`control_room_status` stays read-only. `control_room_propose_goal` always asks through the native approval path.

## Reminder loop

When Control Room is on, the mod can use the `turn_end` event as a lightweight self-check loop. It injects a continuation only when state may need attention:

- goal or next step is missing
- mode is `stuck` or `handoff`
- verification is `unknown`, `checking`, or `stale`
- a meaningful change or verification signal happened after the last reminder

Reminder text:

```text
Control Room checkpoint: state may need an update. If needed, call `control_room_update` or `control_room_propose_goal`; otherwise continue normally.
```

Important: the reminder uses a pending flag so it does **not** recursively trigger itself.

The loop is:

```text
assistant turn finishes
-> Control Room may inject one checkpoint reminder
-> agent either updates Control Room or continues normally
-> the reminder follow-up does not cause another reminder
```

`/cr off` pauses that reminder loop and renders the cockpit as paused:

```text
CR [off] paused | /cr on to resume | workspace
```

`/cr on` resumes it.

## Harness signals

Control Room observes supported Letta Code mod events when available:

```text
conversation_open
conversation_close
turn_start
turn_end
tool_start
tool_end
compact_start
compact_end
llm_start
llm_end
```

Event handlers are capability-guarded, so unavailable surfaces are skipped instead of failing mod load.

## Persistent state

State lives at:

```text
~/.letta/mods/control-room.state.json
```

State is keyed by workspace/cwd so separate projects keep separate Control Room context.

The mod migrates older v1-shaped state into the v2 structure on load.

## Installation

After this package is published in the Letta mods catalog:

```bash
letta install npm:@letta-ai/control-room
```

Then reload mods in Letta Code:

```text
/reload
```

For local development, copy or symlink `mods/index.ts` into `~/.letta/mods/control-room.ts`, then run `/reload`.

## Longer demo script

Use this when you want to show the trust mechanism, not just the pretty line.

### 1. Set the human goal

```text
/cr goal Build Control Room v2 contest demo
/cr mode edit
/cr next Verify the cockpit, tools, lock, and approval flow
```

### 2. Show the cockpit and provenance

```text
/cr
/cr detail
```

### 3. Let the agent narrate progress

Agent calls:

```text
control_room_update({
  "mode": "verify",
  "next": "Run live smoke checks",
  "checkpoint": "Ready to test runtime behavior"
})
```

### 4. Show Trust Guard

```text
/cr safe
```

Agent calls `control_room_update`; the user gets an approval prompt. After approval, execution proceeds.

```text
/cr lock
```

Agent calls `control_room_update`; the tool is denied.

```text
/cr unlock
```

Agent progress updates are allowed again.

### 5. Show the reminder loop

```text
/cr on
```

When Control Room detects stale/checking/missing state at the end of a turn, it reminds the agent to update or continue normally.

```text
/cr off
```

The reminder loop pauses.

## Safety notes

- Control Room stores local workflow state only under `~/.letta/mods/control-room.state.json`.
- It does not modify project files.
- It does not call an LLM.
- It does not import Letta Code internals.
- Its permission overlay applies to its own agent update tool, not arbitrary project tools.
- Human-owned goal and human verification remain separate from agent claims.
