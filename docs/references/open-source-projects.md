# 开源项目代码参考

调研日期：2026-10-16。状态：参考资料与采用建议，尚未确定以任何项目作为代码底座。当前只研究代码风格、可读性和可维护性，不编写产品代码。

本轮查阅项目仓库及下列业务文件。项目规模、技术栈和业务规则各有不同；以下“适合借鉴”是结合本系统需求作出的判断，不代表对整个项目代码的评价。源码链接固定到本次查阅的提交，方便后续一起阅读。

## 推荐顺序

| 项目 | 在本系统中的参考用途 | 适合借鉴 | 应保留的本系统选择 |
|---|---|---|---|
| Woodpecker | Pipeline 的主要代码参考 | Gin 入口与业务流程分工；按创建、审批、取消、入队等业务动作组织代码 | GORM；平台编排、Jenkins 执行；手动触发针对单个 Job |
| Gitea | 通用 Go 业务代码的补充参考 | 功能命名；HTTP 入口调用业务函数；状态约束和解释原因的注释 | 按业务模块就近组织文件；显式传递事务；按需抽象 |
| Harbor | 后续后台执行与并发专题参考 | 明确的任务状态；任务级与执行池级并发的区分 | Worker 管理编排与 Jenkins 交互；具体并发策略另行评审 |
| Gin-Vue-Admin | Gin/GORM CRUD 的补充对照 | 参数绑定、业务调用、响应处理；直接可读的 GORM 数据操作 | 构造函数传入依赖；明确的返回类型；Next.js/React 前端 |

建议当前优先阅读 Woodpecker 和 Gitea 的少量业务文件。Harbor 留到执行需求讨论；Gin-Vue-Admin 用来对照常规 CRUD 写法，不作为整套系统框架。

## Woodpecker：优先参考 Pipeline 业务代码

