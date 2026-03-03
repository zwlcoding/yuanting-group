# 渊渟集团 灾备恢复手册

> 本文档用于服务器损坏或环境迁移时的快速恢复。按照步骤执行可完全恢复 AI 基础设施运转。

---

## 恢复目标

- 恢复 OpenClaw 运行环境
- 恢复所有 Skills（9个）
- 恢复 Mem0 记忆系统连接
- 恢复团队工作流能力

---

## 前置要求

- Linux/macOS 系统（推荐 Ubuntu 22.04+）
- Node.js v22+ 
- Git
- 网络连接（可访问 npm、GitHub、api.mem0.ai）
- 备份的 API Key（Mem0）

---

## 恢复步骤

### 步骤 1：安装 OpenClaw

```bash
# 安装 OpenClaw
npm install -g openclaw

# 验证安装
openclaw --version
```

### 步骤 2：克隆仓库

```bash
# 克隆渊渟集团仓库
git clone https://github.com/zwlcoding/yuanting-group.git

# 进入工作区目录（或复制到 ~/.openclaw/workspace）
cd yuanting-group
```

### 步骤 3：恢复主配置

```bash
# 创建 OpenClaw 配置目录
mkdir -p ~/.openclaw

# 从仓库复制 infrastructure.md 中的配置内容
# 创建 ~/.openclaw/openclaw.json，内容如下：
```

**~/.openclaw/openclaw.json**（关键配置）：

```json5
{
  meta: {
    lastTouchedVersion: "2026.3.1"
  },
  auth: {
    profiles: {
      "kimi-coding:default": {
        provider: "kimi-coding",
        mode: "api_key"
      },
      "github-copilot:github": {
        provider: "github-copilot",
        mode: "token"
      }
    }
  },
  agents: {
    defaults: {
      model: {
        primary: "github-copilot/gpt-5.2-codex"
      },
      workspace: "/root/.openclaw/workspace"
    }
  },
  plugins: {
    allow: ["openclaw-mem0"],
    slots: {
      memory: "openclaw-mem0"
    },
    entries: {
      openclaw-mem0: {
        enabled: true,
        config: {
          apiKey: "${MEM0_API_KEY}",  // 需要设置环境变量
          userId: "渊渟集团"
        }
      }
    }
  }
}
```

### 步骤 4：设置环境变量

```bash
# 设置 Mem0 API Key（从安全备份获取）
export MEM0_API_KEY="你的-mem0-api-key"

# 添加到 ~/.bashrc 或 ~/.zshrc 使其永久生效
echo 'export MEM0_API_KEY="你的-mem0-api-key"' >> ~/.bashrc
```

### 步骤 5：安装 Mem0 插件

```bash
# 安装 Mem0 插件
openclaw plugins install @mem0/openclaw-mem0

# 验证插件状态
openclaw mem0 stats
```

### 步骤 6：安装 Exa MCP

```bash
# 安装 mcporter
npm i -g mcporter@0.7.3

# 配置 Exa（免费版，无需 API key）
mcporter config add exa "https://mcp.exa.ai/mcp?tools=web_search_exa,web_search_advanced_exa,people_search_exa"

# 验证
mcporter list
```

### 步骤 7：安装其他 Skills

```bash
# 从仓库复制 skills 到工作区
cp -r yuanting-group/skills/* ~/.openclaw/workspace/skills/

# 或使用仓库作为工作区
cd yuanting-group
openclaw workspace use $(pwd)
```

### 步骤 7.1：安装 agent-browser CLI

```bash
# 安装 CLI
npm install -g agent-browser

# 安装浏览器运行时（推荐含依赖）
agent-browser install --with-deps
```

### 步骤 7.2：安装 summarize CLI

```bash
npm install -g summarize
```

### 步骤 8：重启 Gateway

```bash
# 重启 Gateway 加载插件
openclaw gateway restart

# 验证状态
openclaw gateway status
openclaw mem0 stats
```

### 步骤 9：验证恢复

```bash
# 测试 Mem0 连接
openclaw mem0 search "测试" --scope long-term

# 测试 Exa 搜索
mcporter call 'exa.web_search_exa(query: "渊渟集团", numResults: 3)'
```

---

## 关键文件清单

恢复后确认以下文件存在：

```
~/.openclaw/
├── openclaw.json              # 主配置文件 ✓
├── workspace/                 # 工作区
│   ├── skills/                # 技能目录
│   │   ├── exa-web-search-free/
│   │   ├── healthcheck/
│   │   ├── tmux/
│   │   └── ...
│   ├── memory/                # 本地记忆文件
│   └── MEMORY.md              # 长期记忆
└── extensions/
    └── openclaw-mem0/         # Mem0 插件
```

---

## 备份要点

### 必须备份的内容

1. **~/.openclaw/openclaw.json**
   - 包含所有配置和 API key 引用
   - 每次修改后需提交到仓库

2. **yuanting-group/ 仓库**
   - 包含 docs/（团队、项目、基础设施文档）
   - 包含技能配置

3. **Mem0 API Key**
   - 单独安全存储
   - 可从 Mem0 Dashboard 重新生成

### 可选备份

- `~/.openclaw/workspace/memory/`（本地记忆，如使用本地模式）
- `~/.openclaw/agents/`（会话历史）

---

## 故障排查

### Gateway 无法启动

```bash
# 检查日志
tail -f /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log

# 重置 Gateway
openclaw gateway stop
openclaw gateway start
```

### Mem0 连接失败

```bash
# 检查 API Key
openclaw mem0 stats

# 测试连通性
curl -X POST https://api.mem0.ai/v1/memories/search \
  -H "Authorization: Token $MEM0_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query":"ping","user_id":"渊渟集团"}'
```

### 技能未加载

```bash
# 列出已安装技能
ls ~/.openclaw/workspace/skills/

# 手动复制
openclaw skills install /path/to/skill
```

---

## 恢复验证清单

- [ ] OpenClaw 版本正确（2026.3.1+）
- [ ] Gateway 运行正常
- [ ] Mem0 插件已加载
- [ ] Mem0 连接正常（`openclaw mem0 stats`）
- [ ] Exa MCP 可用（`mcporter list`）
- [ ] 所有 9 个技能可用
- [ ] agent-browser CLI 可用（`agent-browser --version`）
- [ ] Chromium runtime 可用（`agent-browser open https://example.com`）
- [ ] summarize CLI 可用（`summarize "https://example.com" --length short`）
- [ ] 可以搜索记忆
- [ ] 可以存储新记忆

---

## 联系与支持

如有恢复问题：
1. 检查本手册的故障排查部分
2. 查阅 [OpenClaw 文档](https://docs.openclaw.ai)
3. 查阅 [Mem0 文档](https://docs.mem0.ai)

---

**最后更新**: 2026-03-03  
**维护者**: 韦拙 (亏总) 🎲  
**版本**: v1.0
