# 数据模型接口与事务契约

本章把前面功能映射到数据与API，供逐项实施使用；它是设计清单，不是已实现OpenAPI或建表SQL。所有字段和约束按当前唯一契约实施，不预留旧别名与多版本执行分支。

## 公共字段与归属

业务实体使用UUID字符串，库内建议BINARY(16)，边界统一转换；eventId为BIGINT，JSON中以十进制字符串传递。UTC时间使用微秒精度；版本号是递增整数。空间业务表必须有space_id；created_at、updated_at、created_by及可编辑记录的version按需设置。状态字段使用可识别字符串与显式允许值，不能散落魔法数字。

两个服务各有数据库账号和迁移目录，内部引用可使用外键/唯一约束，跨服务ID只作逻辑引用并通过API校验。key在对应空间/应用范围唯一，归档不释放已使用key，避免历史身份歧义。执行查询列与不可变JSON快照分离。

## CMDB 数据实体

| 表 | 关键字段 | 约束与修改规则 |
|---|---|---|
| users | username、display_name、password_hash、enabled、must_change_password | username唯一；不存可逆密码；不能停用唯一空间Owner |
| sessions | token_hash、user_id、csrf_binding、idle/absolute_expiry、revoked_at | 仅存会话摘要；退出/改密撤销；不记录原Cookie |
| service_identities | name、secret_hash/外部证书标识、scope、revoked_at | CMDB服务调用授权；Runner身份由Pipeline按空间维护 |
| spaces | key、name、owner关系、status、version | key唯一，成员事务保护至少一Owner |
| space_members | space_id、user_id、role、version | space+user唯一，一人一主角色 |
| subsystems | space_id、key、name、owner_id、status、version | space+key唯一，迁移只在同空间 |
| applications | space_id、subsystem_id、key、name、owner、maintainers、language、status | space+key唯一，引用归属一致 |
| environments | space_id、key、name、protected、enabled、order | space+key唯一；保护标志不是名称推断 |
| application_delivery_configs | application_id、repository_url、default_ref、source_credential、image_repository、build_defaults、context_version | application唯一；更新相关字段递增context_version |
| deployment_targets | application_id、environment_id、target_identity_id、cluster_id、namespace、workload、container、credential_ref、enabled | app+env唯一当前配置；快照保留历史目标身份，本期仅Deployment |
| deployment_target_identities | cluster_identity、namespace、workload_kind/name、owner_app/env、enabled | 集群+Namespace+Deployment全局唯一；旧目标身份不分配给其他应用环境 |
| clusters / registries | endpoint、display_name、cluster_identity（集群）、allowed_space_ids、enabled、配置版本 | 系统管理；同一真实集群身份唯一，URL受allowlist控制，不存浏览器可取密钥 |
| credential_references | space_id、controller_id、external_id、kind、purpose、display_name、status、version | space+controller+external_id唯一；只有元数据 |
| releases | space_id、key、name、environment_id、owner、planned_at、status、version | space+key唯一；READY后计划固定 |
| release_applications | release_id、application_id、planned_commit_sha、source_ref_label、note | release+app唯一；commit明确，不存另一份执行状态 |
| cmdb_audit_events | actor、object、action、reason、diff、request_id、time | 追加写；敏感值不进diff |

创建/修改交付上下文需在本库一致读取目标、环境、凭据引用和配置版本；contextVersion可以是相关版本集合摘要，不要求一个修改全空间的热点行。返回当前确切字段集合与摘要，快照持久化保持原值。

## Pipeline 数据实体

