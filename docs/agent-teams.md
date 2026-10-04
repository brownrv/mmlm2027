# Agent Teams — Master Reference

Working reference for planning, spawning, and running Claude Code agent teams in this project.
Source: https://code.claude.com/docs/en/agent-teams (captured 2026-10-04). Agent teams are
**experimental**; re-check the source page when behavior seems to differ from this guide.

---

## 1. Quick facts

| Item | Value |
| :- | :- |
| Enable | `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` (set in `.claude/settings.local.json` for this project) |
| Disable | Set it to `0`. Settings-file `env` changes apply to the running session on save |
| Requires | An interactive session. `-p` / headless / Agent SDK sessions never spawn teammates |
| How a teammate is created | The lead calls the Agent tool **with a `name`** while teams are enabled (not a fork, and no `isolation` on the call). No confirmation prompt |
| Team shape | 1 lead (fixed for the session) + N teammates. One team per session. No nested teams |
| Display mode here | **In-process**. Split panes aren't supported in the VS Code terminal or Windows Terminal, and this is a Windows + VS Code setup |
| Recommended size | 3–5 teammates, with 5–6 tasks per teammate |

**Side effect to remember:** while teams are enabled, *any* subagent that Claude names launches as
a teammate. If you only want a plain subagent, don't pass a `name` (or turn teams off).

---

## 2. Decide: team, subagent, or single session?

Use this check before forming a team. Teams use **significantly more tokens** (each teammate is a
full Claude instance with its own context) and add coordination overhead.

| Situation | Use |
| :- | :- |
| Sequential steps, heavy dependencies, edits to the same files | Single session |
| Focused side task where only the result matters (search, verify, summarize) | Subagent |
| Parallel workers that must **share findings, challenge each other, or self-coordinate** | Agent team |
| Separate sessions you run yourself that just need to pass notes | Cross-session messaging |
| Manual parallel implementation without automated coordination | Git worktrees |

**Strong team use cases:**
- **Research and review**: parallel lenses on one problem, then cross-examination.
- **New modules or features**: each teammate owns a separate piece or set of files.
- **Debugging with competing hypotheses**: adversarial investigators avoid anchoring on the first plausible theory.
- **Cross-layer changes**: data, model, evaluation and tests each owned by a different teammate.

**Red flags against a team:** the work is mostly sequential, more than one teammate would touch the
same file, the task is small enough that coordination costs more than it saves, or the work is routine.

---

## 3. Architecture

| Component | What it is | Where it lives |
| :- | :- | :- |
| Team lead | Main session; spawns, assigns, synthesizes | — |
| Teammates | Independent Claude Code sessions | — |
| Team config | Runtime state; `members` array (name, agent ID, agent type) | `~/.claude/teams/{team-name}/config.json` (deleted at session end) |
| Mailbox | One JSON inbox per agent | `~/.claude/teams/{team-name}/inboxes/{agent-name}.json` |
| Task list | Shared work items with dependencies | `~/.claude/tasks/{team-name}/` (persists; follows `cleanupPeriodDays`) |

- `team-name` = `session-` + the first 8 characters of the session ID. It's auto-generated.
- **Never hand-edit or pre-author the team config.** It's overwritten on each state update. A
  project file like `.claude/teams/teams.json` is *not* recognized as configuration.
- **Reusable roles belong in subagent definitions** (`.claude/agents/*.md`), not in team config.
- Teammates can read the team config to discover other members.

### Tasks
- States: `pending` → `in progress` → `completed`.
- A pending task whose dependencies aren't done can't be claimed. Completing a task automatically
  unblocks the tasks that depend on it.
- Claiming uses file locking, so there are no double-claims.
- Assignment: either the lead assigns, or a teammate **self-claims** the next unassigned,
  unblocked task when it finishes one.
- Agents without the Task tools (`TaskCreate`, `TaskGet`, `TaskList`, `TaskUpdate`) coordinate by
  message only.

### Messaging
- `SendMessage` to a teammate **by name**. There's no broadcast: send one message per recipient.
- Delivery is automatic and the lead doesn't poll. A message counts as sent only when the write to
  the recipient's inbox succeeds; otherwise the sender gets an error.
- **Idle notification**: when a teammate stops, the lead is notified automatically and the
  notification includes the teammate's final answer (or the error text if it failed).
- Messaging a stopped in-process teammate revives it in the same session, with its saved
  conversation and the message as its next prompt. This doesn't work after `/resume`.
- Messages between agents are marked as coming from another Claude session, not the user. A
  teammate **can't grant consent or approve permissions** on the user's behalf, and can't relay a
  denied action to another teammate. In auto mode, the classifier reviews every inter-agent message
  and treats relayed approvals as untrusted.

---

## 4. Context: what a teammate knows

A teammate **does** load:
- CLAUDE.md, MCP servers and skills from project/user settings (respecting `--setting-sources`).
- The **spawn prompt** from the lead.

