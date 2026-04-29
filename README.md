# Agent Skills

A collection of reusable skills for agent-based development tools.

## Overview

This repo is a growing set of skills that plug into various agent coding environments. More will be added over time.

## Adding a Skill

1. Pick the skill you want from this repo.
2. Copy it to the appropriate directory for your tool.
3. Reload or restart your agent session.
4. Reference the skill by name in your prompts (e.g., "use the `lint-fix` skill").

## Installation Examples

### OpenCode

1. Copy skill files into `~/.config/opencode/skills/` (global) or `.opencode/skills/` in your project root (project-local).
2. Restart the session.
3. Verify with `/skills`.

### Claude Code

1. Copy the desired skill directory into `~/.claude/skills/` (global) or `.claude/skills/` (project-local).
2. Restart the session.
3. The skill should appear in the available tools list.

## Contributing

Got a skill to share? Open a PR.

## Status

Work in progress — more skills coming.
