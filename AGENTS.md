# AI-batch-raw-to-offline-guide — 维护索引

## 全局规则
- 架构（为什么）→ [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)
- 字段手册（改 Schema 前必读）→ [docs/schema-fields.md](docs/schema-fields.md)
- 生成器手册 → [generator/README.md](generator/README.md)
- 脚本手册 → [scripts/README.md](scripts/README.md)
- 决策记录 → [.agents/notes/](.agents/notes/)
- 双件分离：AGENTS.md 只写规则，README.md 只写是什么/怎么改

## 常用命令（uv 环境，真实可跑）

- 本地钩子：`pre-commit install`（每个 clone 一次；本体 `uv tool install pre-commit`）——提交前自动 `ruff check --fix` + `ruff format`；CI 只读跑同一组检查（部署链仍在 static.yml）；全量跑 `pre-commit run --all-files`；临时跳过 `git commit --no-verify`；定义见 [.pre-commit-config.yaml](.pre-commit-config.yaml)
- uv run python scripts/schema_validator.py（数据验证）
- uv run python generator/schema_generator.py（生成 HTML）
- uv run python scripts/stats.py · analyze_data.py · export_xlsx.py
- uv run pytest（tests/ 冒烟测试）
- uv run ruff check . — Lint（ruff 默认规则集，列宽默认 88）
- uv run ruff format . — 格式化（`--check` 只看不改）

## 验证快照（2026-11-18 复跑）
- schema_validator: PASS（唯一ID 311 / 有效引用 779 / 双向链接 234）
- pytest: 2 passed / 0 failed
- Ruff: `check` 0 发现；`format --check` 全绿（全量格式化已落地）
- uv.lock registry: 11 个包全部 `https://pypi.org/simple`，无镜像残留

## 活跃坑
- output/ 被 .gitignore 排除，clone 后需先跑生成器
- Windows 控制台中文乱码不影响校验结果
- data/ 是运行真相，src/ 只是素材参考
- **改依赖后必查 uv.lock 的 registry**：本机全局 `%APPDATA%\uv\uv.toml` 配了清华镜像，会让 `uv lock` 把锁文件整片改写成镜像源（2026-09-27 d9c38de 就这么中招）。pyproject 已用 `[[tool.uv.index]]` 钉死官方源，改完仍要确认 `uv.lock` 里没有 `tsinghua` 字样，否则境外 CI 会 tcp 超时。细节见 [.agents/notes/2026-11-18-uv-lock-mirror-repoisoning.md](.agents/notes/2026-11-18-uv-lock-mirror-repoisoning.md)
- ~~`tools/screenshot` 的 Dependabot 告警提不了 PR（extract-zip / basic-ftp 上游无修复版）~~ **2026-11-18 已解除**：puppeteer 24→25 后 `@puppeteer/browsers` 3.x 改用 `modern-tar`，依赖树里 `extract-zip`、`basic-ftp`、`ip-address` 全部消失，`npm audit` 报 0 vulnerabilities。**不要再按旧结论跳过这两个告警**。
- `tools/screenshot` 现在要求 **Node >= 22.12.0**（puppeteer 25 是纯 ESM 包，`require()` 加载 ESM 需要 22.12+）。升级 Node 前先确认这一点，否则截图脚本会报模块加载错误。真机已验证：Node 24.18 下 `require('puppeteer')` 正常、截图产物正常（2.2 MB PNG），`headless: 'new'` 仍可用。

## 文档地图
- 用户教程 → [docs/usage.md](docs/usage.md)
- 产品设计 → [docs/design.md](docs/design.md)
- 技术实现 → [docs/workflow.md](docs/workflow.md)
- 测试 → [tests/README.md](tests/README.md)
- 验证报告 → [VERIFICATION_REPORT.md](VERIFICATION_REPORT.md)
- 变更日志 → [changelog.md](changelog.md)