# PingRunner Benchmark 版 PRD（基于 ScriptRunner for Jira Cloud 深度研究）

> 本文档只覆盖 **Ping Runner**。高级工作流、Ping Clone / Ping Clone++、飞书推送（SLA）等章节见原 PRD `PingCode能力PRD.md`，本版不改。
>
> 来源：用户提供的《ScriptRunner for Jira Cloud 深度研究：作为 PingRunner 的产品与技术 Benchmark》PDF（15 页，截至 2026-10-30），主要一手资料为 Adaptavist ScriptRunner Cloud 官方文档、Atlassian Forge 官方限制与共享责任模型文档、PingCode Developer Center 公开文档。

## 1. 一句话定位

PingRunner 是基于 PingCode Nexus 的**管理员级脚本与业务扩展产品层**，让管理员不开发完整 Nexus App 就能用脚本扩展工作流、事件自动化与数据处理。

三层边界（本版最重要的架构结论）：

```text
PingCode 产品 = 提供稳定 Extension Contract 的业务平台
Nexus        = Forge / P2 级扩展基础设施（Functions / Events / Queue / Storage / Exposer / UI 扩展点）
PingRunner   = ScriptRunner 级脚本产品层（Script / Trigger / Context / Run-As / Editor / History / Logs）
```

**不是**「PingRunner 绕过 Nexus 直连 PingCode 内部服务」——那会重蹈 Jira DC 插件时代的高耦合，失去 Cloud 平台的安全与演进优势。

## 2. 为什么不是照抄 ScriptRunner 菜单

ScriptRunner Cloud 的产品结构可以拆成四层，这是它真正值得 Benchmark 的地方：

| 层 | ScriptRunner 对应 | PingRunner 落点 |
| --- | --- | --- |
| 开发层 | Script Console、Script Manager | 脚本仓库、脚本控制台 |
| 绑定层 | Listener / Scheduled Job / Workflow Rules / Scripted Field | 触发器绑定 |
| 运行层 | Groovy Runtime、HAPI、Run-As | 运行身份与密钥（Run-As / Secrets / Sandbox / Timeout） |
| 治理层 | Execution History、Logs、Audit Logs | 运行日志（五分类）、审计 |

但 ScriptRunner 历史形成的「模块菜单驱动」信息架构（先分别进入 Listener / Job / Workflow / Field 各自页面）**不必复制**。MVP 只保留一个核心概念：**Script**。

一份脚本的完整流程：

```text
基本信息 → 触发器 → 上下文 → 代码 → 权限 → 测试 → 发布 → 历史
```

## 3. Cloud vs DC：执行架构的根本差异

| 维度 | Jira DC | Cloud（Connect → Forge Remote） |
| --- | --- | --- |
| 运行位置 | Jira JVM 内 | Jira 进程外（Forge 模块 + 远端 Runtime） |
| Jira Java API | 可直接访问 | 不可，必须走 REST / HAPI |
| 同步事务能力 | 接近内核 | 明显受平台约束 |
| 核心风险 | 插件影响 JVM | 平台限制、事件顺序、延迟、配额 |

已公开的平台风险（Forge 迁移期）：Post Function / Listener 可能延迟数分钟；同一工作项多个 Post Function 顺序可能变化；被禁用的 Workflow Rule 可能在保存后重新启用；Workflow Condition 在 transition screen 场景可能基于错误数据状态执行。

**这直接导出本版最重要的反面 Benchmark**：不要让 Runner 的业务语义依赖一个不保证 latency / order / transaction 的通用异步平台。同步工作流扩展、事件异步扩展、普通脚本任务必须在**平台契约层**被明确区分。

## 4. 原生平台必须开放什么（Runner 替代不了的部分）

| 原生能力 | Runner 用途 | 可否由 Runner 自己替代 |
| --- | --- | --- |
| Workflow Condition Hook | 决定转换是否可用 | 不能，必须原生 |
| Workflow Validator Hook | 阻止转换并回传错误 | 不能，必须原生（Nexus 已有「工作项-工作流验证」） |
| Workflow Post-Transition Hook | 转换后动作 | 不能完全替代 |
| Form / UI Modification API | 动态表单行为 | 不能 |
| UI Extension Slots | 界面扩展 | 不能 |
| Product Event Bus | 事件监听 | 不能 |
| Domain REST APIs | 脚本读写业务数据 | 底层必须原生，Runner 只封装 |
| Identity & Permission Context | Run-As | 不能 |
| Custom Field / 计算字段生命周期 | 脚本字段 | 部分可模拟，性能与索引取决于平台 |
| Search / Query Extension | PQL 函数 | 若无原生扩展，只能影子索引 |
| Scheduler / Async Runtime | 定时任务 | 可由 Nexus 提供 |
| Secure Storage | 密钥 | 可由 Nexus 提供 |
| Custom API Exposer | 自定义 REST | 可由 Nexus 提供（此项 PingCode 领先标准 ScriptRunner Cloud） |

### Nexus GAP 优先级

