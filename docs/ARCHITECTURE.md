# 技术架构与模块边界

状态：目标架构，须由 P0 审计校准。原则：外挂式扩展、渐进迁移、单一人格、权限前置。

## 1. 架构
```text
DingTalk / Web
      |
Channel Adapter
      |
Gateway + Identity Resolver
      |
Companion Core API
 |       |       |       |
Memory Conversation Tasks Policy
      |
Agent Runtime / Tool Router
      |
Worker + Scheduler
      |
PostgreSQL (+ pgvector later)
```

DSH 保留现有对话运行职责；Companion Core 管理长期状态和跨会话能力。通过经过验证的插件事件或适配器交换必要数据，不把 DSH 内部数据库当业务 API。

## 2. 模块职责
- Channel Adapter：规范化平台消息、发送回复和主动提醒，保留群聊、@、附件等平台能力。
- Gateway & Identity：验证事件、生成 request_id/event_id、解析用户/群组/会话身份、幂等处理。
- Companion Core：统一编排人格、上下文、记忆、工具和模型；对外始终是大肥鱼一个人格。
- Memory：提取、归档、检索、冲突检测、版本和过期；检索前 ACL 过滤。
- Agent Runtime：路由 Research、Planner、Browser 等工具；工具声明权限、超时、成本和副作用。
- Worker/Scheduler：异步任务、重试、续跑和定时任务；初期可合并进程。

## 3. 候选技术栈
| 领域 | 候选 | 原则 |
|---|---|---|
| 运行时 | Node LTS + TypeScript | 先核对现有 DSH 运行时 |
| API | Fastify | P1 再引入 |
| 数据库 | PostgreSQL | 业务状态主存储 |
| 向量 | pgvector | P2 评估后启用 |
| 队列 | PostgreSQL 任务表 | 有瓶颈再加 Redis |
| 浏览器 | Playwright | P5，沙箱和授权先行 |
| 部署 | Docker Compose | 新服务器独立部署 |

Tobs、Mem0 等仅作为专项候选。避免同时引入多个 Agent 编排框架或记忆中心；须核对版本、维护、许可证、成本和真实集成结果。

## 4. 数据边界
PostgreSQL 是 Companion Core 业务事实来源。DSH 配置和运行数据仍由 DSH 管理。钉钉凭据由现有插件/安全配置管理。原始消息、记忆摘要和任务记录分层保存并设置保留周期。向量索引不替代权限和来源。

## 5. 消息流程
1. 钉钉插件收到消息。
2. Adapter 验证并规范化事件，保留平台 ID 和会话范围。
3. Identity Resolver 解析内部身份。
4. Policy Engine 建立权限上下文。
5. Core 加载人格、近期消息和 ACL 过滤后的记忆。
6. Agent Runtime 决定直接回答或调用工具。
7. 回复经已验证的钉钉发送链路发出。
8. 原始事件和必要状态异步持久化，记忆提取异步执行。

新增同步依赖必须有超时和降级；记忆写入失败不应阻断普通聊天。

## 6. 初步 API
统一使用 /v1，并携带 request_id：
- POST /v1/identity/resolve
- POST /v1/events/inbound、/v1/events/outbound
- GET/POST /v1/conversations、/v1/messages
- POST /v1/memories/search、/v1/memories/extract
- GET/PATCH /v1/memories/:id、GET /v1/memories/:id/sources
- POST /v1/tasks、GET/PATCH /v1/tasks/:id、POST /v1/tasks/:id/resume
- POST /v1/proactive/evaluate
- GET/POST /v1/approvals
- GET /v1/health、/v1/usage

接口细节在 P1 用 OpenAPI 固化。不能信任请求体中未经验证的 owner_id、group_id 或权限声明。

## 7. 可靠性
事件唯一键去重；任务状态采用 queued、running、waiting_approval、succeeded、failed、cancelled；重试有上限和退避；外部副作用使用幂等键；Core 不可用时 bridge 可跳过增强功能并保留原链路。