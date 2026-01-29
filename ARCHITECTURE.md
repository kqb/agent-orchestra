# Agent Orchestra — Architecture

## Overview

Agent Orchestra replaces ad-hoc multi-agent orchestration (tmux scraping, unstructured callbacks, polling) with structured protocols, event-driven communication, and task dependency management.

Designed for Clawdbot (Zoid) orchestrating Claude Code agents, but protocol-agnostic — any agent runtime that can read/write files and execute commands can participate.

---

## Current State (What We're Replacing)

```
Orchestrator (Zoid)                    Agent (Claude Code)
      │                                      │
      │── exec pty:true "claude ..."  ──────▶│
      │                                      │
      │   (no communication channel)         │── working...
      │                                      │
      │── tmux capture-pane ──────────▶ screen scrape
      │   (poll every 30s, parse ANSI)       │
      │                                      │
      │◀── gateway wake "Done: text" ────────│
      │   (unstructured, one-shot)           │
      │                                      │
      │── tmux send-keys "1" Enter ─────────▶│ (manual approval)
```

**Problems:**
- Communication is unidirectional (orchestrator → agent via tmux keys)
- Callbacks are unstructured free text
- Monitoring requires polling + ANSI parsing
- No error recovery protocol
- No way for agents to ask questions
- Permission approval is regex-on-screen-output

---

## Target Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Orchestrator (Zoid)                    │
│                                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────────┐   │
│  │ Task DAG │  │ State    │  │ Event Processor      │   │
│  │ Executor │  │ Machine  │  │ (FSWatch → actions)  │   │
│  └────┬─────┘  └────┬─────┘  └──────────┬───────────┘   │
│       │              │                   │               │
└───────┼──────────────┼───────────────────┼───────────────┘
        │              │                   │
        ▼              ▼                   ▼
┌─────────────────────────────────────────────────────────┐
│                     Agent Bus                            │
│                                                          │
│  ~/clawd/agent-bus/                                      │
│  ├── tasks/           # Task manifests (orchestrator →)  │
│  ├── events/          # Event stream (agents →)          │
│  ├── agents/          # Agent state + inbox              │
│  │   ├── {id}/                                           │
│  │   │   ├── state.json    # Lifecycle state             │
│  │   │   ├── inbox.jsonl   # Messages to agent           │
│  │   │   └── outbox.jsonl  # Messages from agent         │
│  └── decisions/       # Shared context across agents     │
│                                                          │
└───────┬──────────────┬───────────────────┬───────────────┘
        │              │                   │
        ▼              ▼                   ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│   Agent A    │ │   Agent B    │ │   Agent C    │
