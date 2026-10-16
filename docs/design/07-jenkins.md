# Jenkins 接入与执行资源方案

覆盖 JENKINS-01 至 JENKINS-08。平台编排 Stage/Job，Jenkins 执行一个原子任务；手动等待存在平台数据库，不能靠 Jenkins input 挂住 Executor。

## JENKINS-01 Controller 配置

平台管理员在“系统设置 → 执行器”登记 Controller：名称、HTTPS baseUrl、部署密钥引用、enabled、允许空间、请求超时、容量和诊断链接。支持登记多个实例，首期每个空间固定一个 Controller；单 Run 固定其 Controller，不在故障时自动转移到其他实例。

“测试连接”验证 TLS、认证、Jenkins 当前支持版本、队列/构建读取与指定 Runner 构建权限，并返回细项结果和时间。只做无副作用探测；“测试执行”明确创建专用健康任务。用户名/token 不在 URL、不回显，服务使用最小权限，普通用户不能修改服务器 URL。

禁用阻止新提交，已有构建继续观测/取消。更换 URL、凭据或空间映射前展示活动执行；不能在活动 Run 中改变逻辑 Controller。网络 allowlist 与证书校验默认开启。

**验收：** 连接成功与执行成功分别显示；停用不丢正在运行的 build；无权服务账号不能配置或删除任意 Jenkins Job。

## JENKINS-02 平台任务映射与 Runner 协议

每空间配置固定 Folder 与一个受管理的 atomic-runner Pipeline Job，允许并发构建；提交参数只有 executionId 和当前协议标记等受控字段。用户不选择 Jenkins Job，不上传 Jenkinsfile。Runner 来自 pipeline 仓库 ops/jenkins 中固定发布的可信代码，所有空间使用同一逻辑版本。

Runner 顶层 agent none，在可信控制步骤认证并调用内部 claim，再申请指定 Agent；因此无业务等待时不占 Agent Executor。认领提供 controllerId、jobFullName、buildNumber、executionId；服务身份绑定 Controller/空间/允许 Job，不能信任请求体自行声明。平台使用 Jenkins API 验证构建参数和归属，网络验证在事务外；事务内再次检查准入与唯一 canonical build。

同 buildKey 重复认领幂等，不同 buildKey 只允许一个成功。额外 Jenkins build 在执行脚本前退出，平台保留 duplicate 标记；spec 只给 canonical build。控制身份仅在可信控制步骤持有，不进入用户仓库脚本环境。普通用户无 Jenkins Job 配置、重放或 Script Console 权限。

spec 固定输入、源码 commit、脚本摘要/文本、节点池标签、凭据绑定、超时、输出 Schema；Runner 不读“最新草稿”或重新解析分支。执行后归档 result.json，平台轮询验证。

除源码准备和业务Job外，配置验证、当前部署观测和Controller健康构建使用diagnostic_operations，复用executionId、提交、认领、观测和日志代码。只读诊断不占业务Run槽，但受空间/Controller/节点池配额与每目标30秒一次的频率限制；只能运行平台固定探测，不执行任意市场脚本。模板测试则是正常占应用环境槽的TEMPLATE_TEST Run，不能混用诊断通道执行部署。

**验收：** 重复提交最多一个构建取得脚本执行资格；伪造空间或 build 不能读取 spec；Runner 协议不符明确拒绝；无“执行整条流水线”的巨型 Jenkinsfile。

## JENKINS-03 Agent 与节点池

推荐在专用 Jenkins 中使用 Kubernetes 动态 Agent：每个执行一个 Pod、每 Pod 一个 Executor；构建池和部署池分开 ServiceAccount、镜像、网络和资源。控制节点 Executor=0，不运行应用构建。现有环境可使用隔离的静态 Agent，但同样每 Executor 独立工作目录、不得混跑不同信任等级。

