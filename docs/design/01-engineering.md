# 工程代码规范与仓库组织

覆盖 ARCH-01 至 ARCH-08。按用户授权形成推荐基线；Go/Gin/GORM、Next.js/React、分仓库、API/Worker 分进程和不写兼容代码为已确认约束。规范服务于可读性和当前需求，不建设通用开发框架。

## ARCH-01 代码风格与可维护性

**用户与维护者任务：** 找到一个功能，读懂主要流程，修改一处规则时清楚影响范围。

后端按业务就近放置 handler.go、service.go、model.go；SQL 复杂或重复时增加 repository.go，输入输出确有差异时增加 dto.go。模块增大后按 trigger.go、cancel.go、dispatch.go 等业务动作拆文件。每个模块不要求创建同样的文件清单。

Handler 绑定和校验 HTTP 输入、调用业务、映射响应；Service 写清规则和事务；外部 HTTP 集中在 Jenkins/CMDB client。普通函数和结构体优先，不强制域模型、端口、适配器、多层 DTO 或接口。命名使用业务动作与明确对象；错误提前返回，正常流程保持直接；不以函数行数决定拆分。

错误使用可识别业务错误与保留原因的包装，禁止比较字符串。入口记录一次最终失败，内部返回错误；日志有 request_id、run_id 等关联信息。注释解释非直观原因，Go 使用 gofmt，不复制作者、参数和返回值模板注释。原有 [具体示例](../requirements/architecture/ARCH-01-code-organization.md) 与 [开源参考](../references/open-source-projects.md) 作为阅读材料。

**验收：** 新增手动触发功能能在 execution 找到入口、状态保护、写入和测试；没有只做转发的多层文件；普通业务错误不 panic、不吞错、不返回假成功。

## ARCH-02 数据访问与显式事务

MySQL 推荐 8.4 LTS；GORM 做常规操作，复杂领取、聚合和查询用参数化 SQL。Service 中直接开启 Transaction，相关函数显式接收同一个 tx；不把事务放到 context，不混用全局 db，不在事务内等待网络。跨服务没有分布式数据库事务。

