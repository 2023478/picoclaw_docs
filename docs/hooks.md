---
id: hooks
title: Hook System
---

# Hook System

PicoClaw exposes a hook system that lets you observe events, intercept LLM and tool calls, and gate tool execution with approval logic — all without modifying core code.

## Hook Types

| Type | Interface | Stage | Can modify data |
|------|-----------|-------|-----------------|
| Observer | EventObserver | EventBus broadcast | No |
| LLM interceptor | LLMInterceptor | before_llm / after_llm | Yes |
| Tool interceptor | ToolInterceptor | before_tool / after_tool | Yes |
| Tool approver | ToolApprover | approve_tool | No, returns allow/deny |

## Hook Points

- **before_llm** — fired before every LLM request. Interceptors may rewrite the request.
- **after_llm** — fired after the LLM responds. Interceptors may rewrite the response.
- **before_tool** — fired before a tool executes. Interceptors may rewrite arguments.
- **after_tool** — fired after a tool executes. Interceptors may rewrite the result.
- **approve_tool** — fired before a tool executes (after before_tool). Approvers return allow or deny.

## Execution Order

1. **In-process hooks** run first.
2. **Process hooks** run second.
3. Within each group, hooks are sorted by **priority** (lower number runs first).
4. If two hooks share the same priority, **name** (lexicographic) is used as a tiebreaker.

## Timeouts

Global defaults are configured under `hooks.defaults`:

| Field | Description |
|-------|-------------|
| `observer_timeout_ms` | Maximum time an observer callback may take before it is cancelled. |
| `interceptor_timeout_ms` | Maximum time an interceptor may take before it is cancelled. |
| `approval_timeout_ms` | Maximum time an approver may take before the tool call is denied by default. |

## Quick Start

Add the following to your PicoClaw configuration to enable a Python process-hook:

```json
{
  "hooks": {
    "enabled": true,
    "processes": {
      "py_review_gate": {
        "enabled": true,
        "priority": 100,
        "transport": "stdio",
        "command": ["python3", "/tmp/review_gate.py"],
        "observe": ["tool_exec_start", "tool_exec_end", "tool_exec_skipped"],
        "intercept": ["before_tool", "approve_tool"],
        "env": {
          "PICOCLAW_HOOK_LOG_FILE": "/tmp/picoclaw-hook-review-gate.log"
        }
      }
    }
  }
}
```

## Go In-Process Example

Register a hook directly in Go:

```go
package main

import (
    "context"
    "log"

    "github.com/anthropics/picoclaw/hook"
)

type auditHook struct{}

func (h *auditHook) Name() string { return "audit" }

func (h *auditHook) BeforeTool(ctx context.Context, req *hook.ToolRequest) (*hook.ToolRequest, error) {
    log.Printf("tool=%s args=%v", req.Name, req.Args)
    return req, nil // pass through unmodified
}

func init() {
    hook.Register(&auditHook{})
}
```

## Python Process-Hook Example

The following `review_gate.py` implements a process-hook that observes tool events and participates in before_tool interception and approve_tool approval. It only logs and never rewrites or denies.