节点池元数据包括 controllerId、role=SOURCE/BUILD/DEPLOY、label、工具镜像 digest、CPU/内存请求与上限、最大Pod数、允许空间、工具清单。模板声明角色与必需工具，由映射选池，不接受用户任意 label。源码准备使用轻量 SOURCE 池。

首期Agent统一Linux，架构与工具镜像固定；起始资源建议SOURCE请求0.25CPU/256Mi、上限1CPU/1Gi，BUILD请求2CPU/4Gi、上限4CPU/8Gi，DEPLOY请求0.25CPU/256Mi、上限1CPU/1Gi。每池最大Pod数与平台配额一并设置，实际构建资源通过模板测试和压测调整，不把这些起点当作实测容量。

构建推荐 rootless BuildKit/受控远端 builder，禁用挂载共享宿主 Docker socket；示例中的 docker buildx 使用预配置 builder。部署池只含 kubectl 与受限目标访问，不运行应用测试脚本。工具版本和真实 Agent/镜像进入执行记录。

**验收：** 构建任务不能选择部署池；不同空间 Agent 互不可读工作目录；Pod 资源不足显示等待原因；工具缺失在模板测试或预检阶段发现。

## JENKINS-04 Folder、凭据与空间隔离

Folder 建议 spaces/{spaceKey}，内部固定 atomic-runner；Folder 路径由后台配置生成，不直接拼接用户名称。外部凭据按空间 Folder 管理，source、registry、deploy 分用途。部署集群凭据限制 Namespace 与 Deployment 所需权限，不能授予 cluster-admin。

CMDB 登记引用：id、spaceId、controllerId、externalCredentialId、kind、purpose、displayName、status；不保存明文。平台管理员在 Jenkins/JCasC 管理密钥，空间管理员绑定引用。验证通过受控 Runner 探测用途，不依赖不存在的通用凭据枚举 REST 接口。

不同 Controller 的凭据 ID 不可互换，节点池允许空间与凭据所在空间必须同时满足。轮换允许同一逻辑引用更新外部密钥，实际执行记录引用修订/检查时间，不能虚构已知明文版本。撤销立即阻止后续绑定，已绑定远端脚本需显式取消。

**验收：** 跨 Folder 凭据拒绝；source.test 无 deploy 凭据；表单、spec 浏览器响应、日志均不出现 token；控制 API 身份不泄露到 Agent。

## JENKINS-05 工作空间、源码与缓存

逻辑路径为 runs/{runId}/executions/{executionId}/{buildNumber}，由受控 ID 构造，Jenkins 实际分配路径另行记录。每次执行全新目录，不能依赖上次 workspace；源码按 commit checkout，检出后验证 HEAD 一致并设定安全工作目录。路径真实解析和符号链接检查在使用前完成。

Job 间不共享源码目录或临时文件，镜像/制品通过仓库和已校验输出传递。同一 Run 的多个 Job 各自检出相同 commit；通过只读 Git 镜像或安全缓存优化，不牺牲隔离。缓存 key 包含空间、应用、工具镜像/架构和 lockfile 摘要，不包含密钥；不跨信任空间写共享缓存。

成功和失败默认均清理 workspace。调试保留须平台管理员对专用测试运行显式开启，先剔除凭据/输入敏感文件，最长24小时，仅诊断管理员可访问。正常日志与 result.json 在清理前归档；动态 Pod 清理失败记录并由定期清理器回收，静态目录失败则隔离，不能再给其他任务复用。

**验收：** 并行执行不覆盖源码；节点重用读不到上个任务密钥；缓存损坏可删除重建；清理失败可追踪且不会伪装清理成功。

## JENKINS-06 并发容量与两级队列

平台配额决定是否提交，Jenkins Executor/Agent 容量决定何时真正运行。配置每 Controller 逻辑在途上限和节点池最大活动数，首期不根据瞬时 Executor 读数自动无限扩容。可用并发取平台、空间、单 Run、Controller 与池的共同约束。

