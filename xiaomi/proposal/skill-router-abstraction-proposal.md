# Skill Router Abstraction Layer Proposal

> Author: Lance (Xiaomi) | Date: 2026-05-27

**TLDR**: 在 `buildSkillsSection` 之前插入一个抽象的 `SkillRouter` 接口，由 plugin 提供实现，未配置时 fallback 回现有的全量 description 注入。core 只定义协议，不绑定任何路由策略。

> 本文聚焦于抽象层的设计思路和接口方向，具体实现细节待社区对齐后再展开讨论。

## 1. Problem

当 skill 数量多时，OpenClaw 将所有 skill 的描述信息注入 system prompt（通过 `buildSkillsSection`）：

```
当前：system prompt 中注入所有 skill 的 name + description → LLM 从中选择
```

这导致：

- **上下文浪费**：大量无用 skill 描述占用 context window
- **干扰决策**：skill 越多，LLM 越容易误选或忽略
- **无扩展点**：不同场景对精度和成本的需求差异巨大，但无法定制路由策略

## 2. Existing Efforts

社区已有多次尝试，但均未形成统一抽象：

| 方案                                                                  | 思路                                               | 状态                 |
| --------------------------------------------------------------------- | -------------------------------------------------- | -------------------- |
| [PR #85359](https://github.com/openclaw/openclaw/pull/85359)          | LLM 调用 `local_skill_route` 工具，本地 token 匹配 | OPEN，未合入         |
| [Skillporter](https://github.com/JuanHuaXu/skillporter/tree/openclaw) | 独立 sidecar HTTP 服务，关键词 + 权重打分          | 外部项目             |
| [SkillPilot](https://github.com/RealTapeL/SkillPilot)                 | 关键词 + 向量，三阶段路由                          | 外部项目             |
| [Skill Router v2](https://github.com/openclaw/openclaw/issues/81826)  | BM25 + Embedding + LLM 精排，96% 正确率            | 外部项目，issue 已关 |
| [#61740](https://github.com/openclaw/openclaw/issues/61740)           | 提议 `skill_search` 工具做 runtime discovery       | OPEN                 |
| [#76782](https://github.com/openclaw/openclaw/issues/76782)           | 提议 SKILL.md 增加 `triggers` 字段                 | OPEN                 |
| [#21823](https://github.com/openclaw/openclaw/issues/21823)           | 提议 plugin hook 动态过滤 skill                    | CLOSED               |

共同问题：各自独立实现，无法复用和组合；且多数方案作为 LLM 工具调用，增加了 round-trip 和 token 消耗。

**目标**：在 `buildSkillsSection` 之前引入一个抽象的 `SkillRouter`，由 plugin 提供实现，未配置时 fallback 回现有的全量 description 注入。core 只定义协议，不绑定任何路由策略。

## 3. Design

### 3.1 Core Interface

```typescript
interface SkillCandidate {
  name: string;
  description?: string;
  filePath?: string;
}

type SkillRouteResult =
  /** 唯一高置信匹配，框架直接加载 SKILL.md */
  | { mode: "direct"; name: string; skillMd?: string }
  /** 多个高置信匹配，框架将候选列表注入，由 LLM 或用户选择 */
  | { mode: "ambiguous"; candidates: Array<{ name: string; score: number }> }
  /** 无匹配，跳过 skill，LLM 自行处理 */
  | { mode: "nomatch" };

interface SkillRouter {
  readonly name: string;
  route(query: string, candidates: SkillCandidate[]): Promise<SkillRouteResult>;
}
```

### 3.2 Plugin Registration

core 只认 `SkillRouter` 接口，不硬编码任何具体实现。通过注册机制，plugin 安装时声明自己提供哪种 router，用户通过配置选择使用哪个。本质是依赖注入：core 定义接口，plugin 提供实现，配置选择用哪个。

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

Plugin 注册：

```typescript
// extensions/my-router/src/index.ts
import { registerSkillRouter } from "openclaw/plugin-sdk";

registerSkillRouter("my-router", (config) => new MyRouter(config));
```

配置：

```json
{ "skills": { "router": { "name": "my-router", "config": {} } } }
```

### 3.3 Integration: Pre-LLM Routing

接口定义了"怎么路由"，这部分说明"插在哪里"。路由在构建 LLM 请求之前由框架代码执行，结果注入为首条 tool result，LLM 不感知路由过程：

```
┌─────────────────────────────────────┐
│  静态 system prompt（不变，可缓存）    │
│  - "见下方 tool result 中的技能指令"   │
└─────────────────────────────────────┘
┌─────────────────────────────────────┐
│  动态 tool result（首条）             │
│  - 路由匹配到的 SKILL.md 内容         │
│  - 或无匹配，不注入                   │
└─────────────────────────────────────┘
┌─────────────────────────────────────┐
│  user message                       │
└─────────────────────────────────────┘
```

```typescript
// agent runner 伪代码（消息格式为简化示意，实际需遵循 OpenClaw 的 tool result 协议）
async function runAgentTurn(params) {
  const messages = [];

  // 前置路由
  const router = resolveSkillRouter(config.skills.router.name, config.skills.router.config);
  if (router) {
    const result = await router.route(params.userMessage, params.skillCandidates);
    if (result.mode === "direct" && result.skillMd) {
      // 唯一高置信匹配 → 直接注入 SKILL.md
      messages.push({ role: "tool_result", content: result.skillMd });
    } else if (result.mode === "ambiguous") {
      // 多个高置信匹配 → 注入候选列表，LLM 选择最合适的
      const list = result.candidates.map((c) => `- ${c.name} (score: ${c.score})`).join("\n");
      messages.push({
        role: "tool_result",
        content: `Ambiguous skill match, pick the most specific:\n${list}`,
      });
    }
    // nomatch → 不注入，LLM 自行处理
  }

  messages.push({ role: "user", content: params.userMessage });
  return callLLM({ systemPrompt: STATIC_SYSTEM_PROMPT, messages });
}
```

**无 router 时**：不注入 tool result，全量 description 仍在 system prompt 中，行为与现在一致。

## 4. Migration Path

1. core 新增 `SkillRouter` 接口 + `registerSkillRouter` / `resolveSkillRouter`
2. `buildSkillsSection` 之前插入前置路由步骤：有 router 则路由，无 router 则保持现有全量注入

## 5. Related Considerations

以下议题与本提案相关，后续可单独展开讨论：

- **Prefix Cache**：路由结果注入为 tool result 而非 system prompt，保持 system prompt 静态，保护 cache 命中率
- **Diagnostic Events**：路由过程需要 emit 诊断事件（call/result/fallback），复用现有 `diagnostic-events.ts` 体系
- **SKILL.md 元数据**：`triggers`（[#76782](https://github.com/openclaw/openclaw/issues/76782)）、`tags`、`model`（[#58142](https://github.com/openclaw/openclaw/issues/58142)）等字段可提升路由精度，作为接口的可选扩展
- **Plugin Hook**：插件可通过 `PluginHookBeforeAgentStartResult` 注入路由结果（[#21823](https://github.com/openclaw/openclaw/issues/21823)）
- **社区已有方案**：[Skillporter](https://github.com/JuanHuaXu/skillporter/tree/openclaw)、[SkillPilot](https://github.com/RealTapeL/SkillPilot)、[Skill Router v2](https://github.com/openclaw/openclaw/issues/81826) 等均可适配为 `SkillRouter` plugin
