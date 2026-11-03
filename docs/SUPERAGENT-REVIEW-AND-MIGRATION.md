# DeepSeek Super-agent 源码审查与官方产品迁移预案

状态：架构决策基线（2026-11-03）。本文件基于 DeepSeek 官方公开仓库 `deepseek-ai/dsh-superagent-prototype` 的 README、package.json、SETUP、Domain model、Authorization、Execution、Architecture，以及关键源码文件审查。它是研究原型审查，不代表已在用户的 DSH 环境完成安装或兼容测试。

上游：
- https://github.com/deepseek-ai/dsh-superagent-prototype
- MIT License
- 上游自述：research prototype for reference only, not an official DeepSeek product；不是生产就绪产品，公开代码与内部版本/文章实现不完全相同，部分设计尚未实现。

## 1. 结论

**采用“自有 Companion Core + 可替换 Agent Runtime + 可替换 Channel Adapter”的路线。**

不把大肥鱼直接建立在 dsh-superagent-prototype 之上，也不把 DSH 内部 Session、SQLite schema、Cordis service 或插件 UI 作为大肥鱼长期数据契约。

复用其架构思想：
- Trigger / Thread / Eval / Decision / Mandate / Ticket / Assignment 的任务领域模型；
- Policy gate 与模型 assessment 分离，授权先于推理；
- Engine 作为领域状态唯一写入者，事务内变更状态、事务后执行副作用；
- durable effects、Outbox receipt、幂等键、重试退避；
- 每个工作单元独立 Worker Session / workspace；
- 重启恢复、并发槽位、健康状态、Observe-first 主动策略；
- Source Adapter / Executor Port / Assessor Port 等边界。

不直接复用：
- GitHub / Feishu 的业务 Source 实现和账号授权流程；
- GitHub PR、CI、review、auto-merge 规则；
- Coding Agent 专用提示词、工具集和面板；
- DSH Session 作为 Ticket 的持久化权威；
- 上游 SQLite schema 和迁移脚本；
- 其 Cordis 插件装配与 UI bundle 结构。

原因：大肥鱼是个人/群聊数字伙伴，不是单仓库 Coding Agent；它必须有 DingTalk、个人记忆 ACL、人格、研究计划、生活任务和未来 Computer 能力。

## 2. 上游结构审查

### 2.1 领域流

上游主流程：
`Source → Trigger → Thread → Eval → Decision → Ticket(Mandate) → Assignment → Executor → Channel`

- Source 负责事件接入、去重游标和渠道地址。
- Trigger 是不可变事件，是审计和幂等单位。
- Thread 聚合会话上下文；同一 Thread 的 Eval 串行。
- Eval 分 Gate（确定性准入）和 Assessment（模型研判）。
- Mandate 记录授权依据、范围、Policy 版本。
- Ticket 是可恢复的工作单元；Assignment 是某次执行尝试。
- Executor 在领域模型之外，通过接口执行任务。
- Channel 是回复出口。

**大肥鱼映射：**
- Source → DingTalk / Web / Scheduler / Email 等 Adapter。
- Trigger → 入站消息、定时事件、任务到期、外部状态变化。
- Thread → 私聊会话、群会话、任务线程、项目线程。
- Eval → 是否响应、是否创建任务、是否触发主动行为。
- Mandate → 主人授权、群权限、任务授权和有效期。
- Ticket → Research / Planning / Reminder / Computer 工作单。
- Assignment → Worker 的一次执行。
- Channel → DingTalk 私聊/群聊、Web 通知等。

注意：上游 Thread 的授权语义围绕 GitHub/Feishu 对话，不可直接照搬为大肥鱼的记忆访问权限。记忆 ACL 必须独立于任务订阅关系。

### 2.2 代码边界

审查到的关键实现：
- `src/core/ports.ts`：Policy、Assessor、Executor、Outbox、SourceScope 等端口定义。
- `src/core/engine.ts`：Engine 是 Trigger、Thread、Decision、Ticket、Assignment 的唯一写入入口，命令在 repository transaction 内执行。
- `src/app/runtime.ts`：组合根，构建 Store、Engine、SourceRegistry、Effects、Outbox、Passes、Sessions、Executor、Assessor；启动时锁定 state directory。
- `src/app/effects.ts`：SQLite 持久化副作用队列，按 kind/key 幂等合并，支持顺序、并发限制、退避与健康状态。
- `src/app/store.ts`：SQLite/WAL、聚合存储、版本化设置、Outbox receipts。
- `src/app/outbox.ts`：先保存发送 receipt 并排队，再经 Source 发出；支持 pending/sent/dropped 与重试。
- `src/app/migration.ts`：显式 schema 版本迁移；当前实现只接受特定旧 schema 范围，过旧/过新的库拒绝启动。