Runner Job 必须允许并发；不能用 Jenkins 的禁止并发把全空间串行化。应用环境互斥由平台槽位处理；Jenkins Build Number 不能充当业务锁。Kubernetes 池上限与集群资源配额一致，排队视图分别显示“平台队列”“Jenkins队列/等待节点”。

平台提交后即占逻辑执行配额，直到确定该执行不能再运行或已结束；canonical build 失联继续占用。重复 build 单独观测并取消，不额外分配平台业务资格。容量变更只影响新分配，不终止已在途任务。

**验收：** 同 Stage Job 在容量允许时确实并行；人工等待不消耗 Executor；配额降低不会随机停止部署；Dashboard 能定位容量瓶颈。

## JENKINS-07 配置交付、版本与升级

Jenkins 采用专用 LTS 基线，实施时固定具体 core、Java、插件和 Agent 镜像组合；本方案不声称兼容任意版本。使用 JCasC 管理系统配置，固定 Job DSL/受管理脚本生成空间 Runner，插件清单固定版本并在测试环境验收。[JCasC 官方插件](https://plugins.jenkins.io/configuration-as-code/)

最低能力包括 Pipeline、Credentials Binding、Folder、工作目录清理、归档输出；动态 Agent 增加 Kubernetes plugin；可信控制调用使用受控 HTTP 步骤。能用内置步骤时不额外安装插件。配置与 Runner 随 pipeline 仓库版本交付，密钥由部署 Secret 注入，不进入 Git。

升级步骤：冻结新提交 → 收敛活动 Run/模板测试 → 备份 Jenkins home、配置和数据库 → 测试升级 → 同步当前 API/Worker/Runner → 验证连接、认领、日志、取消 → 放开入口。活动构建不能悄悄切 Runner 版本。恢复旧发行包需要匹配备份与数据契约，不维护新旧执行协议并行。

**验收：** 从固定清单可重建测试 Jenkins；升级前后真实交付与故障场景通过；备份能恢复 credential 引用及构建记录；无漂移“latest”镜像。

## JENKINS-08 状态、结果、日志与重启

Worker 用参数化构建接口提交，保存 submission、响应 Location/Queue ID，轮询 queue executable 定位 Job URL 与 Build Number；Runner claim 也可更早提供 build 归属。Queue 消失不能直接判失败，先按 executionId 查构建/认领记录。[Remote Access API](https://www.jenkins.io/doc/book/using/remote-access-api/)

canonical build 结束后读取 result.json，核对协议、executionId、jobRunId、脚本摘要、输出 Schema 和镜像来源。SUCCESS 但缺合法结果仍失败；FAILURE/ABORTED 保留原始 Jenkins 结果和平台原因。UNKNOWN 表示暂不可观测，不把404、超时或 Jenkins 重启等同于业务失败。

可信Runner在finally中尽力归档诊断结果，包括部署前后观测；失败输出不进入下游outputs，但已验证的外部变化可进入部署记录。Agent丢失而未取得结果时保存“部署结果未知”，通过只读目标观测确认，不推断“没有部署”。诊断执行的结果以operationId替代jobRunId，协议ownerKind明确，不能伪装成业务Job。

日志通过 API 代理 Jenkins progressiveText，游标维护字节偏移、构建归属与UTF-8分块；浏览器不访问 Jenkins token。平台 Worker 持续归档脱敏日志到对象存储，Jenkins 重启后续读；原始构建记录被清理则显示归档或 LOG_EXPIRED/MISSING。

**验收：** 提交响应丢失能通过 claim/参数对账；Jenkins 重启不重跑已认领脚本；取消时 Agent 离线保持核对；日志断线续读不乱码、不拼错 build。

Jenkins agent none、Agent 分配与 Pipeline 语法依据 [官方语法说明](https://www.jenkins.io/doc/book/pipeline/syntax/)。这些机制支持本方案，但远端副作用仍按执行章节的结果核对规则处理。
