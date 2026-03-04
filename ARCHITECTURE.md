# 渊渟集团多 Agent 协作系统架构文档

## 概述

本文档详细描述渊渟集团（Yuanting Group）的多 Agent 协作系统架构，包括系统设计原理、通信机制、工作流实现及最佳实践。

---

## 目录

1. [系统架构概览](#系统架构概览)
2. [核心设计原则](#核心设计原则)
3. [Agent 角色定义](#agent-角色定义)
4. [通信机制详解](#通信机制详解)
5. [工作流设计](#工作流设计)
6. [技术实现细节](#技术实现细节)
7. [配置说明](#配置说明)
8. [故障排查指南](#故障排查指南)
9. [最佳实践](#最佳实践)

---

## 系统架构概览

### 架构设计图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           渊渟集团多 Agent 协作系统                            │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                              用户交互层 (Discord)                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │   #公告      │  │   #闲聊      │  │  #测试工作流  │  │ #复盘汇总    │        │
│  │  (require    │  │  (require   │  │  (require   │  │  (require   │        │
│  │   Mention)   │  │   Mention)  │  │   Mention)  │  │   Mention)  │        │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘        │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          OpenClaw Gateway 路由层                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                        Session 管理器                                   │ │
│  │  ┌────────────────┐ ┌────────────────┐ ┌────────────────┐              │ │
│  │  │  Session 路由   │ │  Session 隔离   │ │  Session 状态   │              │ │
│  │  └────────────────┘ └────────────────┘ └────────────────┘              │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                      │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                      消息路由与分发                                     │ │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                    │ │
│  │  │  @mention   │  │  关键词触发   │  │  工作流推进   │                    │ │
│  │  │   过滤器    │  │   处理器     │  │   协调器     │                    │ │
│  │  └─────────────┘  └─────────────┘  └─────────────┘                    │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Agent 层 (4个角色)                               │
│                                                                             │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐          │
│  │   亏总 🎲        │    │   摸总 🐱        │    │   混子 💻        │          │
│  │  (战略总监)      │    │  (产品经理)      │    │  (开发工程师)    │          │
│  │                 │    │                 │    │                 │          │
│  │ • 流程优化       │    │ • 需求分析       │    │ • 技术决策       │          │
│  │ • 复盘组织       │    │ • PRD撰写       │    │ • 代码实现       │          │
│  │ • 争议仲裁       │    │ • 产品验收       │    │ • 架构设计       │          │
│  │                 │    │                 │    │                 │          │
│  │ Session:        │    │ Session:        │    │ Session:        │          │
│  │ agent:kuizong:* │    │ agent:mozong:*  │    │ agent:hunzi:*   │          │
│  └─────────────────┘    └─────────────────┘    └─────────────────┘          │
│                                                                             │
│  ┌─────────────────┐                                                        │
│  │   八阿哥 🐛👑    │                                                        │
│  │  (测试工程师)    │                                                        │
│  │                 │                                                        │
│  │ • 质量决策       │                                                        │
│  │ • 测试策略       │                                                        │
│  │ • Bug管理       │                                                        │
│  │                 │                                                        │
│  │ Session:        │                                                        │
│  │ agent:bage:*    │                                                        │
│  └─────────────────┘                                                        │
│                                                                             │
│  每个 Agent 独立的 workspace:                                                │
│  ~/.openclaw/agents/{agentId}/                                               │
│  ├── AGENTS.md          # 工作区配置                                         │
│  ├── IDENTITY.md        # 身份定义                                           │
│  ├── SOUL.md            # 核心性格                                           │
│  ├── USER.md            # 用户信息                                           │
│  ├── DISCORD_FORMAT.md  # Discord 格式规范                                   │
│  ├── TASK_CHECKLIST.md  # 任务检查清单                                       │
│  └── workspace/         # 工作目录                                           │
│      └── shared-projects/  # 共享项目                                         │
│          └── FocusPaw/     # 具体项目                                         │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           通信协议层 (双通道设计)                               │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                        内部通道 (sessions_send)                       │  │
│  │  ┌────────────────────────────────────────────────────────────────┐  │  │
│  │  │  协议: OpenClaw Internal Protocol                              │  │  │
│  │  │  目标: agent:{target}:discord:channel:{channelId}              │  │  │
│  │  │  特点: • 直接发送到目标 agent 的 session                       │  │  │
│  │  │       • 如果对方在线，立即处理                                 │  │  │
│  │  │       • 支持 timeout 控制                                     │  │  │
│  │  │  限制: • 对方必须在对应 session 中才能收到                     │  │  │
│  │  │       • session 隔离设计导致跨 session 不可见                  │  │  │
│  │  └────────────────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                        外部通道 (Discord message)                     │  │
│  │  ┌────────────────────────────────────────────────────────────────┐  │  │
│  │  │  协议: Discord Bot API                                         │  │  │
│  │  │  目标: Discord 频道中的 @mention                                │  │  │
│  │  │  特点: • 通过 Discord 服务器中转                               │  │  │
│  │  │       • 无论对方在哪个 session 都能收到通知                    │  │  │
│  │  │       • 支持高亮提醒和通知推送                                 │  │  │
│  │  │       • 老板可以看到完整的工作流进度                           │  │  │
│  │  └────────────────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              配置管理层                                       │
│                                                                             │
│  openclaw.json                                                              │
│  ├── tools.profile: "full"          # 完整工具权限                           │
│  ├── tools.sessions.visibility: "all"  # 允许跨 session 访问                 │
│  ├── tools.agentToAgent.enabled: true  # 启用 agent 间通信                   │
│  └── channels.discord.*            # Discord 频道配置                        │
│                                                                             │
│  每个 Agent 的 AGENTS.md                                                    │
│  ├── 启动检查清单                                                           │
│  ├── 响应规则                                                              │
│  ├── 工作流通信协议 (混合方案)                                               │
│  └── 消息模板                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 核心设计原则

### 1. Session 隔离设计

OpenClaw 的核心设计是 **Session 完全隔离**。

```
Session 类型:
├── Main Session: agent:{agentId}:main
│   └── 用于直接消息和内部通信
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

### 2. 双通道通信原理

为什么需要双通道？

```
┌────────────────────────────────────────────────────────────┐
│                     消息发送场景分析                         │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  场景1: 摸总发送任务给八阿哥                                 │
│  ┌──────────────┐          ┌──────────────┐               │
│  │   摸总        │          │   八阿哥      │               │
│  │  (Discord    │          │  (在线？      │               │
│  │   Channel)   │          │   在 Main？)  │               │
│  └──────────────┘          └──────────────┘               │
│         │                         │                       │
│         │ sessions_send           │                       │
│         │ to agent:bage:main      │                       │
│         ▼                         ▼                       │
│  ┌─────────────────────────────────────────┐              │
│  │  ❌ 失败！八阿哥在 Discord Channel，      │              │
│  │     不在 main session，收不到消息        │              │
│  └─────────────────────────────────────────┘              │
│                                                            │
│  场景2: 使用混合方案（双通道）                              │
│  ┌──────────────┐          ┌──────────────┐               │
│  │   摸总        │          │   八阿哥      │               │
│  │  (Discord    │          │  (Discord    │               │
│  │   Channel)   │          │   Channel)   │               │
│  └──────────────┘          └──────────────┘               │
│         │                         ▲                       │
│         │                         │                       │
│         ├───── sessions_send ─────┤                       │
│         │   to Discord channel    │                       │
│         │   (如果对方在线立即处理) │                       │
│         │                         │                       │
│         ├────── Discord @ ────────┤                       │
│         │   (通知必定送达)        │                       │
│         │                         │                       │
│         └────── 双保险 ───────────┘                       │
│                                                            │
│  ✅ 成功！无论八阿哥在哪个 session，都能收到通知            │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### 3. 双通道设计详解

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

---

## Agent 角色定义

### 角色职责矩阵

| 角色 | 花名 | Emoji | 真名 | 核心职责 | 绝对权力 | 协作对象 |
|------|------|-------|------|---------|---------|---------|
| 战略总监 | 亏总 | 🎲 | 韦拙 | 基础设施、复盘、流程优化 | 流程优化权 | 所有人（复盘阶段） |
| 产品经理 | 摸总 | 🐱 | 池闲 | 需求分析、PRD撰写、验收 | 需求决策权 | 八阿哥（需求传递）、混子（验收） |
| 开发工程师 | 混子 | 💻 | 程简 | 技术架构、开发实现 | 技术决策权 | 八阿哥（测试交接） |
| 测试工程师 | 八阿哥 | 🐛👑 | 沈慎 | 质量保证、测试策略 | 质量决策权 | 混子（测试反馈）、摸总（验收） |

### 各角色的 Session Key 格式

```
亏总:   agent:kuizong:discord:channel:{channelId}
摸总:   agent:mozong:discord:channel:{channelId}
混子:   agent:hunzi:discord:channel:{channelId}
八阿哥: agent:bage:discord:channel:{channelId}
```

### Discord User ID 映射

```yaml
boss:   1227127547477753967  # waylon
kuizong: 1478290704684814390  # 亏总
mozong:  1478295402779381894  # 摸总
hunzi:   1478296740670083103  # 混子
bage:    1478296381973069895  # 八阿哥
```

---

## 通信机制详解

### 混合通信方案（三步保险）

```
┌─────────────────────────────────────────────────────────────────┐
│                     消息发送流程（三步保险）                       │
└─────────────────────────────────────────────────────────────────┘

步骤1: 查找目标 Session
┌─────────────────────────────────────────┐
│ Tool: sessions_list                     │
│                                         │
│ Request:                                │
│ {                                       │
│   "kinds": ["group"],                   │
│   "messageLimit": 5                     │
│ }                                       │
│                                         │
│ Response:                               │
│ [                                       │
│   {                                     │
│     "key": "agent:bage:discord:channel: │
│             1478694700176248862",       │
│     "agentId": "bage",                  │
│     "channel": "discord",               │
│     ...                                 │
│   }                                     │
│ ]                                       │
└─────────────────────────────────────────┘
                    │
                    ▼
步骤2: 内部通信（并行执行）
┌─────────────────────────────────────────┐
│ Tool: sessions_send                     │
│                                         │
│ Request:                                │
│ {                                       │
│   "sessionKey": "agent:bage:discord:    │
│                  channel:1478694700...",│
│   "message": "任务：FocusPaw PRD...",    │
│   "timeoutSeconds": 10                  │
│ }                                       │
│                                         │
│ Note: 短超时（10秒），避免阻塞 workflow  │
│       即使失败也不影响下一步             │
└─────────────────────────────────────────┘
                    │
                    │ 并行执行
                    ▼
步骤3: 外部通知（必须执行）
┌─────────────────────────────────────────┐
│ Tool: message                           │
│                                         │
│ Request:                                │
│ {                                       │
│   "channel": "discord",                 │
│   "content": "<@1227127547477753967>    │
│               📨 任务流转\n             │
│               摸总 → <@1478296381...>\n │
│               任务：FocusPaw...\n       │
│               状态：已通知"             │
│ }                                       │
│                                         │
│ Note: 这是双保险，必须执行！             │
│       确保对方收到通知                   │
│       老板能看到工作流进度               │
└─────────────────────────────────────────┘
```

### 通信时序图

```
┌─────────┐     ┌──────────────┐     ┌──────────────┐     ┌─────────────┐
│  摸总    │     │ OpenClaw     │     │   Discord    │     │   八阿哥     │
│ (发送者) │     │   Gateway    │     │   Server     │     │  (接收者)    │
└────┬────┘     └──────┬───────┘     └──────┬───────┘     └──────┬──────┘
     │                 │                    │                    │
     │ 1. sessions_list │                    │                    │
     │ ────────────────>│                    │                    │
     │                 │                    │                    │
     │ 2. 返回八阿哥     │                    │                    │
     │    的 sessionKey │                    │                    │
     │ <────────────────│                    │                    │
     │                 │                    │                    │
     │ 3. sessions_send │                    │                    │
     │ ────────────────>│                    │                    │
     │   (并行)         │                    │                    │
     │                 │ 4. 路由到八阿哥      │                    │
     │                 │    的 session        │                    │
     │                 │ ────────────────────>│                    │
     │                 │                    │ 5. 如果在线，       │
     │                 │                    │    立即处理         │
     │                 │                    │ ─────────────────>│
     │                 │                    │                    │
     │ 6. message      │                    │                    │
     │ ────────────────>│ (并行)            │                    │
     │                 │ 7. 发送 Discord    │                    │
     │                 │    消息            │                    │
     │                 │ ────────────────────>│                    │
     │                 │                    │ 8. 推送通知        │
     │                 │                    │ ─────────────────>│
     │                 │                    │                    │
     │                 │ 9. 返回发送成功     │                    │
     │ <────────────────│                    │                    │
     │                 │                    │                    │
     │ 10. 八阿哥回复确认 │                   │                    │
     │ <────────────────────────────────────────────────────────│
     │                 │                    │                    │
```

---

## 工作流设计

### 完整工作流图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           FocusPaw 项目工作流                                 │
└─────────────────────────────────────────────────────────────────────────────┘

阶段1: PRD 撰写与审批
┌──────────┐    ┌──────────┐    ┌──────────┐
│   老板    │    │   摸总    │    │   八阿哥   │
│  (触发)   │───>│  (撰写)   │───>│  (等待)   │
└──────────┘    └──────────┘    └──────────┘
     │                │                │
     │ "@摸总 写PRD"  │                │
     │───────────────>│                │
     │                │                │
     │                │ 完成PRD        │
     │                │ 发送审批请求    │
     │<───────────────│                │
     │                │                │
     │ "批准"         │                │
     │───────────────>│                │
     │                │                │
     │                │ workflow推进   │
     │                │───────────────>│

阶段2: 测试用例编写与评审
┌──────────┐    ┌──────────┐    ┌──────────┐
│   摸总    │    │   八阿哥   │    │   混子    │
│  (等待)   │<───│  (编写)   │───>│  (评审)   │
└──────────┘    └──────────┘    └──────────┘
     │                │                │
     │                │ 完成测试用例    │
     │                │ 发送评审请求    │
     │                │───────────────>│
     │                │                │
     │                │                │ "测试用例确认通过"
     │                │<───────────────│
     │                │                │
     │                │ workflow推进   │
     │<───────────────│                │

阶段3: 开发与测试
┌──────────┐    ┌──────────┐    ┌──────────┐
│   混子    │    │   八阿哥   │    │   摸总    │
│  (开发)   │───>│  (测试)   │───>│  (验收)   │
└──────────┘    └──────────┘    └──────────┘
     │                │                │
     │ 开发完成       │                │
     │ 发送测试请求    │                │
     │───────────────>│                │
     │                │                │
     │                │ 测试通过       │
     │                │ 发送验收请求    │
     │                │───────────────>│
     │                │                │
     │                │                │ 验收通过
     │                │<───────────────│
     │                │                │
     │                │ workflow推进   │
     │<───────────────│                │

阶段4: 复盘
┌──────────┐    ┌──────────┐
│   摸总    │    │   亏总    │
│  (等待)   │───>│  (复盘)   │
└──────────┘    └──────────┘
     │                │
     │ 发送复盘通知    │
     │───────────────>│
     │                │
     │                │ 48小时内
     │                │ 组织复盘
     │                │
     │<───────────────│ 复盘完成
```

### 工作流状态机

```
                    ┌─────────────────┐
                    │    Start        │
                    │  (需求提出)      │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
         ┌─────────│   PRD 撰写      │
         │         │   (摸总🐱)       │
         │         └────────┬────────┘
         │                  │ PRD 完成
         │                  ▼
         │         ┌─────────────────┐
         │         │   PRD 审批      │◄──────────────────┐
         │         │   (老板)         │                   │
         │         └────────┬────────┘                   │
         │                  │ 批准                      │
         │                  ▼                           │
         │         ┌─────────────────┐                   │
         │         │ 测试用例编写    │                   │
         │         │   (八阿哥🐛👑)   │                   │
         │         └────────┬────────┘                   │
         │                  │ 测试用例完成              │
         │                  ▼                           │
         │         ┌─────────────────┐                   │
         │         │ 测试用例评审    │                   │
         │         │   (混子💻)       │                   │
         │         └────────┬────────┘                   │
         │                  │ 评审通过                  │
         │                  ▼                           │
         │         ┌─────────────────┐                   │
         │         │   开发实现      │                   │
         │         │   (混子💻)       │                   │
         │         └────────┬────────┘                   │
         │                  │ 开发完成                  │
         │                  ▼                           │
         │         ┌─────────────────┐                   │
         │         │   测试验收      │                   │
         │         │   (八阿哥🐛👑)   │                   │
         │         └────────┬────────┘                   │
         │                  │ 测试通过                  │
         │                  ▼                           │
         │         ┌─────────────────┐                   │
         │         │   产品验收      │                   │
         │         │   (摸总🐱)       │                   │
         │         └────────┬────────┘                   │
         │                  │ 验收通过                  │
         │                  ▼                           │
         │         ┌─────────────────┐                   │
         └────────>│    复盘        │                   │
                   │   (亏总🎲)       │───────────────────┘
                   │                 │    (新需求循环)
                   └─────────────────┘
```

---

## 技术实现细节

### 1. Session 查找算法

```javascript
// 算法：查找目标 agent 在当前 Discord channel 的 session

async function findTargetSession(targetAgentId) {
  // 1. 获取当前 channel ID（从环境或上下文）
  const currentChannelId = getCurrentChannelId(); // e.g., "1478694700176248862"
  
  // 2. 列出所有 sessions
  const sessions = await sessions_list({
    kinds: ["group"],  // 只查找 group/channel 类型的 session
    messageLimit: 0
  });
  
  // 3. 筛选目标 agent 的 Discord channel session
  const targetSession = sessions.find(s => {
    return s.agentId === targetAgentId &&
           s.channel === "discord" &&
           s.key.includes(`channel:${currentChannelId}`);
  });
  
  if (targetSession) {
    return targetSession.key; // e.g., "agent:bage:discord:channel:1478694700176248862"
  }
  
  // 4. 如果没找到，返回 null（后续只发 Discord 消息）
  return null;
}
```

### 2. 并行双发实现

```javascript
// 算法：并行发送内部消息和 Discord 通知

async function sendTaskWithDualChannel(targetAgentId, taskContent) {
  const results = {
    internal: null,
    external: null
  };
  
  // 1. 查找目标 session
  const targetSessionKey = await findTargetSession(targetAgentId);
  
  // 2. 并行执行双发
  const promises = [];
  
  // 2.1 内部通道（如果有 session）
  if (targetSessionKey) {
    promises.push(
      sessions_send({
        sessionKey: targetSessionKey,
        message: taskContent,
        timeoutSeconds: 10  // 短超时
      }).then(result => {
        results.internal = result;
      }).catch(err => {
        results.internal = { error: err.message };
        // 不抛出错误，继续执行
      })
    );
  }
  
  // 2.2 外部通道（必须执行）
  promises.push(
    message({
      channel: "discord",
      content: formatDiscordMessage(targetAgentId, taskContent)
    }).then(result => {
      results.external = result;
    })
  );
  
  // 3. 等待所有完成
  await Promise.all(promises);
  
  return results;
}
```

### 3. 错误处理策略

```javascript
// 错误处理：确保 workflow 不中断

async function robustSend(targetAgentId, taskContent) {
  try {
    // 尝试双发
    const results = await sendTaskWithDualChannel(targetAgentId, taskContent);
    
    // 检查外部通道是否成功（最关键）
    if (!results.external) {
      throw new Error("Discord 消息发送失败");
    }
    
    // 内部通道失败只记录，不影响 workflow
    if (results.internal?.error) {
      console.log("内部通道发送失败，但 Discord 已成功：", results.internal.error);
    }
    
    return { success: true, results };
    
  } catch (error) {
    // 严重错误：Discord 也失败了
    console.error("消息发送完全失败：", error);
    
    // 重试机制
    return await retrySend(targetAgentId, taskContent, maxRetries = 3);
  }
}
```

---

## 配置说明

### 核心配置文件：openclaw.json

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
            "1478623271393038356": { "allow": true, "requireMention": true },  // #公告
            "1478623331551805572": { "allow": true, "requireMention": false }, // #闲聊
            "1478623393992675338": { "allow": true, "requireMention": true },  // #复盘汇总
            "1478623533474119791": { "allow": true, "requireMention": true },  // #focuspaw
            "1478694700176248862": { "allow": true, "requireMention": true }   // #测试工作流
          }
        }
      }
    }
  },
  
  "tools": {
    "profile": "full",
    "sessions": {
      "visibility": "all"  // 关键配置：允许跨 session 访问
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

### 关键配置项说明

| 配置项 | 值 | 说明 |
|--------|-----|------|
| `tools.sessions.visibility` | `"all"` | 允许跨 session 通信，关键配置！ |
| `tools.agentToAgent.enabled` | `true` | 启用 agent 间通信 |
| `tools.agentToAgent.allow` | `[...]` | 允许通信的 agent 列表 |
| `channels.discord.allowBots` | `true` | 允许 bot 间通信 |
| `requireMention` | `true/false` | 频道级别的 @mention 要求 |

---

## 故障排查指南

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

### 调试命令

```bash
# 查看所有 sessions
openclaw sessions --json

# 查看特定 agent 的 sessions
openclaw sessions --json | grep "agent:bage"

# 查看 Gateway 状态
openclaw status

# 查看 Gateway 日志
journalctl -u openclaw-gateway -f

# 重启 Gateway
openclaw gateway restart

# 验证配置
openclaw doctor --config
```

---

## 最佳实践

### 1. 消息发送 checklist

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

### 2. Session Key 速查

```
通用格式：agent:{agentId}:discord:channel:{channelId}

亏总:   agent:kuizong:discord:channel:1478694700176248862
摸总:   agent:mozong:discord:channel:1478694700176248862
混子:   agent:hunzi:discord:channel:1478694700176248862
八阿哥: agent:bage:discord:channel:1478694700176248862

注意：channelId 根据实际频道变化
```

### 3. Discord ID 速查

```
老板:   <@1227127547477753967>
亏总:   <@1478290704684814390>
摸总:   <@1478295402779381894>
混子:   <@1478296740670083103>
八阿哥: <@1478296381973069895>
```

### 4. 消息模板

**任务流转通知**：
```
<@1227127547477753967> 📨 任务流转
[发送者] ([角色]) → <@目标ID> ([角色])
任务：[任务描述]
PRD/代码位置：[路径]
状态：已通知，等待执行
```

**收到任务确认**：
```
<@1227127547477753967> ✅ [接收者] 已收到任务
来自：[发送者]
任务：[任务描述]
状态：开始执行
```

**进度汇报**：
```
<@1227127547477753967> 📊 [执行者] 进度汇报
阶段：[当前阶段]
状态：[进行中/已完成]
成果：[关键产出]
下一步：[下一步动作]
```

### 5. 性能优化建议

1. **timeout 设置**：10 秒足够，避免阻塞 workflow
2. **并行发送**：sessions_send 和 Discord message 同时执行
3. **错误处理**：内部通道失败不影响 workflow，继续执行外部通道
4. **session 缓存**：可以缓存 sessions_list 结果，避免重复查询

### 6. 安全注意事项

1. 不要将 `.env` 或 token 提交到 Git
2. 定期轮换 Discord Bot Token
3. 限制 `tools.allow` 范围，只开放必要工具
4. 监控 Gateway 日志，及时发现异常

---

## 附录

### A. 项目文件结构

```
~/.openclaw/
├── openclaw.json              # 主配置文件
├── agents/
│   ├── kuizong/              # 亏总工作区
│   │   ├── AGENTS.md
│   │   ├── IDENTITY.md
│   │   ├── SOUL.md
│   │   ├── USER.md
│   │   ├── DISCORD_FORMAT.md
│   │   ├── TASK_CHECKLIST.md
│   │   └── workspace/
│   │       └── shared-projects/
│   │           └── FocusPaw/
│   ├── mozong/               # 摸总工作区（结构类似）
│   ├── hunzi/                # 混子工作区（结构类似）
│   └── bage/                 # 八阿哥工作区（结构类似）
└── workspace/
    └── shared-projects/
        ├── BEHAVIOR.md       # 工作流行为规则
        ├── workflow-templates.md  # 消息模板
        └── FocusPaw/         # 具体项目
            ├── AGENTS.md
            ├── PRD.md
            ├── MEMORY.md
            └── README.md
```

### B. 相关资源

- OpenClaw 官方文档：https://docs.openclaw.ai
- Discord Developer Portal：https://discord.com/developers
- 本项目 GitHub：https://github.com/zwlcoding/yuanting-group

### C. 更新日志

**2026-03-04**: 
- 初始版本
- 实现混合通信方案（双通道）
- 解决 session 隔离问题
- 完成 workflow 端到端测试

---

*本文档由 渊渟集团技术团队 维护*  
*最后更新：2026-03-04*