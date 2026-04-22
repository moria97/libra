# Agent 托管平台 — 统一设计文档 (Draft v0.1)

*企业级 Agent-as-a-Service 平台——用自然语言创建 Agent，通过飞书/钉钉即时协作，沙箱隔离运行，全生命周期托管。*

**灵感来源：** [Nous Research hermes-agent](https://github.com/nousresearch/hermes-agent)

---

## 目录

1. [产品定位与市场分析](#1-产品定位与市场分析)
2. [目标用户与场景](#2-目标用户与场景)
3. [核心用户旅程](#3-核心用户旅程)
4. [系统架构](#4-系统架构)
5. [AgentSpec 配置模型](#5-agentspec-配置模型)
6. [核心操作与算法设计](#6-核心操作与算法设计)
7. [IM Gateway 与多 Session 管理](#7-im-gateway-与多-session-管理)
8. [沙箱隔离架构](#8-沙箱隔离架构)
9. [记忆系统](#9-记忆系统)
10. [多 Agent 协作](#10-多-agent-协作)
11. [Skill Store（技能商店）](#11-skill-store技能商店)
12. [评测引擎](#12-评测引擎)
13. [后端 API 与数据模型](#13-后端-api-与数据模型)
14. [技术选型](#14-技术选型)
15. [企业级管理](#15-企业级管理)
16. [落地路径与 MVP 规划](#16-落地路径与-mvp-规划)
17. [成功指标](#17-成功指标)
18. [开放问题](#18-开放问题)

---

## 1. 产品定位与市场分析

### 一句话定位

> 企业级 Agent-as-a-Service 平台——用自然语言创建 Agent，通过飞书/钉钉即时协作，沙箱隔离运行，全生命周期托管。

### 与 hermes-agent 的关系

hermes-agent 解决的是**单 agent 如何运行**的问题（skill 系统、memory、terminal/messaging 入口）。我们的平台解决的是**如何托管和编排 N 个 agent**的问题：

| 维度 | hermes-agent | 本平台 |
|---|---|---|
| 定位 | 个人 Agent runtime | 企业级多 Agent 托管平台 |
| 部署 | 用户自己安装 | 平台按需拉起 |
| 隔离 | 单容器 | 多级沙箱 + 多租户 |
| 多 Agent | subagent delegation | 平台级 DAG 编排 |
| Memory | 单 agent 三层 | 四层 + 跨 agent 共享 |
| IM | Telegram/Discord/Slack | 企业 IM（飞书/钉钉） |
| 管理 | 无 | 多租户/RBAC/审计/合规 |

可借鉴 hermes-agent 的 skill 标准（兼容 agentskills.io）和 MCP 集成设计。但 agent runtime 建议自研，托管平台对生命周期管理、多租户隔离、资源调度有更严格的要求。

### 竞品格局

| 平台 | 定位 | IM集成(飞书/钉钉) | 多Agent协作 | 托管/自建 | 企业级 | 状态 |
|---|---|---|---|---|---|---|
| **Coze (字节)** | IM Bot 构建平台 | **原生支持** | 工作流级编排 | 两者都有 | 商业版有 | 活跃 |
| **Dify** | LLM应用开发平台 | 社区插件 | 有限 | 两者都有 | 企业版 | 活跃(139K⭐) |
| **FastGPT** | 知识库QA平台 | 飞书原生 | 有限 | 两者都有 | 基础 | 活跃(27.8K⭐) |
| **CrewAI** | 多Agent编排框架 | Slack/Teams | **核心特性** | 两者都有 | SSO/RBAC | 活跃(49.4K⭐) |
| **Langflow** | 可视化Agent构建 | Slack | 通过CrewAI | 两者都有 | 联系销售 | 活跃(147K⭐) |
| **AutoGen** | 多Agent研究框架 | 无 | **核心特性** | 仅自建 | 无 | 维护模式 |
| **hermes-agent** | 个人Agent runtime | Telegram/Discord/Slack | Subagent | 用户自装 | 无 | 活跃(107K⭐) |

### 核心差异化

**没有一个平台同时做到以下三点：**

1. **企业 IM 原生集成**（飞书+钉钉）— 只有 Coze 做到了，但 Coze 的 Agent 是"Bot"级别，不是可自主运行的长期 Agent
2. **全托管 Agent 生命周期** — 从自然语言描述 → 自动配置 → 部署 → 评测 → 维护
3. **真正的多 Agent 协作 + 多 Session** — CrewAI 有多 Agent 但没有 IM 入口；Coze 有 IM 但多 Agent 能力弱

**vs Coze 的本质区别：**
- Coze 的 Agent = 轻量 Bot（对话式，工作流驱动，无独立运行时）
- 我们的 Agent = 有独立沙箱的**自主 Agent**（能执行代码、管理文件、长期运行任务、自主学习技能）
- 这是 "Chatbot 平台" vs "Agent 托管平台" 的本质区别

---

## 2. 目标用户与场景

### Primary：企业中台/效率团队（v1）
- **场景：** 给内部各业务线按需创建专属 Agent——代码审查、竞品分析、客服、数据分析
- **痛点：** 每个需求都要开发一个 Bot，成本高，维护难
- **价值：** 自然语言描述 → 分钟级拉起 Agent → 飞书群直接用

### Secondary：开发者/技术团队（v1-v2）
- **场景：** 需要可编程的 Agent 能力——自定义 Skill、接入私有 API、精细控制沙箱
- **痛点：** 自己搭 Agent 基础设施太重，但又需要比 Coze 更强的定制性
- **价值：** AgentSpec 声明式配置 + Skill Store + 企业级沙箱

### Tertiary：SaaS 服务商（v3）
- **场景：** 想在自己的产品里嵌入 Agent 能力
- **价值：** 白标 Agent 平台，API 集成

---

## 3. 核心用户旅程

### 旅程 1：自然语言创建 Agent

```
用户在飞书发消息给 @平台助手:
  "帮我创建一个竞品分析Agent，能搜网页、读PDF、生成Markdown报告"
     ↓
平台Agent解析意图，生成AgentSpec预览（卡片消息展示）:
  · 角色：竞品分析师
  · 技能：web_search, pdf_reader, report_generator
  · 模型：Claude Sonnet 4.6
  · 沙箱：标准级
  · 记忆：跨Session持久化
     ↓
用户确认/调整 → 点击"创建"
     ↓
Agent实例拉起（< 30秒）→ 自动创建飞书群 → Agent加入群组
     ↓
用户直接在群里和Agent对话，开始工作
```

### 旅程 2：多任务并行（解决 IM 弱点）

```
用户在Agent群里：

[主对话] "帮我分析三个竞品：A、B、C"
     ↓
Agent 创建3个话题Thread：
  · 📊 Thread: 竞品A分析
  · 📊 Thread: 竞品B分析
  · 📊 Thread: 竞品C分析
     ↓
[主对话] Agent 发送任务面板（互动卡片）：
  ┌─────────────────────────┐
  │ 📋 任务看板               │
  │ #1 竞品A分析 ⏳ 进行中    │
  │ #2 竞品B分析 ⏳ 进行中    │
  │ #3 竞品C分析 🔄 排队中    │
  │                           │
  │ [查看详情] [新增任务]      │
  └─────────────────────────┘
     ↓
各Thread内独立推进，完成后主对话汇总
```

### 旅程 3：多 Agent 协作

```
一个飞书群内有多个Agent：

用户: "先做竞品分析，然后基于分析结果出产品方案"
     ↓
Coordinator 自动编排：
  Step 1: @分析师Agent 做竞品分析 (Thread A)
  Step 2: @产品Agent 基于分析结果出方案 (Thread B)
     ↓
TaskHandoff: 分析师 → 产品Agent（摘要+报告文件传递）
     ↓
用户全程在群里可见进度，可随时介入任一Thread
```

---

## 4. 系统架构

### 整体分层

```
┌─────────────────────────────────────────────────────────────┐
│                     IM Gateway Layer                        │
│            (飞书 Bot / 钉钉 Bot / Web Chat)                  │
└────────────────────────┬────────────────────────────────────┘
                         │ WebSocket / Webhook
┌────────────────────────▼────────────────────────────────────┐
│                   Session Router                            │
│         (消息路由 + 多任务分发 + Thread 映射)                  │
└────────────────────────┬────────────────────────────────────┘
                         │ gRPC / Internal API
┌────────────────────────▼────────────────────────────────────┐
│                  Agent Orchestrator                         │
│    (Agent 生命周期管理 + 多 Agent 编排 + 资源调度)             │
├─────────────┬──────────────┬──────────────┬─────────────────┤
│ Agent       │ Skill        │ Memory       │ Sandbox         │
│ Registry    │ Registry     │ Service      │ Manager         │
└─────────────┴──────────────┴──────────────┴─────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│                  Infrastructure Layer                       │
│    (Container Runtime + Model Router + Object Storage)      │
└─────────────────────────────────────────────────────────────┘
```

### 核心组件职责

| 组件 | 职责 | 关键设计 |
|---|---|---|
| **IM Gateway** | 飞书/钉钉协议适配，消息收发 | 协议归一化 → 统一 `ChatEvent` 输出；未来加 Slack/Teams 只需写新 adapter |
| **Session Router** | 消息路由、多任务分发、Thread映射 | 一个 IM 对话框背后 N 个并发 Session；基于 thread_id 路由 |
| **Agent Orchestrator** | Agent 生命周期、多Agent编排、资源调度 | Agent Provisioning、Health Check、Multi-Agent DAG |
| **Agent Registry** | Agent 模板、配置、版本管理 | PostgreSQL + S3 |
| **Skill Registry** | Skill 包注册、依赖管理、权限控制 | PostgreSQL + OCI Registry |
| **Memory Service** | 四层记忆管理 | Redis (L1) + PG+pgvector (L2/L3) + Neo4j (L4) |
| **Sandbox Manager** | 容器生命周期、资源配额、隔离执行 | 三级隔离模型，统一 API |

### 后端服务拆分

```
                          ┌──────────────┐
                          │  API Gateway │  (Kong / APISIX)
                          │  认证/限流/路由 │
                          └──────┬───────┘
              ┌─────────────┬────┴────┬──────────────┐
              ▼             ▼         ▼              ▼
      ┌──────────┐  ┌───────────┐ ┌─────────┐ ┌──────────┐
      │ agent-svc│  │session-svc│ │skill-svc│ │ eval-svc │
      │ (核心)    │  │(会话管理)  │ │(技能商店)│ │(评测引擎) │
      └────┬─────┘  └─────┬─────┘ └────┬────┘ └────┬─────┘
           │              │             │            │
      ┌────▼─────┐  ┌─────▼─────┐     │       ┌────▼─────┐
      │sandbox-  │  │ memory-   │     │       │ metrics- │
      │manager   │  │ svc       │     │       │ store    │
      └──────────┘  └───────────┘     │       └──────────┘
                                      │
                               ┌──────▼──────┐
                               │  skill-     │
                               │  registry   │
                               └─────────────┘
```

### 部署拓扑

```
┌─────────────────────────────────────────────────┐
│                 K8s Cluster                      │
│                                                  │
│  ┌─────────┐  ┌─────────┐  ┌─────────────────┐  │
│  │ Gateway  │  │ Router  │  │  Orchestrator   │  │
│  │ (飞书)   │  │         │  │                 │  │
│  ├─────────┤  │         │  │                 │  │
│  │ Gateway  │  │         │  │                 │  │
│  │ (钉钉)   │  │         │  │                 │  │
│  └─────────┘  └─────────┘  └─────────────────┘  │
│                                                  │
│  ┌──────────────────────────────────────────┐    │
│  │         Agent Sandbox Pool               │    │
│  │  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐       │    │
│  │  │ A-1 │ │ A-2 │ │ A-3 │ │warm │       │    │
│  │  └─────┘ └─────┘ └─────┘ └─────┘       │    │
│  └──────────────────────────────────────────┘    │
│                                                  │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐    │
│  │  PG    │ │ Redis  │ │ Neo4j  │ │  S3    │    │
│  └────────┘ └────────┘ └────────┘ └────────┘    │
└─────────────────────────────────────────────────┘
```

---

## 5. AgentSpec 配置模型

AgentSpec 是平台的核心数据结构——用户的一切意图最终都编译成 AgentSpec：

```yaml
apiVersion: agent/v1
kind: AgentSpec
metadata:
  name: "code-reviewer"
  owner: tenant-123
  version: "1.2.0"
spec:
  persona:
    role: "代码审查专家"
    system_prompt: "你是一个严谨的代码审查专家..."
  model:
    provider: anthropic
    model: claude-sonnet-4-6
    fallback: claude-haiku-4-5
  skills:
    - name: github-integration
      version: "^2.0"
      config:
        repos: ["org/repo-a", "org/repo-b"]
    - name: code-analysis
      version: "^1.5"
  memory:
    working_memory: 128k        # L1 context window
    session_ttl: 7d              # L2 session 保留时长
    persistent: true             # L3 agent-level memory
    shared_namespace: "eng-team" # L4 跨 agent 共享
  sandbox:
    security_level: standard     # standard | elevated | strict
    resources:
      cpu: "2"
      memory: "4Gi"
    network:
      policy: restricted         # none | restricted | full
      allowed_domains:
        - "github.com"
        - "api.anthropic.com"
    filesystem:
      writable_paths: ["/workspace", "/tmp"]
      persistent_volume: 10Gi
  im_channels:
    - platform: feishu
      type: group
      name: "代码审查助手"
  scaling:
    idle_timeout: 300s           # 休眠超时
    max_concurrent_sessions: 5
    warm_pool: 1                 # 预热实例数
  lifecycle:
    auto_sleep: 30m
    wake_on_message: true
```

**安全级别映射（对用户透明）：**

| security_level | 实现 | 隔离度 | 启动速度 | 适用场景 |
|---|---|---|---|---|
| `standard` | gVisor (runsc) 容器 | 中 | 2-5s | 标准 agent、代码执行 |
| `elevated` | Firecracker microVM | 高 | 3-8s | 企业客户、敏感数据 |
| `strict` | 专属 VM | 最高 | 5-15s | 金融/医疗等强合规场景 |

企业管理员可设 org 级最低安全策略，agent 创建时自动取 `max(用户选择, org 策略)`。

---

## 6. 核心操作与算法设计

### 6.1 Agent 配置生成（NL → AgentSpec）

用户用自然语言描述 agent 需求 → 平台 LLM 解析为结构化 AgentSpec。

本质上是一个**约束满足问题**：用户意图 + skill 依赖图 + 资源约束 → 可行配置。

- 维护 **Skill Registry**（技能注册表），每个 skill 带语义标签、依赖关系、资源需求
- 用 embedding 做 skill 匹配 + 依赖图做合法性校验
- LLM 解析用户意图 → 初步 AgentSpec → 校验约束 → 生成预览卡片 → 用户确认

### 6.2 Agent 生命周期管理

使用 Temporal Workflow 编排长链路：

```
CreateAgentWorkflow:
  1. 解析 AgentSpec → 校验配置合法性
  2. 解析 Skill 依赖 → 拉取 skill 包 → 构建运行时镜像
  3. 申请沙箱资源 → K8s 创建 Pod（对应 runtime class）
  4. 注入 AgentSpec + Skill 包 + Memory 初始化
  5. 启动 Agent Runtime 进程
  6. 健康检查通过 → 注册 IM Bot → 通知用户

  补偿逻辑（任一步失败）:
  - 回滚已创建资源
  - 通知用户失败原因 + 建议
```

**休眠/唤醒（Temporal Signal）：**
- `SleepSignal`: 保存 memory state → 释放 Pod（runtime 丢弃，memory 持久化在 PG/pgvector）
- `WakeSignal`: 分配 Pod → 从 memory 恢复上下文 → 重连 NATS → 消费积压消息
- MVP 阶段：仅支持"丢弃 runtime + memory 恢复"模式；Phase 3 加 runtime checkpoint

### 6.3 多 Agent 协作调度

DAG-based 任务编排：

```
用户请求 → Task Planner (LLM) → Task DAG
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
              Agent A (code)   Agent B (review)  Agent C (test)
                    │               ▲               ▲
                    └───────────────┘               │
                            └───────────────────────┘
```

- 优先级队列 + 依赖拓扑排序
- 每个 agent 维护自己的 task queue，平台级 scheduler 做全局协调
- Coordinator 层做拓扑验证，防止环形依赖（调度前拒绝，而非运行时死锁）

---

## 7. IM Gateway 与多 Session 管理

### IM Gateway 设计

飞书和钉钉各有不同的事件模型和卡片 SDK，Gateway 层做**协议归一化**：

```
飞书/钉钉 Webhook
        │
        ▼
  IM Gateway (Go)
  ├── 验签 + 解密
  ├── 协议归一化 → ChatEvent{platform, user_id, thread_id, content, ...}
  └── 发送到 NATS: "im.events.{tenant_id}"
        │
        ▼
  Session Router (Go)
  ├── 查 thread_id → session_id 映射 (Redis)
  ├── 无映射 → 创建新 Session + 分配 Thread
  ├── 有映射 → 加载 Session 上下文
  └── 转发到 NATS: "agent.{agent_id}.messages"
        │
        ▼
  Agent Runtime (Python, 沙箱内)
  ├── 消费 NATS 消息
  ├── 执行 LLM + tool calling loop
  ├── 流式输出 → NATS: "agent.{agent_id}.responses"
  └── 回传给 Session Router → IM Gateway → 飞书/钉钉
```

### 多 Session / 多任务管理（三层方案）

IM 对话框天然是单线程的，但 agent 需要并发处理多任务。解决方案：

| 层级 | 方案 | 说明 |
|---|---|---|
| 主力 | **话题线程 (Thread)** | 飞书话题/钉钉话题 = 1个 Session，最自然的映射 |
| 辅助 | **指令切换** | `/session new` `/session list` `/session 3` |
| 兜底 | **自动识别** | LLM 判断话题切换，自动分 session |

**IM 交互模型：**

```
IM 消息 → Gateway → Session Router
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
         主对话区     Thread A    Thread B
        (状态面板)   (Task #1)   (Task #2)
```

- **主对话区**：作为"控制台"，显示 agent 状态面板、任务摘要、快捷指令
- **Thread**：每个长期任务自动开一个 IM Thread，agent 在 thread 内完成具体工作
- **卡片消息**：利用飞书/钉钉 Interactive Card 做富交互——任务列表、进度条、审批按钮、session 切换器

**Session 状态机：**

```
每个 agent 实例维护:
├── Active Sessions[] — 当前进行中的对话
│   ├── session_id
│   ├── task_context (任务上下文快照)
│   ├── state: idle | working | waiting_input | blocked
│   └── priority: int
├── Task Queue — 待处理任务队列（优先级堆）
└── Context Switcher — 上下文切换引擎
```

**上下文切换算法：**
- 每个 session 的上下文做 checkpoint（关键状态 + 摘要），存入 memory store
- 切换时只加载目标 session 的 checkpoint + 最近 N 条消息
- 类似 OS 进程调度的 context switch，但"寄存器"是 LLM 的 conversation context

### Slash Commands

```
/tasks             → 📋 当前任务列表 + 状态
/task new <描述>   → 创建新任务
/task 1 status     → 查看任务详情和进度
/task 1 cancel     → 取消任务
/agent list        → 查看可用 Agent
/agent create <描述> → 用自然语言创建新 Agent
/session new       → 新建会话
/session list      → 列出所有会话
/session 3         → 切换到指定会话
/status            → 全局状态面板
```

---

## 8. 沙箱隔离架构

### 隔离模型

```
Platform
├── Tenant (企业/用户)
│   ├── Agent Instance 1
│   │   ├── Sandbox (容器/gVisor/Firecracker)
│   │   ├── Resource Quota (CPU, memory, API calls)
│   │   └── Permission Scope (文件系统, 网络, 工具调用)
│   └── Agent Instance 2
│       └── ...
└── Shared Services
    ├── Skill Registry
    ├── Model Router
    └── Memory Store
```

### 沙箱存储架构：分层镜像 + 持久卷

Agent 沙箱需要保证：(1) 已安装的依赖不随休眠丢失，(2) 中间文件可持久保存，(3) Skill 文件依赖可靠加载。

核心方案：**K8s Pod + PersistentVolume**，而非纯临时容器。

```
┌─────────────────────────────────┐
│        Agent 沙箱 (Pod)          │
│                                  │
│  Layer 1: Base Image (只读)      │  ← Python runtime + 基础库
│  Layer 2: Skill Image (只读)     │  ← 预构建的 skill 依赖层
│  Layer 3: PV /workspace (读写)   │  ← 用户安装的依赖 + 中间文件
│  Layer 4: PV /agent-state (读写) │  ← agent 配置 + skill 运行时状态
└─────────────────────────────────┘
```

**Layer 1: Base Image**（平台维护）：包含 Python、Node.js 等 runtime + 常见依赖。不同安全级别可选不同 base image。

**Layer 2: Skill Image**（按 AgentSpec 构建）：平台根据 skill manifest 的依赖列表构建中间镜像层，**相同 skill 组合的 agent 共享同一镜像**（缓存 key = `sha256(sorted_skill_deps)`）。

**Layer 3: PV /workspace**：Agent 运行期间 `pip install` 的额外依赖、生成的中间文件、下载的数据等写入 PV。**休眠时 Pod 销毁但 PV 保留**，唤醒时新 Pod 挂载同一 PV。Agent runtime 自动维护 `requirements.lock` 记录所有 pip install 操作，存到 `/agent-state/`，PV 损坏时可从 lock 文件重装。

**Layer 4: PV /agent-state**：Agent 配置、skill 运行时状态、checkpoint。与 workspace 分离，便于独立备份和迁移。

**PV 存储注意事项：**
- 使用 ReadWriteMany 存储（如 NFS/EFS/CephFS）确保跨 node 调度时 PV 可挂载。MVP 阶段推荐 EFS (AWS) 或 NFS。
- 长期休眠的 agent 分级处理存储成本：< 1h PV 在线（< 5s 唤醒）；1h-7d 降级到低成本存储类（< 10s）；> 7d 快照到 S3 + 释放 PV（30-60s 唤醒）。

### Skill 文件依赖加载

```
Skill Registry (S3/OCI)
        │
        ▼ skill install 时
   Sandbox Manager
   ├── 拉取 skill 包
   ├── 解压到 /agent-state/skills/{skill-name}/
   ├── 执行 skill 的 setup.sh（安装依赖到 /workspace/venv/）
   └── 注册 tool schema 到 Agent Runtime
        │
        ▼ 休眠后唤醒
   /agent-state/skills/ 仍在 PV 上
   → Agent Runtime 启动时扫描已安装 skill → 直接加载，无需重装
```

### 休眠 & 唤醒

Agent 空闲时保存 memory state → 释放计算资源，收到消息时秒级唤醒：

```
休眠：
  1. Agent Runtime 优雅停止（flush 缓存、保存状态）
  2. Pod 销毁（释放 CPU/内存）
  3. PV 保留（/workspace + /agent-state 不动）
  4. 镜像 tag 记录在 AgentSpec.status 中

唤醒：
  1. 从 AgentSpec.status 读取镜像 tag
  2. 创建新 Pod，挂载原 PV
  3. Agent Runtime 启动，检测 /agent-state 存在 → 恢复模式
  4. 所有依赖、中间文件、skill 状态原封不动
  → 对用户来说，就像 Agent 一直在运行
```

- **Memory state**（L2/L3 记忆）：持久化在 PG/pgvector，不随休眠丢失
- **Runtime state**（进程状态、临时文件）：MVP 阶段"丢弃 runtime + PV 恢复"模式；Phase 3 加进程级 checkpoint

**关键性能指标：** 冷启动（休眠→唤醒）< 5秒，通过预热 Pod 池 + 分层镜像实现。

---

## 9. 记忆系统

### 四层记忆架构

```
Memory Architecture:
├── L1: Working Memory (当前 session 上下文, in-context)
│       存储: Redis | 生命周期: 会话内
├── L2: Session Memory (FTS5 索引 + 向量检索, 跨 session)
│       存储: PG + pgvector | 生命周期: 按 session_ttl 配置
├── L3: Agent Memory (agent 级持久知识)
│       存储: PG + pgvector | 生命周期: agent 生命周期
└── L4: Platform Memory (跨 agent 共享知识, 组织级)
        存储: Neo4j (knowledge graph) | 生命周期: 组织级
```

### 关键算法

**记忆检索：** Hybrid retrieval（BM25 + 向量 + 时间衰减加权）
- 最近的记忆权重更高
- 重要的历史记忆通过 "importance score" 保持可达
- 性能目标：< 200ms（pgvector HNSW 索引 + Redis 缓存 top-K）

**记忆压缩：** 长对话自动摘要
- 保留关键决策点
- 结构化提取（entities, decisions, action items）

**跨 Agent 记忆共享：** L4 层通过 knowledge graph 实现
- Agent A 学到的项目上下文可以被 Agent B 检索到
- 通过 `shared_namespace` 配置控制共享范围

---

## 10. 多 Agent 协作

### Coordinator 模式

在 IM 群里引入多个 Agent + 一个 Coordinator：
- `@forge` → Platform Agent（管理指令）
- `@分析师` → Agent A（竞品分析）
- `@设计师` → Agent B（UI 设计）
- 不@任何人 → **Coordinator 自动路由**到最合适的 Agent

**Coordinator 本身是独立 Agent 实例**，skill 为 `[routing, orchestration, task_planning]`，可版本管理和替换。

### TaskHandoff 协议

Agent 之间的任务交接通过受控协议进行（而非共享全量上下文）：

```yaml
handoff:
  from: agent-a
  to: agent-b
  context_summary: "..."      # LLM 生成的摘要
  artifacts: ["report.md"]    # 传递的文件
  thread_id: "..."            # IM thread 用于用户可见
```

**IM 可见性：** Handoff 发生时在 IM 发送可见卡片消息：

> 📋 任务交接：@分析师 → @设计师
> 摘要：竞品分析报告已完成，包含 Google/Meta/Apple 三家对比数据
> 附件：report.md
> [查看详情]

### 跨沙箱通信安全

Agent 之间不直接通信，通过平台级 Message Bus 中转：

```
Agent A (沙箱内)                    Agent B (沙箱内)
    │                                    ▲
    │ delegate_task(target="agent-B")    │
    ▼                                    │
NATS: "agent.routing.delegate"     NATS: "agent.{B}.messages"
    │                                    ▲
    └────► Orchestrator 路由 ────────────┘
           (权限校验 + 参数过滤)
```

即使 Agent A 被注入恶意指令，也无法直接访问 Agent B 的文件系统或内存。

---

## 11. Skill Store（技能商店）

- **内置技能**：web_search、code_executor、file_ops、pdf_reader、image_gen、data_analysis
- **MCP 协议技能**：Slack、GitHub、Jira、Database 等标准 MCP 集成
- **自生成技能**：Agent 使用中自动生成并优化技能（参考 hermes-agent 模式）
- **用户自定义**：支持上传自定义 MCP Server 扩展能力

每个 skill 带有语义标签、依赖关系、资源需求，支持版本管理和权限控制。

---

## 11.5 用户数据与知识库集成

Agent 常需要访问用户的本地文件、Wiki、文档库。提供三种集成方式：

### 方式 A：IM 直接上传（MVP）

用户在飞书/钉钉群里直接发送文件（PDF/Word/图片/代码），IM Gateway 接收后存入 S3，挂载到 Agent 的 `/workspace/uploads/`。Agent 自动感知新文件并加载到上下文或知识库。

- 对用户零门槛——发文件就行
- 适合少量文件场景

### 方式 B：Knowledge Volume（Phase 2）

用户通过 Web 管理后台或 API 批量上传文件，绑定到 Agent：

```yaml
# AgentSpec 扩展
spec:
  knowledge:
    volumes:
      - name: "product-docs"
        source: s3://tenant-123/knowledge/product-docs/
        mount: /workspace/knowledge/product-docs
        sync: on_change    # 源文件变化时自动同步
      - name: "wiki-export"
        source: s3://tenant-123/knowledge/wiki/
        mount: /workspace/knowledge/wiki
        sync: manual       # 手动触发同步
```

- 文件存在 S3，通过 PV 或 sidecar 挂载到沙箱
- 支持自动同步——源文件更新时 Agent 自动获取最新版本（通过 S3 Event Notification 触发）
- 适合团队知识库、产品文档等场景
- **Knowledge Indexer**（自动触发）：上传文件 → 提取文本 + 切片 + 向量化 → 写入 Memory Service (L3)。Agent 既能做语义检索（"找关于XX的文档段落"），也能直接读取原始文件。

### 方式 C：外部数据源连接器（v1+）

Agent 通过 MCP Skill 直接连接用户的外部系统（Notion、Confluence、GitHub Wiki、Google Drive 等）：

```yaml
spec:
  skills:
    - name: notion-connector
      config:
        workspace_id: "xxx"
        api_key: "${secrets.NOTION_KEY}"  # 加密存储
    - name: confluence-connector
      config:
        base_url: "https://company.atlassian.net"
        api_token: "${secrets.CONFLUENCE_TOKEN}"
```

- Agent 实时拉取最新内容，不需要手动同步
- 凭证通过平台 Secrets Manager 加密存储

### 优先级

| 方式 | 阶段 | 复杂度 | 用户体验 |
|---|---|---|---|
| **A: IM 上传** | MVP (Phase 1) | 低 | 发文件即可，零门槛 |
| **B: Knowledge Volume** | Phase 2 | 中 | 批量上传 + 自动同步 |
| **C: 外部连接器** | Phase 3 (v1) | 高 | 实时连接，最强大 |

---

## 12. 评测引擎

### 自动评测维度

| 维度 | 指标 | 采集方式 |
|---|---|---|
| **任务完成率** | agent 完成用户请求的成功率 | messages 表 + 用户反馈 |
| **交互效率** | 完成任务所需的来回次数 | session 消息计数 |
| **Skill 利用率** | 安装的 skill 中实际被使用的比例 | tool_calls 日志 |
| **Memory 命中率** | 跨 session 记忆检索的相关性评分 | memory service 指标 |
| **用户满意度** | 显式反馈（👍/👎）+ 隐式信号 | IM 反馈 + 追问率 |
| **响应延迟** | 首 token 延迟、总响应时间 | OpenTelemetry trace |
| **Token 消耗** | 每任务 token 用量 | LLM 调用日志 |

### A/B 测试框架

- 同一 agent 配置可灰度发布不同版本（不同 prompt、model、skill 组合）
- 流量按比例分配，用 Sequential Probability Ratio Test 判断显著性
- 结果反馈到 Skill Registry，自动调整推荐权重

### 输出

- Web Dashboard 可视化报表
- IM 定期推送评测报告摘要

---

## 13. 后端 API 与数据模型

### API 设计

#### Agent 管理

```
POST   /api/v1/agents                    # 创建 agent（AgentSpec）
POST   /api/v1/agents/from-description   # 自然语言 → AgentSpec
GET    /api/v1/agents                     # 列出当前租户所有 agent
GET    /api/v1/agents/{id}               # agent 详情 + 运行状态
PATCH  /api/v1/agents/{id}               # 更新配置
DELETE /api/v1/agents/{id}               # 销毁 agent
POST   /api/v1/agents/{id}/wake          # 手动唤醒
POST   /api/v1/agents/{id}/sleep         # 手动休眠
POST   /api/v1/agents/{id}/clone         # 克隆 agent
GET    /api/v1/agents/{id}/health        # 健康检查
```

#### Session 管理

```
POST   /api/v1/agents/{id}/sessions                        # 创建新会话
GET    /api/v1/agents/{id}/sessions                        # 列出所有会话
GET    /api/v1/agents/{id}/sessions/{sid}                  # 会话上下文摘要
POST   /api/v1/agents/{id}/sessions/{sid}/messages         # 发送消息
POST   /api/v1/agents/{id}/sessions/{sid}/messages/stream  # SSE 流式响应
DELETE /api/v1/agents/{id}/sessions/{sid}                  # 关闭会话
```

#### Skill 管理

```
GET    /api/v1/skills                     # 技能商店列表
GET    /api/v1/skills/{name}/versions     # 技能版本列表
POST   /api/v1/skills                     # 发布自定义技能
POST   /api/v1/agents/{id}/skills         # 为 agent 安装技能
DELETE /api/v1/agents/{id}/skills/{name}  # 卸载技能
```

### 核心数据模型（PostgreSQL）

```sql
-- 租户隔离
CREATE TABLE tenants (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        TEXT NOT NULL,
    plan        TEXT NOT NULL DEFAULT 'free',
    quota_json  JSONB NOT NULL DEFAULT '{}',
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Agent 定义
CREATE TABLE agents (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID NOT NULL REFERENCES tenants(id),
    name        TEXT NOT NULL,
    spec        JSONB NOT NULL,
    state       TEXT NOT NULL DEFAULT 'creating',
    sandbox_id  TEXT,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name)
);

-- 会话
CREATE TABLE sessions (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agent_id         UUID NOT NULL REFERENCES agents(id),
    im_thread_id     TEXT,
    im_platform      TEXT,
    state            TEXT NOT NULL DEFAULT 'active',
    context_snapshot JSONB,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_active      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- 消息记录
CREATE TABLE messages (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id  UUID NOT NULL REFERENCES sessions(id),
    role        TEXT NOT NULL,
    content     TEXT NOT NULL,
    tool_calls  JSONB,
    token_count INT,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_messages_session ON messages(session_id, created_at);

-- 向量记忆
CREATE TABLE memories (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agent_id    UUID NOT NULL REFERENCES agents(id),
    layer       TEXT NOT NULL,
    namespace   TEXT,
    content     TEXT NOT NULL,
    embedding   vector(1536),
    importance  FLOAT DEFAULT 0.5,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    accessed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_memories_vec ON memories USING ivfflat (embedding vector_cosine_ops);

-- 技能注册表
CREATE TABLE skills (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        TEXT NOT NULL UNIQUE,
    version     TEXT NOT NULL,
    description TEXT,
    type        TEXT NOT NULL,
    manifest    JSONB NOT NULL,
    package_url TEXT,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## 14. 技术选型

| 组件 | 选型 | 理由 |
|---|---|---|
| **平台服务** | Go (gin/grpc) | 高并发、沙箱调度低延迟 |
| **IM Gateway** | Go | 飞书/钉钉 SDK 都有 Go 版本 |
| **Agent Runtime** | Python | LLM SDK 生态优先 |
| **消息队列** | NATS JetStream | 轻量、持久化、适合 agent 异步通信 |
| **数据库** | PostgreSQL + pgvector | 结构化数据 + 向量检索一体 |
| **缓存** | Redis Cluster | Session 上下文缓存、agent 状态 |
| **知识图谱** | Neo4j | L4 跨 agent 共享记忆 |
| **对象存储** | MinIO / S3 | Skill 包、agent 快照、文件附件 |
| **容器编排** | K8s + gVisor/Kata | Agent 沙箱运行时 |
| **任务编排** | Temporal | 长链路 workflow（agent 创建/休眠/唤醒） |
| **API Gateway** | Kong / APISIX | 认证、限流、路由 |
| **前端** | React / Vue | Web 管理后台 |
| **可观测性** | OpenTelemetry | 端到端 tracing + 评测指标采集 |

---

## 14.5 Provider 抽象架构

平台采用 **Provider 插件化**设计，每个核心层定义统一接口，由不同 Provider 实现。目标：开源社区可在任意环境（单机/云/混合）部署，不绑定特定云厂商。

### 14.5.1 Provider 分层

| 层 | 接口名 | 职责 | MVP Provider | 扩展 Provider |
|---|---|---|---|---|
| **Runtime** | `RuntimeProvider` | Agent 实例的生命周期管理（创建/启动/停止/销毁） | 单机 Docker | K8s、Nomad、ECS |
| **Sandbox** | `SandboxProvider` | 代码执行隔离、syscall 控制 | gVisor (本地) | E2B、Kata Containers、Firecracker、云厂商沙箱 |
| **Storage** | `StorageProvider` | 文件持久化（Agent 工作区、快照、附件） | 本地目录 | S3、Aliyun OSS、MinIO、GCS |
| **Model** | `ModelProvider` | LLM 调用抽象（chat/completion/embedding） | OpenAI API | Claude、本地模型（Ollama）、Azure OpenAI、各云厂商 LLM |
| **Memory** | `MemoryProvider` | Agent 记忆存储与检索 | SQLite + 本地向量（hnswlib） | PG+pgvector、Redis、Qdrant、云向量DB |
| **Messaging** | `MessagingProvider` | 平台内部异步消息通信 | 内存队列 | NATS JetStream、Kafka、RabbitMQ |
| **IM** | `IMProvider` | 外部 IM 平台对接 | 飞书 | 钉钉、Slack、Discord、企业微信 |

### 14.5.2 核心接口示例

```go
// RuntimeProvider — Agent 运行时管理
type RuntimeProvider interface {
    Create(ctx context.Context, spec AgentSpec) (Instance, error)
    Start(ctx context.Context, instanceID string) error
    Stop(ctx context.Context, instanceID string) error
    Destroy(ctx context.Context, instanceID string) error
    Status(ctx context.Context, instanceID string) (InstanceStatus, error)
}

// StorageProvider — 文件存储抽象
type StorageProvider interface {
    Put(ctx context.Context, key string, data io.Reader) error
    Get(ctx context.Context, key string) (io.ReadCloser, error)
    Delete(ctx context.Context, key string) error
    List(ctx context.Context, prefix string) ([]ObjectInfo, error)
}

// ModelProvider — LLM 调用抽象
type ModelProvider interface {
    Chat(ctx context.Context, req ChatRequest) (ChatResponse, error)
    Embed(ctx context.Context, texts []string) ([][]float32, error)
    ListModels(ctx context.Context) ([]ModelInfo, error)
}
```

每个 Provider 接口保持 **3-5 个核心方法**，从至少 2 个实际实现中提炼，避免过度抽象。

### 14.5.3 平台配置（platform.yaml）

AgentSpec 描述单个 Agent，`platform.yaml` 描述平台级的 Provider 选型与配置：

```yaml
# platform.yaml — 平台 Provider 配置
version: "1"

providers:
  runtime:
    type: docker           # docker | k8s | nomad
    config:
      socket: /var/run/docker.sock

  sandbox:
    type: gvisor           # gvisor | e2b | kata | firecracker
    config:
      runtime_class: runsc

  storage:
    type: local            # local | s3 | oss | minio
    config:
      base_path: /data/agents

  model:
    type: openai           # openai | claude | ollama | azure
    config:
      api_key: ${OPENAI_API_KEY}
      default_model: gpt-4o

  memory:
    type: sqlite           # sqlite | postgres | qdrant
    config:
      db_path: /data/memory.db

  messaging:
    type: memory           # memory | nats | kafka
    config: {}

  im:
    - type: feishu         # feishu | dingtalk | slack | discord
      config:
        app_id: ${FEISHU_APP_ID}
        app_secret: ${FEISHU_APP_SECRET}
```

### 14.5.4 部署拓扑

**单机模式（开发/个人使用）：**
```
Docker Compose 一键启动
├── Platform Service (Go)
├── Agent Runtime (Python, gVisor 容器)
├── SQLite (Memory + 元数据)
├── 本地目录 (Storage)
└── 内存队列 (Messaging)
```

**集群模式（团队/企业）：**
```
K8s 集群
├── Platform Service (Deployment, 多副本)
├── Agent Runtime (Pod per agent, gVisor/Kata RuntimeClass)
├── PostgreSQL + pgvector (Memory + 元数据)
├── S3/OSS/MinIO (Storage)
├── NATS JetStream (Messaging)
├── Redis (缓存)
├── Temporal (Workflow)
└── Kong/APISIX (API Gateway)
```

### 14.5.5 Provider 优先级

实现顺序按对平台可用性的影响排序：

1. **Runtime** — 决定 Agent 能否运行（Docker → K8s）
2. **Storage** — 决定数据在哪里存（本地 → S3/OSS）
3. **Model** — 决定 LLM 调用通路（OpenAI → Claude → 本地）
4. **Sandbox** — 决定隔离级别（gVisor → E2B/Kata）
5. **Memory** — 决定记忆容量与性能（SQLite → PG+pgvector）
6. **Messaging** — 决定通信吞吐（内存 → NATS）
7. **IM** — 决定入口渠道（飞书 → 钉钉 → Slack）

### 14.5.6 设计原则

- **MVP 零外部依赖**：默认 Provider 全部用本地/内嵌实现，`docker compose up` 即可运行
- **渐进式替换**：改一行 `platform.yaml` 即可切换 Provider，无需改代码
- **接口最小化**：先实现再抽象，从 2+ 个具体实现中提炼通用接口
- **Provider 独立发版**：每个 Provider 作为独立 Go module，按需引入，不膨胀核心包

---

## 15. 企业级管理

- **多租户**：Organization → Workspace → Agent 三级隔离
- **RBAC**：Admin / Manager / Developer / Viewer 四角色
- **审计**：操作日志 + 对话审计 + API 调用追踪
- **合规**：PII 脱敏、加密存储、等保就绪
- **计费**：按实例时长 + Token 消耗 + Skill 调用混合计费

---

## 16. 落地路径与 MVP 规划

### Phase 1 — MVP（1-2 周）：证明核心闭环

| 功能 | 优先级 |
|---|---|
| Platform Agent（NL → AgentSpec → 创建） | P0 |
| 飞书 Bot Gateway | P0 |
| Docker 沙箱 + 基础隔离 | P0 |
| 3-5 个基础 Skill（搜索/文件/代码） | P0 |
| 基础记忆（L1/L2） | P1 |

**目标：** 验证 "自然语言描述 → Agent 拉起 → 飞书对话" 核心闭环。

### Phase 2（3-4 周）：多任务 + 持久化

| 功能 | 优先级 |
|---|---|
| Session Router + Thread 映射 | P1 |
| 任务管理（Slash Commands） | P1 |
| 记忆持久化（L2/L3） | P1 |
| 互动卡片（任务面板、配置确认） | P1 |
| 钉钉 Gateway | P2 |

### Phase 3（5-6 周）：协作 + 管理

| 功能 | 优先级 |
|---|---|
| 多 Agent 协作 + Coordinator | P1 |
| Skill Store + 自定义 Skill | P1 |
| Web 管理后台 | P2 |
| 基础评测 Dashboard | P2 |
| RBAC | P2 |

### Phase 4（后续）：企业级

- 多租户完整方案
- 审计日志 + 合规
- A/B 测试 + 自动优化
- Runtime checkpoint（代码执行 agent）
- 白标 / API 嵌入

---

## 17. 成功指标

### MVP 阶段

| 指标 | 目标 |
|---|---|
| Agent 创建成功率 | > 90% |
| 创建到可用时间 | < 60 秒 |
| 首次对话任务完成率 | > 70% |
| 冷启动（休眠→唤醒） | < 5 秒 |

### v1 阶段

| 指标 | 目标 |
|---|---|
| 企业试点 | 3-5 家 |
| 月活 Agent 数 | > 50 |
| 用户满意度 | > 4/5 |
| 多 Agent 协作完成率 | > 60% |

---

## 18. 开放问题

1. **项目名称**：暂定 AgentForge？还是其他？
2. **GitHub Repo**：是否需要新建独立 repo？
3. **MVP 第一个 IM**：先做飞书还是钉钉？
4. **Agent Runtime 语言**：沙箱内 runtime 用 Python 还是也支持其他语言？
5. **模型默认值**：v0 默认用 Claude 还是做 pluggable？→ 已确认：Provider 化，MVP 默认 OpenAI，可切换
6. **开源策略**：完全开源还是 open-core？→ 已确认方向：开源 platform 服务
7. **Provider 接口粒度**：各层 Provider 接口的具体方法定义需要在实现中迭代确认
8. **部署基线**：MVP 单机模式的最低硬件要求？

---

## 贡献者

| 角色 | 负责模块 |
|---|---|
| **@Alice** (PM) | 产品定位、竞品分析、用户旅程、功能优先级、成功指标 |
| **@Eric** (架构师) | 系统分层、组件边界、AgentSpec、沙箱架构、部署拓扑 |
| **@Yoshua** (算法) | NL→Config、DAG调度、Session状态机、记忆检索、评测框架 |
| **@Tody** (前端) | 产品设计、IM交互流程、多Session UX、Slash Commands、MVP排期 |
| **@Jack** (后端) | API设计、数据模型、服务拆分、NATS消息管道、Temporal工作流 |