[项目仓库](https://github.com/woodpecker-ci/woodpecker)。本次提交：`4179f32ac58d85e5d5173a564e20d9d72024528c`。

查阅入口：

- [server/api/pipeline.go](https://github.com/woodpecker-ci/woodpecker/blob/4179f32ac58d85e5d5173a564e20d9d72024528c/server/api/pipeline.go#L52-L90)：CreatePipeline 解析请求、获取所需信息、调用 pipeline.Create、处理响应。
- [server/pipeline/approve.go](https://github.com/woodpecker-ci/woodpecker/blob/4179f32ac58d85e5d5173a564e20d9d72024528c/server/pipeline/approve.go)：审批业务放在 pipeline 包；先检查状态，再处理配置和后续启动。
- [server/pipeline/queue.go](https://github.com/woodpecker-ci/woodpecker/blob/4179f32ac58d85e5d5173a564e20d9d72024528c/server/pipeline/queue.go#L28-L80)：pipelineTasks 组装队列任务，函数注释明确真正入队由 scheduler 负责。
- [server/store/datastore/pipeline.go](https://github.com/woodpecker-ci/woodpecker/blob/4179f32ac58d85e5d5173a564e20d9d72024528c/server/store/datastore/pipeline.go)：按流水线业务含义命名查询方法，集中处理数据操作。
- [server/pipeline/pipeline_status_test.go](https://github.com/woodpecker-ci/woodpecker/blob/4179f32ac58d85e5d5173a564e20d9d72024528c/server/pipeline/pipeline_status_test.go)：状态相关测试同时检查运行时间、审批人等字段。

适合借鉴的是业务动作清楚、职责容易找到。例如 execution 代码增长后，可以按需要拆为 trigger.go、cancel.go、dispatch.go，而不把所有行为长期塞进一个大文件。

适用边界：该项目使用 Gin 和 XORM，见 [go.mod](https://github.com/woodpecker-ci/woodpecker/blob/4179f32ac58d85e5d5173a564e20d9d72024528c/go.mod)。其审批是流水线级，执行依赖自己的 Agent。我们继续使用 GORM 和 Jenkins，手动等待发生在 Job 级。所查代码中的全局服务访问、从 context 取得 Store、错误记录方式及模型兼容字段，也不转为本项目规范。

## Gitea：参考业务命名与流程表达

[项目仓库](https://github.com/go-gitea/gitea)。本次提交：`1c117a9352e4a64cd301028403c33949ab146424`。

查阅入口：

- [routers/api/v1/repo/actions_run.go](https://github.com/go-gitea/gitea/blob/1c117a9352e4a64cd301028403c33949ab146424/routers/api/v1/repo/actions_run.go#L135-L215)：取消、审批等入口有明确名称；不满足状态条件时提前返回，再调用业务服务。
- [services/actions/approve.go](https://github.com/go-gitea/gitea/blob/1c117a9352e4a64cd301028403c33949ab146424/services/actions/approve.go)：ApproveRuns 集中表达审批和状态变化；部分注释解释并发限制与状态处理的原因。
- [services/actions/approve_test.go](https://github.com/go-gitea/gitea/blob/1c117a9352e4a64cd301028403c33949ab146424/services/actions/approve_test.go)：与业务代码就近放置的审批行为测试。

适合借鉴的是动作名称、提前返回、关键约束的说明，以及业务代码与测试容易对应。我们仍按业务模块组织少量文件，随实际规模拆分。

适用边界：Gitea 的所查实现使用 db.WithTx 并通过 context 传递事务范围；我们的事务继续显式传递 tx。它的整体目录、审批语义、函数规模和历史兼容需求也不直接变为本系统要求。

## Harbor：后续研究后台任务时参考

[项目仓库](https://github.com/goharbor/harbor)。本次提交：`502a51bdecf21ed942386b75f4671a478f254dcf`。

查阅入口：

- [src/jobservice/job/status.go](https://github.com/goharbor/harbor/blob/502a51bdecf21ed942386b75f4671a478f254dcf/src/jobservice/job/status.go)：定义任务状态及终态判断。
- [src/jobservice/job/interface.go](https://github.com/goharbor/harbor/blob/502a51bdecf21ed942386b75f4671a478f254dcf/src/jobservice/job/interface.go)：表达任务运行、参数校验、重试和任务类型并发能力，并区分任务类型并发与 WorkerPool 并发。
- [src/jobservice/core/controller.go](https://github.com/goharbor/harbor/blob/502a51bdecf21ed942386b75f4671a478f254dcf/src/jobservice/core/controller.go)：任务提交、状态查询和控制的入口。

可用于后续讨论“状态放在哪里”“并发限制控制什么”“取消后如何核对结果”。其 JobService 有自己的任务运行体系，我们的实际任务由 Jenkins 执行；本文不据此引入 Harbor 的接口、队列或重试默认值，也不采用其已弃用的状态方法。

## Gin-Vue-Admin：对照 Gin/GORM 的常规写法

[项目仓库](https://github.com/flipped-aurora/gin-vue-admin)。本次提交：`e1c3cc415b5c3638e904924e78898bba10169a92`。

查阅入口：

- [server/api/v1/example/exa_customer.go](https://github.com/flipped-aurora/gin-vue-admin/blob/e1c3cc415b5c3638e904924e78898bba10169a92/server/api/v1/example/exa_customer.go#L25-L46)：参数绑定、校验、业务调用和响应的顺序。
- [server/service/example/exa_customer.go](https://github.com/flipped-aurora/gin-vue-admin/blob/e1c3cc415b5c3638e904924e78898bba10169a92/server/service/example/exa_customer.go)：直接使用 GORM 完成常规数据操作。
- [server/router/example/exa_customer.go](https://github.com/flipped-aurora/gin-vue-admin/blob/e1c3cc415b5c3638e904924e78898bba10169a92/server/router/example/exa_customer.go)：路由注册的具体写法。

可以对照 CMDB 的简单 CRUD 如何写清楚。所查实现中的全局数据库、全局服务实例、interface{} 列表返回和重复参数注释不作为本项目写法；依赖通过构造函数传入，返回类型明确，注释解释必要原因。前端继续使用 Next.js/React。

## 对 ARCH-01 的建议

结合这些代码，建议继续收敛以下几条，尚未因调研而视为已确认：

1. 文件名和函数名首先表达业务动作，让人能找到创建、触发、取消、调度和查询的位置。
2. HTTP 处理与可供 API/Worker 共用的业务逻辑分清；简单功能用少量文件完成。
3. 同一模块内容增多后按动作拆文件，不为预想能力创建层级、接口和框架。
4. 状态规则集中、错误保留原因、事务显式；具体正确性机制按自己的需求设计。
5. 阅读外部项目时记录“问题、做法、适用条件”，不因为它采用某个模式就添加为本系统约束。

下一次可以沿 Woodpecker 的 CreatePipeline → pipeline.Create 这条业务链讨论每段代码承担什么职责，再收敛本系统的命名、文件拆分和错误规范。只讨论对当前需求有帮助的写法。