重要启示：可复用的是领域规则和端口设计，不是上游的具体 Store。大肥鱼需要自己的 schema、数据导出和迁移层。

### 2.3 授权与主动行为

上游 Policy 将“谁可以发起工作”与“模型是否认为值得做”分开。Observe 模式会记录“如果开启自动接单本会接单”的判断，但不执行推断型任务；只有明确请求可在 Observe 下正常执行。

大肥鱼采用更严格的默认值：
- 默认 Observe；主动规则先记录候选，不直接行动。
- 私聊、群聊、定时任务分别定义授权范围。
- 群聊内容不能自行提升为个人记忆或全局事实。
- 群消息中的提示词、链接、附件和引用均为不可信输入，不能改变系统权限。
- 对外发送、修改数据、提交表单、购买/付款、删除等副作用按风险等级审批。
- 所有授权应有主体、资源范围、动作、有效期、撤销方式和审计记录。

## 3. 大肥鱼目标架构

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
    |             |                |
    +--------- Context Builder -----+
                  |
          Agent Runtime Port
          /       |        \
    DSH Runtime  Official   Other Runtime
                 Agent
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

Companion Core 是产品和数据权威；Agent Runtime 是可替换的推理/执行后端。任何 Runtime 都只能通过受控接口读取上下文、请求工具和提交结构化结果，不能直接持有数据库管理员权限。

### 3.1 必须稳定的内部契约

先定义版本化、与供应商无关的接口（命名为概念契约，不要求第一版全部实现）：

- `AgentRuntime.run(task, context, capabilities)`：执行一次有边界的任务。
- `AgentRuntime.resume(runId, input)`：恢复任务。
- `AgentRuntime.cancel(runId)`：取消任务。
- `AgentRuntime.health()`：能力和健康状态。
- `ModelProvider.generate(...)`：模型调用，与 Agent Runtime 分开。
- `ToolProvider.invoke(tool, args, authContext)`：工具调用经过权限代理。
- `ChannelAdapter.receive(event)` / `send(message)`：渠道输入输出。
- `MemoryProvider.search(query, principal, scope)` / `write(candidate, provenance)`：记忆检索/写入。
- `TaskStore` / `EventStore` / `Outbox`：持久化状态和可靠副作用。

契约应包含：
- `contract_version`、`capability_manifest`、`request_id`、`task_id`、`idempotency_key`；
- 结构化状态、错误码、超时、取消和重试语义；
- 输入/输出 JSON Schema；
- 工具权限范围和副作用等级；
- 可观测的 token、成本、耗时与来源元数据。

不得把某一供应商的 Session ID 当作全局 task ID。需要保存映射：`internal_task_id ↔ runtime_kind/runtime_instance_id/external_run_id`。

### 3.2 数据所有权

Companion Core 永远持有：
- 用户、平台账号、群组、会话和 ACL；
- 原始事件索引、长期记忆、记忆来源、版本和纠错记录；
- 任务、授权 Mandate、计划、审批、审计和 Outbox；
- 人格版本、偏好和技能登记表；
- Runtime 映射、迁移 checkpoint 和兼容状态。

Runtime 只持有执行所需的临时上下文、运行日志和工作目录。允许缓存，不允许成为唯一数据源。

记忆导出采用供应商无关格式（JSONL + 附件清单，必要时 Markdown 可读导出）；包含 schema version、稳定 ID、ACL、来源、时间、置信度、状态和引用关系。向量 embedding 不作为唯一可迁移内容，迁移时可重建。

## 4. 官方产品出现后的迁移设计

未来若 DeepSeek 发布官方 Super Agent / Agent Runtime，执行“并行接入 → 影子验证 → 小流量切换 → 分类型迁移 → 旧 Runtime 退役”，不是原地覆盖。

### 阶段 A：官方能力评估
核对官方产品是否支持：
- 稳定 API/SDK、服务端部署和版本策略；
- 外部消息渠道或标准 Webhook；
- 独立任务、异步运行、暂停/恢复/取消；
- 工具调用授权、MCP 或等价工具接口；
- 任务事件/运行状态回调；
- 外部记忆或自定义上下文；
- 数据导出、删除、备份和迁移；
- 企业级密钥隔离、审计和权限；
- 许可证、商用限制、地域可用性和成本。

缺少其中关键项时，只替换适配器或模型层，不强行替换 Core。

### 阶段 B：适配器与能力映射
新增 `OfficialDeepSeekRuntimeAdapter`，实现内部 AgentRuntime 契约。先做 capability mapping：
- 支持的能力 → 映射到内部标准能力；
- 不支持的能力 → 由 Core 保留或由现有 Worker 补足；
- 语义不一致 → 写兼容层，不伪装为完全兼容；
- 需要外部数据的功能 → 通过 Core API 受控提供。

