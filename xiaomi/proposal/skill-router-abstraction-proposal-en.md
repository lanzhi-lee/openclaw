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

interface SkillRouteContextMessage {
  role: "user" | "assistant";
  text: string;
}

interface SkillRouteContext {
  /**
   * A small, framework-selected recent conversation window. Plugins receive
   * structured messages instead of a pre-rendered string so each router can
   * decide how much context to use.
   */
  recentMessages: SkillRouteContextMessage[];
}

type SkillRouteResult =
  /** Single high-confidence match — framework resolves this name from the prompt-visible candidates */
  | { mode: "direct"; name: string }
  /** Multiple high-confidence matches — framework injects only these prompt-visible candidates */
  | { mode: "ambiguous"; candidates: Array<{ name: string; score: number }> }
  /** No match — skip skill, LLM handles on its own */
  | { mode: "nomatch" };

interface SkillRouter {
  readonly name: string;
  route(
    query: string,
    candidates: SkillCandidate[],
    ctx?: SkillRouteContext,
  ): Promise<SkillRouteResult>;
}
```

The router returns names, not raw `SKILL.md` content. Core resolves those names
against the same prompt-visible candidate set that would have been rendered in
`<available_skills>`, so disabled skills, agent skill filters, and runtime tool
restrictions cannot leak extra skills into routing.

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

### 3.3 Integration: Pre-Routing / Pre-Filter

Section 3.1 defines "how to route"; this section explains "where to plug in".
Routing is executed by framework code before the final model request is built.
The model is not asked to perform routing. When a router matches, core
suppresses the full skill catalog from the static system prompt and injects the
narrowed skill context as dynamic, prompt-local content before the user message.

Earlier versions of this proposal described the dynamic payload as a first tool
result. The important contract is narrower: the routed skill payload must stay
out of the static system prompt so prefix cache behavior is protected. The exact
carrier should follow OpenClaw's runtime message model; the current implementation
uses prompt-local context rather than a synthetic tool result.

```
┌──────────────────────────────────────┐
│  Static system prompt (unchanged,    │
│  cacheable)                          │
│  - no per-turn routed skill payload  │
└──────────────────────────────────────┘
┌──────────────────────────────────────┐
│  Dynamic prompt-local context        │
│  - Matched <available_skills> subset │
│  - Or no match, no skill catalog     │
└──────────────────────────────────────┘
┌──────────────────────────────────────┐
│  User message                        │
└──────────────────────────────────────┘
```

```typescript
// Agent runner pseudocode (message format is simplified; actual implementation
// should use OpenClaw's prompt-local context/message composition primitives)
async function runAgentTurn(params) {
  let systemPrompt = STATIC_SYSTEM_PROMPT;
  let userPrompt = params.userMessage;
  let suppressFullSkillCatalog = false;

  // Pre-routing / pre-filtering
  const router = resolveSkillRouter(config.skills.router.name, config.skills.router.config);
  if (router) {
    const result = await router.route(params.userMessage, params.skillCandidates, {
      recentMessages: params.recentMessages,
    });

    if (result.mode === "direct") {
      const matched = findCandidateByName(params.skillCandidates, result.name);
      if (matched) {
        userPrompt = `${formatSkillsForPrompt([matched])}\n\n${userPrompt}`;
        suppressFullSkillCatalog = true;
      }
    } else if (result.mode === "ambiguous" && result.candidates.length > 0) {
      const matched = result.candidates
        .map((candidate) => findCandidateByName(params.skillCandidates, candidate.name))
        .filter(Boolean);
      if (matched.length > 0) {
        userPrompt = `${formatSkillsForPrompt(matched)}\n\n${userPrompt}`;
        suppressFullSkillCatalog = true;
      }
    } else if (result.mode === "nomatch") {
      suppressFullSkillCatalog = true;
    }
  }

  if (!suppressFullSkillCatalog) {
    systemPrompt = appendFullSkillsCatalog(systemPrompt, params.skillCandidates);
  }

  return callLLM({ systemPrompt, userPrompt });
}
```

**When no router is configured**: no dynamic routed context is injected; all
skill descriptions remain in the system prompt, so behavior is identical to
today. If a configured router errors or returns an invalid name, core should log
the failure and fall back to the full catalog.

## 4. Migration Path

1. Core adds the `SkillRouter` interface + `registerSkillRouter` / `resolveSkillRouter`
2. Build router candidates from the same prompt-visible skill set used by the normal skill catalog
3. Insert a pre-routing step before final prompt construction: route if a router is configured, otherwise keep the current full injection
4. Pass a small structured recent-message context to help routers handle short follow-up prompts without forcing every plugin into the same string rendering

## 5. Related Considerations

The following topics are related to this proposal and can be discussed separately:

- **Prefix Cache**: routing results are injected outside the static system prompt, keeping the cacheable prefix stable. A tool result is one possible carrier, but prompt-local dynamic context also satisfies this requirement.
- **Recent Context**: routers can receive a bounded structured recent-message list so short prompts such as "do this too" can be routed without giving plugins a pre-rendered transcript string.
- **Diagnostic Events**: the routing process should emit diagnostic events (call/result/fallback), reusing the existing `diagnostic-events.ts` system
- **SKILL.md Metadata**: `triggers` ([#76782](https://github.com/openclaw/openclaw/issues/76782)), `tags`, `model` ([#58142](https://github.com/openclaw/openclaw/issues/58142)) and other fields can improve routing accuracy as optional interface extensions
- **Plugin Hook**: plugins can inject routing results via `PluginHookBeforeAgentStartResult` ([#21823](https://github.com/openclaw/openclaw/issues/21823))
- **Community Solutions**: [Skillporter](https://github.com/JuanHuaXu/skillporter/tree/openclaw), [SkillPilot](https://github.com/RealTapeL/SkillPilot), [Skill Router v2](https://github.com/openclaw/openclaw/issues/81826) etc. can all be adapted as `SkillRouter` plugins
