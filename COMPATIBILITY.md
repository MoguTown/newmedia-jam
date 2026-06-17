# 跨平台兼容性指南

> Newmedia Jam 基于 [Agent Skills 开放标准](https://agentskills.io) 设计，可在 Claude、Codex、GitHub Copilot 等支持该标准的智能体平台上运行。本文件说明各平台上的适配策略和已知差异。

---

## 一、支持的平台

| 平台 | 状态 | 说明 |
|------|------|------|
| Claude (Desktop / Code) | ✅ 原生支持 | 所有功能完整可用 |
| OpenAI Codex CLI | ✅ 兼容 | 内部推理步骤以不同方式呈现，无网络时降级 |
| GitHub Copilot | ⚠️ 可运行 | 需手动加载 references，搜索能力有限 |
| 其他 Agent Skills 兼容平台 | ✅ 理论兼容 | 遵循标准 SKILL.md 格式 |

---

## 二、Claude 特有机制及跨平台适配

### 2.1 "不展示给用户"的内部推理

Claude 支持原生的"思维"步骤，可将某些检查步骤标记为不显示。在 Codex 等平台上，这些步骤的处理方式：

- **概念校准 / 内部推理**（Step 1）：在 Codex 上，可将推理结果以简短前缀的形式输出给用户（如"基于我的理解，这个选题的核心是……"），然后继续提问
- **张力审计**（Step 2）：在 Codex 上，简要告知用户"团队构成已做角色审计，具备以下碰撞潜力……"
- **强制检查 / 红线扫描**（Step 3）：在 Codex 上可简化为每轮末尾的 1-2 句快速自检
- **自检清单**（Step 4 / Step 6）：在 Codex 上可在输出末尾附加简短的"✓ 自检通过"标注，不逐条展示

**统一渲染规则**：如果平台不支持隐藏内部推理，将"不展示给用户"的内容用以下任一方式处理：
1. HTML 注释包裹：`<!-- 内部推理：张力审计通过 -->`
2. Obsidian callout：`> [!note] 内部检查：张力审计通过`
3. 极简标注：`✓ 张力审计通过`（附在回复末尾）

**关键原则**：逻辑必须执行，仅呈现方式因平台而异。

### 2.2 联网搜索能力

- **Claude**：内置 WebSearch / WebFetch
- **Codex**：需用户配置 MCP 或 web 工具，否则不可用
- **降级策略**（无网络环境下统一适用）：
  1. Step 1 跳过概念搜索，基于内置知识推理 → 标注"⚠️ 未联网，理解可能不够精准"
  2. Step 3 平台数据引用使用 `production-platforms.md` 的静态数据 → 标注"⚠️ 基于 2026-05 数据"
  3. Step 5 突发事件卡牌使用预设场景（预算砍半 / 时间压缩 / 选题转向……）
  4. Step 6 AIGC 工具验证跳过 → 统一标注"⚠️ 价格信息未联网验证"

### 2.3 references/ 文件加载

- **Claude**：通过 `Read` 工具在流程内按需加载
- **Codex**：SKILL.md 中的 `allowed-tools` 已声明 `Read` 权限；无 `Read` 工具时，依赖模型训练知识补位
- **降级策略**：异常处理表中已包含"参考文件读取失败"的降级方案

### 2.4 PKB 联动检测

Jam 在 Step 1 前自动检测 PKB 环境。检测逻辑使用多信号确认，确保跨平台兼容：

| 确认信号 | Claude | Codex | 说明 |
|----------|--------|-------|------|
| `CLAUDE.md` | ✅ 常见 | ❌ 不存在 | Claude Code 项目配置 |
| `wiki/` 目录 | ✅ 初始化后有 | ✅ 初始化后有 | 最通用的信号 |
| `PKB操作指南.md` | ✅ 初始化后有 | ✅ 初始化后有 | 初始化产物，跨平台有效 |
| `PKB-dashboard.md` | ✅ 初始化后有 | ✅ 初始化后有 | 初始化产物，跨平台有效 |

**Codex 用户注意**：由于 Codex 没有 `CLAUDE.md`，PKB 必须**先运行 `wiki维护` 完成初始化**，生成 `wiki/` 目录和 `PKB操作指南.md`，Jam 才能检测到。

**未初始化场景**：如果 Jam 检测到 `raw/` 但无任何确认信号，会温和提示用户先初始化 PKB，不会报错或阻断流程。

### 2.5 Sub-agent / fork 模式

newmedia-jam **未使用** Claude 特有的 sub-agent 或 fork 模式，因此不产生跨平台兼容问题。

---

## 三、平台特有的发现路径

各平台从不同目录发现 skills，如需在多个平台间共享本项目：

| 平台 | 安装路径 | 安装命令 |
|------|---------|---------|
| Claude | `~/.claude/skills/newmedia-jam/` | 复制或软链接到该目录 |
| Codex | `~/.agents/skills/newmedia-jam/` | `ln -s /path/to/newmedia-jam ~/.agents/skills/newmedia-jam` |
| 项目级 | `./.claude/skills/` 或 `./.agents/skills/` | 同上 |

**推荐方案**：将本项目放置在共享路径，然后软链接到各平台目录。

---

## 四、兼容性前端字段

SKILL.md frontmatter 已添加 Codex 兼容字段：

```yaml
compatibility: "需要文件读写权限。联网搜索用于选题调研..."
license: "MIT"
allowed-tools: "Read, Write, Grep, Glob, Bash, WebSearch, WebFetch"
```

- `compatibility`：Codex 读取并展示给用户
- `license`：符合 Codex 的必填/推荐字段
- `allowed-tools`：声明 skill 所需的工具权限，逗号分隔，与 Codex 标准格式一致

---

## 五、已知差异总结

| 功能 | Claude | Codex/其他 | 影响 |
|------|--------|-----------|------|
| 内部推理隐藏 | 原生支持 | 以简短输出替代 | 用户体验略有差异 |
| 联网搜索 | WebSearch 内置 | 需额外配置 | 无网络时降级 |
| bash/文件操作 | 全部可用 | 部分受限 | 需检查 allowed-tools 声明 |
| references 文件加载 | Read 工具按需加载 | Read 工具按需加载 | 无差异 |
| PKB 联动 | 自动检测 PKB 目录（多信号确认：CLAUDE.md / wiki/ / PKB操作指南.md / PKB-dashboard.md，任一满足即可） | 需 PKB 已初始化且可访问，无 Bash 时降级 | 低风险 |
| emoji 支持 | 完整渲染 | 大部分平台支持 | 低风险 |
| 中文内容 | 原生支持 | 多数平台支持 | 低风险 |

---

## 六、移植检查清单

在 Codex 或其他平台上首次使用前，验证以下项：

- [ ] SKILL.md 可被平台扫描识别（`name` 字段符合 `^[a-z0-9]+(-[a-z0-9]+)*$`）
- [ ] references/ 目录下的文件可被 `Read` 工具访问
- [ ] 联网搜索功能已配置或已接受降级策略
- [ ] emoji 渲染正常（⚠️ ✅ ❌ 🔴 🟡 🟢 等）
- [ ] NPC 角色名中文显示正常
- [ ] Markdown 表格渲染正常

---

> 📅 最后更新：2026-06-16 | 基于 Agent Skills 开放标准 v1.0