| 优先级 | 能力 | 当前公开状态 | 需要补什么 |
| --- | --- | --- | --- |
| P0 | Workflow Validator | 存在（expression / resolver） | transition 表单新值、错误字段定位、timeout、同步 SLA |
| P0 | Workflow Condition | 未找到明确独立扩展点 | 原生同步 Hook，控制转换可用性 |
| P0 | Post Function | 部分存在（自动化 Action） | transition 生命周期 hook、执行顺序、同步 / 异步语义 |
| P0 | Event Bus | 存在 | delivery guarantee、retry、ordering、dedup、loop protection |
| P0 | Function Runtime | 存在 | 脚本沙箱、CPU / 内存 / 时间限制、kill、context 注入 |
| P0 | Async Queue | 存在 | retry / backoff / DLQ / 幂等元数据 / 运行历史 |
| P0 | Run-As | 部分存在（userToken / appToken） | 统一 current-user / app / 指定用户身份模型 |
| P0 | Execution Observability | 未知 | correlation ID、执行记录、duration、status、trigger |
| P1 | Form Behaviour | 未找到等价 API | onLoad / onChange、字段 visible / required / options |
| P1 | Computed / Script Field | 未知 | 字段计算生命周期、物化、索引、刷新 |
| P1 | UI Fragments | 已有大量扩展点 | 把扩展点动态脚本化，而不是要求改 App manifest |
| P1 | Scheduled Trigger | SDK 有 scheduled Event | cron、时区、misfire、并发、Run Now |
| P1 | Secrets | 待确认 | 加密变量 + 脱敏 + 轮换 |
| P2 | Query Extension | 未知 | 查询函数 / 索引扩展，或 Runner 侧影子索引 |
| P2 | Remote Runtime | 存在 | 客户自托管 runtime：治理、签名、数据驻留 |

## 5. MVP 范围与路线图

**P0（本版原型已覆盖）**

| 模块 | 能力 | 验收要求 |
| --- | --- | --- |
| 脚本仓库 | Script / Folder / Draft-Published 版本、Clone | 版本可回滚；改草稿不影响已发布 |
| 脚本控制台 | Run / Cancel / Context / Output | 能执行一次 CRUD / API 操作，可终止 |
| Workflow Validator | 绑定项目 / 类型 / Transition | 返回 error 时同步阻止流转 |
| Post Action | 转换成功后执行 | 明确异步；失败不把业务状态回滚为「假成功」 |
| Event Listener | workitem created / updated / comment | eventId、retry、loop detection |
| Scheduled Job | interval / cron、Run Now | 时区、timeout、并发策略 |
| Runtime | sandbox、API SDK、context、Run-As | 不可访问宿主机 / 其他租户数据 |
| Secrets | 加密变量 | UI 永不回显明文，日志自动脱敏 |
| Test Runner | 模拟 / 选择真实工作项 Context | 可在发布前运行 |
| 运行日志 | status、trigger、actor、duration、correlationId | 所有 P0 触发器统一记录 |

**不建议首版做**：Behaviours、Scripted Fields、PQL 查询函数、UI Fragments、HTTP Endpoint。原因不是不重要，而是它们分别要求 Form Engine、Field lifecycle、Query Engine、UI Extension、API Gateway 提供更成熟的原生能力；先把通用 Runtime / Trigger / Repository / Observability 做扎实，后续模块才能复用同一底座。

**P1**：动态表单行为、脚本字段、界面扩展、批处理 / Escalation Service、共享依赖库、Git / API 导出、环境 promotion、发布审批。
**P2**：PQL 查询函数、自定义 REST 端点、第三方 Runtime、高级 profiler、AI 编码助手、完整 CI/CD 与包市场。

## 6. 触发器契约：同步与异步必须分开

| 场景 | 建议目标 | 上限 / 策略 | 约束 |
| --- | --- | --- | --- |
| Workflow Condition | P95 < 100 ms | 硬上限 300–500 ms | 禁止外部 IO，只做判断 |
| Validator | P95 < 300 ms | 硬上限 1–2 s | 同步阻止流转并回传错误字段 |
| UI Behaviour | P95 < 50 ms | 本地计算 | 禁止常规远程 IO |
| Post Action | 接收 < 1 s | 后台执行 | 与流转非同事务，最终一致 |
| Event Listener | P95 开始 < 3 s | 重试最多 4 次 | at-least-once，必须幂等 |
| Scheduled Job | 调度偏差 P95 < 30 s | timeout 可配 | misfire / 并发策略 |
| 普通脚本 | 默认 30–60 s | 可配上限 | 长任务走 Async / Batch，不抬高 timeout |

核心原则：**同步路径必须能力受限，异步路径可以能力丰富**。不照抄单一 240 秒超时，也不让同步判断去等远程 IO。

### 统一执行对象模型

```json
{
  "executionId": "exec_8f2c91",
  "scriptId": "script_defect_owner",
  "scriptVersion": 17,
  "trigger": { "type": "workitem.updated", "eventId": "event_4a71" },
  "principal": { "mode": "app-user" },
  "context": { "teamId": "t_01", "projectId": "XJ", "workitemId": "XJ-2481" },
  "idempotencyKey": "XJ-2481:defect_owner:17:event_4a71",
  "deadline": "2026-10-30T10:41:52+08:00",
  "correlationId": "corr_9d1e77"
}
```