### 阶段 C：影子与双跑
- 复制脱敏任务样本到官方 Runtime 做离线评测，不复制无关私聊/群聊隐私。
- 对比任务成功率、事实准确性、工具调用、权限遵循、成本、延迟和可恢复性。
- 先对只读 Research/Planning 任务影子运行。
- 不允许两个 Runtime 同时执行同一项有副作用任务。
- 每个 task 通过 lease + fencing token 保证同一时刻只有一个执行者；切换前旧 Worker 必须停止或确认已失去租约。

### 阶段 D：逐步切换
按能力和任务类型切换，不按整个 Agent 一次性切换：
1. 只读问答/Research；
2. Planning；
3. 提醒和低风险主动任务；
4. 受控工具调用；
5. Computer 写操作（最后迁移）。

保留每类任务的 Runtime 路由开关和回滚开关。切换失败时将未完成任务恢复到旧 Runtime，前提是任务状态可序列化且工具副作用已核对。

### 阶段 E：数据和会话迁移
- Core 数据库不迁移到官方 Runtime；继续作为大肥鱼数据主库。
- 导出正在运行的任务快照、计划步骤、已完成结果、待处理输入和审批状态。
- 外部 Session 历史若不能导出，不把它视为迁移阻断项；用 Core 保存的摘要、事件和任务状态重建上下文。
- 记忆通过标准导出格式导入/重建；权限与来源元数据必须完整保留。
- 逐步废弃旧 Runtime 的执行映射，保留审计记录和必要归档。

## 5. 技术选型调整

| 层 | 选择 | 说明 |
|---|---|---|
| Companion Core | TypeScript / Node LTS | 供应商无关业务层 |
| Domain Engine | 自建轻量 Engine + Ports | 借鉴上游状态机，避免引入完整 Agent 框架 |
| Database | PostgreSQL | 长期业务事实来源；事务/约束/备份 |
| Queue | PostgreSQL durable jobs / outbox | 初期不引入 Redis |
| Vector | pgvector optional | 先 ACL 过滤和关键词检索，后评估向量 |
| Current Runtime | DSH adapter | 兼容性经 P0/P1 实测后接入 |
| Future Runtime | Official DeepSeek adapter | 未来可插拔，当前不预设其接口 |
| Channel | 独立 DingTalk adapter | 不与 Runtime 绑定 |
| Computer | 隔离 Browser/Computer worker | 单独权限、凭据、网络和审计边界 |

## 6. 当前 DSH 兼容风险

上游当前 package.json 声明：
- `engines.dsh: ^0.2.1-alpha.2`
- peer dependencies 多数为 `@deepseek-ai/* 0.2.1-alpha.2`
- Node `^22.19.0 || >=24.0.0`
- 安装文档示例固定 DSH `0.2.1-rc.3`

用户环境目前只确认到 DSH 0.2.x，精确版本未知。因此：
- 不能推断兼容；
- 不要在现有 profile 安装原型；
- 不要运行其 profile 初始化命令覆盖配置；
- 不要把其 SQLite 数据库与大肥鱼业务数据库混用；
- 先采集 `dsh --version`、Node 版本、插件清单和独立 profile 状态，再用隔离 profile/测试机验证。

## 7. 分阶段实施更新

- P0：保持只读环境审计和备份；收集 DSH 精确版本及插件入口。
- P1：先建立 Core 契约、Runtime Port、DingTalk Adapter contract 和持久化任务 PoC；暂不绑定具体 DSH 内部接口。
- P2：记忆与 ACL 由 Core 独立拥有，提供标准导入/导出。
- P3：Research/Planning 通过 AgentRuntime 与 ToolProvider 运行。
- P4：Proactive 基于 Trigger/Policy/Observe + durable scheduler。
- P5：Computer/Skill Sandbox 独立执行，未来 Runtime 可替换。
- P6（新增）：官方 Runtime 兼容评估、影子测试、灰度迁移和旧 Runtime 退役。

## 8. 验收清单

- [ ] 关闭/替换 DSH Runtime，不丢失用户记忆、人格、任务、授权和审计。
- [ ] 更换模型供应商不改变数据 schema。
- [ ] DingTalk Adapter 不依赖某个 Runtime 的 Session ID。
- [ ] 同一任务切换期间不会双重执行外部副作用。
- [ ] 任务可从 Core 状态恢复，而非只依赖 Runtime 历史对话。
- [ ] 私聊/群聊 ACL 在 Runtime 切换前后结果一致。
- [ ] 记忆 JSONL 导出后可在空环境重建并通过权限测试。
- [ ] 官方产品不支持某能力时，Core 能保留该能力或明确降级。
- [ ] 可按任务类型回滚 Runtime。
- [ ] 全过程保留迁移日志、审批记录和版本信息。
