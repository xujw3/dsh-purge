# dsh-purge 适配 DSH 0.2.2 记录（1.1.72）

> 目标宿主：**DSH 0.2.2-rc.1**（npm `latest`）。仓库上一版目标为 0.2.0-rc.2。
> 0.2.2-rc.2（npm `next`）与 rc.1 有 29/55 个被补丁的文件不同，本版**未逐项复测**；以 rc.1 为准。
> 本文件记录「为什么改、改在哪、怎么验证」，供下次升级宿主时复用。

## 1. 关键破坏点与修复

| # | 0.2.2 的变化 | 对插件的影响 | 修复 |
| --- | --- | --- | --- |
| 1 | `system-prompt` 不再注册 `{{cwd}}` 变量（工作目录改由 PromptContext `working-directory:current` 提供），`renderPrompt` 对未知变量**严格抛错** | 插件的 `cordis.patch.yml` `personaSuffix: Your working directory is {{cwd}}.` 与 identity 改写会在**每轮发请求前**抛 `UNKNOWN: unknown prompt variable "{{cwd}}"` | `lib/identity.js` 新增 `resolveWorkingDirectory()` / `installCwdPromptVariable()`；动态段落 `interpolate:false`；`overlayOfficialIdentity` 用真实目录；`cordis.patch.yml` personaSuffix 置空；`lib/index.js` 的 `dsh-purge:post` 也 `interpolate:false` |
| 2 | `dsh-headless` 自己覆盖 `sandbox-policy.mode` 默认 `workspace-write` | 补丁 #7/#8 只改 `dsh-base`：base 变 `danger-full-access`+`never`，headless 仍是 `workspace-write` → `permission: composed sandbox and approval defaults match no preset`，patched headless 起不来 | #7 `file` 扩为 `["base","headless"]`（#8 无需第二目标，headless 不覆盖审批） |
| 3 | 升级自愈 `refreshStaleBaksForUpgrade()` 在第二次 apply 时把**不含 `dsh-purge` 字样的补丁产物**当成官方新版，覆盖 `.dshpurge.bak` | 点两次「应用」后，revert 无法还原 8 个文件 | 记录补丁产物 sha256（vault meta `patchedHash`），自愈时命中即跳过；revert 后清理记录 |
| 4 | 5 处宿主代码搬迁 | 旧 pattern 在 0.2.2 上静默跳过：#32/#53 → `dsh-harness-dev-context`；#54 → `dsh-deliverable-file-references`；#60 → `dsh-tool-executor-ptc`；#62 → `dsh-tools/lib/pipeline.js`（+ types） | `targetFiles()` 增加新键（`nestedPkg` 兼容 npm 嵌套/扁平布局）；按 `analysis/stale-patches.md` 的新锚点重锚 |
| 5 | 其余迁移/失效补丁 | #12 #14 #40 #67 #74 #87 #88 代码/文案变化；#17 `editIntent`/`deleteIntent` 移到共享 `requirePresent`；#77 锚点失配（0.2.2 在 `this.revised.clear()` 前加了 `if (this.revised.size > 0)` 守卫） | 逐条重锚（#74+#87 合一）；#77 按新锚点重锚并应用（复用 #56 注入的 `isRewindMarker`，带 `typeof` 守卫）；#71（梁神预设不在宿主）/ #80（旧版追加工具的空补丁）已从补丁表删除 |
| 6 | 演练台/子代理契约变化 | `dsh-tool-subagent-control/list-agents` 子路径删除；`toolFilter`/`backgroundMode`/`modelSelectionSettings` 移除；预设需 `time-context`；`SubagentCatalogEntry` 去掉 `kind`/`activity`；`agentPresets.standingKeyFor()` 移除；`registry.register` 变 async | 见 `presets/redteam/agent.cordis.yml`、`lib/redteam/index.js`、`lib/redteam/tools.js`：新增 `redteam-role` 子预设（`agentPreset` + `maxDepth:1`）、旧预设启动自愈（先写 `.bak`）、`listChildren` 新形状、`acquireScope` |
| 7 | Web client 契约 | `dsh-client-runtime` / `dsh-client-ui-slots` 不再是可安装包；bundle 协议/槽位本身兼容 | `package.json` `dsh.client.inject` 改为 `@deepseek-ai/dsh-client-ui-renderer` + `@deepseek-ai/dsh-client-locale`；`lib/redteam/client.js` 源漂移修正（否则重建会把演练台 UI 回退，`client.js` 字节不变） |
| 8 | 其它小 API | `Session.projectionValues` 删除；`ReactLoopInbox.hasPending` 变 getter | `identity.js` 用 `ctx.sessionProjections.snapshot()`；`continue-retry.js` 兼容 getter |

