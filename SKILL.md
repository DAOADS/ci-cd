---
name: ci-cd
description: "CI/CD 流水线：GitHub Actions 工作流、四阶段门禁自动化、发布自动化、制品管理。组合 github-pr-workflow + github-repo-management + github-code-review。"
version: "1.1.0"
author: Hermes Agent
license: MIT
platforms: [windows, linux, macos]
metadata:
  hermes:
    tags: [ci, cd, github-actions, pipeline, release]
    category: devops
    skill_type: tool
    requires_toolsets: [terminal, file]
    local_override: true
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

### 4. S5 发布门禁校验（来自 Profile release_checklist）

**必须全部通过，任一失败即阻断**：

|| 校验项 | 标准 | 失败处理 ||
|--------|------|----------|
| 版本号语义化 | MAJOR.MINOR.PATCH，符合语义化版本规范 | 阻断，要求修正 |
| CHANGELOG 生成 | `CHANGELOG.md` 从 Conventional Commits 自动生成 | 阻断，要求生成 |
| 所有测试通过 | 单元/集成/E2E/GUI 全层测试 100% 通过 | 阻断，修复失败测试 |
| 覆盖率 ≥90% | `pytest --cov --cov-fail-under=90` | 阻断，补全测试 |
| 安全扫描无高危 | `uv-audit` / `bandit` / `trivy` 无高危 | 阻断，修复漏洞 |
| 依赖审计无高危 | `uv-audit` 无高危漏洞 | 阻断，升级/替换依赖 |
| DB 迁移回滚就绪 | 迁移脚本含双向操作，回滚测试通过 | 阻断，补全回滚脚本 |
| 灰度发布策略 | 10% → 50% → 100% 流量切分方案就绪 | 阻断，提供策略文档 |
| 监控告警规则更新 | 新版本监控指标/告警已同步 | 阻断，更新监控配置 |
| **性能基线回归门禁** | `s4-validation/perf-baseline --action gate --threshold-profile core` | **exit≠0 则阻断发布** |

**门禁命令**：
```bash
hermes gate-check s5
# 期望：PASS
```

## Output

- `.github/workflows/sdlc-pipeline.yml`
- `.github/workflows/release.yml`
- `.github/dependabot.yml`
- Kanban：`artifact: CI_CD_CONFIG`
- S5 发布门禁校验记录

## Verification

- 工作流语法正确（`actionlint` 通过）
- 推送触发正常执行
- 四阶段门禁在 CI 中同步运行
- 发布流程端到端验证通过
- S5 发布门禁校验 PASS

## Pitfalls

1. **不要在 CI 中硬编码密钥**：用 GitHub Secrets / OIDC
2. **不要跳过门禁**：CI 必须跑完整 `make pipeline`
3. **不要并行有依赖的阶段**：S1→S2→S3→S4 串行，S4 内部并行
4. **忘记数据库回滚脚本** → 必须双向迁移
5. **灰度策略缺失** → 必须有流量切分方案
6. **监控未更新** → 新版本无观测性
7. **版本号不规范** → 语义化版本是硬性要求

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
