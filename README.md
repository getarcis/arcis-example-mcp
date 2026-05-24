[![CI](https://github.com/getarcis/arcis-example-mcp/actions/workflows/ci.yml/badge.svg)](https://github.com/getarcis/arcis-example-mcp/actions/workflows/ci.yml)

# arcis-example-mcp

> Demo: 9 tool-call results land in an AI agent. 7 are prompt-injection payloads from the V32 toolcall / DAN / `<system>`-tag families. Arcis catches every malicious one. Safe results pass through.

## What this is

A minimal demonstration of Arcis's V32 agent-toolcall-injection detectors (shipped in v1.6.0) and the older DAN / `<system>`-tag jailbreak signatures, all running through `detectPromptInjection` from `@arcis/node`.

This is the same function the [`@arcis/mcp`](https://www.npmjs.com/package/@arcis/mcp) server exposes as the `arcis_detect_prompt_injection` MCP tool. Any MCP-aware AI agent (Cursor or any other MCP client) can invoke it on every tool-call result before forwarding the result to the model.

Files:

- [`demo.js`](./demo.js): 9 payloads (2 safe, 5 V32 toolcall injection, 2 classic jailbreaks). Runs each through `detectPromptInjection` and reports which were caught.

Total dependencies: `@arcis/node`. Nothing else.

## Run it

```bash
npm install
npm run demo
```

Expected output:

```
Arcis prompt-injection demo against detectPromptInjection
------------------------------------------------------------------------
OK     safe     plain tool result: clean (passed through, as expected)
OK     safe     JSON tool result: clean (passed through, as expected)
BLOCK  v32      agent-toolcall-marker: caught (rule=agent-toolcall-marker, severity=high)
BLOCK  v32      agent-tool-name-spoof: caught (rule=agent-tool-name-spoof, severity=high)
BLOCK  v32      agent-tool-result-marker: caught (rule=agent-tool-result-marker, severity=high)
BLOCK  v32      ansi-escape-sequence: caught (rule=ignore-previous-instructions, severity=high)
BLOCK  v32      claude-tool-use-tags: caught (rule=claude-tool-use-tags, severity=high)
BLOCK  classic  fake-system-tag: caught (rule=ignore-previous-instructions, severity=high)
BLOCK  classic  DAN jailbreak: caught (rule=jailbreak-dan, severity=high)
------------------------------------------------------------------------
7 injections caught, 2 safe calls passed, 0 unexpected
```

(The ANSI-escape and fake-system-tag payloads also trigger `ignore-previous-instructions` from the older signature library — the detector reports the highest-severity match. Both rules fire on those payloads.)

## V32 toolcall injection — what it catches

Five new patterns shipped in Arcis v1.6.0, specifically aimed at the AI-agent runtime where one compromised tool result can pivot an entire session:

| Rule | Catches |
|---|---|
| `agent-toolcall-marker` | Strings that mimic the JSON shape of a tool call: `{"tool_call": ...}`, `{"function_call": ...}`, `{"call_tool": ...}` |
| `agent-tool-name-spoof` | Tool-name strings that target dangerous primitives: `exec`, `shell`, `run_command`, `eval`, `read_file`, `write_file`, `delete_file` |
| `agent-tool-result-marker` | Strings that mimic the JSON shape of a tool result: `{"tool_result": ...}`, `{"tool_output": ...}` |
| `ansi-escape-sequence` | ANSI control sequences in tool output. Used in terminal-clear injections (`\x1b[2J\x1b[H...`) that pivot the conversation by visually rewriting prior context |
| `claude-tool-use-tags` | Tag-style invocations like `<tool_use>`, `<invoke>`, `<function_calls>` that some agent runtimes literally execute when echoed |

## How an AI agent uses this in production

```js
import { detectPromptInjection } from '@arcis/node';

async function runToolAndForward(toolName, toolArgs) {
  const result = await callTool(toolName, toolArgs);

  const finding = detectPromptInjection(result);
  if (finding) {
    return {
      role: 'tool',
      name: toolName,
      content: `[Arcis blocked: ${finding.rule}]`,
    };
  }

  return { role: 'tool', name: toolName, content: result };
}
```

The MCP form ([`@arcis/mcp`](https://www.npmjs.com/package/@arcis/mcp)) wraps the same logic as the `arcis_detect_prompt_injection` tool, so an agent that speaks MCP can invoke it the same way it invokes any other registered tool.

## Sister examples

| Framework | Repo |
|---|---|
| Express | [`arcis-example-express`](https://github.com/getarcis/arcis-example-express) |
| FastAPI | [`arcis-example-fastapi`](https://github.com/getarcis/arcis-example-fastapi) |
| Gin (Go) | [`arcis-example-gin`](https://github.com/getarcis/arcis-example-gin) |
| Bun + Hono | [`arcis-example-bun`](https://github.com/getarcis/arcis-example-bun) |
| NestJS | [`arcis-example-nestjs`](https://github.com/getarcis/arcis-example-nestjs) |
| Next.js | [`arcis-example-nextjs`](https://github.com/getarcis/arcis-example-nextjs) |

## License

MIT.
