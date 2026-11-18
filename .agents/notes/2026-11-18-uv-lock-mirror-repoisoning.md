# 决策：锁文件源由 pyproject 钉死，不再依赖「本机别配镜像」的自觉（2026-11-18）

状态：生效

## 问题

2026-08-29 的决策（[uv 包源切回官方](2026-08-29-uv-index-rollback.md)）要求 uv.lock 固定官方源，
但只靠「锁文件里写官方 URL」这一层，挡不住后续任何一次 `uv lock`。

- 本机全局 `%APPDATA%\uv\uv.toml` 配了 `index-url = https://pypi.tuna.tsinghua.edu.cn/simple/`
- 该全局配置的优先级高于锁文件里既有的 registry 记录，`uv lock` 会把 103 行 registry 整片改写成镜像
- 2026-09-27 `d9c38de`（引入 Ruff）就是这样把锁文件从官方源改回镜像的：提交信息只提 lint，
  没人注意到 uv.lock 里 84→103 行 URL 一起变了
- 后果不在本地暴露（本机连镜像很快），只在境外 CI 暴露：GitHub Actions 拉清华镜像 tcp 超时

2026-11-18 `50c2c86` 合并后 Pages 部署失败，即此因：

```
error: Failed to download `markupsafe==3.0.3` -> pypi.tuna.tsinghua.edu.cn ... deadline has elapsed
```

## 决策

- `pyproject.toml` 用 `[[tool.uv.index]]` 显式声明官方源并 `default = true`。
- 该声明属于项目配置，优先级高于用户级 `uv.toml`，因此 `uv lock` 不再被本机镜像改写。
- 判定标准：在配了镜像的机器上，不带任何参数跑 `uv lock`，`uv.lock` 里 `tsinghua` 出现次数必须为 0。

## 替代方案

- **只改锁文件、不改 pyproject**：已被现实否决——这就是 8 月到 11 月之间发生的事，
  锁文件是生成物，任何一次 relock 都会让手工修正失效
- **`[tool.uv] default-index = "..."`**：uv 0.11.32 不认这个字段，报 `unknown field` 并整体
  忽略 `[tool.uv]`，等于没写（实测），必须用 `[[tool.uv.index]]` 表
- **让每个人删掉本机 uv.toml 的 index-url**：属个人环境，换机器/换人即失效，不可作为仓库约束
- **CI 侧覆盖 `UV_DEFAULT_INDEX`**：能救 CI，但本地 relock 仍会污染锁文件，问题只被推迟

## 影响

- 收益：锁文件不再可能被本机镜像悄悄改写；CI 与本地解析同一组 URL，可复现
- 代价：本机下载依赖走官方源，速度低于镜像（如需加速，用 `uv sync` 时的命令行 `--default-index` 临时覆盖，不落盘）
- 关联：取代 [2026-08-29-uv-index-rollback.md](2026-08-29-uv-index-rollback.md) 中「仅靠锁文件固定源」的做法，目标不变、机制加强
