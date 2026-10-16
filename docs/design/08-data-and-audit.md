# 执行数据日志与审计

覆盖 DATA-01 至 DATA-09。运行详情必须能回答代码、配置、镜像、执行节点、日志、人员和部署结果的来源；最新配置、流水线成功和实际部署是不同事实。

## DATA-01 代码来源

每个 Run 保存 applicationId、repositoryUrl（无内嵌密码）、requestedRef、refKind、resolvedCommitSha、resolvedAt、commitSubject、author、committedAt 和源码准备 executionId。SHA 是执行身份，分支/标签只表示请求来源；摘要来自已验证仓库，不能由用户随意填写冒充 Git 事实。

创建时保存请求 ref，PREPARING 成功后 commit 从空写入一次；各源码 Job 检出并校验同一 commit。部署专用 Run 从已登记镜像生产记录取得 commit。首期一个应用一个源码仓库；子模块/多仓库不静默忽略。

详情头部显示短 SHA，点击复制完整值，外链到允许 Git 主机的 commit 页面。原仓库归档或 commit 后来无法读取，不删除历史 SHA；新运行需要重新验证可检出。

**验收：** 排队期间分支变化有明确解析时点；不同 Job 不各自漂移；用户输入的分支标签不能替换实际 SHA。

## DATA-02 配置与执行快照

创建 Run 时保存 Pipeline 身份/修订与正文、ExecutionPlan、模板版本及脚本/Schema摘要与执行内容、CMDB contextVersion 与交付字段、节点池映射、普通参数、Release 与计划项快照、创建者与请求来源。密钥仅保存引用。

快照用于重建当次事实，不随草稿、模板描述或 CMDB 修改。commit、解析输入、canonical build 等在各自首次确定时只补写一次；生命周期、观测和日志清单另列可变字段。原始快照摘要写入审计，读取展示不依赖最新模板仍存在。

配置页和运行页明确分为“当前应用配置”“本次运行快照”；差异可读，不能在历史详情中静默显示新的集群或分支。历史格式可按原文查看，只有显式迁移并重新发布的当前契约能新执行。

**验收：** 修改应用目标后旧 Run 仍显示旧目标；模板封禁不阻止历史查看；密钥扫描确认快照不含明文。

## DATA-03 输入输出与绑定来源

JobRun 保存原始 bindings、resolvedInputs、resolvedAt、inputDigest、validatedOutputs、outputDigest，以及每个来源的类型与引用 ID。literal 来自修订，run 来自 Run 参数，cmdb 来自上下文版本，job 来自明确的 producerJobRunId/outputName/artifactId。

输入解析在 Job 获得执行资格、依赖已满足后完成并固定；解析失败产生明确错误，不把 null、未定义、空串混同。输出只在 canonical 成功、Schema 与业务语义校验后写入；失败执行的未验证输出保存在诊断结果中，不能给下游消费。

输入输出页按字段显示值和来源，长对象折叠，密钥槽位仅显示引用名；最大64 KiB，错误值脱敏和截断。导出审计必须同样按权限与敏感规则处理。

**验收：** 下游可追到生产 Job；输出非法不进入业务 outputs；修改上游模板不会改变历史字段解释。

## DATA-04 镜像制品与部署事实

ImageArtifact 表示一次生产记录，字段：space/application、repository、tag、digest、imageRef、commitSha、runId、producerJobRunId、outputName、createdAt、availability、verificationAt。一个 digest 可以由多个 Job/commit 产生记录，不建立 digest→commit 的全局唯一推断；生产 Job+输出名唯一。

镜像 tag 默认 runId-jobKey 便于检索，部署使用 digest。可用性包含 AVAILABLE、MISSING、UNKNOWN，镜像后来被清理不删除来源元数据；启动部署前再核对可拉取。制品元数据归 Pipeline，镜像二进制在 registry，平台首期不自动删除 registry 镜像。

DeploymentRecord 记录目标 cluster/namespace/workload/container、前后镜像、预期与观测 commit、Deployment UID、generation/observedGeneration、request/applied/observed 时间、rollout、JobRun 与操作者。先记录意图，尽可能持续记录已确认变化，即使 Job 最终失败；不能因 Run 失败删除部署事实。

无法通过实际镜像生产记录确认的commit保持未知，不把Run期望commit写成集群已观测事实。Runner未能归档时保留UNKNOWN；受控只读诊断可以刷新目标事实，但不能仅因镜像当前正确就推断失联脚本已经结束。

应用“当前部署”来自最近确认的观测，显示时间与 UNKNOWN/STALE；不是对集群持续控制的承诺。人工在集群外改镜像时，通过显式刷新发现偏离，并标“非本平台变更”，不伪造归属 Run。发布计划的历史达成与当前环境镜像分开。

**验收：** 部署后验证失败仍能看到镜像已变更；来源不符拒绝部署；外部 tag 覆盖不影响 digest；外部改镜像不会冒充平台发布。

## DATA-05 节点与执行环境

ExecutorExecution 保存 controllerId、runnerJobFullName、canonicalBuildNumber/URL、queueIds、executionProtocol、runnerVersion、template/script摘要、nodePoolId、agentName/Pod UID、toolImageDigest、工具版本、资源请求/限制、实际workspace逻辑标识和检查时间。Submission 独立保存每次请求与结果，不覆盖未知或重复提交证据。

Controller 与节点池配置在创建/执行时分别记录选定值，实际 Agent 由 Runner 上报并由观测交叉检查。节点池“计划镜像”与实际镜像分字段，避免只记录期望。标准用户看到名称和必要资源信息，详细内网路径、Jenkins 管理链接放在诊断授权区。

