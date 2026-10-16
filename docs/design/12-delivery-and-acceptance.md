# 实施顺序与验收方案

本章给出推荐设计完成后的实施路线和验收证据要求。当前交付仅为方案文档，不启动以下开发或连接真实执行环境。每项实现关联需求ID，开发自检通过后再进入用户验收。

## 分阶段实施

| 阶段 | 交付范围 | 完成证据与准入条件 |
|---|---|---|
| M0 工程基础 | 三仓库、Go/Gin/GORM、Next.js/TS、配置、迁移、日志、OpenAPI、CI与同源网关 | 代码可读、启动失败清楚、两API及Worker可独立运行；无兼容层或通用业务框架 |
| M1 应用交付CMDB | 账号/会话、空间角色、子系统、应用、环境、目标、引用、交付上下文与对应前端 | 可登记一个测试应用、越权测试通过、上下文一致、敏感值不泄漏 |
| M2 市场与配置 | 模板草稿/测试发布、Pipeline身份、编辑器、绑定编译、不可变修订 | 示例完整编辑发布；不完整草稿/冲突/失效引用可恢复；测试执行器验证契约 |
| M3 执行闭环 | 持久化队列、状态机、人工等待、配额、失败/取消/超时、快照/事件/日志界面 | 模拟Jenkins通过并发、重复、网络结果不明、Worker重启等故障场景 |
| M4 真实执行 | Jenkins配置、Runner认领、Agent隔离、三种首批能力、镜像/部署/日志归档 | 真实构建→人工部署闭环；取消/超时/失联通过；来源和实际目标可追溯 |
| M5 上线收敛 | 发布计划、保留清理、当前部署观测、统计容量、备份恢复、升级手册、UX回归 | 完整需求追踪、恢复演练、容量测试和操作手册；用户验收后才能上线 |

市场测试运行最终复用M3执行逻辑；M2可先用同契约的测试执行器验证，不建永久的第二套引擎。数据和前端随业务一起交付，不延后补记历史。M0锁定确切依赖，M4锁定并验证Jenkins/插件/Agent组合，M5记录实际容量与恢复指标。

## 关键场景与证据

每个场景至少保留操作步骤、预期、实际结果、相关请求/Run/事件ID与必要截图；并发场景同时检查数据库约束和远端实际执行次数。

