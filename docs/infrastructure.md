# 渊渟集团 AI 基础设施文档

> 本文档记录 OpenClaw 助手的技能配置与记忆系统基础设施，用于灾备恢复与团队知识传承。

---

## 更新时间

- **更新日期**: 2026-03-03
- **更新人**: 亏总 (AI 助理)
- **版本**: v1.0

---

## 一、已安装 Skills

当前工作区 (`/root/.openclaw/workspace`) 已安装以下技能插件：

| 技能名称 | 用途 | 状态 |
|---------|------|------|
| **exa-web-search-free** | AI 搜索引擎，支持网页/代码/公司研究搜索 | ✅ 已配置 |
| **find-skills** | 技能发现与搜索 | ✅ 已安装 |
| **github** | GitHub 操作集成 | ✅ 已安装 |
| **self-improving-agent** | 自我改进代理 | ✅ 已安装 |
| **skill-vetter** | 技能审核工具 | ✅ 已安装 |
| **summarize** | 内容摘要生成 | ✅ 已安装 |

### 1.1 Exa Web Search (Free) 配置

**安装命令**:
```bash
npm i -g mcporter@0.7.3
mcporter config add exa "https://mcp.exa.ai/mcp?exaApiKey=${EXA_API_KEY}&tools=web_search_exa,web_search_advanced_exa,people_search_exa"
```

**可用工具**:
- `web_search_exa` - 快速网页搜索
- `web_search_advanced_exa` - 高级搜索（支持域名/日期过滤）
- `people_search_exa` - 人员搜索

**配置位置**: `/root/.openclaw/workspace/config/mcporter.json`

---

## 二、记忆系统 (Mem0)

### 2.1 概述

采用 **Mem0 Platform** 作为长期记忆后端，替代默认的本地文件记忆系统。

**特点**:
- 云端托管，跨会话持久化
- 支持语义搜索与自动召回
- 自动捕获对话中的重要信息
- 支持用户级记忆隔离

### 2.2 配置详情

**插件**: `@mem0/openclaw-mem0@0.1.2`

**安装命令**:
```bash
openclaw plugins install @mem0/openclaw-mem0
```

**配置文件**: `~/.openclaw/openclaw.json`

```json5
{
  plugins: {
    allow: ["openclaw-mem0"],
    slots: {
      memory: "openclaw-mem0"
    },
    entries: {
      openclaw-mem0: {
        enabled: true,
        config: {
          apiKey: "${MEM0_API_KEY}",  // 从环境变量读取
          userId: "渊渟集团"
        }
      },
      memory-core: {
        enabled: false
      },
      memory-lancedb: {
        enabled: false
      }
    }
  }
}
```

### 2.3 核心参数

| 参数 | 值 | 说明 |
|------|-----|------|
| `mode` | `platform` | 使用 Mem0 托管云服务 |
| `userId` | `渊渟集团` | 记忆命名空间，区分不同用户/组织 |
| `autoRecall` | `true` | 自动回忆相关记忆 |
| `autoCapture` | `true` | 自动捕获新记忆 |
| `topK` | `5` | 每次召回最多 5 条记忆 |
| `searchThreshold` | `0.3` | 相似度阈值 (0-1) |
| `enableGraph` | `false` | 实体关系图（未启用） |

### 2.4 用户标识 (userId)

当前配置的 `userId` 为 **"渊渟集团"**。

**作用**:
- 所有记忆数据以此 ID 为命名空间存储
- 隔离不同组织/用户的记忆数据
- 便于多租户场景下的数据管理

**如需修改**:
编辑 `~/.openclaw/openclaw.json` 中 `plugins.entries.openclaw-mem0.config.userId` 字段，重启 Gateway 生效。

### 2.5 记忆类型

Mem0 自动管理两种记忆范围：

| 类型 | 标识 | 生命周期 | 用途 |
|------|------|---------|------|
| **Session (短期)** | `run_id` | 单次会话 | 当前对话上下文 |
| **User (长期)** | `user_id` | 永久 | 跨会话持久化知识 |

### 2.6 可用工具

安装后，AI 助手可在对话中使用以下记忆工具：

| 工具 | 功能 |
|------|------|
| `memory_search` | 语义搜索记忆 |
| `memory_list` | 列出所有记忆 |
| `memory_store` | 显式存储记忆 |
| `memory_get` | 获取特定记忆 |
| `memory_forget` | 删除记忆 |

### 2.7 CLI 命令

```bash
# 查看记忆统计
openclaw mem0 stats

# 搜索记忆（全部范围）
openclaw mem0 search "关键词"

# 搜索长期记忆
openclaw mem0 search "关键词" --scope long-term

# 搜索会话记忆
openclaw mem0 search "关键词" --scope session
```

### 2.8 API 端点

- **Base URL**: `https://api.mem0.ai`
- **Search**: `POST /v1/memories/search`
- **Add**: `POST /v1/memories/add`
- **认证**: `Authorization: Token <MEM0_API_KEY>`

---

## 三、环境信息

| 项目 | 值 |
|------|-----|
| OpenClaw 版本 | 2026.3.1 |
| Node.js 版本 | v22.22.0 |
| 操作系统 | Linux 6.8.0-87-generic (x64) |
| 工作区 | `/root/.openclaw/workspace` |
| Gateway 端口 | 18789 |
| Gateway 绑定 | 127.0.0.1 (loopback) |

---

## 四、备份与恢复

### 4.1 关键文件备份清单

```
/root/.openclaw/
├── openclaw.json              # 主配置（含 mem0 API key）
├── workspace/
│   ├── config/
│   │   └── mcporter.json      # Exa MCP 配置
│   ├── skills/                # 自定义 skills
│   ├── memory/                # 本地记忆文件（如使用本地模式）
│   └── MEMORY.md              # 长期记忆汇总
└── extensions/
    └── openclaw-mem0/         # 插件代码
```

### 4.2 恢复步骤

1. **安装 OpenClaw**
   ```bash
   npm install -g openclaw
   ```

2. **恢复配置文件**
   ```bash
   cp backup/openclaw.json ~/.openclaw/openclaw.json
   ```

3. **安装插件**
   ```bash
   openclaw plugins install @mem0/openclaw-mem0
   ```

4. **安装 Exa MCP**
   ```bash
   npm i -g mcporter
   mcporter config add exa "https://mcp.exa.ai/mcp?exaApiKey=YOUR_KEY&tools=web_search_exa,web_search_advanced_exa,people_search_exa"
   ```

5. **重启 Gateway**
   ```bash
   openclaw gateway restart
   ```

6. **验证**
   ```bash
   openclaw mem0 stats
   ```

---

## 五、注意事项

1. **API Key 安全**: Mem0 API key 存储在 `openclaw.json` 中，请妥善保管，勿提交到公共仓库
2. **网络依赖**: Mem0 Platform 模式需要外网访问 `api.mem0.ai`
3. **数据隔离**: 确保 `userId` 设置正确，避免与其他组织数据混淆
4. **版本兼容**: `@mem0/openclaw-mem0` 为早期版本，关注官方更新

---

## 六、相关链接

- [Mem0 官方文档](https://docs.mem0.ai)
- [OpenClaw 文档](https://docs.openclaw.ai)
- [Exa MCP 文档](https://docs.exa.ai)
- [插件 GitHub](https://github.com/mem0ai/openclaw-mem0) *(待确认)*

---

*本文档由 亏总 🎲 自动生成于 2026-03-03*