```python
#!/usr/bin/env python3
"""review_gate.py – PicoClaw process-hook (JSON-RPC over stdio).

Supports:
  hook.hello      – handshake
  hook.event      – observe events (log only)
  hook.before_tool – intercept before tool execution (pass-through)
  hook.approve_tool – approve tool execution (always allow)
"""

import json
import os
import sys

LOG_FILE = os.environ.get("PICOCLAW_HOOK_LOG_FILE", "/tmp/picoclaw-hook-review-gate.log")


def _log(msg: str) -> None:
    with open(LOG_FILE, "a") as f:
        f.write(msg + "\n")


def _respond(id: int | str | None, result: dict) -> None:
    payload = {"jsonrpc": "2.0", "id": id, "result": result}
    line = json.dumps(payload)
    sys.stdout.write(line + "\n")
    sys.stdout.flush()


def handle_hello(id, params):
    _log(f"hello: protocol_version={params.get('protocol_version')}")
    _respond(id, {"name": "py_review_gate", "protocol_version": 1})


def handle_event(id, params):
    _log(f"event: {params.get('type')} — {json.dumps(params.get('data', {}))}")
    _respond(id, {})


def handle_before_tool(id, params):
    tool = params.get("name", "<unknown>")
    _log(f"before_tool: {tool}")
    # Pass through unmodified
    _respond(id, {"args": params.get("args", {})})


def handle_approve_tool(id, params):
    tool = params.get("name", "<unknown>")
    _log(f"approve_tool: {tool} → allow")
    _respond(id, {"allow": True})


DISPATCH = {
    "hook.hello": handle_hello,
    "hook.event": handle_event,
    "hook.before_tool": handle_before_tool,
    "hook.approve_tool": handle_approve_tool,
}


def main() -> None:
    _log("review_gate started")
    for line in sys.stdin:
        line = line.strip()
        if not line:
            continue
        try:
            msg = json.loads(line)
        except json.JSONDecodeError:
            _log(f"bad json: {line}")
            continue

        method = msg.get("method", "")
        handler = DISPATCH.get(method)
        if handler:
            handler(msg.get("id"), msg.get("params", {}))
        else:
            _log(f"unknown method: {method}")
    _log("review_gate exiting")


if __name__ == "__main__":
    main()
```

## Process-Hook Protocol

Process hooks communicate with PicoClaw over **JSON-RPC 2.0** on **stdio** (one JSON object per line).

1. PicoClaw spawns the process and sends `hook.hello` with `{"protocol_version": 1}`.
2. The process must reply with `{"name": "<hook_name>", "protocol_version": 1}`.
3. PicoClaw then sends `hook.event`, `hook.before_tool`, `hook.after_tool`, or `hook.approve_tool` messages as appropriate.
4. The process replies with a JSON-RPC response for each request.

All communication is synchronous from PicoClaw's perspective: it sends a request and waits for exactly one response (subject to the configured timeout).

## Configuration Reference

### Builtin hooks — `hooks.builtins.<name>`

| Field | Type | Description |
|-------|------|-------------|
| `enabled` | bool | Whether this builtin hook is active. |
| `priority` | int | Execution order (lower = earlier). |
| `config` | object | Hook-specific configuration passed to the builtin. |

### Process hooks — `hooks.processes.<name>`

| Field | Type | Description |
|-------|------|-------------|
| `enabled` | bool | Whether this process hook is active. |
| `priority` | int | Execution order (lower = earlier). |
| `transport` | string | Transport protocol. Currently only `"stdio"` is supported. |
| `command` | string[] | Command and arguments to spawn the process. |
| `dir` | string | Working directory for the spawned process. |
| `env` | object | Extra environment variables passed to the process. |
| `observe` | string[] | List of event types this hook wants to observe. |
| `intercept` | string[] | List of hook points this hook wants to intercept. |

## Scope and Limits

The hook system is best suited for:

- **LLM request rewriting** — normalize prompts, inject system context, enforce policies.
- **Tool argument normalization** — sanitize or transform arguments before execution.
- **Pre-execution tool approval** — gate dangerous operations with custom logic.
- **Auditing** — log all LLM and tool activity for compliance or debugging.

Not yet supported:

- External hooks sending messages back into the conversation.
- Suspending execution for human-in-the-loop approval.
- Full message-level interception (only LLM request/response and tool call/result are hookable).

## Troubleshooting

- **Hook not firing** — Verify `enabled: true` and that the event type or hook point is listed in `observe` or `intercept`.
- **Timeout errors** — Increase the relevant timeout in `hooks.defaults`. Check that the process hook flushes stdout after every response.
- **Process hook crashes on startup** — Run the command manually to check for missing dependencies or syntax errors.
- **JSON parse errors** — Ensure exactly one JSON object per line, no extra output on stdout (use stderr or a log file for debug output).