比各模块各自拥有不同 execution context 更利于长期治理。

### 可靠性契约

- 投递语义：**at-least-once**，不承诺 exactly-once；有副作用的脚本必须按幂等设计
- 幂等键：SDK 提供 `execution.idempotencyKey`，默认 `workitemId + scriptVersion + eventId`
- 重试：指数退避（1s / 5s / 30s / 5m），最多 4 次，超限进 DLQ
- 顺序：同一工作项事件按 partition key 串行
- 自触发保护：同一脚本自触发超过 10 层即截断
- 事务边界：Post Action 不保证与工作流事务同事务，UI 与文档必须明确「最终一致」

## 7. 运行身份、密钥与沙箱

**Run-As 三种身份**

| 身份 | 说明 | 典型场景 |
| --- | --- | --- |
| Current User | 继承触发人权限 | Validator、部分 Post Action |
| App User | 固定应用身份，权限稳定 | Scripted Field、Listener |
| 指定用户 | 绑定服务账号，跨项目批处理 | Scheduled Job、批处理 |

**密钥**：加密存储；UI 永不回显明文；日志输出自动脱敏；支持轮换与到期提醒。

**超时与终止**：脚本级 timeout 默认 60 s 可配；超时自动终止并标记 `TIMEOUT`；支持手动 Kill；长任务走 Async / Batch，不抬高普通 runtime timeout。

**沙箱与安全边界**：租户级隔离；禁止 filesystem / socket / process 任意访问；外部域名 allowlist；API scope 最小化；代码修改与发布全程 Audit；作者与执行者分离（谁写的 ≠ 以谁的权限跑）。

**角色矩阵**：Runner Admin（发布 / 改触发器 / 管密钥 / 看全量日志）、Script Editor（编辑草稿、测试运行、看自己脚本日志）、Viewer（只读）。

## 8. 运行日志与可观测

运行日志不是「附属日志」，是 P0 核心产品能力。本版原型收口为五个分类：运行记录（含执行详情四段：Time / Logs / Payload / Timing，Timing 区分 Elapsed 与 CPU Time）、耗时分析（P50 / P95 / 慢执行排行 / 按入口分布）、失败与异常（堆栈、重试策略）、触发来源（占比与趋势）、采集与留存（按入口定级别、脱敏、采样、导出 / Webhook）。

保留策略（对标 ScriptRunner：Script Logs 14 天、Audit Logs 无限期、Behaviour Logs 最新 50 条）：本版建议按入口分档，高频入口（脚本字段）30 天，其余 180 天，审计日志长期保留。

## 9. API 建议

```text
POST /runner/scripts                       GET /runner/scripts/{id}
PUT  /runner/scripts/{id}/draft             POST /runner/scripts/{id}/test
POST /runner/scripts/{id}/publish           POST /runner/scripts/{id}/disable
POST /runner/triggers                       PUT  /runner/triggers/{id}
GET  /runner/executions                     GET /runner/executions/{executionId}
GET  /runner/executions/{executionId}/logs  POST /runner/executions/{executionId}/retry
POST /runner/secrets                        GET /runner/secrets   DELETE /runner/secrets/{id}
```

## 10. 脚本资产治理（容易被忽略的一条）

ScriptRunner Cloud 的脚本与部分数据**不属于 Jira backup**——脚本存在 Adaptavist 托管基础设施里，Jira 导出不会自动带走脚本。

因此「脚本资产导出、版本恢复、环境迁移」必须是 PingRunner 自己的明确产品能力：支持按脚本 / 按文件夹导出，跨环境导入时自动检测依赖与缺失密钥。不能假定业务平台备份会覆盖它。

## 11. 原型对照（Benchmark 版）

| 原型页面 | 对应章节 |
| --- | --- |
| Ping Runner · 能力总览（四层架构 + 八步流程） | §2 |
| Ping Runner · 平台契约与 GAP（原生十类能力 / GAP / 路线图 / 反面 Benchmark） | §3–§5 |
| Ping Runner · 脚本仓库（Draft-Published、回滚、引用） | §5、§10 |
| Ping Runner · 脚本详情（八段：基本信息→触发器→上下文→代码→权限→测试→发布→历史） | §2、§5、§7 |
| Ping Runner · 脚本控制台（Run-As / 超时 / 终止） | §5、§7 |
| Ping Runner · 触发器绑定（同步异步契约 / SLA 分级 / 执行对象模型 / 可靠性） | §6 |
| Ping Runner · 运行身份与密钥（Run-As / Secrets / 沙箱 / 角色矩阵） | §7 |
| Ping Runner · 运行日志（五分类） | §8 |
| Ping Runner · 工作流挂载点 / 动态表单行为 / 动态字段 / 界面扩展 | 标注 P1，沿用原 PRD 设计 |

## 12. 与本版无关的既有能力

高级工作流（流转四段式、动态表单三态、关系映射）、Ping Clone（单实例）、Ping Clone++（跨实例）、飞书推送（SLA）均**不在本版改动范围**，沿用 `PingCode能力PRD.md`。
