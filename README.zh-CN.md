# Code Layer Linter（代码分层位置检查）

一个 AI 编程助手 Skill，在合并前检查每个代码改动点是否放在了正确的架构层。

**不限语言** — Java / Go / Python / TypeScript 等分层架构项目均适用。
**不限平台** — Claude Code、Cursor、Codex、Windsurf 或任何能读 Markdown 指令的 AI 编程助手均可使用。

## 它做什么

给定一个分支的 diff：

1. **拆分**改动为具体的改动点（锁、事务边界、远程调用、校验、缓存、消息队列、状态转换……）
2. **逐个判定**是否符合 6 条业界分层原则（P1–P6）
3. **输出**一张 ✅ / ⚠️ / ❓ 结论表，附带挪动建议

只出建议，不阻断合并——修不修由提交人决定。

## 6 条原则

| # | 原则 | 一句话判定 |
|---|------|-----------|
| P1 | 入口层只翻译协议 | Controller/Facade/消费者/定时任务入口不出现业务逻辑、事务、锁 |
| P2 | 保证跟着规则走 | 锁、事务、幂等、状态推进放 core 层——换个入口调同一业务，保证还在吗？ |
| P3 | 技术细节藏在基础设施后面 | SQL 放 DAL，外部调用放 integration client，core 不直接碰 |
| P4 | 依赖单向向下 | 下层不引上层类型，common 只放纯工具/常量 |
| P5 | 横切关注点走框架机制 | 仓库已有切面/拦截器就用它，不手抄 |
| P6 | 模型转换在边界 | DTO 转换放 facade/integration 边界，不穿透 core |

## 安装

### Claude Code

```bash
mkdir -p .claude/skills/code-layer-linter
cp SKILL.md .claude/skills/code-layer-linter/
```

手动调用：`/code-layer-linter` 或直接说"帮我跑一下 layer lint"。

### Cursor

```bash
mkdir -p .cursor/skills/code-layer-linter
cp SKILL.md .cursor/skills/code-layer-linter/
```

### Codex (OpenAI)

将 `SKILL.md` 内容追加到 `AGENTS.md`，或引用它：

```markdown
<!-- 在 AGENTS.md 中 -->
合并任何分支前，按 .codex/code-layer-linter.md 的指令检查每个改动点的分层位置。
```

### Windsurf / 其他 AI 助手

把 `SKILL.md` 放到你的工具读取自定义指令的位置（如 `.windsurfrules`），或将内容粘贴到系统提示词中。

## 自动触发：让它在每次合并前自动运行

装了 Skill 是第一步。第二步是确保它在代码合并前**真的会跑**：

### Claude Code — Hook（推荐）

在项目的 `.claude/settings.json` 中添加：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash(git push|gh pr create)",
        "hook": "echo 'IMPORTANT: Before pushing or creating a PR, run /code-layer-linter to check layer placement of all change points.'"
      }
    ]
  }
}
```

或在项目 `CLAUDE.md` 中添加：

```markdown
## 合并前检查清单
合并任何分支前，运行 `/code-layer-linter` 检查分层位置。
```

### Cursor — Rules

添加 `.cursor/rules/pre-merge.mdc`：

```markdown
创建 PR 或合并前，运行 code-layer-linter skill 检查 diff 中每个改动点的分层位置。
```

### Codex — AGENTS.md

在 `AGENTS.md` 中添加：

```markdown
## 合并前关卡
完成任何涉及生产代码修改的任务前，按 .codex/code-layer-linter.md 对分支 diff 做分层位置检查。
```

> **说明**：实际分析由 AI 助手读取 `SKILL.md` 完成，不是独立的 CLI 工具。如需全自动化，需要将 AI 调用封装为脚本。

## 示例输出

```
Layer map: *-service-impl (facade) → *-core-service (core) → *-dal (DAL) / integration

| # | 改动点 | 当前位置 | 结论 | 依据 | 建议 |
|---|--------|---------|------|------|------|
| 1 | 初始化并发锁 | SetupFacade#init / facade | ⚠️ | P2 | 挪到 SetupCoreService |
| 2 | 订单状态推进 | OrderCoreService#submit / core | ✅ | P2 | — |
| 3 | 外部API超时配置 | PaymentCoreService#charge / core | ⚠️ | P3 | 挪到 PaymentClient (integration) |

结论：✅ 1 / ⚠️ 2 / ❓ 0
```

## 检查范围

- **管**：仓库内分层位置（锁该在 core 还是 facade？SQL 是否藏在 repository 后面？）
- **不管**：跨服务归属、命名、代码风格、bug、性能

## 许可

MIT
