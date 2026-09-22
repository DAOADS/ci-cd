---
name: ci-cd
description: "CI/CD 流水线：GitHub Actions 工作流、四阶段门禁自动化、发布自动化、制品管理。组合 github-pr-workflow + github-repo-management + github-code-review。"
version: "1.0.0"
author: Hermes Agent
license: MIT
platforms: [windows, linux, macos]
metadata:
  hermes:
    tags: [ci, cd, github-actions, pipeline, release]
    category: devops
    skill_type: tool
    requires_toolsets: [terminal, file]
---

# CI/CD Skill

> **职责**：持续集成/部署 — GitHub Actions 工作流、门禁自动化、语义化发布、制品管理

## When to Use

- 仓库创建后：配置 CI/CD
- 流水线变更：增删阶段、调整阈值
- 发布自动化：标签触发、构建、发布

## Don't Use For

- 本地开发流水线（见 `Makefile`）
- 门禁判定逻辑（见 `quality-gate`）
- 代码实现/测试（各阶段 Skill）

## Procedure

### 1. 生成标准工作流

```yaml
# .github/workflows/sdlc-pipeline.yml
name: SDLC Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  workflow_dispatch:
    inputs:
      stage:
        type: choice
        options: [s1, s2, s3, s4, all]
        default: all

jobs:
  gate-s1:
    if: github.event.inputs.stage == 's1' || github.event.inputs.stage == 'all' || github.event_name != 'workflow_dispatch'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Setup uv
        uses: astral-sh/setup-uv@v3
      - name: Run S1 Gate
        run: make gate-s1

  gate-s2:
    needs: gate-s1
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Setup uv
        uses: astral-sh/setup-uv@v3
      - name: Run S2 Gate
        run: make gate-s2

  gate-s3:
    needs: gate-s2
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Setup uv
        uses: astral-sh/setup-uv@v3
      - name: Run S3 Gate
        run: make gate-s3

  gate-s4:
    needs: gate-s3
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Setup uv
        uses: astral-sh/setup-uv@v3
      - name: Run S4 Gates (parallel)
        run: |
          make test-unit &
          make test-integration &
          make test-gui &
          make test-golden &
          wait
      - name: Quality Gate
        run: hermes skill run quality-gate --stage all
      - name: Upload Reports
        uses: actions/upload-artifact@v4
        with:
          name: test-reports
          path: docs/REPORTS/

  release:
    needs: gate-s4
    if: startsWith(github.ref, 'refs/tags/v')
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build & Publish
        run: |
          uv build
          uv publish
```

### 2. 语义化发布配置

```yaml
# .github/workflows/release.yml
# 触发：推送 v*.*.* 标签
# 动作：构建、测试、发布 PyPI、生成 changelog、创建 GitHub Release
```

### 3. 依赖更新自动化

```yaml
# .github/workflows/dependabot.yml
# Dependabot PR 自动合并（通过门禁后）
```

## Output

- `.github/workflows/sdlc-pipeline.yml`
- `.github/workflows/release.yml`
- `.github/dependabot.yml`
- Kanban：`artifact: CI_CD_CONFIG`

## Verification

- 工作流语法正确（`actionlint` 通过）
- 推送触发正常执行
- 四阶段门禁在 CI 中同步运行
- 发布流程端到端验证通过

## Pitfalls

1. **不要在 CI 中硬编码密钥**：用 GitHub Secrets / OIDC
2. **不要跳过门禁**：CI 必须跑完整 `make pipeline`
3. **不要并行有依赖的阶段**：S1→S2→S3→S4 串行，S4 内部并行

## References

- [references/github-actions-patterns.md](references/github-actions-patterns.md)
- [references/semantic-release.md](references/semantic-release.md)
- [references/artifact-management.md](references/artifact-management.md)

## Templates

- [templates/sdlc-pipeline.yml](templates/sdlc-pipeline.yml)
- [templates/release.yml](templates/release.yml)
- [templates/dependabot.yml](templates/dependabot.yml)

## Scripts

- [scripts/generate_workflows.py](scripts/generate_workflows.py)
- [scripts/validate_workflows.py](scripts/validate_workflows.py)