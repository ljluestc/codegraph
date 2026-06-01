# PR 标题（建议）
feat(installer): 新增 gptme 安装目标并自动写入 CodeGraph MCP 配置

## 关联 Issue
- Closes #382

## 背景
当前 `codegraph install --target=...` 已支持 Claude、Cursor、Codex、opencode、Hermes、Gemini、Antigravity、Kiro，但尚未覆盖 gptme。  
gptme 已支持 MCP（`[mcp]` / `[[mcp.servers]]`），因此用户需要手动编辑配置才能接入 `codegraph serve --mcp`，安装体验不一致，也提高了接入门槛。

## 问题拆解（对应 #382 提问）
1. `codegraph install --target=claude-code`（当前实现中的 `claude` 目标）写入路径在哪里？  
当前实现会写 MCP 配置到 `~/.claude.json`（global）或 `./.mcp.json`（local），权限在 `~/.claude/settings.json` 或 `./.claude/settings.json`；指引文件不再写入 `CLAUDE.md`，而是由 MCP `initialize` 返回统一指导信息。

2. MCP 配置路径是否与 OS 相关，还是固定为仓库根目录 `gptme.toml`？  
gptme 当前文档默认全局配置在 `~/.config/gptme/config.toml`。项目级可使用 `gptme.toml`（并支持 `gptme.local.toml` 分层覆盖）。因此安装器需明确 global/local 两种写入策略并做路径探测。

3. 如何落地 gptme install target？  
按现有 installer target 架构新增 `gptme` 目标，实现 `detect/install/uninstall/printConfig/describePaths`，并将其接入 `registry` 与 CLI `--target` 解析链路。

## 方案概述
新增 `gptme` 目标，复用现有 MCP server 标准配置：

```toml
[[mcp.servers]]
name = "codegraph"
command = "codegraph"
args = ["serve", "--mcp"]
```

并在安装时根据 location 写入：
- global: `~/.config/gptme/config.toml`
- local: `./gptme.toml`（若存在本地分层配置，则保留并仅更新 `codegraph` 对应 server 条目）

卸载时只移除 `codegraph` 对应 MCP server，保留用户其他 server 与其他配置项。

## 主要改动（计划）
1. Installer target 实现
   - 新增 `src/installer/targets/gptme.ts`
   - 实现 TOML 读写（保持幂等、尽量保留用户现有内容）
   - 支持 `global` + `local` 双 location

2. 注册与类型
   - 更新 `src/installer/targets/types.ts` 的 `TargetId`，增加 `gptme`
   - 更新 `src/installer/targets/registry.ts`，将 `gptmeTarget` 加入 `ALL_TARGETS`

3. 测试
   - 扩展 `__tests__/installer-targets.test.ts`
   - 覆盖 `detect/install/uninstall`、幂等重入、只删除 `codegraph` 条目不误删其他条目

4. 文档
   - 更新 `README.md` 和安装文档，补充 gptme 为受支持目标
   - 增加 gptme 手动配置片段示例

## 兼容性与风险
- 对现有目标（claude/cursor/codex/opencode/hermes/gemini/antigravity/kiro）无行为改动。  
- gptme 配置文件可能存在用户自定义结构；本改动按“最小侵入”策略，仅增删 `codegraph` MCP server 节点。  
- 若用户同时使用 `gptme.toml` 与 `gptme.local.toml`，安装器默认只管理主配置中的 `codegraph` 条目，不覆盖用户本地 secret 分层。

## 测试计划（建议）
- 单测：installer target 相关测试全量通过。  
- 手工验证：
  1) `codegraph install --target=gptme --location=global --yes`  
  2) `codegraph install --target=gptme --location=local --yes`  
  3) 重复执行 install，验证幂等（`unchanged`）  
  4) `codegraph uninstall --target=gptme --yes` 仅移除 `codegraph` server，不影响其他 server

## 预期用户收益
- gptme 用户可一键接入 CodeGraph MCP，无需手工改 TOML。  
- 降低首次接入成本，统一多 Agent 安装体验。  
- 通过 installer 保障配置形态一致，减少路径/格式错误导致的接入失败。

## Release Note
添加 gptme 安装目标，支持通过 `codegraph install --target=gptme` 自动配置 CodeGraph MCP。
