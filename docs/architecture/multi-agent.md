# 渊渟集团多‑Agent 架构（v2 - 实施版）

> 基于实际实施经验的多 Agent 协作系统架构文档

---

## 更新记录

- **v2.0** (2026-03-04): 基于实际实施经验大幅更新，添加双通道通信方案、Session 隔离详解、故障排查
- **v1.0** (2026-03-03): 初始架构设计

---

## 一、角色与职责映射（已实施）

| 角色 | AgentId | 人设定位 | 绝对权力 | Discord ID | 主要职责 |
|------|---------|----------|----------|-----------|----------|
| 亏总 🎲 | `kuizong` | 统筹/流程/基础设施 | 流程优化权 | 1478290704684814390 | 任务拆解、复盘、架构维护 |
| 摸总 🐱 | `mozong` | 产品经理 | 需求决策权 | 1478295402779381894 | PRD、需求评审结论 |
| 混子 💻 | `hunzi` | 开发工程师 | 技术架构权 | 1478296740670083103 | 代码实现、技术方案 |
| 八阿哥 🐛👑 | `bage` | 测试工程师 | 质量门禁权 | 1478296381973069895 | 测试报告、上线门禁 |

> 备注：所有 Agent 必须在职责范围内输出，跨界任务需由 亏总 🎲 统一协调。

---

## 二、系统结构（实际部署）

### 2.1 整体架构

```
用户 (Discord)
    │
    ▼
┌─────────────────────────────────────────────┐
│           Discord 服务器                    │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐    │
│  │ #公告   │  │ #闲聊   │  │ #测试   │    │
│  └─────────┘  └─────────┘  └─────────┘    │
└─────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────┐
│        OpenClaw Gateway (单进程)            │
│  ┌─────────────────────────────────────┐   │
│  │          Session 管理器              │   │
│  │  • Session 路由                     │   │
│  │  • Session 隔离（关键！）           │   │
│  │  • 消息分发                         │   │
│  └─────────────────────────────────────┘   │
│                                             │
│  ┌─────────────────────────────────────┐   │
│  │        多 Agent 层 (4个角色)         │   │
│  │  ┌─────────┐ ┌─────────┐           │   │
│  │  │ kuizong │ │ mozong  │           │   │
│  │  │ (亏总)  │ │ (摸总)  │           │   │
│  │  └─────────┘ └─────────┘           │   │
│  │  ┌─────────┐ ┌─────────┐           │   │
│  │  │  hunzi  │ │  bage   │           │   │
│  │  │ (混子)  │ │ (八阿哥) │           │   │
│  │  └─────────┘ └─────────┘           │   │
│  └─────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
```

### 2.2 Session 隔离（核心设计）

OpenClaw 的核心设计是 **Session 完全隔离**。

**Session 类型**:

```
Session 类型:
├── Main Session: agent:{agentId}:main
│   └── 用于直接消息和内部通信（但其他 Agent 不一定在监听！）
├── Discord DM: agent:{agentId}:discord:direct:{userId}
│   └── 用于 Discord 私聊
├── Discord Channel: agent:{agentId}:discord:channel:{channelId}
│   └── 用于 Discord 频道（本项目主要使用）
└── Group/Other: agent:{agentId}:{channel}:group:{groupId}
    └── 用于群组聊天
```

**关键理解**：
- 每个 Session 是独立的"沙盒"
- Agent 只监听**当前活跃**的 Session
- `sessions_send` 只能发送到**存在的** Session
- 发送到 `agent:xxx:main` 时，如果对方在 Discord 频道，是收不到的！

---

## 三、双通道通信方案（实施经验）

### 3.1 为什么要双通道？

**场景分析**：

```
场景1: 摸总发送任务给八阿哥
┌──────────────┐          ┌──────────────┐
│   摸总        │          │   八阿哥      │
│  (Discord    │          │  (在线？      │
│   Channel)   │          │   在 Main？)  │
└──────────────┘          └──────────────┘
       │                         │
       │ sessions_send           │
       │ to agent:bage:main      │
       ▼                         ▼
┌─────────────────────────────────────────┐
│  ❌ 失败！八阿哥在 Discord Channel，      │
│     不在 main session，收不到消息        │
└─────────────────────────────────────────┘

场景2: 使用混合方案（双通道）
┌──────────────┐          ┌──────────────┐
│   摸总        │          │   八阿哥      │
│  (Discord    │          │  (Discord    │
│   Channel)   │          │   Channel)   │
└──────────────┘          └──────────────┘
       │                         ▲
       │                         │
       ├───── sessions_send ─────┤
       │   to Discord channel    │
       │   (如果对方在线立即处理) │
       │                         │
       ├────── Discord @ ────────┤
       │   (通知必定送达)        │
       │                         │
       └────── 双保险 ───────────┘

✅ 成功！无论八阿哥在哪个 session，都能收到通知
```

### 3.2 双通道设计

#### 通道1：内部通道 (sessions_send)

