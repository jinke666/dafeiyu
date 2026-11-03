# DSH 与 Agent Runtime 兼容策略

## 1. 已知基线
- DSH 主版本范围：0.2.x，补丁版本待核实。
- 人格插件 dsh-persona-dafeiyu 已安装，版本/路径待核实。
- 钉钉通过独立插件接入，准确名称、版本、仓库和接口待核实。
- 曾提及腾讯云 2GB RAM 环境，是否仍为当前生产环境待确认。

这些是规划输入，不等于已验证的运行时事实。

## 2. 原则
1. 不因新项目直接升级 DSH。
2. 不修改 DSH 核心源码，不直接写 DSH 内部数据库。
3. 不替换现有人格插件或钉钉插件。
4. 只使用当前版本真实存在且经过验证的事件接口。
5. 先只读审计、备份，再隔离环境 PoC，最后才考虑生产接入。
6. 增强服务故障时原聊天链路可降级。
7. 不把适用于旧版本的第三方插件直接安装到 0.2.x。
8. DSH 只是当前候选 Runtime；未来 DeepSeek 官方 Agent 产品通过独立 Adapter 接入，不能成为 Core 数据和权限的唯一所有者。

## 3. 当前 dsh-superagent-prototype 评估
官方公开原型声明为研究用途，不是官方产品，不保证生产就绪。其 package.json 声明 `engines.dsh: ^0.2.1-alpha.2`，依赖多个 0.2.1-alpha.2 DSH 组件；SETUP/部署文档示例使用 DSH 0.2.1-rc.3。用户环境只确认 DSH 0.2.x，精确补丁未知，因此当前不能认定兼容。

不在现有 profile 安装原型；不执行会覆盖 profile 的初始化命令；不把原型 SQLite 数据库与 Companion Core 数据库混用。可借鉴任务状态机、durable effects、Outbox、Worker 隔离、恢复与 Observe-first Policy。

详细源码审查见 [迁移预案](SUPERAGENT-REVIEW-AND-MIGRATION.md)。

## 4. 建议集成
新增独立 dafeiyu-bridge：订阅已验证的 DSH 会话/消息事件，转换为 Core 内部事件；从 Core 获取增强结果；通过已验证回复接口返回。不得覆盖原钉钉入站/出站流程，除非完成专项回归测试。

Runtime 与 Channel 解耦：DingTalk Adapter 只负责消息输入输出，不依赖某个 DSH Session ID。Agent Runtime Port 负责启动/恢复/取消任务；未来官方 Runtime 使用独立 adapter。

若 DSH 当前事件接口不足，先用独立 Webhook/API 或只读旁路记录；不要用脆弱的内部文件监听做生产双向同步。

## 5. P0 采集信息
OS/架构/运行用户；DSH 完整版本和安装路径；Node/npm/pnpm 版本；DSH_HOME/Profile/启动方式；人格插件版本、路径、来源；钉钉插件名称、版本、仓库、入口和收发机制；当前群聊、@、附件、主动发送能力；备份路径、校验和与恢复结果。

环境报告可能含敏感信息，分享前脱敏；密钥和完整配置不得提交 GitHub。

## 6. 验收
私聊、群聊、@、图片/文件等原有能力无回退；重复事件不重复回复；Core 超时/宕机时原链路可用；重启后任务恢复；群组身份不串；禁用 bridge 后能回到原始行为；替换 Runtime 后记忆、权限、任务和审计不丢失。

## 7. 回滚
变更前记录版本和配置校验和；备份 DSH、人格插件、钉钉插件及必要数据；一次只发布一个可关闭变更；验收失败即禁用 bridge 或恢复备份，不叠加未经验证的升级。
