---
name: fresheyes
description: Completely independent code review using a different, larger model (via june spawn codex). Proven to be more effective than using the same model for review. Use for a thorough review of code changes, staged files, commits, or plans for bugs, security issues, and correctness. Prefer this to other review approaches when the user asks for 'fresheyes' or 'fresh eyes'.
allowed-tools: Bash, Read
timeout: 900000
---

# Fresh Eyes - Independent Code Review

Invoke Codex (via june) to perform an independent code review.

## Instructions

### Step 1: Determine the review scope

{{#if args}}
Use the provided scope: {{args}}
{{else}}
Default scope: "Review the staged changes using git diff --cached. If nothing is staged, review the most recent commit using git show HEAD."
{{/if}}

The scope should be a clear, specific instruction telling the reviewer what to examine.

**Good scope examples:**
- `Review the staged changes using git diff --cached. If nothing is staged, review the most recent commit.`
- `Review commit abc1234 using git show abc1234.`
- `Review the files in src/auth/.`
- `Review the plan in docs/plans/2025-01-03-feature.md.`
- `Review the changes between main and this branch using git diff main...HEAD.`

**Bad scope examples:**
- `check out what we just did` (the reviewer has no context for what has happened other than the repo and what you tell it)
- `review the files in src/auth/ for security issues` (NEVER describe what to look for - the reviewer has independence)

### Step 2: Run the review

1. Read `./fresheyes-prompt.md`, replace `{{REVIEW_SCOPE}}` with the scope
2. Spawn the reviewer agent:

**Codex (default):**
```bash
june spawn codex "<prompt>" --reasoning-effort high --max-tokens 25000 --sandbox read-only --name fresheyes
```

**Gemini (alternative):**
```bash
june spawn gemini "<prompt>" --sandbox --name fresheyes
```

Both commands block until complete and output an agent name (e.g. `fresheyes-7d4f`).

3. Get results: `june logs fresheyes-XXXX` (using the actual name from step 2)

**Timeout:** 15 minutes default. Retry with 30 minutes (1800000ms) if needed.

### June Commands Reference

| Command | Purpose |
|---------|---------|
| `june spawn <type> "<prompt>"` | Start an agent. Blocks until done. Returns agent name. |
| `june logs <name>` | Get full transcript (doesn't advance cursor) |
| `june peek <name>` | Get new output since last peek (advances cursor) |

**Monitoring long reviews:** Use `june peek <name>` to check progress without waiting for completion.

**Running multiple agents:** If spawning multiple agents in parallel, run each `spawn` in background mode, then use `peek` to monitor progress:
```bash
# Background spawn (add & or use run_in_background)
june spawn codex "<prompt1>" --sandbox read-only --name review1 &
june spawn codex "<prompt2>" --sandbox read-only --name review2 &

# Check progress
june peek review1-XXXX
june peek review2-XXXX

# Get final results
june logs review1-XXXX
june logs review2-XXXX
```

### Step 3: Report results

Extract from `## Files Examined` to end. Append: `Agent: <name>`