│ (Claude Code)│ │ (Claude Code)│ │ (Claude Code)│
│              │ │              │ │              │
│ Reads:       │ │ Reads:       │ │ Reads:       │
│  task.json   │ │  task.json   │ │  task.json   │
│  inbox.jsonl │ │  inbox.jsonl │ │  inbox.jsonl │
│              │ │              │ │              │
│ Writes:      │ │ Writes:      │ │ Writes:      │
│  state.json  │ │  state.json  │ │  state.json  │
│  outbox.jsonl│ │  outbox.jsonl│ │  outbox.jsonl│
│  events/     │ │  events/     │ │  events/     │
└──────────────┘ └──────────────┘ └──────────────┘
```

---

## Protocols

### 1. Task Manifest Protocol

The orchestrator creates a task manifest before spawning an agent. The agent reads it on startup.

**Location:** `~/clawd/agent-bus/tasks/{task_id}.json`

```json
{
  "schema": "agent-orchestra/task/v1",
  "task_id": "localllm-hub-build",
  "created_at": "2026-01-29T00:00:00Z",
  "timeout_minutes": 30,
  
  "objective": "Build localllm-hub packages per ARCHITECTURE.md",
  "acceptance_criteria": [
    "All 5 packages load without errors",
    "CLI --help shows all commands",
    "npm install succeeds with 0 vulnerabilities"
  ],
  
  "context": {
    "primary_files": ["ARCHITECTURE.md", "README.md"],
    "reference_files": [
      "~/Projects/emailctl/lib/classifier.js",
      "~/clawd/scripts/semantic-search.js"
    ]
  },
  
  "agent": {
    "model": "claude-sonnet-4-5",
    "workdir": "~/Projects/localllm-hub",
    "permissions": "skip",
    "flags": ["--dangerously-skip-permissions"]
  },
  
  "communication": {
    "bus_dir": "~/clawd/agent-bus",
    "agent_id": "localllm-hub-build",
    "callback": "clawdbot gateway wake",
    "result_file": "~/clawd/agent-bus/agents/localllm-hub-build/result.json"
  },
  
  "dependencies": [],
  "priority": "normal"
}
```

### 2. Agent State Protocol

Each agent maintains its lifecycle state. Updated by the agent wrapper (not by Claude Code directly).

**Location:** `~/clawd/agent-bus/agents/{agent_id}/state.json`

```json
{
  "schema": "agent-orchestra/state/v1",
  "agent_id": "localllm-hub-build",
  "task_id": "localllm-hub-build",
  "pid": 12345,
  "tmux_session": "localllm-hub-build",
  
  "lifecycle": "working",
  "lifecycle_history": [
    { "state": "spawned", "at": "2026-01-29T00:00:00Z" },
    { "state": "initializing", "at": "2026-01-29T00:00:05Z" },
    { "state": "working", "at": "2026-01-29T00:00:12Z" }
  ],
  
  "progress": {
    "phase": "writing packages",
    "files_created": 15,
    "files_total_estimate": 25,
    "percent": 60
  },
  
  "health": {
    "last_activity_at": "2026-01-29T00:05:30Z",
    "idle_seconds": 0,
    "errors": 0,
    "approvals_pending": 0
  },
  
  "updated_at": "2026-01-29T00:05:30Z"
}
```

**Lifecycle states:**

```
SPAWNED ──▶ INITIALIZING ──▶ WORKING ──▶ COMPLETING ──▶ DONE
                │                │              │
                │                ▼              │
                │           BLOCKED ────────────┤
                │           (needs approval     │
                │            or input)           │
                │                │              │
                ▼                ▼              ▼
              ERROR          TIMEOUT        FAILED
```

| State | Meaning | Orchestrator Action |
|-------|---------|---------------------|
| `spawned` | Process created, not yet running | Wait |
| `initializing` | Claude Code loading, reading context | Wait |
| `working` | Actively processing task | Monitor health |
| `blocked` | Waiting for approval or input | Approve or answer |
| `completing` | Running final validation | Wait |
| `done` | Task complete, result ready | Read result, cleanup |
| `error` | Unrecoverable error | Log, notify, maybe retry |
| `timeout` | Exceeded timeout_minutes | Kill, notify |
| `failed` | Completed but acceptance criteria not met | Review, maybe retry |

### 3. Event Protocol

Agents and orchestrator communicate via an append-only event stream.

**Location:** `~/clawd/agent-bus/events/{timestamp}-{type}.json`

**Event types:**

```json
// Agent spawned
{
  "schema": "agent-orchestra/event/v1",
  "type": "agent.spawned",
  "agent_id": "localllm-hub-build",
  "task_id": "localllm-hub-build",
  "timestamp": "2026-01-29T00:00:00Z",
  "data": {
    "model": "claude-sonnet-4-5",
    "workdir": "~/Projects/localllm-hub"
  }
}

// Progress update
{
  "type": "agent.progress",
  "agent_id": "localllm-hub-build",
  "timestamp": "2026-01-29T00:03:00Z",
  "data": {
    "phase": "writing packages",
    "detail": "Created embeddings, classifier, working on triage",
    "files_changed": 15
  }
}

// Agent needs input
{
  "type": "agent.needs_input",
  "agent_id": "localllm-hub-build",
  "timestamp": "2026-01-29T00:04:00Z",
  "data": {
    "question": "Should classifier use from_email or from for the sender field?",
    "options": ["from_email (backward compat)", "from (cleaner API)"],
    "blocking": true,
    "timeout_minutes": 10
  }
}

// Orchestrator answers
{
  "type": "orchestrator.answer",
  "agent_id": "localllm-hub-build",
  "timestamp": "2026-01-29T00:04:30Z",
  "data": {
    "answer": "Use from — cleaner API, we'll handle migration",
    "in_response_to": "agent.needs_input@2026-01-29T00:04:00Z"
  }
}

