# 系统需求拆分与讨论顺序

本清单包含 10 个专题、76 个独立需求。编号用于关联讨论结论、设计、实现任务和验收证据。它也是完整推荐方案的索引，原问题已在对应章节给出推荐答案；推荐不冒充用户已逐项确认。

当前方向已按用户澄清调整：工程部分聚焦代码风格、可读性和可维护性，保持简单；撤回强制多层目录、公开契约层、UnitOfWork/TxStores 和最小端口拆分。Go、Gin、GORM、业务范围及 API/Worker 分进程选择保留。用户现已授权全部按推荐方案写全后集中审阅，仍只生成文档，未创建产品代码。完整入口见 [推荐方案总览](../design/00-overview.md)，覆盖与验收见 [追踪表](../design/12-delivery-and-acceptance.md)。

全局已确认约束：[ENG-001 不写兼容性代码，只保留当前正确方案](../engineering-constraints.md)。该规则作用于所有专题，不能在单项设计中默认保留旧实现。

## 一 代码结构设计规范与解耦

| 需求 ID | 需求 | 必须讨论的内容 | 当前状态 |
|---|---|---|---|
| [ARCH-01](../design/01-engineering.md#arch-01-代码风格与可维护性) | 代码风格可读性与可维护性 | 命名、函数、注释、错误、职责、文件组织与修改成本 | 推荐方案已生成，待整体审阅 |
| [ARCH-02](../design/01-engineering.md#arch-02-数据访问与显式事务) | 数据访问与事务代码规范 | GORM、SQL 集中管理、直接显式事务、错误与并发保护 | 推荐方案已生成，待整体审阅 |
| [ARCH-03](../design/01-engineering.md#arch-03-接口与复用) | 按需使用接口与代码复用 | 何时提取接口、普通函数与结构体、实际复用、依赖创建 | 推荐方案已生成，待整体审阅 |
| [ARCH-04](../design/01-engineering.md#arch-04-api-与编码约定) | 编码与 API 规范 | 命名、DTO、错误、HTTP 状态、校验、分页、幂等、乐观锁 | 推荐方案已生成，待整体审阅 |
| [ARCH-05](../design/01-engineering.md#arch-05-配置与进程生命周期) | 配置与运行基础设施 | 配置加载、密钥、日志、时钟、ID、进程生命周期与退出 | 推荐方案已生成，待整体审阅 |
| [ARCH-06](../design/01-engineering.md#arch-06-测试与质量门禁) | 测试与质量门禁 | 规则测试、MySQL 集成测试、接口契约、故障模拟与 CI | 推荐方案已生成，待整体审阅 |
| [ARCH-07](../design/01-engineering.md#arch-07-前端结构) | 前端代码结构 | 路由、功能模块、共享组件、数据请求、表单、权限与状态 | 推荐方案已生成，待整体审阅 |
| [ARCH-08](../design/01-engineering.md#arch-08-跨仓库契约与升级) | 跨仓库契约与依赖 | OpenAPI、客户端生成、当前契约一致性、协同升级与发布顺序 | 推荐方案已生成，待整体审阅 |

文档：[ARCH-01](architecture/ARCH-01-code-organization.md)、[ARCH-02](architecture/ARCH-02-persistence.md)、[ARCH-03](architecture/ARCH-03-ports-and-injection.md)。

## 二 CMDB

| 需求 ID | 需求 | 必须讨论的内容 |
|---|---|---|
| [CMDB-01](../design/02-cmdb.md#cmdb-01-空间成员与权限) | 空间成员与权限 | 空间生命周期、账号、成员、角色、授权继承与移除效果 |
| [CMDB-02](../design/02-cmdb.md#cmdb-02-子系统管理) | 子系统管理 | 标识、负责人、归属、调整及删除限制 |
| [CMDB-03](../design/02-cmdb.md#cmdb-03-应用管理) | 应用管理 | 应用身份、必填字段、技术栈、负责人、迁移与归档 |
| [CMDB-04](../design/02-cmdb.md#cmdb-04-环境与部署目标) | 环境与部署目标 | 环境层级、目标集群、Namespace、工作负载及多目标需求 |
| [CMDB-05](../design/02-cmdb.md#cmdb-05-源码与构建元数据) | 代码与构建元数据 | 仓库数量、默认分支、构建配置、镜像仓库和凭据引用 |
| [CMDB-06](../design/02-cmdb.md#cmdb-06-发布计划与应用版本) | 发布版本与应用计划 | 应用关联、计划 ref、环境、发布状态及变更规则 |
| [CMDB-07](../design/02-cmdb.md#cmdb-07-交付上下文接口) | 交付上下文接口 | 提供给 Pipeline 的字段、版本、一致性、缓存与不可用策略 |
| [CMDB-08](../design/02-cmdb.md#cmdb-08-变更引用与历史保护) | 元数据变更与引用保护 | 删除、归档、迁移、历史执行引用及空间隔离 |

## 三 Pipeline Job 市场

| 需求 ID | 需求 | 必须讨论的内容 |
|---|---|---|
| [JOB-01](../design/03-job-catalog.md#job-01-原子能力与对象边界) | 原子能力边界 | 一个 Job 应做多大、能力分类、模板与实例的区别 |
| [JOB-02](../design/03-job-catalog.md#job-02-输入输出与表单) | 输入输出契约 | 参数类型、默认值、表单、输出验证、语义与产物引用 |
| [JOB-03](../design/03-job-catalog.md#job-03-脚本与执行环境) | 脚本与执行环境 | 脚本存放、解释器、工具依赖、镜像或 Agent、工作目录 |
| [JOB-04](../design/03-job-catalog.md#job-04-凭据与上下文) | 凭据与上下文 | 凭据槽位、空间授权、上下文注入、脱敏与最小授权 |
| [JOB-05](../design/03-job-catalog.md#job-05-版本生命周期) | 模板版本生命周期 | 草稿、发布、变更、下架、废弃及既有引用处理 |
| [JOB-06](../design/03-job-catalog.md#job-06-测试与发布) | 模板测试与发布 | 调试、测试空间、发布校验、失败回退与权限 |
| [JOB-07](../design/03-job-catalog.md#job-07-市场检索与使用) | 市场检索与使用 | 分类、搜索、详情、版本选择、使用示例及可见范围 |
| [JOB-08](../design/03-job-catalog.md#job-08-首批能力与成功判据) | 首批能力清单 | 构建、测试、镜像、部署的参数、输出及成功判据 |

## 四 Pipeline 管理

管理的是流水线资源身份和生命周期，不包含 Stage 编辑或某次运行状态。

| 需求 ID | 需求 | 必须讨论的内容 |
|---|---|---|
| [PIPE-01](../design/04-pipeline-management.md#pipe-01-应用与流水线关系) | 应用与流水线关系 | 一对多、用途、命名、环境适用范围与默认流水线 |
| [PIPE-02](../design/04-pipeline-management.md#pipe-02-列表与查询) | 列表与查询 | 空间和应用筛选、搜索、分页、最近运行与权限过滤 |
| [PIPE-03](../design/04-pipeline-management.md#pipe-03-创建与复制) | 创建与复制 | 初始配置、跨应用复制、引用重绑定与复制权限 |
| [PIPE-04](../design/04-pipeline-management.md#pipe-04-启用与停用) | 启用与停用 | 对新运行、排队运行、活动运行和历史记录的影响 |
| [PIPE-05](../design/04-pipeline-management.md#pipe-05-归档删除与恢复) | 删除与归档 | 软删除、恢复、清理前提、Release 和历史引用 |
| [PIPE-06](../design/04-pipeline-management.md#pipe-06-操作权限与审计) | 操作权限与审计 | 查看、编辑、发布、执行、取消等动作的空间角色映射 |

## 五 Pipeline 配置与组装

配置是“如何运行”的定义，运行记录是“某次实际发生了什么”。

| 需求 ID | 需求 | 必须讨论的内容 |
|---|---|---|
| [CFG-01](../design/05-pipeline-editor.md#cfg-01-草稿-crud) | 配置草稿 CRUD | 草稿结构、保存、读取、重置、删除与不完整配置 |
| [CFG-02](../design/05-pipeline-editor.md#cfg-02-stage-增删改与排序) | Stage CRUD | 新增、删除、复制、排序、空 Stage 和引用影响 |
| [CFG-03](../design/05-pipeline-editor.md#cfg-03-job-节点增删改) | Job 节点 CRUD | 市场选择、复制、移除、触发方式、版本升级与参数保留 |
| [CFG-04](../design/05-pipeline-editor.md#cfg-04-参数与数据绑定) | 参数与数据绑定 | 固定值、运行参数、CMDB、前序输出及类型匹配 |
| [CFG-05](../design/05-pipeline-editor.md#cfg-05-校验与执行计划) | 校验与执行计划 | 结构、依赖、字段、资源冲突、错误定位与编译产物 |
| [CFG-06](../design/05-pipeline-editor.md#cfg-06-发布差异与配置回退) | 发布修订与差异 | 草稿到发布、版本号、修订比较、回退与不可变性 |
| [CFG-07](../design/05-pipeline-editor.md#cfg-07-定义格式导入导出与迁移) | 定义格式与升级 | JSON Schema、导入导出、单一运行格式、显式迁移及旧输入拒绝 |
| [CFG-08](../design/05-pipeline-editor.md#cfg-08-多人编辑与断线保护) | 多人编辑与自动保存 | 乐观锁、冲突处理、离开提示和编辑器预览 |

## 六 Pipeline 执行

| 需求 ID | 需求 | 必须讨论的内容 |
|---|---|---|
| [RUN-01](../design/06-execution.md#run-01-创建运行与准备) | 创建执行与准备 | 修订、环境、参数、Release 关联、快照及 commit 解析时点 |
| [RUN-02](../design/06-execution.md#run-02-状态机与-stage-屏障) | Stage 与 Job 状态机 | 串行屏障、并行激活、聚合状态及成功判据 |
| [RUN-03](../design/06-execution.md#run-03-单个手动-job-放行) | 手动触发 | 等待时机、触发权限、参数能否修改、重复点击与竞态 |
| [RUN-04](../design/06-execution.md#run-04-队列与并发) | 平台队列与并发 | 空间、应用、环境、Run、Job 限流及排队公平性 |
| [RUN-05](../design/06-execution.md#run-05-失败部分结果与外部副作用) | 失败策略 | 兄弟 Job、未启动 Job、后续 Stage、输出不合法及部分成功 |
| [RUN-06](../design/06-execution.md#run-06-取消与终止) | 取消与终止 | 停止调度、取消队列、终止执行、迟到结果及实际副作用 |
| [RUN-07](../design/06-execution.md#run-07-超时) | 超时策略 | 人工等待、平台队列、Jenkins 排队、执行和清理超时 |
| [RUN-08](../design/06-execution.md#run-08-持久化租约与故障恢复) | 持久化与故障恢复 | 重启、多副本、租约、重复提交、状态核对及人工介入 |
| [RUN-09](../design/06-execution.md#run-09-重新运行与修复) | 重试重跑与恢复 | 整体重跑、单 Job 重试、失败点恢复、配置和产物沿用 |

## 七 Jenkins 接入与执行资源

| 需求 ID | 需求 | 必须讨论的内容 |
|---|---|---|
| [JENKINS-01](../design/07-jenkins.md#jenkins-01-controller-配置) | Controller 配置 | 单或多个 Jenkins、连接配置、认证、验证与归属 |
| [JENKINS-02](../design/07-jenkins.md#jenkins-02-平台任务映射与-runner-协议) | Runner 与映射 | 平台 Job 如何落到 Jenkins、Runner 发布、参数与认领协议 |
| [JENKINS-03](../design/07-jenkins.md#jenkins-03-agent-与节点池) | Agent 与节点池 | 静态或动态 Agent、标签、工具、镜像、隔离和资源规格 |
| [JENKINS-04](../design/07-jenkins.md#jenkins-04-folder凭据与空间隔离) | 凭据与空间隔离 | Jenkins Folder、凭据可见范围、服务身份与密钥轮换 |
| [JENKINS-05](../design/07-jenkins.md#jenkins-05-工作空间源码与缓存) | 工作空间与缓存 | 路径、唯一性、源码检出、Job 间文件、缓存、清理及失败保留 |
| [JENKINS-06](../design/07-jenkins.md#jenkins-06-并发容量与两级队列) | Jenkins 并发容量 | Executor、Controller、Agent、平台限额及资源不足时的队列 |
| [JENKINS-07](../design/07-jenkins.md#jenkins-07-配置交付版本与升级) | 配置交付与升级 | 配置即代码、插件、Runner 代码、固定支持版本、备份及协同升级 |
| [JENKINS-08](../design/07-jenkins.md#jenkins-08-状态结果日志与重启) | 状态日志与异常 | 队列到构建映射、日志游标、结果、网络中断和 Jenkins 重启 |

## 八 执行数据与追溯

每类数据必须明确来源、写入时点、归属、可变性、存储位置、保留周期和读取权限。

| 需求 ID | 需求 | 必须讨论的内容 |
|---|---|---|
| [DATA-01](../design/08-data-and-audit.md#data-01-代码来源) | 代码来源 | 仓库、请求 ref、commit、分支标签、提交说明及多仓库需求 |
| [DATA-02](../design/08-data-and-audit.md#data-02-配置与执行快照) | 配置快照 | Pipeline 修订、Stage/Job 节点、模板脚本、CMDB 与工具版本 |
| [DATA-03](../design/08-data-and-audit.md#data-03-输入输出与绑定来源) | 输入输出与来源 | 原始参数、解析值、上游引用、输出验证及字段脱敏 |
| [DATA-04](../design/08-data-and-audit.md#data-04-镜像制品与部署事实) | 镜像与制品 | tag、digest、制品地址、校验和、来源 commit 与生产执行 |
| [DATA-05](../design/08-data-and-audit.md#data-05-节点与执行环境) | 节点与执行环境 | Controller、Queue、Build、Agent、资源、工作目录与运行工具 |
| [DATA-06](../design/08-data-and-audit.md#data-06-日志与实时事件) | 日志 | 采集、实时推送、断线续读、脱敏、检索、落盘与归档 |
| [DATA-07](../design/08-data-and-audit.md#data-07-历史时间线与审计) | 执行历史与审计 | Run/Stage/Job 时间线、触发人、人工动作、失败原因及部署事实 |
| [DATA-08](../design/08-data-and-audit.md#data-08-保留清理与资源回收) | 保留与清理 | 元数据、日志、工作空间、缓存、制品的独立保留规则 |
| [DATA-09](../design/08-data-and-audit.md#data-09-查询容量与统计) | 查询与容量 | 筛选、分页、索引、统计、数据量估算与大表治理 |

## 九 通用运行与交付细节

| 需求 ID | 需求 | 必须讨论的内容 |
|---|---|---|
| [OPS-01](../design/09-operations.md#ops-01-登录与跨服务授权) | 登录与跨服务授权 | 会话、过期、撤销、鉴权调用、SSE 和内部服务身份 |
| [OPS-02](../design/09-operations.md#ops-02-密钥生命周期) | 密钥生命周期 | 存储、引用、注入、轮换、审计和日志脱敏 |
| [OPS-03](../design/09-operations.md#ops-03-外部依赖故障) | 外部依赖故障 | CMDB、Jenkins、Git、镜像仓库不可用时的行为 |
| [OPS-04](../design/09-operations.md#ops-04-可观测性) | 可观测性 | 请求与运行关联、结构化日志、指标、告警和定位入口 |
| [OPS-05](../design/09-operations.md#ops-05-部署退出与升级) | 系统部署与升级 | 本地环境、容器、配置、优雅退出、升级与程序回滚 |
| [OPS-06](../design/09-operations.md#ops-06-数据迁移备份与恢复) | 数据变更与恢复 | 显式迁移、备份、恢复演练及历史记录保存 |
| [OPS-07](../design/09-operations.md#ops-07-联调与完整验收) | 联调与端到端验收 | 模拟执行器、真实 Jenkins、K8s 环境与完整交付场景 |

## 十 前端操作流程

| 需求 ID | 需求 | 必须讨论的内容 |
|---|---|---|
| [WEB-01](../design/10-user-experience.md#web-01-导航空间与应用上下文) | 空间与应用导航 | 空间切换、权限、子系统和应用入口、深链接 |
| [WEB-02](../design/10-user-experience.md#web-02-表单列表与-job-市场) | CMDB 与市场表单 | 字段、校验、列表、版本选择、空状态和错误反馈 |
| [WEB-03](../design/10-user-experience.md#web-03-编辑器布局与操作) | 流水线编辑器 | Stage/Job 布局、增删改、引用配置、发布和冲突呈现 |
| [WEB-04](../design/10-user-experience.md#web-04-运行日志和历史) | 执行操作与日志 | 状态图、手动按钮、取消、排队、日志及历史快照 |
| [WEB-05](../design/10-user-experience.md#web-05-可访问性响应式与异常状态) | 前端验收 | 加载、断线恢复、键盘操作、响应式与权限变化 |

## 依赖与实现顺序

全部专题已按推荐基线展开，待整体审阅后调整。RUN与Jenkins协议、DATA与市场/配置/执行关系联合核对，不把推荐标成用户已确认。

建议实现顺序是工程骨架、CMDB、Job 契约、流水线管理和配置、模拟执行器下的运行引擎、Jenkins 接入、真实交付验收。执行数据和前端随对应功能实现，不作为全部功能完成后的补录工作。最终实现任务以已确认需求及依赖为准。

## 当前讨论结论

2026-10-16，用户澄清其架构关注点是代码风格、可读性、可维护性，要求不要复杂。当前按简单、清楚和按需抽象重新评审代码规范，旧的强制分层与接口设计不再执行。技术栈、业务范围、API/Worker 分进程部署及不写兼容性代码的要求保留；现已授权先生成全部推荐方案再集中修改，不等待每条单独回复；仍不把需求拆分转换成代码框架建设。
