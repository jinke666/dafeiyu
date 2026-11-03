# 技术架构与模块边界

状态：目标架构，须由 P0 审计校准。原则：外挂式扩展、渐进迁移、单一人格、权限前置、Runtime 可替换。

## 1. 架构
```text
DingTalk / Web / Scheduler / Future Channels
                  |
           Channel Adapters
                  |
        Canonical Event Gateway
                  |
        Identity & ACL Resolver
                  |
          Companion Core
    +-------------+----------------+
    |             |                |
 Personality   Memory          Task Engine
    |          + ACL          Trigger/Eval/
    |             |           Ticket/Assignment
    +--------- Context Builder -----+
                  |
          Agent Runtime Port
          /       |        \
     DSH Runtime Official Runtime Other Runtime
                  |
          Capability / Tool Ports
       Research / Planner / Browser /
          Computer / Skill Sandbox
                  |
       Durable Store + Outbox + Queue
                  |
        PostgreSQL (initial target)
        pgvector (optional, later)
```

Companion Core 是产品、权限和数据权威；Agent Runtime 是可替换的推理/执行后端。DSH 保留当前对话运行职责，后续通过 DSH adapter 接入。官方 DeepSeek Agent 产品未来通过独立 adapter 接入，不替换 Core 或主数据库。

## 2. 模块职责
- Channel Adapter：规范化平台消息、发送回复和主动提醒，保留群聊、@、附件等平台能力。
- Gateway & Identity：验证事件、生成 request_id/event_id、解析用户/群组/会话身份、幂等处理。
- Companion Core：统一编排人格、上下文、记忆、任务、权限和工具；对外始终是大肥鱼一个人格。
- Memory：提取、归档、检索、冲突检测、版本和过期；检索前 ACL 过滤；提供供应商无关导入/导出。
- Task Engine：管理 Trigger、Eval、Decision、Mandate、Ticket、Assignment 和生命周期；Core 是任务状态权威。
- Agent Runtime Port：统一启动、恢复、取消任务和能力声明；不得直接访问 Core 数据库。
- Tool Provider：Research、Planner、Browser、Computer 等工具的统一权限代理。
- Worker/Scheduler：异步任务、重试、续跑和定时任务；初期可合并进程。
- Outbox：可靠发送队列，幂等、重试、顺序与回执。

## 3. 候选技术栈
| 领域 | 候选 | 原则 |
|---|---|---|
| 运行时 | Node LTS + TypeScript | 先核对现有 DSH 运行时 |
| API | Fastify | P1 再引入 |
| 数据库 | PostgreSQL | 业务状态主存储 |
| 向量 | pgvector | P2 评估后启用 |
| 队列 | PostgreSQL 任务表 + Outbox | 有瓶颈再加 Redis |
| 浏览器 | Playwright | P5，沙箱和授权先行 |
| 部署 | Docker Compose | 新服务器独立部署 |
| Runtime | DSH adapter / Future official adapter | 只通过稳定内部契约接入 |

不同时引入多个 Agent 编排框架或记忆中心。大肥鱼自有 Core 和数据模型，不因某个 Runtime 更换数据库或重写业务。

## 4. 稳定契约与供应商隔离
内部定义版本化 AgentRuntime、ModelProvider、ToolProvider、ChannelAdapter、MemoryProvider 接口。契约包含 contract_version、capability_manifest、request_id、task_id、idempotency_key、结构化状态、错误码、超时、取消、重试、权限范围和成本元数据。

内部 task_id 是唯一任务标识；外部 DSH Session ID 或官方产品 Run ID 只作为映射字段保存，不得成为主键或跨模块契约。Runtime 只接收最小任务上下文，通过 Tool Provider 请求工具，不直接持有数据库管理员权限。

## 5. 数据边界
PostgreSQL 是 Companion Core 业务事实来源。DSH 配置和运行数据仍由 DSH 管理。钉钉凭据由现有插件/安全配置管理。记忆、任务、授权、人格版本、审批、审计和 Outbox 归 Core 持有。外部 Runtime 历史不可导出时，可由 Core 的任务快照、事件、摘要和记忆重建执行上下文。

记忆导出采用带 schema version 的 JSONL + 附件清单，保留稳定 ID、ACL、来源、时间、置信度、状态和引用关系；向量 embedding 可重建，不作为唯一内容。

## 6. 消息与任务流程
1. 钉钉插件收到消息，DingTalk Adapter 验证并规范化事件。
2. Identity Resolver 解析内部身份与会话范围。
3. Policy Engine 建立授权上下文，记忆检索先做 ACL 过滤。
4. Core 加载人格、近期消息和必要记忆。
5. Task Engine 决定直接回复或创建持久化 Ticket。
6. Agent Runtime 通过 Runtime Port 执行；工具请求经过 Tool Provider。
7. 结构化结果返回 Core，Outbox 经已验证钉钉发送链路发出。
8. 任务、事件和审计异步持久化；记忆提取异步执行。

## 7. 可靠性与迁移
事件唯一键去重；任务状态采用 queued、running、waiting_approval、succeeded、failed、cancelled；副作用使用幂等键、Outbox 和租约/fencing token；重启后从 Core 恢复。Runtime 替换时先影子测试，再按任务类型灰度；同一副作用任务任何时刻只允许一个有效 Worker。Core 不可用时 bridge 可降级并保留原聊天链路。

官方产品迁移预案、上游源码审查和验收清单见 [DeepSeek Super-agent 源码审查与官方产品迁移预案](SUPERAGENT-REVIEW-AND-MIGRATION.md)。
