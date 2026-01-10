# Fresh Eyes

Independent code review for Claude Code.

## Why?

Using the same model to review its own work has blind spots. Fresh Eyes provides two modes:

- **Quick mode** - Fast self-review with structured checklists (2-5 min)
- **Full mode** - Send code to independent agents (Claude subagents, Codex, Gemini) with no context of your conversation

## Prerequisites

**For full mode with Codex/Gemini:** Install the june CLI for spawning external agents.

## Installation

```
/plugin marketplace add danshapiro/fresheyes
/plugin install fresheyes@danshapiro-fresheyes
```

## Usage

In Claude Code:

### Quick Mode (self-review)
- `/fresheyes quick` - Fast checklist-based self-review

### Full Mode (external agents)
- `/fresheyes` - Review with 1x Claude subagent (default)
- `/fresheyes with codex` - Review with Codex
- `/fresheyes 2 claude 1 codex` - Review with 2x Claude + 1x Codex in parallel
- `/fresheyes Review commit abc1234` - Review a specific commit
- `/fresheyes Review the files in src/auth/` - Review specific files

## Credits

Quick mode checklists adapted from [2389-research/claude-plugins/fresh-eyes-review](https://github.com/2389-research/claude-plugins/tree/main/fresh-eyes-review), originally from Harper Reed's dotfiles.

## License

MIT
