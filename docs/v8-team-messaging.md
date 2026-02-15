# v8: Team Messaging

**Core insight: Subagents are dispatched workers. Teammates are colleagues sitting next to you.**

v3 subagents are "divide and conquer": the main agent dispatches a task, the subagent executes, returns a result, and is destroyed.

```sh
v3 Subagent:
  Main Agent -> "explore the codebase" -> Subagent
  Main Agent <- "auth is in src/auth/" <- Subagent
  (subagent destroyed, context gone)
```

For tasks like "develop frontend and backend simultaneously," subagents are not enough: they cannot communicate with each other, cannot share progress, and are destroyed after execution. Teammates solve the **sustained collaboration** problem.

## Subagent vs Teammate

| Feature | Subagent (v3) | Teammate (v8) |
|---------|--------------|---------------|
| Lifecycle | One-shot | Persistent (spawned, works, shuts down) |
| Communication | Return value (one-way) | Message protocol (two-way) |
| Parallelism | Pseudo-parallel (blocks on return) | True parallel (independent threads) |
| Task management | None | Shared Tasks (v6) |
| Use case | One-off tasks | Multi-module long-term collaboration |

## Architecture

```sh
Team Lead (main agent)
  |-- Teammate: frontend   (daemon thread)
  |-- Teammate: backend    (daemon thread)
  +-- Shared:
        |-- .tasks/         <- everyone sees the same board
        +-- .teams/         <- JSONL inbox files per teammate
```

Each Teammate runs as a daemon thread with its own agent loop, its own context window, and runs compression (v5) independently.

## TeammateManager

The `TeammateManager` class handles team lifecycle and messaging. It maintains a registry of teams and their members, protected by a threading lock:

```python
class TeammateManager:
    MESSAGE_TYPES = {
        "message", "broadcast", "shutdown_request",
        "shutdown_response", "plan_approval_response",
    }

    def __init__(self):
        self._teams: dict[str, dict[str, Teammate]] = {}
        self._lock = threading.Lock()
```

Three operations form the team lifecycle:
1. `create_team(name)` -- register a new team, create its directory
2. `spawn_teammate(name, team_name, prompt)` -- start a teammate thread
3. `delete_team(name)` -- send shutdown to all members, remove team

## JSONL Inbox File Format

Each teammate has a dedicated inbox file at `.teams/{team_name}/{name}_inbox.jsonl`. Messages are stored one per line in JSON format:

```json
{"type": "message", "sender": "lead", "content": "Please finish the login page first", "timestamp": 1709234567.89}
{"type": "broadcast", "sender": "backend", "content": "API schema is finalized", "timestamp": 1709234590.12}
```

Reading the inbox consumes all messages and clears the file (read-and-clear pattern). This prevents duplicate processing while keeping the format simple and append-friendly for concurrent writers.

## Message Types

Five message types cover the full range of team communication:

| Type | Scenario | Direction |
|------|----------|-----------|
| `message` | "API docs are at docs/api.md" | Point-to-point |
| `broadcast` | "Database schema changed, everyone take note" | One-to-many |
| `shutdown_request` | "Project done, please wrap up" | Lead -> Teammate |
| `shutdown_response` | "I have wrapped up" | Teammate -> Lead |
| `plan_approval_response` | "Your refactoring plan is approved" | Lead -> Teammate |

## Teammate Data Model

Each teammate is represented by a `Teammate` dataclass:

```python
@dataclass
class Teammate:
    name: str
    team_name: str
    agent_id: str = ""       # Format: "name@team_name"
    status: str = "active"   # active | shutdown
    thread: threading.Thread
    inbox_path: Path         # .teams/{team_name}/{name}_inbox.jsonl
    color: str = ""          # ANSI color for terminal output
```

The `agent_id` field uses the format `name@team_name` (e.g. `backend@rest-to-graphql`) for identification in logs and messages.

## Teammate Colors

Each teammate is assigned a distinct ANSI color for terminal output, cycling through cyan, yellow, magenta, green, and blue. This makes it easy to distinguish which teammate produced which output in parallel execution.

```python
TEAMMATE_COLORS = [
    "\033[36m",   # cyan
    "\033[33m",   # yellow
    "\033[35m",   # magenta
    "\033[32m",   # green
    "\033[34m",   # blue
]
```

## Team Directory Structure

When a team is created, a directory is set up under `.teams/`:

```
.teams/
  rest-to-graphql/
    config.json             <- team metadata and member list
    frontend_inbox.jsonl    <- frontend teammate's inbox
    frontend_inbox.lock     <- lock file for atomic writes
    backend_inbox.jsonl     <- backend teammate's inbox
    backend_inbox.lock
```

The `config.json` file tracks team metadata and current membership:

```json
{
  "name": "rest-to-graphql",
  "created_at": 1709234500.0,
  "members": [
    {"name": "frontend", "agent_id": "frontend@rest-to-graphql", "status": "active"},
    {"name": "backend", "agent_id": "backend@rest-to-graphql", "status": "active"}
  ]
}
```

## Inbox Lock Files

Concurrent writes to inbox files are protected by lock files. The `_write_to_inbox` method uses `os.O_CREAT | os.O_EXCL` for atomic lock acquisition:

```python
def _write_to_inbox(inbox_path, message):
    lock_path = inbox_path.with_suffix(".lock")
    for _ in range(50):   # retry up to 50 times
        try:
            fd = os.open(str(lock_path), os.O_CREAT | os.O_EXCL | os.O_WRONLY)
            os.close(fd)
            break
        except FileExistsError:
            time.sleep(0.05)
    try:
        with open(inbox_path, "a") as f:
            f.write(json.dumps(message) + "\n")
    finally:
        lock_path.unlink(missing_ok=True)
```

This prevents message corruption when multiple threads (e.g. the team lead and another teammate) write to the same inbox simultaneously.

## TEAMMATE_TOOLS vs ALL_TOOLS

Teammates and the Team Lead receive different toolsets:

| Tool | Team Lead | Teammate |
|------|-----------|----------|
| bash, read_file, write_file, edit_file | Yes | Yes |
| TaskCreate, TaskGet, TaskUpdate, TaskList | Yes | Yes |
| SendMessage | Yes | Yes |
| Task (spawn subagents/teammates) | Yes | No |
| Skill | Yes | No |
| TaskOutput, TaskStop | Yes | No |
| TeamCreate, TeamDelete | Yes | No |

Teammates get `BASE_TOOLS + task CRUD (including TaskGet) + SendMessage` -- enough to do work, read task details, update the shared board, and communicate with peers, but not enough to spawn other agents or manage the team itself. This enforces the Team Lead as the orchestrator.

## Three Core Tools

```python
# TeamCreate: create a team
TeamCreate(name="my-project")

# SendMessage: send a message to a teammate
SendMessage(recipient="frontend", content="Please finish the login page first")

# TeamDelete: disband the team
TeamDelete(name="my-project")
```

Teammates are spawned via the Task tool with a `team_name` parameter:

```python
Task(prompt="Handle frontend development", team_name="my-project", name="frontend")
# -> spawns a persistent Teammate, not a one-shot subagent
```

The same Task tool now has three modes:
1. No extra params -- synchronous subagent (v3)
2. `run_in_background=True` -- background subagent (v7)
3. `team_name + name` -- persistent teammate (v8)

## Teammate Work Loop (Simplified)

In v8, the teammate loop is straightforward: receive a prompt, work until done, then shut down. There is no idle cycle or auto-claiming -- those are introduced in v9.

```python
def _teammate_loop(self, teammate, initial_prompt):
    sub_system = f"You are teammate '{teammate.name}' in team '{teammate.team_name}'..."
    sub_messages = [{"role": "user", "content": initial_prompt}]

    while teammate.status != "shutdown":
        teammate.status = "active"

        # Compression before each API call
        sub_messages = CTX.microcompact(sub_messages)
        if CTX.should_compact(sub_messages):
            sub_messages = CTX.auto_compact(sub_messages)

        response = client.messages.create(
            model=MODEL, system=sub_system,
            messages=sub_messages, tools=TEAMMATE_TOOLS, max_tokens=8000,
        )

        if response.stop_reason == "tool_use":
            # Execute tools and continue
            results = [execute(tc) for tc in tool_calls]
            sub_messages.append({"role": "assistant", "content": response.content})
            sub_messages.append({"role": "user", "content": results})
            continue

        # No more tool calls -- check inbox for new instructions
        new_messages = self.check_inbox(teammate.name, teammate.team_name)
        if new_messages:
            if any(m.get("type") == "shutdown_request" for m in new_messages):
                return  # Exit
            sub_messages.append({"role": "user", "content": format(new_messages)})
            continue

        # Nothing left to do
        return
```

## How Broadcast Works

Broadcasting is not a separate method. It uses the same `send_message()` function with `msg_type="broadcast"`:

```python
SendMessage(recipient="anyone", content="Schema is finalized", type="broadcast", team_name="my-project")
```

Internally, the manager iterates through all teammates in the team (excluding the sender) and appends the message to each teammate's JSONL inbox file. The `recipient` field is ignored for broadcasts -- the message goes to everyone.

## Shutdown Protocol

The shutdown sequence is a request-response protocol:

1. **Team Lead** sends `shutdown_request` via `SendMessage(type="shutdown_request")`
2. The message is written to the teammate's JSONL inbox
3. The teammate reads the `shutdown_request` on its next inbox check
4. The teammate sets `status = "shutdown"` and exits its loop
5. Since the thread is a daemon thread, no join is needed

`TeamDelete` sends `shutdown_request` to all teammates in the team simultaneously, then removes the team from the registry.

## Comparison

| Aspect | v3 (Subagent) | v8 (Teammate) |
|--------|--------------|---------------|
| Model | One-shot function call | Persistent worker thread |
| Communication | Return value | Message protocol (5 types) |
| State | Stateless | Stateful (active/shutdown) |
| Task management | None | Shared Tasks |
| Parallelism | Pseudo-parallel | True parallel |

## The Deeper Insight

> **From command to collaboration.**

v3 subagents follow a command pattern: the main agent gives orders, subagents obey. v8 Teammates follow a collaboration pattern: the Team Lead sets direction, Teammates work on shared tasks, communicate with each other through inboxes.

```sh
Subagent  -> do one thing, report back            (intern)
Teammate  -> work on assigned task, communicate   (colleague)
Team Lead -> create team, assign work, coordinate (manager)
```

The next step is making teammates autonomous: instead of waiting for explicit instructions, they find their own work. That is v9.

---

**One agent has limits. A team of agents has reach.**

[← v7](./v7-background-tasks.md) | [Back to README](../README.md) | [v9 →](./v9-autonomous-teams.md)