| 表 | 关键字段 | 约束与修改规则 |
|---|---|---|
| job_templates | key、name、category、effect、visibility、allowed_spaces | key唯一；身份描述可改，脚本在版本中 |
| job_template_versions | template_id、version、status、blocked、schemas、script、executor_requirements、content_digest、last_test_run_id | template+version唯一；PUBLISHED后内容不可变 |
| pipelines | space_id、application_id、key、name、enabled、archived_at、allowed_environments、default_revision_id、version | app+key唯一；一应用多流水线 |
| application_pipeline_preferences | space_id、application_id、default_pipeline_id、version | app唯一；默认流水线事实由Pipeline维护，不在CMDB复制指针 |
| pipeline_drafts | pipeline_id、definition、version、saved_by | pipeline唯一；ETag保护 |
| pipeline_revisions | pipeline_id、revision_no、definition、execution_plan、template_digests、plan_version、message | pipeline+revision_no唯一，不可变 |
| pipeline_runs | kind、space/app/env、pipeline/revision、snapshot、requested_ref、commit、state、sync_state、active_stage、creator、release、re_run_of、stop_reason | kind=PIPELINE/TEMPLATE_TEST；固定字段只首次写入 |
| run_stages | run_id、stage_key、name、order、state、started/finished_at | run+stage_key唯一 |
| run_jobs | run_id、stage_id、job_key、template_ref、trigger_mode、bindings、inputs、outputs、state、manual_actor、deadline | run+job_key唯一；输入输出首次解析成功后固定 |
| executor_executions | owner_kind/id、execution_id、controller/pool、canonical_build、sync_state、protocol、runner_version、deadlines | owner_kind+owner_id唯一；一个逻辑执行 |
| diagnostic_operations | space/app/env、kind、context_version、target_snapshot、actor、state、result、deadline | 固定只读验证/目标观测/健康检查，不执行任意业务脚本 |
| executor_submissions | execution_id、submission_no、request_state、queue_id、observed_build、times、error | execution+submission_no唯一；不覆盖不明提交 |
| scheduler_tasks | kind、aggregate_id、task_key、generation、claimed_generation、next_run_at、status、lease_owner/until/fence、attempts | task_key唯一；同任务新唤醒不会被旧ack吞掉 |
| application_environment_slots | space_id、application_id、environment_id、owner_run_id | space+app+env唯一；非终态不能强制释放 |
| capacities / resource_permits | kind/key、limit、in_use；owner_kind/id、resource_key | 资源key唯一；owner+resource唯一；计数受事务保护 |
| run_events | run_id、event_id、actor、action、old/new_state、reason、details、request_id | run+event_id索引，追加写 |
| image_artifacts | producer_job_run、output_name、space/app、repository、tag、digest、commit、availability | producer+output唯一；digest不全局唯一 |
| deployment_records | job_run、目标快照、前后镜像、实际观测、rollout、时间与原因 | job_run+目标唯一；保存失败后的已知外部变化 |
| log_manifests / log_chunks | execution/build、source_offset范围、object_key、digest、archive_state、expiry | build+起始offset唯一；归档闭合后才complete |
| jenkins_controllers / node_pools / space_executor_bindings | URL/密钥引用、容量、工具镜像、角色、Folder/Runner映射 | 每空间一个启用映射，活动Run固定其值 |
| runner_identities | controller、space、allowed_job、secret_hash、revoked_at | scope限定；不把服务密钥交给仓库脚本 |
| idempotency_records | actor、space、operation、key、request_digest、response、expires_at | 作用域+key唯一，24小时保留 |
| pipeline_audit_events | actor、object、action、reason、diff、request_id | 追加写，非Run的管理行为也可审计 |

CatalogTestRun 是 `pipeline_runs.kind=TEMPLATE_TEST` 的业务视图，不另造执行引擎；pipeline/revision可空，快照为候选模板和一个内部Stage/Job，保存候选摘要。它仍使用同样的空间权限、应用环境槽、配额、日志与取消；普通运行列表默认过滤模板测试，管理员从模板测试页查看。

模板contentDigest为发布时固定UTF-8 JSON存档原始字节的SHA-256，存档以不可变文本/二进制保留，不从MySQL JSON重新序列化后重算；scriptDigest为脚本UTF-8内容SHA-256，不擅自变更换行。这样跨服务可直接验证原始存档，无须引入一套隐式JSON规范化规则。

Execution ownerKind为SOURCE_PREPARATION、JOB_RUN或DIAGNOSTIC_OPERATION。诊断操作没有业务Run时锁diagnostic_operation→Execution，不参与Stage推进，只获取相应执行容量；其spec/result包含operationId与受控probeKind。操作由当前资源权限准入，目标观测不会解除未知业务执行的槽，也不能把只读镜像观测冒充已知脚本停止。

## 事务与并发边界

1. 创建Pipeline：身份+唯一草稿+审计。保存草稿：条件version更新+审计。
2. 发布：指定草稿编译 → 事务锁Pipeline并复核version和模板准入 → 新修订+默认指针+事件；网络调用不在事务中。
3. 创建Run：外部授权/上下文预检 → 本地Run/全部节点/快照/首任务/幂等响应原子提交。
4. 取得执行资格：按稳定key锁容量行 → 应用环境槽 → Run → Stage → Job → Execution；同时写permit和计数。任何事务不得持有Run再反向申请容量/槽。
5. 手动触发：Run→Job，状态变更+操作者+待办+事件；配额由后续调度获取。
6. 执行认领：外部准入与Jenkins归属核对后，Run→Job→Execution复核取消/失败与canonical唯一性；凭据访问最后再核对当前资格。最后一次检查后的撤销不能保证截断已启动脚本，保存检查时间并对后续执行重新校验。
7. 结果/终结：预读资源标识，再按容量→槽→Run→Job→Execution重读锁定，结果/产物/事件/推进任务原子保存；确认终结才释放相关permit和Run槽。
8. 任务领取事务只锁任务行并结束，再进入业务事务。唤醒递增generation；ack只确认claimed_generation，若有更新保留待处理，防止处理期间新事件丢失。lease_fence条件防旧Worker写回。

