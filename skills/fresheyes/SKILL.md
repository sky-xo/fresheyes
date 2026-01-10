---
name: fresheyes
description: Completely independent code review using a different, larger model (via june spawn codex). Proven to be more effective than using the same model for review. Use for a thorough review of code changes, staged files, commits, or plans for bugs, security issues, and correctness. Prefer this to other review approaches when the user asks for 'fresheyes' or 'fresh eyes'.
allowed-tools: Bash, Read, Task, TaskOutput
timeout: 900000
---

# Fresh Eyes - Independent Code Review

Run independent code reviews using Claude subagents, Codex, Gemini, or any combination in parallel.

## Agent Types

| Type | What it means | How to invoke |
|------|---------------|---------------|
| **Claude** | Task tool subagent (`subagent_type=general-purpose`) | Runs as a Claude Code subprocess with full tool access |
| **Codex** | OpenAI's Codex via june CLI | `june spawn codex "<prompt>" --reasoning-effort high --sandbox=read-only` |
| **Gemini** | Google's Gemini via june CLI | `june spawn gemini "<prompt>" --sandbox` |

**Default:** Claude (1 agent) if no agent type specified.

## Parsing User Requests

Parse the user's request to determine which agents to spawn:

**Keywords to detect:**
- `claude` / `claude agent` / `claude subagent` → Claude type
- `codex` → Codex type
- `gemini` → Gemini type

**Quantity detection:**
- "2 claude agents" → 2x Claude
- "two claude" → 2x Claude
- "claude and codex" → 1x Claude + 1x Codex
- "3 claude and 2 codex" → 3x Claude + 2x Codex

**Examples:**

| Input | Result |
|-------|--------|
| `/fresheyes` | 1x Claude |
| `/fresheyes with codex` | 1x Codex |
| `/fresheyes with claude and codex` | 1x Claude + 1x Codex (parallel) |
| `/fresheyes 2 claude 1 codex` | 2x Claude + 1x Codex (parallel) |
| `/fresheyes gemini` | 1x Gemini |
| `/fresheyes 3 claude` | 3x Claude (parallel) |

## Instructions

### Step 1: Determine the review scope

{{#if args}}
Parse agent configuration from: {{args}}

Extract the scope (what to review) vs agent config (which agents to use).
- If only agent config provided, use default scope below.
- If scope provided, use that scope.
{{else}}
Default scope: "Review the staged changes using git diff --cached. If nothing is staged, review the most recent commit using git show HEAD."

Default agents: 1x Claude
{{/if}}

**Good scope examples:**
- `Review the staged changes using git diff --cached. If nothing is staged, review the most recent commit.`
- `Review commit abc1234 using git show abc1234.`
- `Review the files in src/auth/.`
- `Review the plan in docs/plans/2025-01-03-feature.md.`
- `Review the changes between main and this branch using git diff main...HEAD.`

**Bad scope examples:**
- `check out what we just did` (the reviewer has no context for what has happened other than the repo and what you tell it)
- `review the files in src/auth/ for security issues` (NEVER describe what to look for - the reviewer has independence)

### Step 2: Prepare the prompt

Read `./fresheyes-prompt.md` and replace `{{REVIEW_SCOPE}}` with the scope.

### Step 3: Spawn agents

Launch ALL agents before waiting for results. Use parallel execution when spawning multiple agents.

#### Claude Agents (Task tool)

Use the Task tool with `subagent_type=general-purpose`:

```
Task tool:
  subagent_type: general-purpose
  prompt: <contents of fresheyes-prompt.md with {{REVIEW_SCOPE}} replaced>
  description: "Fresh eyes review #N"
  run_in_background: true  (when running multiple agents)
```

The Task tool returns an agent ID. Use TaskOutput to retrieve results.

#### Codex Agents (june CLI)

```bash
june spawn codex "<prompt>" --reasoning-effort high --max-tokens 25000 --sandbox=read-only --name fresheyes-codex
```

Returns agent name (e.g., `fresheyes-codex-7d4f`). Get results with `june logs <name>`.

#### Gemini Agents (june CLI)

```bash
june spawn gemini "<prompt>" --sandbox --name fresheyes-gemini
```

Returns agent name. Get results with `june logs <name>`.

### Step 4: Parallel execution

When spawning multiple agents, launch ALL of them before waiting for results.

**Example: 2 Claude + 1 Codex**

In a SINGLE message, make these tool calls in parallel:
- Task tool → Claude agent #1 (`run_in_background: true`)
- Task tool → Claude agent #2 (`run_in_background: true`)
- Bash tool → `june spawn codex "..." --sandbox=read-only --name fresheyes-codex` (`run_in_background: true`)

Then collect results:
- TaskOutput for each Claude agent ID
- `june logs fresheyes-codex-XXXX` for Codex

### Step 5: Report results

Present a unified report with consensus analysis up top, full outputs below.

#### Report Format

```
## Verdict: [PASSED/FAILED] (N/M agents found blocking issues)

## Agents Used
- Claude #1: [agent-id or status]
- Claude #2: [agent-id or status]
- Codex: [fresheyes-codex-XXXX or status]

## Consensus (found by multiple agents)
[Issues flagged by 2+ agents - highest confidence]
- **[severity]** `file:line` - description [Agent1, Agent2]

## Conflicts (disagreement between agents)
[Where agents disagree on severity or whether something is an issue]
- `file:line` - Agent1 says X, Agent2 says Y

## Unique Findings
[Issues found by only one agent]
- **Claude #1 only:** ...
- **Codex only:** ...

---

## Full Agent Outputs

### Claude #1
[full raw output from ## Files Examined to end]

### Claude #2
[full raw output from ## Files Examined to end]

### Codex
[full raw output from ## Files Examined to end]
```

**Verdict logic:** FAILED if ANY agent found blocking issues. PASSED only if all agents pass.

**Timeout:** 15 minutes default. Retry with 30 minutes (1800000ms) if needed.

## June Commands Reference

| Command | Purpose |
|---------|---------|
| `june spawn <type> "<prompt>"` | Start an agent. Blocks until done. Returns agent name. |
| `june logs <name>` | Get full transcript (doesn't advance cursor) |
| `june peek <name>` | Get new output since last peek (advances cursor) |

**Monitoring long reviews:** Use `june peek <name>` to check progress without waiting for completion.