// Agent needs approval
{
  "type": "agent.needs_approval",
  "agent_id": "localllm-hub-build",
  "timestamp": "2026-01-29T00:05:00Z",
  "data": {
    "action": "bash",
    "command": "npm install",
    "risk": "low"
  }
}

// Agent completed
{
  "type": "agent.completed",
  "agent_id": "localllm-hub-build",
  "timestamp": "2026-01-29T00:07:00Z",
  "data": {
    "status": "success",
    "summary": "Built 5 packages, 25 files, 1810 lines",
    "result_file": "~/clawd/agent-bus/agents/localllm-hub-build/result.json",
    "acceptance_criteria_met": true
  }
}

// Agent errored
{
  "type": "agent.error",
  "agent_id": "localllm-hub-build",
  "timestamp": "2026-01-29T00:06:00Z",
  "data": {
    "error": "Ollama connection timeout",
    "recoverable": true,
    "suggestion": "Restart Ollama and retry"
  }
}
```

### 4. Result Protocol

When an agent completes, it writes a structured result.

**Location:** `~/clawd/agent-bus/agents/{agent_id}/result.json`

```json
{
  "schema": "agent-orchestra/result/v1",
  "agent_id": "localllm-hub-build",
  "task_id": "localllm-hub-build",
  "completed_at": "2026-01-29T00:07:00Z",
  "duration_seconds": 420,
  
  "status": "success",
  "summary": "Built complete localllm-hub with 5 packages",
  
  "deliverables": {
    "files_created": 25,
    "files_modified": 0,
    "lines_added": 1810,
    "commits": ["11950ff feat: build complete localllm-hub unified packages"]
  },
  
  "acceptance_criteria": {
    "All 5 packages load without errors": true,
    "CLI --help shows all commands": true,
    "npm install succeeds with 0 vulnerabilities": true
  },
  
  "tests": {
    "passed": 6,
    "failed": 0,
    "skipped": 0,
    "details": [
      { "name": "transcribe --help", "status": "pass" },
      { "name": "classify bills", "status": "pass" },
      { "name": "embed hello world", "status": "pass", "note": "1024 dims" },
      { "name": "compare cat dog", "status": "pass", "note": "0.6913" },
      { "name": "triage urgent", "status": "pass", "note": "urgency 4" },
      { "name": "search reindex", "status": "pass", "note": "390 chunks" }
    ]
  },
  
  "issues": [
    {
      "severity": "medium",
      "description": "Ollama client timeout on first model load",
      "resolution": "Added 120s fetch timeout in shared/ollama.js"
    },
    {
      "severity": "low",
      "description": "localhost resolves to IPv6 on macOS",
      "resolution": "Changed to 127.0.0.1 in ollama.js"
    }
  ],
  
  "questions_for_orchestrator": [
    "Classifier uses 'from' instead of 'from_email' — OK for backward compat?"
  ]
}
```

### 5. Messaging Protocol (Bidirectional)

For ongoing communication during task execution.

**Agent inbox:** `~/clawd/agent-bus/agents/{agent_id}/inbox.jsonl`
**Agent outbox:** `~/clawd/agent-bus/agents/{agent_id}/outbox.jsonl`

```jsonl
{"ts":"2026-01-29T00:04:00Z","from":"agent","type":"question","text":"Use from_email or from?","blocking":true}
{"ts":"2026-01-29T00:04:30Z","from":"orchestrator","type":"answer","text":"Use from"}
{"ts":"2026-01-29T00:05:00Z","from":"agent","type":"info","text":"Starting npm install"}
{"ts":"2026-01-29T00:05:30Z","from":"agent","type":"warning","text":"Ollama timeout, retrying with longer timeout"}
{"ts":"2026-01-29T00:06:00Z","from":"orchestrator","type":"directive","text":"Skip Ollama tests if it keeps timing out, we'll test manually"}
```

**Message types:**
| Type | Direction | Blocking | Purpose |
|------|-----------|----------|---------|
| `question` | agent → orchestrator | yes/no | Agent needs decision |
| `answer` | orchestrator → agent | — | Response to question |
| `info` | agent → orchestrator | no | Progress update |
| `warning` | agent → orchestrator | no | Non-blocking issue |
| `error` | agent → orchestrator | yes | Blocking error |
| `directive` | orchestrator → agent | — | Change instructions |
| `cancel` | orchestrator → agent | — | Abort task |

---

## Components

### Agent Bus (`agent-bus/`)

File-based pub/sub system. No external dependencies — just filesystem + FSWatch.

```
~/clawd/agent-bus/
├── tasks/                    # Task manifests
│   └── {task_id}.json
├── events/                   # Append-only event stream
│   └── {timestamp}-{type}.json
├── agents/                   # Per-agent state + messaging
│   └── {agent_id}/
│       ├── state.json        # Lifecycle state
│       ├── result.json       # Final result (when done)
│       ├── inbox.jsonl       # Messages TO agent
│       └── outbox.jsonl      # Messages FROM agent
├── decisions/                # Shared context across agents
│   └── {project}/
│       ├── decisions.md      # Architecture decisions made
│       ├── blockers.md       # Known issues
│       └── progress.md       # What's done, what's left
└── config.json               # Bus configuration
```

**Why file-based:**
- No server to run/crash
- Works with any agent runtime (Claude Code, Codex, custom)
- Easy to inspect/debug (just `cat` the files)
- FSWatch is built into Node.js and macOS
- Survives process restarts
- Git-trackable if desired

### Event Watcher (`watcher.js`)

Node.js FSWatch process that:
1. Watches `events/` and `agents/*/outbox.jsonl` for new files/appends
2. Parses events
3. Routes to orchestrator via `clawdbot gateway wake`

```javascript
// Pseudocode
fs.watch(EVENTS_DIR, (event, filename) => {
  const eventData = JSON.parse(fs.readFileSync(path));
  
  switch (eventData.type) {
    case 'agent.needs_input':
      gatewayWake(`Agent ${eventData.agent_id} asks: ${eventData.data.question}`);
      break;
    case 'agent.completed':
      gatewayWake(`Agent ${eventData.agent_id} done: ${eventData.data.summary}`);
      break;
    case 'agent.error':
      gatewayWake(`Agent ${eventData.agent_id} error: ${eventData.data.error}`);
      break;
  }
});
```

### Task DAG Executor (`executor.js`)

Manages task dependencies and parallel execution.

```json
{
  "project": "localllm-hub-v2",
  "tasks": [
    { "id": "shared", "deps": [], "model": "sonnet" },
    { "id": "embeddings", "deps": ["shared"], "model": "sonnet" },
    { "id": "classifier", "deps": ["shared"], "model": "sonnet" },
    { "id": "search", "deps": ["shared", "embeddings"], "model": "sonnet" },
    { "id": "triage", "deps": ["shared"], "model": "sonnet" },
    { "id": "integration-test", "deps": ["embeddings", "classifier", "search", "triage"], "model": "opus" }
  ]
}
```

**Execution:**
```
Time 0:  [shared] ─────────────────────────▶ done
Time 1:  [embeddings] ───▶ done  [classifier] ───▶ done  [triage] ───▶ done
Time 2:  [search] ──────────────▶ done (waited for embeddings)
Time 3:  [integration-test] ────────────────────────────────────▶ done
```

Parallelizes independent tasks. Waits for dependencies. Handles failures (retry, skip, abort).

### Agent Wrapper (`wrapper.sh`)

Replaces raw `claude` invocation. Manages state updates and bus communication.

```bash
#!/bin/bash
# wrapper.sh — wraps Claude Code with protocol support

AGENT_ID="$1"
TASK_FILE="$2"
BUS_DIR="${AGENT_BUS:-$HOME/clawd/agent-bus}"
STATE_FILE="$BUS_DIR/agents/$AGENT_ID/state.json"

# Initialize state
mkdir -p "$BUS_DIR/agents/$AGENT_ID"
write_state "spawned"

# Read task manifest
TASK=$(cat "$TASK_FILE")
WORKDIR=$(echo "$TASK" | jq -r '.agent.workdir')
MODEL=$(echo "$TASK" | jq -r '.agent.model')
OBJECTIVE=$(echo "$TASK" | jq -r '.objective')

# Update state
write_state "initializing"

# Build prompt with protocol instructions
PROMPT="$OBJECTIVE

## Communication Protocol
- Write progress to: $BUS_DIR/agents/$AGENT_ID/outbox.jsonl
- Check for messages in: $BUS_DIR/agents/$AGENT_ID/inbox.jsonl  
- Write final result to: $BUS_DIR/agents/$AGENT_ID/result.json
- If you need input, write to outbox with type 'question' and wait for inbox response

## Acceptance Criteria
$(echo "$TASK" | jq -r '.acceptance_criteria[]' | sed 's/^/- /')

When finished, write result.json and run: clawdbot gateway wake --text 'AGENT_DONE:$AGENT_ID' --mode now"

# Spawn Claude Code
write_state "working"
tmux new-session -d -s "$AGENT_ID" -c "$WORKDIR"
tmux send-keys -t "$AGENT_ID" "claude --model $MODEL --dangerously-skip-permissions '$PROMPT'" Enter

# Monitor health (background)
while tmux has-session -t "$AGENT_ID" 2>/dev/null; do
  update_health
  check_inbox
  sleep 10
done

# Finalize
if [ -f "$BUS_DIR/agents/$AGENT_ID/result.json" ]; then
  write_state "done"
else
  write_state "error"
fi
```

### Orchestrator CLI (`cli.js`)

```bash
# Spawn single agent
node cli.js spawn --task tasks/build-api.json

# Spawn project (DAG)
node cli.js project --dag project.json

# Check status of all agents
node cli.js status
# Output:
#   localllm-build    WORKING   5m elapsed   60% progress
#   auth-refactor     BLOCKED   needs input  "Which OAuth provider?"
#   test-suite        QUEUED    waiting on: localllm-build

# View event stream
node cli.js events --follow
# Output:
#   00:00:05  agent.spawned       localllm-build
#   00:03:00  agent.progress      localllm-build  "writing packages"
#   00:05:00  agent.needs_input   localllm-build  "from_email or from?"

# Send message to agent
node cli.js send localllm-build "Use from — cleaner API"

# Kill agent
node cli.js kill localllm-build

# View result
node cli.js result localllm-build
```

---

## Agent Specialization

Define agent roles for complex projects:

```json
{
  "roles": {
    "architect": {
      "model": "claude-opus-4-5",
      "strengths": ["design", "planning", "review", "complex debugging"],
      "prompt_prefix": "You are the architect. Design and review, don't implement.",
      "max_concurrent": 1
    },
    "builder": {
      "model": "claude-sonnet-4-5",
      "strengths": ["implementation", "fast coding", "file creation"],
      "prompt_prefix": "You are a builder. Implement exactly as specified.",
      "max_concurrent": 5
    },
    "tester": {
      "model": "claude-sonnet-4-5",
      "strengths": ["validation", "edge cases", "integration testing"],
      "prompt_prefix": "You are a tester. Verify thoroughly, find bugs.",
      "max_concurrent": 2
    },
    "reviewer": {
      "model": "claude-opus-4-5",
      "strengths": ["code review", "quality gates", "security audit"],
      "prompt_prefix": "You are a reviewer. Be critical. Find issues.",
      "max_concurrent": 1
    }
  }
}
```

**Workflow:**
```
Architect ──plan──▶ Builder A ──code──▶ Tester ──verify──▶ Reviewer ──approve──▶ Merge
                    Builder B ──code──┘
                    Builder C ──code──┘
```

---

## Directory Structure

```
agent-orchestra/
├── cli.js                    # CLI entry point
├── package.json
│
├── lib/
│   ├── bus.js                # Agent bus (read/write events, state, messages)
│   ├── executor.js           # Task DAG executor
│   ├── watcher.js            # FSWatch event processor
│   ├── wrapper.js            # Agent wrapper (spawn + monitor)
│   ├── state-machine.js      # Agent lifecycle state machine
│   └── protocols/
│       ├── task.schema.json    # Task manifest JSON schema
│       ├── state.schema.json   # Agent state JSON schema
│       ├── event.schema.json   # Event JSON schema
│       ├── result.schema.json  # Result JSON schema
│       └── message.schema.json # Message JSON schema
│
├── templates/
│   ├── task.json             # Task manifest template
│   ├── project.json          # DAG project template
│   └── roles.json            # Agent role definitions
│
├── scripts/
│   ├── wrapper.sh            # Shell wrapper for Claude Code
│   └── approver.js           # Reliable permission approver
│
├── test/
│   ├── bus.test.js           # Bus read/write tests
│   ├── executor.test.js      # DAG execution tests
│   └── state-machine.test.js # Lifecycle tests
│
├── ARCHITECTURE.md           # This file
├── README.md
└── LICENSE
```

---

## Implementation Phases

### Phase 1: Foundation (Week 1)

Build the core bus and protocols.

| Component | Description | Priority |
|-----------|-------------|----------|
| `protocols/*.schema.json` | JSON schemas for all protocols | P0 |
| `lib/bus.js` | Read/write events, state, messages | P0 |
| `lib/state-machine.js` | Agent lifecycle transitions | P0 |
| `scripts/wrapper.sh` | Agent wrapper with protocol support | P0 |
| `cli.js spawn` | Spawn agent with task manifest | P0 |
| `cli.js status` | View agent states | P0 |

**Validation:** Spawn one agent with structured callback, verify state transitions and result.json.

### Phase 2: Communication (Week 2)

Add bidirectional messaging and event watching.

| Component | Description | Priority |
|-----------|-------------|----------|
| `lib/watcher.js` | FSWatch → gateway wake notifications | P0 |
| `cli.js send` | Send message to agent | P0 |
| `cli.js events` | View event stream | P1 |
| `scripts/approver.js` | Reliable permission handling | P1 |
| Inbox polling in wrapper | Agent checks for messages | P1 |

**Validation:** Agent asks question, orchestrator answers, agent continues. No tmux scraping needed.

### Phase 3: Orchestration (Week 3)

Add DAG execution and parallel agents.

| Component | Description | Priority |
|-----------|-------------|----------|
| `lib/executor.js` | Task DAG parser and executor | P0 |
| `cli.js project` | Execute DAG project | P0 |
| Parallel spawning | Git worktrees + concurrent agents | P1 |
| Retry logic | Failed task retry with backoff | P1 |
| `cli.js kill` | Clean shutdown with state update | P1 |

**Validation:** Execute 3-task DAG with dependency. Tasks 1+2 parallel, task 3 waits. All complete with structured results.

### Phase 4: Specialization (Week 4+)

Add agent roles and workflow templates.

| Component | Description | Priority |
|-----------|-------------|----------|
| `templates/roles.json` | Agent role definitions | P1 |
| Role-based prompting | Inject role context into agent prompts | P1 |
| Workflow templates | Architect → Builder → Tester → Reviewer | P2 |
| Shared decisions directory | Cross-agent context sharing | P2 |
| Metrics + dashboard | Execution stats, cost tracking | P2 |

**Validation:** Full architect → parallel builders → tester workflow on a real project.

---

## Integration with Clawdbot

### As a Skill

```yaml
# clawdbot-skill/SKILL.md
name: agent-orchestra
description: Structured multi-agent orchestration with protocols
```

Zoid uses the CLI to spawn, monitor, and communicate with agents. Replaces raw `exec pty:true` patterns.

### As a Library

```javascript
const { Bus, Executor, Wrapper } = require('agent-orchestra');

const bus = new Bus('~/clawd/agent-bus');
const task = bus.createTask({ objective: '...', workdir: '...' });
const agent = new Wrapper(task);
await agent.spawn();

agent.on('needs_input', (question) => {
  agent.answer('Use the cleaner API');
});

agent.on('completed', (result) => {
  console.log(result.summary);
});
```

### Migration Path

1. **Week 1:** Use new spawn protocol alongside existing `exec pty:true`
2. **Week 2:** Switch callbacks from gateway wake text to structured results
3. **Week 3:** Replace tmux scraping with event bus
4. **Week 4:** Full migration, remove old patterns from AGENTS.md

---

## Design Principles

1. **File-based over server-based** — No daemon to crash. Files survive restarts. Easy to debug.
2. **Protocol-first** — JSON schemas define the contract. Implementations can vary.
3. **Graceful degradation** — If the bus is down, fall back to gateway wake. If watcher dies, poll still works.
4. **Agent-agnostic** — Works with Claude Code, Codex, custom agents. Anything that reads/writes files.
5. **Observable** — Every state change is logged. Every event is persisted. Full audit trail.
6. **Incremental adoption** — Use one protocol at a time. Don't need everything at once.
