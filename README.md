# 大肥鱼（Dafeiyu）

大肥鱼是一个长期在线数字伙伴型混合 Agent 项目，以稳定人格和长期记忆为基础，逐步具备研究、规划、主动互动、Computer 操作与受控成长能力。

> 当前状态：方案与兼容性验证阶段（P0）。本文描述目标，不代表所有模块已经实现。

## 目标能力
- Personality：稳定、统一、可版本化的人格。
- Memory：个人、群聊、项目和经历记忆，来源可追溯。
- Research：资料检索、交叉验证、引用和不确定性说明。
- Planning：目标拆解、依赖、时间、风险和验收标准。
- Proactive：授权范围内主动提醒和跟进。
- Computer：在明确授权、可审计、可回滚的范围内操作网页或桌面。
- Growth：积累经验、起草技能；权限和核心环境变更必须审批。

## 当前基础与兼容原则
当前已知基础为 DSH 0.2.x、已安装的人格插件 dsh-persona-dafeiyu，以及独立钉钉插件。具体补丁版本、插件版本、路径和事件接口仍需 P0 审计确认。

采用渐进式外挂架构：
1. 保留现有 DSH、人格插件和钉钉插件。
2. 新建 Companion Core，逐步管理身份、记忆、任务、权限和主动行为。
3. Agent Runtime 与 Channel Adapter 解耦；当前 DSH、未来官方 Agent 产品均通过独立 Adapter 接入。
4. Runtime 可替换，Core 数据、人格、记忆、任务、权限和审计不依赖供应商。
5. 新服务故障时优先降级为原有聊天链路。
6. 未完成兼容验证前，不升级 DSH、不替换钉钉插件。

## 总体架构
```text
DingTalk / Web / Scheduler
      |
Channel Adapters
      |
Gateway & Identity
      |
Companion Core
 |      |       |
Memory Planner Proactive
 |      |       |
Research / Agent Runtime Port / Skill Manager
      |
DSH Runtime / Future Official Runtime
      |
Tool Provider / Isolated Workers
      |
PostgreSQL (+ pgvector later)
```

## 目标目录
```text
apps/       gateway、core、worker、admin
adapters/   dsh、dingtalk、web、agent-runtime/*
packages/   identity、memory、permissions、conversation、tasks、proactive
database/   migrations、seeds、exports
deploy/     Docker Compose 与部署模板
tests/      单元、集成、权限和恢复测试
docs/       产品、架构、兼容和运维文档
```

## 开发阶段
| 阶段 | 目标 |
|---|---|
| P0 | 环境审计、备份、兼容性验证 |
| P1 | Core、稳定契约、身份映射、事件桥接 |
| P2 | 长期记忆与权限隔离、标准导入/导出 |
| P3 | Research 与 Planning |
| P4 | 主动行为 |
| P5 | 受控成长与 Computer |
| P6 | 官方 Runtime 兼容评估与灰度迁移 |

## 安全底线
- 密钥、令牌、环境报告、数据库和备份不得提交 Git。
- 身份和权限由服务端解析，不信任客户端传入的 owner/group。
- 群聊记忆按群隔离；个人私聊信息不得进入群聊检索范围。
- 外部操作和技能执行有权限边界、审批和审计。
- 自动学习不等于自动获得权限。
- Runtime 不得直接拥有 Core 数据库管理员权限。

## 文档
- [产品需求](docs/PRD.md)
- [技术架构](docs/ARCHITECTURE.md)
- [记忆与权限](docs/MEMORY-DESIGN.md)
- [DSH 与 Runtime 兼容策略](docs/DSH-COMPATIBILITY.md)
- [开发路线图](docs/ROADMAP.md)
- [部署规划](docs/DEPLOYMENT.md)
- [安全设计](docs/SECURITY.md)
- [P0 检查清单](docs/P0-CHECKLIST.md)
- [DeepSeek Super-agent 源码审查与官方产品迁移预案](docs/SUPERAGENT-REVIEW-AND-MIGRATION.md)

目前仓库只承载设计文档和实施基线。生产部署、依赖安装和数据迁移须等待 P0 审计结果。