推荐GORM官方MySQL驱动，迁移使用goose维护显式SQL文件，生产不使用AutoMigrate猜测表结构。迁移由独立部署步骤执行，版本/校验和写入库。[GORM连接说明](https://gorm.io/docs/connecting_to_the_database.html)、[goose项目](https://github.com/pressly/goose)

所有空间资源查询同时限定 space_id 与实体 ID。写操作明确列出允许变更的字段，不能把 HTTP 请求直接 Save 到数据库；布尔 false、零值、空字符串的更新必须显式包含，避免 ORM 默认行为漏改。列表必须有排序和上限，预加载有范围，不循环逐条请求远端。

草稿等编辑资源使用 version 乐观锁；调度用行锁和条件更新，检查 affected rows。涉及配额时先按稳定key锁容量行，再按应用环境槽位 → Run → Stage → Job → Execution；领取task的事务先结束再处理业务。数据库时间用于租约和截止时间，服务时钟仅作显示或测试注入。

**验收：** GORM 与 SQL 在同事务中共同回滚；并发放行只写一次；越权替换 ID 无法访问另一空间；真实 MySQL 测试覆盖锁与条件更新。

## ARCH-03 接口与复用

启动时使用普通构造函数创建依赖。API、Worker 复用同一 Service；不复制业务代码，不用 DI 框架、全局服务定位器或泛型 CRUD 框架。Jenkins/CMDB client 管理认证、超时、响应解析，业务规则仍在模块内。

只有真实测试隔离或替换需求时定义使用方需要的接口。模拟 Jenkins 是首期真实需求，可以围绕实际调用方法建接口；不把每个 Service/Repository 或每个操作分别变成接口。重复的同一业务规则提取复用，语义不同的相似代码可保持独立。

模块调用保持清楚且无循环；普通查询可在同服务内按需联查。跨 CMDB/Pipeline 通过 HTTP 契约，禁止跨库查询。短暂网络故障使用当前协议重试，与历史兼容无关。

**验收：** 单元测试可控制 Jenkins 提交和观测结果；运行程序由构造函数装配；实现没有旧字段别名、双读、旧解析器和实现切换开关。

## ARCH-04 API 与编码约定

URL 采用小写复数资源和必要动作；外部前缀分别为 /api/cmdb/v1、/api/pipeline/v1，空间资源包含 /spaces/{spaceId}。JSON 使用 camelCase，Go 缩写使用 ID、URL；UUID 字符串作为实体 ID，时间 RFC3339 UTC，空集合返回 []。执行器上下文和 JSON Schema 中明确约定的字段按其契约保持，不混改。

成功读取返回资源对象；分页返回 items、nextCursor、hasMore。错误为 code、message、requestId、details，details 可包含 fieldPath、stageKey、jobKey。401 未登录，403 已知空间内无动作权限，404 资源不存在或空间不可见，409 业务状态冲突，412 If-Match 不匹配，422 语义校验不通过，429 配额或速率限制，503 依赖不可用；内部错误不向用户泄露 SQL、token 或堆栈。

编辑读取返回 ETag，修改必须 If-Match，缺失返回 428。创建 Run、发布修订、复制、手动放行等使用 Idempotency-Key；作用域为当前主体、空间、操作与键，保存请求摘要和原响应至少 24 小时。同键同内容返回原结果，同键不同内容返回409；资源状态和唯一约束继续防止键过期后的业务重复。

默认分页20，最大100；游标绑定筛选条件，按 createdAt、id 稳定排序，不接受任意 SQL 排序字段。未知写字段拒绝，字符串裁剪规则明确，标识不可隐式重写；表单后端最终校验。

**验收：** 网络重发创建只得到一个 Run；冲突保留用户输入；错误可以定位字段；日志与错误响应可以通过 requestId 关联。

## ARCH-05 配置与进程生命周期

配置分为部署配置与业务配置。数据库地址、服务 URL、服务身份、密钥引用来自部署文件/环境；空间配额、模板、Jenkins 映射存业务库。启动时统一加载并校验，缺必需项立即失败，不在请求中临时猜默认值。密钥不得输出到启动日志。

提供 /health/live 和 /health/ready。数据库、迁移版本与必要配置决定 ready；Jenkins 暂时故障体现在依赖健康与执行排队，不让只读 API 不断重启。SIGTERM 后 API 停止接新写请求并完成短事务；Worker 停止领取任务、结束或释放租约；远端构建继续，由其他 Worker 接管核对。

采用标准结构化日志，UUID v4，由数据库约束保证唯一；事件使用 bigint 游标，HTTP 将其序列化为字符串避免 JS 精度丢失。时钟、随机 ID 仅在确需确定性测试处注入，不引入通用时间框架。

**验收：** 缺密钥启动报清楚配置名；退出不会将远端执行标失败；ready 不报告错误数据库迁移已就绪。

## ARCH-06 测试与质量门禁

测试优先验证规则：定义校验、输入绑定、权限、状态迁移、并发屏障、事务、重复提交和取消。真实 MySQL 验证锁，模拟 Jenkins 验证超时与结果不明，受控真实 Jenkins/K8s 验证完整交付。前端验证编辑器、冲突恢复、手动按钮和日志续读，不堆纯 getter、转发层测试。

合并门禁：格式、lint、Go test 与关键包 race、TypeScript 类型检查、关键前端测试、契约与样例校验、迁移校验、密钥扫描。依赖漏洞按可达性和严重性判定，记录豁免原因与期限；不以任意覆盖率代替场景验收。代码生成结果须可复现、无手改。

**验收：** 每项实现能关联本方案需求 ID；[验收场景](12-delivery-and-acceptance.md) 的关键故障场景进入自动化；CI 失败无法合并。

## ARCH-07 前端结构

共享仓库使用 Next.js App Router 和 TypeScript，路由只装配页面；features/cmdb、catalog、pipelines、runs 就近存组件、表单、查询和类型；components 存真实共享 UI，lib/api 存生成的服务客户端与错误映射。页面和编辑器的局部状态留在本地，服务端数据不复制进一个全局 store。

推荐Ant Design提供表格、表单、抽屉和基础组件，TanStack Query管理浏览器列表、详情、轮询及失效，表单由Ant Design Form统一管理，不叠加第二套表单状态库；实际依赖版本在开工时锁定。编辑器用局部reducer保存完整草稿，生成市场参数表单使用已支持Schema子集。SSR只做页面壳和必要初始数据，不在Next.js再建业务后端。根据[Next.js数据读取说明](https://nextjs.org/docs/app/getting-started/fetching-data)明确Server/Client边界；用户权限与运行数据显式禁用共享缓存。

查询与组件集成入口见[TanStack Query官方文档](https://tanstack.com/query/latest/docs/framework/react/reference/index)和[Ant Design的Next.js集成说明](https://ant.design/docs/react/use-with-next)，采用对应版本的SSR样式注册方式，避免首屏样式闪烁。

同源网关分别路由前端、CMDB、Pipeline；会话 Cookie 由 CMDB 颁发，前端不在 localStorage 保存 token。URL 保存筛选与选中的 Job，深链接刷新可还原。公共状态组件统一 loading、empty、error、forbidden、stale；多条独立数据查询并行，不重复拉整页。

**验收：** 切空间清理对应查询缓存且不闪现另一空间数据；退出登录后缓存清除；编辑输入、服务端草稿与运行快照不会串用。

## ARCH-08 跨仓库契约与升级

仓库建议命名 cmdb、pipeline、devops-web。CMDB 和 Pipeline 各自维护 OpenAPI 与数据库迁移，前端按指定提交生成客户端；Pipeline 生成所需 CMDB 客户端。禁止共享数据库模型或抽出一个包含双方业务的 common 仓库。

跨服务读写使用超时、请求关联与服务身份，并校验空间。发布计划页面由前端组合 CMDB 计划与 Pipeline 汇总，不把执行状态反写成第二份权威状态。跨服务操作没有“全局事务成功”的伪保证，失败可安全重查或重发。

正常运行只支持一套当前协议。破坏性升级安排窗口：阻止新运行，收敛活动执行，备份，先迁移并部署服务端、再部署对应 Worker/Runner 和前端，契约验收后开放。静态资源可留短期旧文件供已打开页面提示刷新，但旧 API/执行协议不保留；版本不一致明确要求刷新。

**验收：** 生成客户端与 OpenAPI 对应同一提交；跨仓库契约测试通过；旧输入明确拒绝；迁移后正常路径没有历史解释器。