A teammate **does not** get:
- The lead's conversation history. Anything the lead learned, the teammate doesn't know unless the
  spawn prompt or a message says it.

**Consequence: every spawn prompt must be self-contained.** See the template in §8.

---

## 5. Models, effort, permissions

### Model selection (first match wins)
1. A model named in the spawn prompt for that teammate.
2. The subagent definition's `model` (`inherit` = the lead's model).
3. `CLAUDE_CODE_SUBAGENT_MODEL`, if it's set and isn't `inherit`.
4. The lead's current model.

- `CLAUDE_CODE_SUBAGENT_MODEL_FORCE=1` skips steps 1–2 (v2.1.257+).
- `teammateDefaultModel` was removed (v2.1.234). Name the model in the prompt instead.
- The org `availableModels` allowlist can substitute a blocked model (family alias → newest
  permitted version; otherwise → the lead's model).
- A teammate's model and fast mode are **fixed at spawn**. `/model` and `/fast` from a teammate's
  view affect the lead only.
- Effort: teammates inherit the lead's effort level by default.

### Permissions
- Teammates start in the lead's permission mode, **except `dontAsk`**, which isn't inherited.
  `--dangerously-skip-permissions` propagates.
- Permission modes can't be set per teammate at spawn. They can only be changed afterwards.
- Teammate permission prompts surface **in the lead session**. To cut friction, pre-approve common
  operations in permission settings *before* spawning.
- Plan approval is the exception: when the lead is in plan mode, spawned teammates work read-only
  until their plan is ready, then send an approval request that the lead **auto-approves without
  review**. Their later edits and commands still go through normal permission prompts.

---

## 6. Subagent definitions as teammate roles

Define a role once in `.claude/agents/<role>.md` and reuse it as either a subagent or a teammate:
`Spawn a teammate using the <role> agent type to ...`

How a definition applies to an **in-process** teammate (the mode used here):

| Field | Applied? |
| :- | :- |
| `tools` | Yes. Restricts the teammate's tools; `SendMessage` and the Task tools are always added |
| `disallowedTools` | Yes, but `SendMessage` and the Task tools can't be removed |
| `model` | Yes, unless the spawn prompt names a model |
| `effort` | Yes |
| Body | **Appended** to the default system prompt (split-pane teammates *replace* it instead) |
| `skills` | **No**. The teammate loads skills from project and user settings |
| `mcpServers` | **No**. The teammate loads MCP from project and user settings (split-pane only) |

- When a revived teammate's definition lives in a project's `.claude/agents/`, the definition is
  re-applied only if **that exact folder** is trusted. Without trust, the teammate comes back
  without the definition's tools and instructions.
- In-process teammates **can't run background subagents**. `background: true` and
  `run_in_background: true` error out or run in the foreground.

---

## 7. Quality gates with hooks

| Hook | Fires when | Exit code 2 |
| :- | :- | :- |
| `TeammateIdle` | A teammate is about to go idle | Sends feedback and keeps it working |
| `TaskCreated` | A task is being created | Blocks creation and sends feedback |
| `TaskCompleted` | A task is being marked complete | Blocks completion and sends feedback |

Typical uses: require tests to pass before a task can complete, reject vague task titles, or stop a
teammate going idle while its owned tasks are unfinished.

---

## 8. Playbook: building an effective team

### Step 1 — Design before spawning
1. Confirm the work passes the §2 decision check.
2. Split it into **self-contained units with clear deliverables** (a function, a test file, a review,
   a findings section). Units that are too small waste coordination; units that are too large run
   too long without check-ins.
3. **Assign file ownership**: each teammate owns a disjoint set of files. Two teammates editing one
   file overwrite each other.
4. Map dependencies into the task list so blocked work can't start early.
5. Choose 3–5 teammates. Three focused teammates beat five scattered ones. With 15 independent
   tasks, start with 3 teammates.
6. Give each teammate a **predictable name** so later prompts and messages can address it.
7. Pick a model per role (for example, a cheaper model for mechanical work and the strongest model
   for architecture or adjudication).
8. Pre-approve the permissions the team will need.

### Step 2 — Spawn prompt template

```text
Name: <short-role-name>
Role: <one line — the lens or ownership area>
Goal: <the concrete deliverable and what "done" means>
Context: <facts from the lead's conversation this teammate needs: paths, decisions made,
          constraints, data locations, conventions. Assume it knows nothing beyond CLAUDE.md.>
Owns (may edit): <files/dirs>
Read-only: <files/dirs it may read but must not edit>
Coordinate with: <teammate names and what to exchange with them>
Tasks: claim from the shared list; mark each task completed as soon as it's done.
Report: <format: severity-ranked findings / summary + diff paths / evidence for or against hypothesis>
Stop when: <exit criteria>. If blocked or erroring, message the lead instead of stopping silently.
```

### Step 3 — Run and steer
- **Monitor** teammates' progress, redirect failing approaches, and synthesize as results arrive.
  Don't leave the team unattended for long.
- **The lead must wait.** It shouldn't start implementing teammates' tasks itself. Synthesize only
  after the idle notifications arrive.
- **Watch task status lag**: if a task looks stuck, check whether the work is actually done, then
  update its status or nudge the owner.
- **The lead can stop too early**: verify every task is completed before declaring the team done.
- **Teammates that stop on errors**: send more instructions, or spawn a replacement with the
  context the original had. Messaging a teammate that's waiting on an API retry makes it retry
  immediately.

### Step 4 — Wrap up
- Ask each teammate by name to shut down. It can accept or reject with a reason, and shutdown waits
  for the current tool call to finish.
- The team directories clean up automatically when the session ends. The task list persists.
- After `/resume` or `/rewind`, in-process teammates are gone. Spawn new ones instead of messaging
  the old names.

---

## 9. Team patterns

### Parallel review (distinct lenses)
```text
Spawn three teammates to review <target>:
- "security": security implications
- "perf": performance impact
- "tests": test coverage
Each reports severity-ranked findings. Synthesize after all three finish.
```
Why it works: a single reviewer tends to fix on one class of issue. Distinct lenses get covered in parallel.

### Competing hypotheses (adversarial debate)
```text
<Symptom>. Spawn 5 teammates, each owning one hypothesis. Have them message each other to
try to disprove each other's theories, like a scientific debate. Record the consensus in
<findings doc>.
```
Why it works: sequential investigation anchors on the first theory, while independent
investigators who attack each other converge on the cause that survives.

### Multi-perspective design exploration
```text
Spawn three teammates to explore <design question>: one on UX/usability, one on technical
architecture, one playing devil's advocate. Lead synthesizes a recommendation.
```

### Parallel implementation (owned modules)
```text
Spawn <N> teammates, each owning <module/dir>. Shared interfaces are defined in <file>
(read-only for all). Each writes code + tests for its module only.
```
Lead first: define the interfaces and contracts, create the tasks with dependencies, then spawn the
teammates. A `TaskCompleted` hook can require tests to pass.

### Plan-then-build (risky changes)
Put the lead in plan mode, then: `Spawn an architect teammate to <refactor X>.` The teammate plans
read-only, the plan is auto-approved, and then it implements. Because approval is automatic, the
lead's spawn prompt should spell out what the plan must contain.

**For newcomers:** start with research and review teams (no code writing) before moving to
parallel implementation.

---

## 10. Troubleshooting

| Symptom | Fix |
| :- | :- |
| No teammates appeared | Check the agent panel (↑/↓, Enter). Maybe Claude used subagents instead; ask explicitly for an *agent team*. The task may also have looked too simple for a team |
| Teammate row vanished | It's hidden, not stopped. Idle rows hide 30s after the whole panel goes idle. Message it by name |
| `N idle agents` row | More than 3 teammates are idle and the extras collapsed. Select it and press Enter to expand |
| Unwanted teammates instead of subagents | Named subagents become teammates while teams are on. Omit `name`, or set the env var to `0` (higher-precedence or managed settings can still force it on) |
| Too many permission prompts | Pre-approve common operations in permission settings before spawning |
| Teammate stopped early | Open its transcript, then send instructions or spawn a replacement |
| Lead declared done too soon | Tell it to keep going until all tasks are completed |
| Task stuck in progress | Status lag. Verify the work, then update the status or nudge the owner |
| Lead messages ghosts after resume | Teammates aren't restored. Spawn new ones |
| Inbox write failed | Disk full or directory not writable. The sender gets an error and nothing is sent |

### In-process controls
- **↑/↓**: select a teammate. **Enter**: open its transcript or message it. **Esc**: clear the
  selection, or interrupt the teammate's turn while viewing it. **x**: stop the selected teammate.
  **Ctrl+T**: toggle the task list.
- While viewing a teammate, plain text and skills go to the teammate, and built-in commands go to
  the lead. `/compact`, `/clear` and `/rewind` ask for confirmation; `/model` and `/fast` are blocked.

---

## 11. Limitations (experimental)

- `/resume` and `/rewind` don't restore in-process teammates.
- Task status can lag and block dependent tasks.
- Shutdown can be slow, because teammates finish the current request first.
- One team per session. Teams can't be shared across sessions.
- No nested teams: only the lead manages the team.
- In-process teammates can't run background subagents.
- The lead is fixed for the session's lifetime.
- Permission modes can't be set per teammate at spawn.
- Split panes need tmux or iTerm2 (`it2`), and aren't supported in VS Code, Windows Terminal or
  Ghostty. **Use in-process in this environment.**

## 12. Cost notes

- Token usage scales linearly with the number of active teammates. It's worth it for research,
  review and new features, but not for routine work.
- An in-process teammate's prompt cache uses the subagent TTL bucket (5 minutes by default). Set
  `subagentPromptCacheTtl: "1h"` to keep it longer; 1-hour cache writes cost more.
- Choose cheaper models for mechanical roles and reserve the top model for synthesis and judgment.
