# ARCH-02 数据访问与事务代码规范

状态：MySQL、GORM和复杂调度显式SQL已确认。本文件保留直接事务写法示例，现行推荐基线见 [工程规范 ARCH-02](../../design/01-engineering.md#arch-02-数据访问与显式事务) 与 [数据事务契约](../../design/11-data-model-and-api.md)。原强制UnitOfWork/TxStores已撤回。

## 数据访问

简单查询与更新直接使用 GORM。复杂或重复 SQL 集中到所在业务模块的 repository.go，使用普通函数或具体结构体即可，不建设统一泛型 CRUD 框架。

业务流程放在 service.go，HTTP 参数与响应放在 handler.go。需要请求校验、脱敏或不同接口表示时使用 DTO；没有差异的内部数据不机械复制多份。

CMDB 与 Pipeline 各自维护数据，通过 API 交互，不跨服务查询业务库。

## 事务写法建议

需要一起成功或失败的操作，在 Service 中显式使用 GORM Transaction。事务内所有 ORM 与显式 SQL 都使用同一个 tx；需要调用其他数据函数时显式传 tx，不藏进 context。

下面仅演示一组原子写入，不是已实现的流水线业务代码：

```go
return s.db.WithContext(ctx).Transaction(func(tx *gorm.DB) error {
    if err := tx.Create(&run).Error; err != nil {
        return err
    }
    return tx.Create(&task).Error
})
```

GORM 的事务回调支持这些操作一起提交或回滚，显式 SQL 也可以通过该 tx 执行。[GORM 事务说明](https://gorm.io/docs/transactions.html)

- 事务内不能混用全局 db 或另外取得的连接。
- 不在事务里等待 Jenkins、CMDB 等网络调用。
- 不把 tx 交给后台 goroutine，回调结束后不继续使用。
- 状态更新需要明确前置条件或并发保护，检查错误及受影响行数。
- context 用于取消、超时和追踪；请求结束不取消已经持久化的后台工作。

不再为上述规则额外定义 UnitOfWork 接口、TxStores 集合或事务依赖注入层。

## 错误与维护

SQL 使用参数绑定，动态排序和字段选择限制在明确允许的范围。资源不存在、状态冲突和数据库错误应能区分，不通过吞错让流程继续。

关键事务和并发规则用真实MySQL验证。迁移遵守ENG-001：显式升级数据，不双读旧格式；MySQL8.4、goose迁移、锁顺序与超时等现行推荐值见design目录。

## 当前结论

数据访问技术选择保留。当前规范建议是 Service 中显式 GORM 事务、SQL 按复杂度集中、同一个事务连接贯穿原子操作；不再要求复杂的事务框架。具体业务数据结构尚未开始实现。

2026-10-16：用户澄清代码可读性和简单性优先，原强制事务抽象被替代。