| 场景ID | 场景 | 必须成立的结果 |
|---|---|---|
| ACC-01 | 首次账号、改密、登录、退出与过期 | 首次强制改密；会话撤销；无明文密码/token；过期保留可恢复输入 |
| ACC-02 | 五角色、跨空间与最后Owner | 权限矩阵有效；替换ID不可越权；唯一Owner不可移除 |
| ACC-03 | 创建应用、子系统移动与归档 | 基本信息可先保存；同空间移动保留身份；有引用不能永久删 |
| ACC-04 | 普通/保护环境与目标配置 | protected按标志判断；禁用/假集群/跨空间凭据拒绝；实际目标清楚 |
| ACC-05 | CMDB并发更新与上下文不可用 | 快照无混合版本；503不创建半个Run；旧快照不随新配置改变 |
| ACC-06 | 发布计划commit冻结与汇总 | READY后不能改计划；环境/应用/commit匹配；测试成功不算部署 |
| ACC-07 | 模板输入、默认值与表单子集 | 未支持Schema拒绝发布；默认值显式保存；未知字段/类型报错 |
| ACC-08 | 模板版本、弃用、封禁与可见性 | 已发布不可变；弃用按规则准入；封禁阻止新执行；历史可看 |
| ACC-09 | 模板升级与参数/输出变化 | 合法字段保留；失效项定位；没有自动猜名或旧别名 |
| ACC-10 | 模板测试与发布摘要绑定 | 非生产测试；候选修改后旧测试失效；复用配额、取消与日志 |
| ACC-11 | 凭据、脚本输入与节点隔离 | 参数作为数据；路径逃逸拒绝；仓库脚本无部署/控制凭据 |
| ACC-12 | Pipeline创建、复制与默认项 | 幂等创建；跨应用复制需重绑定；无修订不能运行 |
| ACC-13 | 停用与既有运行 | 禁止新Run；既有排队/活动Run按规则继续；停用不伪装取消 |
| ACC-14 | 归档、恢复与历史 | 活动Run阻止归档；恢复保持停用；历史链接仍可访问 |
| ACC-15 | 不完整草稿与自动保存乱序 | 可保存但不能发布；单在途保存；无输入丢失或假保存 |
| ACC-16 | 两人/两标签编辑与断网 | 412冲突保留双方内容；不自动覆盖；恢复后核对ETag |
| ACC-17 | Stage/Job增删复制重排 | key稳定；引用影响可见；支持撤销和键盘替代 |
| ACC-18 | 四种绑定与非法前序关系 | 同Stage输出/未来输出拒绝；解析值和来源准确 |
| ACC-19 | 发布校验与数据竞态 | 结构/类型/目标冲突完整检查；发布只针对审阅草稿 |
| ACC-20 | 修订比较与配置回退 | 内容不可变；回退重发新编号；发布不自动运行 |
| ACC-21 | 导入导出和当前格式 | 不导出密钥；旧格式明确拒绝；非法导入不破坏草稿 |
| ACC-22 | Run创建超时重发 | 同幂等键一个Run；同键不同内容冲突；节点任务同事务写入 |
| ACC-23 | 分支排队变化与固定commit | 在准备时明确解析；所有源码Job使用同SHA；只写一次 |
| ACC-24 | Stage串行与Job并行 | 自动Job并行具备资格；下一Stage只激活一次且等全部成功 |
| ACC-25 | 自动+多个手动Job | 自动正常开始；各手动独立；只剩人工时状态正确 |
| ACC-26 | 手动与失败/取消/角色变更竞争 | 同锁复核；只放行一次；取消/失败后不能启动；新权限生效 |
| ACC-27 | 并行失败与兄弟收敛 | 停止新任务；已认领兄弟继续；未启动跳过；不早释放槽 |
| ACC-28 | 取消、迟到结果、Agent离线 | 停止后续Stage；保留真实结果；无法确认时仍取消中 |
| ACC-29 | 各阶段超时及Worker重启 | deadline不重置；最终超时有证据；清理上限有效 |
| ACC-30 | 双Worker、多空间与容量限制 | 不超配、不重复推进；公平调度；同应用环境互斥 |
| ACC-31 | Jenkins已接受但响应丢失 | 按同executionId核对/重提；只有一个canonical执行业务脚本 |
| ACC-32 | 重复/伪造Runner claim与spec | 同build幂等；其他build拒绝；跨空间和假归属拒绝 |
| ACC-33 | API/Worker/Jenkins重启与旧fence | 恢复原状态与commit；旧Worker写回失败；不重跑已认领脚本 |
| ACC-34 | Controller、池、工具与Folder配置 | 连接/执行验证分开；正确映射；禁用只阻止新提交 |
| ACC-35 | workspace、源码、缓存与清理失败 | 执行隔离；相同SHA；缓存可丢；清理失败隔离并有记录 |
| ACC-36 | SUCCESS但输出缺失、UNSTABLE或digest错 | 业务Job失败；下游不启动；诊断与原始结果保留 |
| ACC-37 | 镜像生产记录与历史镜像部署 | 来源应用/commit/仓库匹配；digest可多生产记录；新Run有来源链 |
| ACC-38 | 部署已生效但rollout/后置检查失败 | 保留实际部署与Run失败；不自动回滚、不伪造未部署 |
| ACC-39 | 日志断线、中文、长行、归档延迟 | 无乱码/重复/漏尾；游标绑定build；归档完成真实 |
| ACC-40 | 保留、到期、缺失和清理异常 | 非终态/引用不删；日志过期明确；registry不被平台自动删 |
| ACC-41 | 大数据分页、容量与指标 | 分页稳定；常用索引可用；统计有口径；压测达到建议目标或记录调整 |
| ACC-42 | MySQL/对象/Jenkins备份恢复 | 真实恢复可用；先隔离在途执行；不重复部署；RPO/RTO有证据 |
| ACC-43 | 破坏性迁移与跨仓库升级 | 单当前契约、升级窗口、版本准入；无双读或旧Runner回退 |
| ACC-44 | 真实构建→手动部署→计划汇总 | Git/镜像/K8s事实与页面、日志、审计一致 |
| ACC-45 | 响应式、键盘、长内容和空/错状态 | 375/768/1440关键流程可用；焦点合理；颜色不是唯一信号 |
| ACC-46 | CSRF、来源限制、密钥与SSRF | 非法来源/地址拒绝；无密钥入URL/缓存/日志；下载同样鉴权 |
| ACC-47 | CMDB改变、硬停用与集群外改镜像 | 旧Run目标不变；后续准入受停用限制；外部变更标记非平台来源 |
| ACC-48 | 可读代码、事务与接口测试 | 简单分工、显式tx、ORM+SQL共同回滚；接口只围绕实际隔离需要 |
| ACC-49 | 原Run重新运行与修复部署 | 新ID/lineage、当前上下文差异、原记录不变；无单Job自动重跑 |
| ACC-50 | SSE/日志长连接权限撤销 | 15秒内复核关闭；重连仍鉴权；reset重新拉快照不丢事实 |