```json
{
  "tool": "sessions_send",
  "sessionKey": "agent:bage:discord:channel:1478694700176248862",
  "message": "任务内容...",
  "timeoutSeconds": 10
}
```

**优点**：
- 如果对方在线，立即处理
- 支持任务详情和上下文
- 可以等待响应

**缺点**：
- 必须知道对方当前在哪个 session
- 如果对方不在对应 session，收不到
- 可能超时

#### 通道2：外部通道 (Discord message)

```json
{
  "tool": "message",
  "channel": "discord",
  "content": "<@1478296381973069895> 任务通知..."
}
```

**优点**：
- 必定送达（通过 Discord 服务器）
- 支持 @mention 高亮
- 老板可以看到完整流程
- 不受 session 隔离限制

**缺点**：
- 只是通知，不包含完整任务详情
- 对方可能延迟处理

#### 为什么两者都需要？

| 需求 | sessions_send | Discord message | 结论 |
|------|---------------|-----------------|------|
| 即时处理 | ✅ 如果对方在线 | ❌ 只是通知 | 都需要 |
| 必定送达 | ❌ 受限于 session | ✅ 必定送达 | 都需要 |
| 任务详情 | ✅ 完整上下文 | ❌ 简略通知 | 都需要 |
| 老板可见 | ❌ 内部通信 | ✅ 公开可见 | 都需要 |
| **结论** | **不可少** | **不可少** | **缺一不可** |

### 3.3 混合通信方案（三步保险）

**完整流程**：

```javascript
// 第1步：查找目标 Session
const sessions = await sessions_list({
  kinds: ["group"],
  messageLimit: 5
});
// → 找到八阿哥的 session: agent:bage:discord:channel:1478694700176248862

// 第2步：内部通信（并行执行）
await sessions_send({
  sessionKey: "agent:bage:discord:channel:1478694700176248862",
  message: "任务：FocusPaw PRD 已批准，请编写测试用例。",
  timeoutSeconds: 10  // 短超时，避免阻塞
});

// 第3步：外部通知（必须执行，与第2步并行）
await message({
  channel: "discord",
  content: "<@1227127547477753967> 📨 任务流转\n摸总 → <@1478296381973069895>\n任务：FocusPaw PRD 已批准\n状态：已通知"
});
```

**关键点**：
1. **动态查找 Session**：不写死 main，查找对方当前 channel session
2. **并行执行**：sessions_send 和 Discord message 同时发送
3. **短超时**：10 秒超时，避免阻塞 workflow
4. **双保险**：即使内部通道失败，Discord 也能送达

---

## 四、实施经验与最佳实践

### 4.1 Session Key 格式速查

```
通用格式：agent:{agentId}:discord:channel:{channelId}

亏总:   agent:kuizong:discord:channel:1478694700176248862
摸总:   agent:mozong:discord:channel:1478694700176248862
混子:   agent:hunzi:discord:channel:1478694700176248862
八阿哥: agent:bage:discord:channel:1478694700176248862

注意：channelId 根据实际频道变化
```

### 4.2 Discord ID 速查

```
老板:   <@1227127547477753967>
亏总:   <@1478290704684814390>
摸总:   <@1478295402779381894>
混子:   <@1478296740670083103>
八阿哥: <@1478296381973069895>
```

### 4.3 消息发送 checklist

```
发送任务前检查：
☐ 已使用 sessions_list 查找对方 session
☐ 已获取正确的 sessionKey（Discord channel，不是 main）
☐ 已设置 timeoutSeconds: 10（短超时）
☐ 已并行发送 sessions_send 和 Discord message
☐ Discord 消息中已 @目标 agent
☐ Discord 消息中已 @老板（waylon）
☐ 消息内容清晰，包含任务详情
```

### 4.4 工作流实际运行情况

**已验证的流程**：

```
✅ PRD 撰写 → 审批
   摸总撰写 PRD → 老板审批 → 摸总通知八阿哥

✅ 测试用例编写 → 评审
   八阿哥编写测试用例 → 混子评审 → 八阿哥通知混子开发

✅ 开发 → 提交流程
   混子开发完成 → 混子通知八阿哥测试

🔄 待验证的流程：
   测试验收 → 产品验收 → 复盘
```

---

## 五、故障排查（实际遇到的问题）

### 问题1："No session found with label: xxx"

**症状**：
```
错误：No session found with label: bage
```

**原因**：使用了不支持的 `"label"` 参数

**解决方案**：
```json
// ❌ 错误
{ "label": "bage", "message": "..." }

// ✅ 正确
{ "sessionKey": "agent:bage:discord:channel:1478694700176248862", "message": "..." }
```

### 问题2：消息发送到 main session，但对方在 Discord 频道

**症状**：对方收不到消息

**原因**：session 隔离，对方不在 main session

**解决方案**：
```javascript
// ❌ 错误：发送到 main
{ "sessionKey": "agent:bage:main" }

// ✅ 正确：发送到 Discord channel
{ "sessionKey": "agent:bage:discord:channel:1478694700176248862" }
```

### 问题3："Session send visibility is restricted"

