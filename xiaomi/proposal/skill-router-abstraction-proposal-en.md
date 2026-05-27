# Skill Router Abstraction Layer Proposal

> Author: Lance (Xiaomi) | Date: 2026-05-27

**TLDR**: Introduce an abstract `SkillRouter` interface before `buildSkillsSection`, implemented by plugins. When unconfigured, fall back to the current full description injection. Core only defines the protocol; it does not bind to any routing strategy.

> This document focuses on the abstraction design and interface direction. Implementation details will be discussed after community alignment.

## 1. Problem

When there are many skills, OpenClaw injects all skill descriptions into the system prompt (via `buildSkillsSection`):

```
Current: inject all skill names + descriptions into system prompt → LLM selects from them
```

This causes:

- **Context waste**: large amounts of irrelevant skill descriptions consume the context window
- **Decision interference**: the more skills, the more likely the LLM misselects or overlooks
- **No extension point**: different scenarios have vastly different precision/cost needs, but there's no way to customize routing strategy

## 2. Existing Efforts

The community has made multiple attempts, but none formed a unified abstraction:

| Approach                                                              | Idea                                                        | Status                 |
| --------------------------------------------------------------------- | ----------------------------------------------------------- | ---------------------- |
| [PR #85359](https://github.com/openclaw/openclaw/pull/85359)          | LLM calls `local_skill_route` tool, local token matching    | OPEN, not merged       |
| [Skillporter](https://github.com/JuanHuaXu/skillporter/tree/openclaw) | Standalone sidecar HTTP service, keyword + weighted scoring | External project       |
| [SkillPilot](https://github.com/RealTapeL/SkillPilot)                 | Keywords + vectors, three-stage routing                     | External project       |
| [Skill Router v2](https://github.com/openclaw/openclaw/issues/81826)  | BM25 + Embedding + LLM reranking, 96% accuracy              | External, issue closed |
| [#61740](https://github.com/openclaw/openclaw/issues/61740)           | Proposes `skill_search` tool for runtime discovery          | OPEN                   |
| [#76782](https://github.com/openclaw/openclaw/issues/76782)           | Proposes adding `triggers` field to SKILL.md                | OPEN                   |
| [#21823](https://github.com/openclaw/openclaw/issues/21823)           | Proposes plugin hook for dynamic per-turn skill filtering   | CLOSED                 |

Common problem: each implementation is standalone, not reusable or composable. Most approaches work as LLM tool calls, adding round-trips and token overhead.

**Goal**: Introduce an abstract `SkillRouter` before `buildSkillsSection`, implemented by plugins. When unconfigured, fall back to the current full description injection. Core only defines the protocol; it does not bind to any routing strategy.

## 3. Design

### 3.1 Core Interface

```typescript
interface SkillCandidate {
  name: string;
  description?: string;
  filePath?: string;
}

type SkillRouteResult =
  /** Single high-confidence match — framework loads SKILL.md directly */
  | { mode: "direct"; name: string; skillMd?: string }
  /** Multiple high-confidence matches — framework injects candidate list for LLM or user to choose */
  | { mode: "ambiguous"; candidates: Array<{ name: string; score: number }> }
  /** No match — skip skill, LLM handles on its own */
  | { mode: "nomatch" };

interface SkillRouter {
  readonly name: string;
  route(query: string, candidates: SkillCandidate[]): Promise<SkillRouteResult>;
}
```

### 3.2 Plugin Registration

Core only recognizes the `SkillRouter` interface; it does not hardcode any concrete implementation. Through a registration mechanism, plugins declare which router they provide at install time, and users select one via configuration. This is dependency injection: core defines the interface, plugins provide the implementation, config selects which one to use.

```typescript
// src/agents/skills/router-registry.ts
type SkillRouterFactory = (config: Record<string, unknown>) => SkillRouter;
const registry = new Map<string, SkillRouterFactory>();

export function registerSkillRouter(name: string, factory: SkillRouterFactory) {
  registry.set(name, factory);
}

export function resolveSkillRouter(
  name: string,
  config?: Record<string, unknown>,
): SkillRouter | null {
  const factory = registry.get(name);
  return factory ? factory(config ?? {}) : null;
}
```

Plugin registration:

```typescript
// extensions/my-router/src/index.ts
import { registerSkillRouter } from "openclaw/plugin-sdk";

registerSkillRouter("my-router", (config) => new MyRouter(config));
```

Configuration:

```json
{ "skills": { "router": { "name": "my-router", "config": {} } } }
```

### 3.3 Integration: Pre-LLM Routing

Section 3.1 defines "how to route"; this section explains "where to plug in". Routing is executed by framework code before building the LLM request. The result is injected as the first tool result — the LLM is unaware of the routing process:

```
┌──────────────────────────────────────┐
│  Static system prompt (unchanged,    │
│  cacheable)                          │
│  - "See skill instructions in the    │
│    tool result below"                │
└──────────────────────────────────────┘
┌──────────────────────────────────────┐
│  Dynamic tool result (first item)    │
│  - Matched SKILL.md content          │
│  - Or no match, nothing injected     │
└──────────────────────────────────────┘
┌──────────────────────────────────────┐
│  User message                        │
└──────────────────────────────────────┘
```

```typescript
// Agent runner pseudocode (message format is simplified; actual implementation
// must follow OpenClaw's tool result protocol)
async function runAgentTurn(params) {
  const messages = [];

  // Pre-LLM routing
  const router = resolveSkillRouter(config.skills.router.name, config.skills.router.config);
  if (router) {
    const result = await router.route(params.userMessage, params.skillCandidates);
    if (result.mode === "direct" && result.skillMd) {
      // Single high-confidence match → inject SKILL.md directly
      messages.push({ role: "tool_result", content: result.skillMd });
    } else if (result.mode === "ambiguous") {
      // Multiple high-confidence matches → inject candidate list for LLM to pick
      const list = result.candidates.map((c) => `- ${c.name} (score: ${c.score})`).join("\n");
      messages.push({
        role: "tool_result",
        content: `Ambiguous skill match, pick the most specific:\n${list}`,
      });
    }
    // nomatch → inject nothing, LLM handles on its own
  }

  messages.push({ role: "user", content: params.userMessage });
  return callLLM({ systemPrompt: STATIC_SYSTEM_PROMPT, messages });
}
```

**When no router is configured**: no tool result is injected; all skill descriptions remain in the system prompt — behavior is identical to today.

## 4. Migration Path

1. Core adds the `SkillRouter` interface + `registerSkillRouter` / `resolveSkillRouter`
2. Insert a pre-routing step before `buildSkillsSection`: route if a router is configured, otherwise keep the current full injection

## 5. Related Considerations

The following topics are related to this proposal and can be discussed separately:

- **Prefix Cache**: routing results are injected as tool results rather than into the system prompt, keeping the system prompt static and protecting cache hit rates
- **Diagnostic Events**: the routing process should emit diagnostic events (call/result/fallback), reusing the existing `diagnostic-events.ts` system
- **SKILL.md Metadata**: `triggers` ([#76782](https://github.com/openclaw/openclaw/issues/76782)), `tags`, `model` ([#58142](https://github.com/openclaw/openclaw/issues/58142)) and other fields can improve routing accuracy as optional interface extensions
- **Plugin Hook**: plugins can inject routing results via `PluginHookBeforeAgentStartResult` ([#21823](https://github.com/openclaw/openclaw/issues/21823))
- **Community Solutions**: [Skillporter](https://github.com/JuanHuaXu/skillporter/tree/openclaw), [SkillPilot](https://github.com/RealTapeL/SkillPilot), [Skill Router v2](https://github.com/openclaw/openclaw/issues/81826) etc. can all be adapted as `SkillRouter` plugins
