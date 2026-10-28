# Code Layer Linter

[中文说明](README.zh-CN.md)

An AI coding skill that checks whether each code change sits in the correct architectural layer before merging.

**Language-agnostic** — works with Java, Go, Python, TypeScript, or any layered architecture.
**Platform-agnostic** — works with Claude Code, Cursor, Codex, Windsurf, or any AI coding assistant that reads markdown instructions.

## What It Does

Given a branch diff, this skill:

1. **Splits** the diff into concrete change points (lock, transaction, remote call, validation, cache, MQ, state transition…)
2. **Judges** each point against 6 industry layering principles (P1–P6)
3. **Outputs** a table of ✅ / ⚠️ / ❓ verdicts with move-to suggestions

Advisory only — the submitter decides what to fix.

## The 6 Principles

| # | Principle |
|---|-----------|
| P1 | Entry layer translates protocol only |
| P2 | Guarantees live where every entry point must pass |
| P3 | Technical details hide behind infrastructure |
| P4 | Dependencies flow downward only |
| P5 | Cross-cutting concerns use the framework's mechanism |
| P6 | Model conversion at boundaries |

## Installation

### Claude Code

```bash
mkdir -p .claude/skills/code-layer-linter
cp SKILL.md .claude/skills/code-layer-linter/
```

Invoke manually: `/code-layer-linter` or ask "run a layer lint on my branch."

### Cursor

```bash
mkdir -p .cursor/skills/code-layer-linter
cp SKILL.md .cursor/skills/code-layer-linter/
```

### Codex (OpenAI)

Append the content of `SKILL.md` to your `AGENTS.md`, or reference it:

```markdown
<!-- In AGENTS.md -->
Before merging any branch, follow the instructions in .codex/code-layer-linter.md
to check that each change point sits in the correct architectural layer.
```

### Windsurf / Other AI Assistants

Place `SKILL.md` wherever your tool reads custom instructions (e.g., `.windsurfrules`, project-level config), or paste its content into your system prompt.

## Auto-Trigger: Make It Run on Every Merge

Installing the skill is step one. Step two is making sure it **actually runs** before code gets merged. Here's how for each platform:

### Claude Code — Hook (Recommended)

Add to `.claude/settings.json` in your project:

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

Or add to your project's `CLAUDE.md`:

```markdown
## Pre-merge Checklist
Before merging any branch, run `/code-layer-linter` to verify layer placement.
```

### Cursor — Rules

Add to `.cursor/rules/pre-merge.mdc`:

```markdown
Before creating a PR or merging, run the code-layer-linter skill to check
that each change point in the diff sits in the correct architectural layer.
```

### Codex — AGENTS.md

Add to `AGENTS.md`:

```markdown
## Pre-merge Gate
Before completing any task that modifies production code, run a layer placement
check per .codex/code-layer-linter.md against the branch diff.
```

### CI Integration (Any Platform)

If your AI assistant supports CI integration, add a step that prompts the check:

```yaml
# Example: GitHub Actions reminder (human-in-the-loop)
- name: Layer placement reminder
  run: echo "⚠️ Have you run a code-layer-linter check on this branch?"
```

> **Note**: The actual analysis is done by the AI assistant reading `SKILL.md`, not by a standalone CLI tool. The CI step above is a reminder, not an automated lint. For full automation, you would need to wrap the AI call in a script.

## Example Output

```
Layer map: *-service-impl (facade) → *-core-service (core) → *-dal (DAL) / integration

| # | Change point | Current location | Verdict | Basis | Suggestion |
|---|---|---|---|---|---|
| 1 | Concurrency lock for setup | SetupFacade#init / facade | ⚠️ | P2 | Move to SetupCoreService |
| 2 | Order status transition | OrderCoreService#submit / core | ✅ | P2 | — |
| 3 | External API timeout config | PaymentCoreService#charge / core | ⚠️ | P3 | Move to PaymentClient (integration) |

Summary: ✅ 1 / ⚠️ 2 / ❓ 0
```

## Scope

- **In scope**: intra-repo layer placement (is this lock in core or facade? is this SQL behind a repository?)
- **Out of scope**: cross-service ownership, naming, code style, bugs, performance

## License

MIT