**验收：** 能区分同 execution 的多个 build；实际运行环境可追踪；节点不存在后历史仍可阅读；浏览器不能利用内部地址拼任意代理请求。

## DATA-06 日志与实时事件

状态事件与业务状态同事务写 run_events，按 eventId 稳定排序。SSE 支持 Last-Event-ID，传小事件和必要摘要，客户端按 ID 去重后失效对应查询；游标超过7天实时回放窗口时返回 reset 事件并重新拉快照，历史审计仍保留。

日志由 API 代理活动 Jenkins progressiveText，Worker 独立持续采集并归档对象存储，不依赖用户打开页面。归档块标记 execution/build、原始起止偏移、脱敏后内容摘要、完成状态；幂等对象 key 防重复。UTF-8 边界与跨块过滤由有状态流处理，不把字符索引当 Jenkins 字节偏移。

响应包含 nextCursor、hasMore、complete、archiveState；complete 只在构建终结且尾部采集完成为真。归档落后时先显示可用区段和“日志同步中”，不能把缺口当空日志。只读游标为服务端签名 opaque 值，绑定空间、JobRun 和 build，不能跨任务重用。

界面支持自动跟随、暂停滚动、搜索已加载内容、复制行、下载脱敏日志；搜索范围明确不承诺全量历史全文检索。长日志采用窗口渲染，内存保留最近5,000行，可向前分页。Jenkins 凭据掩码与平台敏感模式过滤是防意外措施，主动泄密脚本需靠可信代码与最小授权阻断；不保存另一份未脱敏平台日志。

**验收：** 断线恢复无重复/乱码/漏尾；权限撤销最迟下一次15秒复核关闭流；Jenkins日志清理后可读取完整归档；缺失返回 LOG_MISSING/EXPIRED，不返回假空白。

## DATA-07 历史、时间线与审计

时间线显示创建、排队、准备、Stage激活、手动等待/放行、提交、认领、输入解析、结果、取消、超时与人工收敛。事件包含 actorType=USER/WORKER/RUNNER/SYSTEM、actorId、reason、requestId、previous/new state、发生与记录时间；事件不由浏览器任意上报。

运行历史支持应用、流水线、环境、状态、时间、commit、触发人、Release、镜像digest筛选。详情展示排队/准备/人工等待/执行时长，不用总时长代表计算耗时。审计追加不可覆盖，涉及删除/迁移保留操作证据；审计管理员权限不等于可读取密钥。

跨服务审计共用 correlationId，CMDB 和 Pipeline 各自记录自己实际完成的事务；不伪造一条跨服务原子成功事件。用户可从业务对象进入时间线，复制深链接供排障。

**验收：** 能重建某次人工触发是谁、何时、什么目标；取消和实际成功结果同时保留；部署事实与运行主状态各自准确。

## DATA-08 保留、清理与资源回收

| 数据 | 默认保留 | 清理条件与用户表现 |
|---|---|---|
| Run/Stage/Job、快照、输出、事件、审计、部署/镜像元数据 | 终结后730天 | 被发布计划或保留标记引用则延长；批次清理关联闭包 |
| 流水线修订与模板版本 | 存在引用期间 | 历史到期且无当前引用后才可管理性清理 |
| 平台归档日志、Jenkins console/result.json | 30天 | Jenkins 至少保留到归档确认完成；失败转清理异常 |
| SSE 实时事件回放 | 7天 | 更老游标重新取快照，审计不因此删除 |
| workspace | 执行后立即清理 | 调试例外最长24小时、剔除凭据 |
| 缓存 | 最近使用后7天、容量上限触发LRU | 可再生成，不影响产物身份 |
| 幂等响应 | 至少24小时 | 资源状态/约束继续保护业务重复 |
| registry 镜像二进制 | 由独立仓库策略管理 | 首期平台不自动删；先核对受保护部署引用 |

归档和清理任务有预览数量、限速、重试、审计与保留标记。非终态 Run、未知远端执行、未完成归档不能因“超过日期”被清理。日志到期详情仍保留结果和“日志已过期”；删除历史不能级联删除应用身份。

**验收：** 清理不影响活动任务和引用闭包；对象存储失败不会使 DB 标记已清理；保留时间变更明确影响后续清理而非立即大批删除。

## DATA-09 查询、容量与统计

关键筛选为普通列并有组合索引，JSON 保存快照、定义和输出，不在大表逐条解析 JSON 做常规列表。run_jobs 索引 runId+jobKey、spaceId+status+updatedAt；runs 索引 spaceId+applicationId+environmentId+createdAt+id；tasks 索引 status+nextRunAt+leaseUntil；事件按 runId+eventId。

按5,000 Run/天、平均10 Job估算约50,000 Job/天，730天约3,650万 Job；若每Job平均200 KiB日志，约9.5 GiB/天、30天约286 GiB，不含副本和索引。它们是容量输入，需要压测与实际采样校准，不能作为已达成指标。

首期用单MySQL实例分服务数据库、索引、游标分页、对象日志与限速清理，不预建分库分表。监控行数/表体积/慢查询，到run_jobs千万行或表100 GiB触发扩容评审；保留期内先加容量/优化查询，若需分区须显式迁移，不临时双读旧表。

概览统计最近7/30天成功率、人工等待、排队时长和各阶段耗时，异步汇总带更新时间；失败率明确分母是否含取消/超时。首轮性能目标在建议数据规模下列表p95<1秒、详情p95<2秒，不含外部日志网络，实施时用压测证据验收。

**验收：** 分页无重复遗漏；筛选不会全表扫描常用大字段；统计可说明口径；历史日志不进入MySQL大字段。