容量与槽不能用普通COUNT再INSERT代替原子保护。死锁/短事务冲突可有限重试当前数据库事务，外部提交不能被放进事务重试循环。

## 外部 API 约定

以下路径除登录/系统管理外均相对服务前缀，空间资源缩写 `S=/spaces/{spaceId}`。读取按当前权限过滤；编辑带If-Match，异步动作返回202+operationId/Run链接；具体错误规则见工程章节。

### CMDB API

| 方法与路径 | 用户能力与关键输入/输出 |
|---|---|
| POST /session；GET /session；DELETE /session | 登录、会话/CSRF/空间摘要、退出 |
| POST /account/change-password | 原密码/新密码，撤销旧会话 |
| GET/POST /admin/users；PATCH /admin/users/{id} | 账号创建、停用、重置凭据与引导改密 |
| GET/POST /spaces；GET/PATCH /spaces/{id} | 有权空间列表、创建、元数据与归档 |
| GET/POST S/members；PATCH/DELETE S/members/{userId} | 角色、成员、最后Owner保护 |
| GET/POST S/subsystems；GET/PATCH/DELETE S/subsystems/{id} | 分类CRUD与归档前条件 |
| GET/POST S/applications；GET/PATCH S/applications/{id} | 身份CRUD、同空间移动、归档/恢复 |
| GET/PUT S/applications/{id}/delivery-config | 基础代码镜像配置、contextVersion、ETag |
| GET/POST S/environments；PATCH S/environments/{id} | 环境、enabled/protected及顺序 |
| GET/PUT S/applications/{id}/targets/{envId} | 目标集群、Namespace、Deployment、容器和凭据引用 |
| POST S/applications/{id}/validations | 配置验证，返回检查操作id和逐项结果 |
| GET S/validations/{id} | 异步只读探测进度与结果 |
| GET/POST S/credential-references；PATCH S/credential-references/{id} | 登记/验证/撤销元数据，绝不回传密钥 |
| GET /admin/clusters、/admin/registries；POST/PATCH对应资源 | 平台连接与空间授权；空间下提供有权候选读取 |
| GET/POST S/releases；GET/PATCH S/releases/{id} | 计划CRUD、应用commit条目、ETag |
| POST S/releases/{id}/ready、/close、/cancel、/copy | 生命周期动作与原因；READY冻结计划 |
| GET S/audit-events | 对象/操作者/时间筛选与脱敏差异 |

### Pipeline 管理、配置与执行 API

| 方法与路径 | 用户能力与关键输入/输出 |
|---|---|
| GET S/job-templates；GET S/job-templates/{id}/versions/{version} | 可见市场、契约、示例、弃用/封禁提示 |
| GET/POST S/pipelines；GET/PATCH S/pipelines/{id} | 列表、身份、环境范围、启停/归档字段 |
| GET/PUT S/applications/{id}/pipeline-preferences | Admin/Owner设置同应用的可用默认流水线，If-Match |
| POST S/pipelines/{id}/copy | 指定草稿/修订、目标应用、重绑定清单 |
| DELETE S/pipelines/{id} | 仅完全未使用资源，ETag与二次条件校验 |
| GET/PUT S/pipelines/{id}/draft | 不完整草稿、ETag；保存冲突412 |
| POST S/pipelines/{id}/draft/reset | 从指定修订重置或清空，If-Match |
| POST S/pipelines/{id}/validate | 指定draftVersion或完整定义，返回问题与预览计划 |
| POST S/pipelines/{id}/revisions | If-Match+幂等键+说明，创建不可变修订 |
| GET S/pipelines/{id}/revisions；GET .../revisions/{revisionId} | 历史配置与差异查询 |
| POST S/pipelines/{id}/run-previews | 环境/ref/参数/计划/镜像，预检摘要；不创建Run、不提前声称解析分支 |
| POST S/pipelines/{id}/runs | 修订/环境/ref/参数/计划/原因+幂等键，202 Run |
| GET S/runs；GET S/runs/{runId} | 筛选历史；图、计数、快照、availableActions |
| POST S/runs/{runId}/reruns | 选择当前上下文的新运行，返回新Run与reRunOfId |
| POST S/job-runs/{id}/trigger | 原因+幂等键，放行单个Job |
| POST S/runs/{id}/cancel | 持久化停止意图、返回取消操作 |
| GET S/runs/{id}/events | SSE、Last-Event-ID、权限复核与reset |
| GET S/job-runs/{id}/logs；GET .../logs/download | opaque cursor分页、脱敏下载或明确过期/缺失 |
| GET S/pending-actions | 按当前用户空间/环境权限筛选手动等待 |
| GET S/artifacts；GET S/deployments | 同应用来源筛选、目标观测、部署事实 |
| POST S/config-validations；GET S/diagnostic-operations/{id} | 仅CMDB服务或有权管理员创建固定只读探测，绑定已授权配置版本 |
| POST S/applications/{id}/deployment-observations | 只读刷新当前目标状态，异步受配额控制 |
| GET S/release-summaries?releaseId=... | 前端组合CMDB计划与执行/部署；强校验同空间 |
| GET S/audit-events；GET S/metrics-summary | 管理审计、带口径与更新时间的概览 |