## 2. 元数据

- `package.json`：`version 1.1.72`、`dshTarget: "0.2.2-rc.1"`、`engines.dsh: "^0.2.2-rc.1"`、`engines.node: ">=24.9.0"`、`peerDependencies."@deepseek-ai/cordis": "~4.1.0"`、`dsh.client.inject` 更新。
- README（中文 / en / zh-CN 副本）：版本号、DSH 徽章、Node 要求、适配范围。
- `release-notes.md` 顶部新增 `# 1.1.72`（发布 CI 要求首行等于 package 版本）。

## 3. 验证（全部在隔离环境，未改动真实安装）

- **补丁层（Harness A，14/14 断言通过）**：官方 0.2.2-rc.1 的完整副本 → `applyPatches` 两遍（第二遍必须 0 变更）→ `revertAll` 后逐文件 SHA-256 必须等于官方原文；`node --check` 全部改动 JS；额外校验「第二次 apply 后官方 `.dshpurge.bak` 仍未被污染」。84 个 patch 中 73 应用、11 na（未装 hooks / 桌面壳 / 可选预设）、0 missing_file、0 pending/unmatched；清洗面板 67/67。
- **运行时层（Harness B + C，8/8 断言通过）**：scratch `$DSH_HOME`（`profiles/web` + `profiles/headless`，用 junction 挂本仓库，**不安装**到用户环境）：
  - 未打补丁宿主：`dsh --profile headless "say hi"` 修复前 = `UNKNOWN: unknown prompt variable "{{cwd}}"`，修复后 = 只到 `MISSING_CREDENTIAL`（装配/渲染/预步骤全部通过）；
  - 打过补丁的宿主副本：headless + web 启动 stderr 为空/无 `permission:`、无 `did not activate`（除 headless 本就没有 webServer 导致的 redteam-ui 等待，属预期）；`GET /dsh-purge/status` 200（ready、patches_applied 67 / pending 0）、`POST /redteam/api` 200；
  - **mock Anthropic-Messages 请求捕获**：未打补丁 2758B/24 tools → 打补丁 47318B/27 tools；注入正文在 system 开头，prompt-inject seed 的 147/147 条特征行逐字出现在请求里；无字面 `{{...}}` 残留；`purge_apply/revert/status` 已注册；工具描述中性化生效。
- **演练台**：scratch home 中 `redteam/preset-declare.json` → `phase: registered`，`redteam` / `redteam-role` 均 `broken: null`；56/56 redteam 工具符合 0.2.2 `ToolDefinition`；23 个技能可见；并发闸门按新 `listChildren` 形状正确计数。
- **构建可复现**：`node scripts/patch-client-dock.mjs && node scripts/build-client.mjs` 两次输出字节一致，且与提交的 `client.js` 相同。

## 4. 残余风险

- 官方桌面 EXE（app.asar 解包/重启路径）本机未安装，**未实测**；相关补丁按旧锚点保留。
- 0.2.2-rc.2（next）未逐项复测：29/55 个被补丁文件不同，部分补丁会软跳过。
- 未使用真实模型凭据跑完整越狱对话；以 mock Anthropic-Messages 捕获的请求（system prompt / tools）做前后对比。
- `redteam-role` 会出现在 Web 模式下拉里（0.2.2 预设注册表没有 hidden 标记）。
- 0.2.2 中 `agent/request` 已 deprecated：anchor-gate 只改 `maxTokens`，仍可用，但未来宿主可能移除。