**症状**：
```
错误：Session send visibility is restricted. 
      Set tools.sessions.visibility=all to allow cross-agent access.
```

**原因**：没有配置 `tools.sessions.visibility`

**解决方案**：
```json
{
  "tools": {
    "sessions": {
      "visibility": "all"  // 添加这行
    }
  }
}
```

### 问题4：workflow 中断，对方没响应

**症状**：任务发送后，对方没有回复

**可能原因**：
1. 只发了 `sessions_send`，没发 Discord 消息
2. 对方不在线或不在对应 session
3. Discord 配置错误

**排查步骤**：
1. 检查是否使用了双通道（sessions_send + Discord）
2. 检查 Discord 消息是否正确发送
3. 检查对方是否被正确 @mention
4. 查看 Gateway 日志

---

## 六、配置文件示例

### 6.1 openclaw.json（核心配置）

```json
{
  "agents": {
    "defaults": {
      "provider": "kimi-coding",
      "model": "k2p5"
    },
    "list": [
      {
        "id": "kuizong",
        "name": "kuizong",
        "workspace": "/root/.openclaw/agents/kuizong"
      },
      {
        "id": "mozong",
        "name": "mozong",
        "workspace": "/root/.openclaw/agents/mozong"
      },
      {
        "id": "hunzi",
        "name": "hunzi",
        "workspace": "/root/.openclaw/agents/hunzi"
      },
      {
        "id": "bage",
        "name": "bage",
        "workspace": "/root/.openclaw/agents/bage"
      }
    ]
  },
  
  "channels": {
    "discord": {
      "enabled": true,
      "allowBots": true,
      "groupPolicy": "allowlist",
      "guilds": {
        "1478291192700604426": {
          "users": ["1227127547477753967"],
          "channels": {
            "1478623271393038356": { "allow": true, "requireMention": true },
            "1478623331551805572": { "allow": true, "requireMention": false },
            "1478623393992675338": { "allow": true, "requireMention": true },
            "1478623533474119791": { "allow": true, "requireMention": true },
            "1478694700176248862": { "allow": true, "requireMention": true }
          }
        }
      }
    }
  },
  
  "tools": {
    "profile": "full",
    "sessions": {
      "visibility": "all"
    },
    "agentToAgent": {
      "enabled": true,
      "allow": ["main", "kuizong", "mozong", "hunzi", "bage"]
    }
  },
  
  "bindings": [
    { "agentId": "kuizong", "match": { "channel": "discord", "accountId": "kuizong" } },
    { "agentId": "mozong", "match": { "channel": "discord", "accountId": "mozong" } },
    { "agentId": "hunzi", "match": { "channel": "discord", "accountId": "hunzi" } },
    { "agentId": "bage", "match": { "channel": "discord", "accountId": "bage" } }
  ]
}
```

### 6.2 Agent 配置模板（AGENTS.md）

每个 Agent 的 `AGENTS.md` 关键章节：

```markdown
### 🚨 启动检查（每次启动必须执行）

**检查清单：**
- [ ] 已阅读 IDENTITY.md 中的"⚠️ 工作流强制规则"
- [ ] 已确认 Discord mention 格式：使用 `<@ID>` 而非 `@名字`
- [ ] 已确认 workflow 双通道：先 `sessions_send`，再 Discord 消息

**记忆口诀：**
> **"先查 session，再双发，缺一不可！"**
```

```markdown
## 工作流通信协议（新）- 混合方案

### 🚀 混合通信方案（三步保险）

**第1步：查找对方在当前频道的 session**
```json
{
  "tool": "sessions_list",
  "kinds": ["group"],
  "messageLimit": 5
}
```

**第2步：内部通信（sessions_send）【并行执行】**
```json
{
  "tool": "sessions_send",
  "sessionKey": "agent:target:discord:channel:1478694700176248862",
  "message": "任务内容...",
  "timeoutSeconds": 10
}
```

**第3步：外部通知（Discord message）【必须执行，与第2步并行】**
```json
{
  "tool": "message",
  "channel": "discord",
  "content": "<@1227127547477753967> 📨 任务流转..."
}
```
```

---

## 七、验收标准（已达成）

- [x] 需求/技术/测试/统筹各自独立对话与记忆
- [x] 任务拆解明确、无跨角色抢活
- [x] PRD‑开发‑测试‑复盘闭环可重复执行
- [x] 复盘结论落入 MEMORY.md
- [x] 双通道通信方案验证通过
- [x] Session 隔离与跨 session 通信验证通过
- [x] 完整架构文档发布

---

## 八、相关文档

- [完整架构文档](../../ARCHITECTURE.md) - 包含详细的双通道通信原理、时序图、故障排查
- [基础设施配置](../infrastructure.md) - OpenClaw 配置详情
- [团队信息](../team.md) - 角色定义与 Discord ID
- [FocusPaw 项目](../projects/FocusPaw.md) - 工作流验证项目

---

*维护人：韦拙（亏总）🎲*  
*最后更新：2026-03-04*