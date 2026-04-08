# Agent Orchestra

Multi-agent orchestration framework for Clawdbot.
> **Lineage:** Built for Clawdbot (codename: Zoid), a personal AI agent system. [OpenClaw](https://github.com/kqb/openclaw) is its public successor. Path references like `~/clawd/` reflect that system layout.

Structured communication protocols, event-driven coordination, and task dependency management for AI coding agents.

## Problem

Current multi-agent orchestration relies on:
- Screen-scraping tmux output (fragile, ANSI parsing)
- Unstructured `gateway wake` callbacks (free text, no schema)
- Poll-based monitoring (expensive token burn)
- No bidirectional communication (agents can't ask questions)
- Manual permission approval (brittle auto-approver scripts)

## Solution

Structured protocols for agent lifecycle, communication, and coordination.

## Components

- **agent-bus/** — File-based event bus with FSWatch notifications
- **protocols/** — JSON schemas for tasks, callbacks, messages
- **orchestrator/** — Task DAG execution, parallel spawning, state management
- **approver/** — Reliable permission handling for Claude Code

## Quick Start

```bash
npm install
# Spawn an agent with structured protocol
node cli.js spawn --task task.json --workdir ~/Projects/myapp
# Monitor all agents
node cli.js status
# View event stream
node cli.js events --follow
```

## Status

🚧 In development — see ARCHITECTURE.md for full design.

## License

MIT