## 文档覆盖与实现追踪

下表由需求目录与各设计章节对应，状态统一为“推荐设计已生成，待用户审阅”。阶段是实施建议，不表示已经开工。场景ID为最低验收覆盖，实际测试仍需覆盖章节中的全部约束。

| 需求 | 设计章节 | 推荐阶段 | 最低验收场景 |
|---|---|---|---|
| ARCH-01 | [代码风格与可维护性](01-engineering.md#arch-01-代码风格与可维护性) | M0 | ACC-48 |
| ARCH-02 | [数据访问与显式事务](01-engineering.md#arch-02-数据访问与显式事务) | M0 | ACC-48, ACC-30 |
| ARCH-03 | [接口与复用](01-engineering.md#arch-03-接口与复用) | M0 | ACC-48, ACC-31 |
| ARCH-04 | [API 与编码约定](01-engineering.md#arch-04-api-与编码约定) | M0 | ACC-16, ACC-22 |
| ARCH-05 | [配置与进程生命周期](01-engineering.md#arch-05-配置与进程生命周期) | M0 | ACC-33, ACC-43 |
| ARCH-06 | [测试与质量门禁](01-engineering.md#arch-06-测试与质量门禁) | M0 | ACC-44, ACC-48 |
| ARCH-07 | [前端结构](01-engineering.md#arch-07-前端结构) | M0 | ACC-15, ACC-39, ACC-45 |
| ARCH-08 | [跨仓库契约与升级](01-engineering.md#arch-08-跨仓库契约与升级) | M0 | ACC-43 |
| CMDB-01 | [空间成员与权限](02-cmdb.md#cmdb-01-空间成员与权限) | M1 | ACC-01, ACC-02 |
| CMDB-02 | [子系统管理](02-cmdb.md#cmdb-02-子系统管理) | M1 | ACC-03 |
| CMDB-03 | [应用管理](02-cmdb.md#cmdb-03-应用管理) | M1 | ACC-03, ACC-05 |
| CMDB-04 | [环境与部署目标](02-cmdb.md#cmdb-04-环境与部署目标) | M1 | ACC-04, ACC-47 |
| CMDB-05 | [源码与构建元数据](02-cmdb.md#cmdb-05-源码与构建元数据) | M1 | ACC-05, ACC-11 |
| CMDB-06 | [发布计划与应用版本](02-cmdb.md#cmdb-06-发布计划与应用版本) | M5 | ACC-06, ACC-44 |
| CMDB-07 | [交付上下文接口](02-cmdb.md#cmdb-07-交付上下文接口) | M1 | ACC-05, ACC-47 |
| CMDB-08 | [变更、引用与历史保护](02-cmdb.md#cmdb-08-变更引用与历史保护) | M1 | ACC-03, ACC-14, ACC-47 |
| JOB-01 | [原子能力与对象边界](03-job-catalog.md#job-01-原子能力与对象边界) | M2 | ACC-07, ACC-12 |
| JOB-02 | [输入输出与表单](03-job-catalog.md#job-02-输入输出与表单) | M2 | ACC-07, ACC-18, ACC-36 |
| JOB-03 | [脚本与执行环境](03-job-catalog.md#job-03-脚本与执行环境) | M2 | ACC-11, ACC-34, ACC-35 |
| JOB-04 | [凭据与上下文](03-job-catalog.md#job-04-凭据与上下文) | M2 | ACC-11, ACC-32, ACC-46 |
| JOB-05 | [版本生命周期](03-job-catalog.md#job-05-版本生命周期) | M2 | ACC-08, ACC-09 |
| JOB-06 | [测试与发布](03-job-catalog.md#job-06-测试与发布) | M2 | ACC-10 |
| JOB-07 | [市场检索与使用](03-job-catalog.md#job-07-市场检索与使用) | M2 | ACC-08, ACC-12 |
| JOB-08 | [首批能力与成功判据](03-job-catalog.md#job-08-首批能力与成功判据) | M4 | ACC-36, ACC-37, ACC-38, ACC-44 |
| PIPE-01 | [应用与流水线关系](04-pipeline-management.md#pipe-01-应用与流水线关系) | M2 | ACC-12, ACC-04 |
| PIPE-02 | [列表与查询](04-pipeline-management.md#pipe-02-列表与查询) | M2 | ACC-12, ACC-41 |
| PIPE-03 | [创建与复制](04-pipeline-management.md#pipe-03-创建与复制) | M2 | ACC-12 |
| PIPE-04 | [启用与停用](04-pipeline-management.md#pipe-04-启用与停用) | M2 | ACC-13 |
| PIPE-05 | [归档、删除与恢复](04-pipeline-management.md#pipe-05-归档删除与恢复) | M2 | ACC-14 |
| PIPE-06 | [操作权限与审计](04-pipeline-management.md#pipe-06-操作权限与审计) | M2 | ACC-02, ACC-26 |
| CFG-01 | [草稿 CRUD](05-pipeline-editor.md#cfg-01-草稿-crud) | M2 | ACC-15 |
| CFG-02 | [Stage 增删改与排序](05-pipeline-editor.md#cfg-02-stage-增删改与排序) | M2 | ACC-17 |
| CFG-03 | [Job 节点增删改](05-pipeline-editor.md#cfg-03-job-节点增删改) | M2 | ACC-09, ACC-17 |
| CFG-04 | [参数与数据绑定](05-pipeline-editor.md#cfg-04-参数与数据绑定) | M2 | ACC-18, ACC-37 |
| CFG-05 | [校验与执行计划](05-pipeline-editor.md#cfg-05-校验与执行计划) | M2 | ACC-19 |
| CFG-06 | [发布、差异与配置回退](05-pipeline-editor.md#cfg-06-发布差异与配置回退) | M2 | ACC-20 |
| CFG-07 | [定义格式、导入导出与迁移](05-pipeline-editor.md#cfg-07-定义格式导入导出与迁移) | M2 | ACC-21, ACC-43 |
| CFG-08 | [多人编辑与断线保护](05-pipeline-editor.md#cfg-08-多人编辑与断线保护) | M2 | ACC-16 |
| RUN-01 | [创建运行与准备](06-execution.md#run-01-创建运行与准备) | M3 | ACC-22, ACC-23 |
| RUN-02 | [状态机与 Stage 屏障](06-execution.md#run-02-状态机与-stage-屏障) | M3 | ACC-24, ACC-25 |
| RUN-03 | [单个手动 Job 放行](06-execution.md#run-03-单个手动-job-放行) | M3 | ACC-25, ACC-26 |
| RUN-04 | [队列与并发](06-execution.md#run-04-队列与并发) | M3 | ACC-30 |
| RUN-05 | [失败、部分结果与外部副作用](06-execution.md#run-05-失败部分结果与外部副作用) | M3 | ACC-27, ACC-38 |
| RUN-06 | [取消与终止](06-execution.md#run-06-取消与终止) | M3 | ACC-28 |
| RUN-07 | [超时](06-execution.md#run-07-超时) | M3 | ACC-29 |
| RUN-08 | [持久化、租约与故障恢复](06-execution.md#run-08-持久化租约与故障恢复) | M3 | ACC-31, ACC-32, ACC-33 |
| RUN-09 | [重新运行与修复](06-execution.md#run-09-重新运行与修复) | M3 | ACC-49 |
| JENKINS-01 | [Controller 配置](07-jenkins.md#jenkins-01-controller-配置) | M4 | ACC-34 |
| JENKINS-02 | [平台任务映射与 Runner 协议](07-jenkins.md#jenkins-02-平台任务映射与-runner-协议) | M4 | ACC-31, ACC-32 |
| JENKINS-03 | [Agent 与节点池](07-jenkins.md#jenkins-03-agent-与节点池) | M4 | ACC-11, ACC-34 |
| JENKINS-04 | [Folder、凭据与空间隔离](07-jenkins.md#jenkins-04-folder凭据与空间隔离) | M4 | ACC-11, ACC-32 |
| JENKINS-05 | [工作空间、源码与缓存](07-jenkins.md#jenkins-05-工作空间源码与缓存) | M4 | ACC-35 |
| JENKINS-06 | [并发容量与两级队列](07-jenkins.md#jenkins-06-并发容量与两级队列) | M4 | ACC-30, ACC-34 |
| JENKINS-07 | [配置交付、版本与升级](07-jenkins.md#jenkins-07-配置交付版本与升级) | M4 | ACC-43, ACC-44 |
| JENKINS-08 | [状态、结果、日志与重启](07-jenkins.md#jenkins-08-状态结果日志与重启) | M4 | ACC-31, ACC-33, ACC-39 |
| DATA-01 | [代码来源](08-data-and-audit.md#data-01-代码来源) | M3/M4/M5 | ACC-23 |
| DATA-02 | [配置与执行快照](08-data-and-audit.md#data-02-配置与执行快照) | M3/M4/M5 | ACC-05, ACC-20, ACC-47 |
| DATA-03 | [输入输出与绑定来源](08-data-and-audit.md#data-03-输入输出与绑定来源) | M3/M4/M5 | ACC-18, ACC-36 |
| DATA-04 | [镜像制品与部署事实](08-data-and-audit.md#data-04-镜像制品与部署事实) | M3/M4/M5 | ACC-37, ACC-38, ACC-47 |
| DATA-05 | [节点与执行环境](08-data-and-audit.md#data-05-节点与执行环境) | M3/M4/M5 | ACC-34, ACC-35 |
| DATA-06 | [日志与实时事件](08-data-and-audit.md#data-06-日志与实时事件) | M3/M4/M5 | ACC-39, ACC-50 |
| DATA-07 | [历史、时间线与审计](08-data-and-audit.md#data-07-历史时间线与审计) | M3/M4/M5 | ACC-06, ACC-26, ACC-38 |
| DATA-08 | [保留、清理与资源回收](08-data-and-audit.md#data-08-保留清理与资源回收) | M3/M4/M5 | ACC-40 |
| DATA-09 | [查询、容量与统计](08-data-and-audit.md#data-09-查询容量与统计) | M3/M4/M5 | ACC-41 |
| OPS-01 | [登录与跨服务授权](09-operations.md#ops-01-登录与跨服务授权) | M1/M3/M5 | ACC-01, ACC-02, ACC-46, ACC-50 |
| OPS-02 | [密钥生命周期](09-operations.md#ops-02-密钥生命周期) | M1/M3/M5 | ACC-11, ACC-46 |
| OPS-03 | [外部依赖故障](09-operations.md#ops-03-外部依赖故障) | M1/M3/M5 | ACC-05, ACC-28, ACC-31, ACC-33 |
| OPS-04 | [可观测性](09-operations.md#ops-04-可观测性) | M1/M3/M5 | ACC-41 |
| OPS-05 | [部署、退出与升级](09-operations.md#ops-05-部署退出与升级) | M1/M3/M5 | ACC-33, ACC-43 |
| OPS-06 | [数据迁移、备份与恢复](09-operations.md#ops-06-数据迁移备份与恢复) | M1/M3/M5 | ACC-42, ACC-43 |
| OPS-07 | [联调与完整验收](09-operations.md#ops-07-联调与完整验收) | M1/M3/M5 | ACC-44 |
| WEB-01 | [导航、空间与应用上下文](10-user-experience.md#web-01-导航空间与应用上下文) | 随业务阶段 | ACC-02, ACC-45 |
| WEB-02 | [表单、列表与 Job 市场](10-user-experience.md#web-02-表单列表与-job-市场) | 随业务阶段 | ACC-03, ACC-07, ACC-45 |
| WEB-03 | [编辑器布局与操作](10-user-experience.md#web-03-编辑器布局与操作) | 随业务阶段 | ACC-15, ACC-16, ACC-17, ACC-19, ACC-45 |
| WEB-04 | [运行、日志和历史](10-user-experience.md#web-04-运行日志和历史) | 随业务阶段 | ACC-25, ACC-28, ACC-38, ACC-39 |
| WEB-05 | [可访问性、响应式与异常状态](10-user-experience.md#web-05-可访问性响应式与异常状态) | 随业务阶段 | ACC-45, ACC-50 |

## 完成与上线判断

设计交付完成要求76项都有明确功能/规则、交互或维护流程、失败边界与验收，并核对章节、示例、默认值和接口互相一致。当前阶段不以代码编译或真实执行报告冒充设计验收。

实现完成要求所有首期条目有代码、必要迁移、当前契约、前端可操作流程与自动/人工验收证据。真实Jenkins组合、负载、备份恢复尚须在M4/M5实测；设计中的容量和时限是可评审目标，不是已实现服务承诺。

## 本次设计交付核对

2026-10-16完成设计交付核对：10个专题的76个需求均有独立详细章节与验收条目，76条追踪关系关联50个核心验收场景；总览、需求索引、数据/API、实施计划及重点取舍相互链接。原编排长草案已归并为当前方案导航，避免维护不同默认策略。

核对内容包括：本地文档链接与章节锚点、Markdown代码块、JSON语法、Definition的2020-12 Schema、当前定义正例及5个拒绝用例、模板与ExecutionSpec的输入/脚本/输出契约一致性、Result归属与摘要、示例前序绑定和数量限制。拒绝用例覆盖旧协议、未知字段、空Stage集合、非法触发方式和Stage超限。

设计交叉检查覆盖数据拥有方、应用环境互斥、全局目标身份归属、配额锁顺序、手动截止竞态、诊断执行隔离、取消证据、部署副作用、日志与保留周期、当前协议升级。尚未进行产品编译、性能测试、真实Jenkins执行或备份恢复；这些是已列入实施阶段的验证工作。