### Pipeline 管理员与内部 API

| 方法与路径 | 约束 |
|---|---|
| /admin/job-templates及其drafts/versions/test-runs | 平台管理员CRUD草稿、测试、发布、弃用、封禁；业务内容发布后不可变 |
| /admin/controllers、/node-pools、/space-executor-bindings | 连接/映射CRUD、If-Match、权限与活动引用检查 |
| POST /admin/controllers/{id}/connection-tests、/execution-tests | 区分无副作用验证与真实健康构建 |
| GET /admin/executions/uncertain；POST .../{id}/reconcile | 当前已知事实、重新核对，不开启替代脚本 |
| POST /admin/executions/{id}/confirm-termination | 必须证据与原因，可信终止/隔离后收敛，不提供强制成功 |
| GET /admin/cleanup-operations；POST .../{id}/retry | 归档与清理异常；重试不删除正在使用的资源 |
| CMDB POST /internal/v1/authorize | 服务身份+用户会话摘要/动作/空间，返回当前判定，不接受客户端自报角色 |
| CMDB GET /internal/v1/spaces/{s}/applications/{a}/delivery-context?environmentId=... | 同空间一致快照与contextVersion，无明文密钥 |
| CMDB POST /internal/v1/execution-admissions | 当前空间/目标/凭据准入，不改快照、不跨库事务 |
| Pipeline POST /internal/v1/executions/{id}/claim | 受限Runner身份、build归属核对、canonical唯一 |
| Pipeline GET /internal/v1/executions/{id}/spec | 仅canonical可读，scope严格校验，控制身份不进入Agent |
| Pipeline POST /internal/v1/executions/{id}/observations | 可选加速上报，不替代Worker最终轮询与输出核对 |

管理员资源CRUD遵循GET集合/详情、POST创建、PATCH编辑；动作使用POST且明确具体效果，实施OpenAPI须展开表内缩写，不允许以泛型动态操作端点实现。

## 关键返回示意

```json
{
  "code": "REFERENCE_NOT_PREVIOUS_STAGE",
  "message": "镜像输入只能引用前序 Stage 的输出",
  "requestId": "req-example",
  "details": [{
    "stageKey": "quality",
    "jobKey": "deploy_app",
    "input": "image_ref",
    "fieldPath": "stages[0].jobs[1].inputs.image_ref"
  }]
}
```

Run详情的availableActions包含动作名与不可用原因，前端用它保持交互一致；操作端点仍重新鉴权、锁定和检查。运行状态、同步健康、日志归档状态分别返回，不用一个FAILED字段表达所有故障。

## 完整数据示例

- [CMDB内部交付上下文](../examples/cmdb.delivery-context.v1.json)：当前CMDB事实与凭据引用，无明文密钥。Controller/节点池映射由Pipeline补入，不由CMDB重复维护。
- [Definition](../examples/pipeline.definition.v1.json)与[结构Schema](../schemas/pipeline.definition.v1.schema.json)：编辑表示及完整发布的结构要求。
- [模板版本](../examples/job-template-version.v1.json)：原子能力契约、脚本与效果。
- [ExecutionSpec](../examples/job.execution-spec.v1.json)：已解析输入、固定commit、当前ownerKind、选定资源与摘要。
- [Result](../examples/job.result.v1.json)：canonical结果封装与受Schema约束的输出。

CMDB HTTP字段使用camelCase，内部Runner的固定上下文字段按示例使用snake_case，模板输入/输出key由模板Schema定义。client在真实服务边界做一次显式映射；不是旧字段别名，也不能同时接受两套写法。
