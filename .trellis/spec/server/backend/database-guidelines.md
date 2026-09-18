# SQLite、事务与迁移

## 连接与查询

**当前实现**：`server/src/db/database.ts` 的 `initDb()` 打开 better-sqlite3，设置 busy timeout、durability pragmas、foreign_keys，再依次调用 createTables、runMigrations、runSeeds。默认数据库在 server/data，测试进程用 `:memory:`；journal/synchronous 可配置，不应声称所有部署强制 WAL。

**新增代码约束**（`server/CLAUDE.md`）：Nest 代码注入 `DatabaseService`，不新建连接或导入全局 db。值用 `?` 绑定，SQL 标识符只能来自字面白名单；不能拼接用户字段名或环境变量到 SQL。`DatabaseService.get<T>` 的泛型只是静态声明，不是对 DB 行做了运行时验证。

`server/src/nest/collections/collections.service.ts` 的 `accessibleCollectionIds`、`isVisible` 使用绑定参数，限制 owner/accepted member；不能只检查资源 ID 存在而不检查调用者与父 trip/collection 的归属。`DatabaseService.rosterUserIds` 用于验证被引用的参与人属于同一行程，访客参与人也在 roster 中。

## 多步写入必须同步成组

`server/src/nest/database/database.service.ts` 的 `transaction(fn)` 执行同步 better-sqlite3 transaction。多表创建/删除、重排、金额分配必须一起成功或回滚，不手写 BEGIN/COMMIT，不把 async callback 或网络请求放进事务。网络、文件等异步操作先设计失败顺序，事务后再通知消费者；不要把一次 SQL 成功误认为整个业务完成。

参考 `server/src/nest/budget/budget.service.ts` 的事务化写入及结算逻辑；为新写入增加中途失败的回滚测试，不只断言调用了 transaction。

## 迁移只能追加

`server/src/db/migrations.ts` 的 `runMigrations()` 以数组位置对应 schema_version，不能重排、中插或删除。修复已发布步骤应追加纠正迁移，否则已升级数据库不会重新运行。

**当前实现**：普通函数迁移与 `UPDATE schema_version` 在同一事务内；失败中止启动。特殊 `{ raw }` 步骤因 PRAGMA 等需要在事务外运行，版本写入也是独立语句，这是受限例外，不可推广成普通迁移默认模式。

**测试证据**：`server/tests/unit/db/migration-version-atomicity.test.ts` 的 `MIGRATE-ATOMIC-001` 在隔离内存 DB 回放末尾迁移，确认版本更新时 `db.inTransaction === true`。不要用生产 DB 回退版本测试。

## 金额：内部整数分不等于线协议全是分

`BudgetService.splitEqualShares(totalCents, members, itemId)` 输入/输出整数分，使用 floor 和余数轮转，退款负金额同样必须精确加回总额。`calculateSettlement` 在行程币种中净额结算，优先冻结汇率，最后整体换算显示币种，避免动态汇率产生微额残差。

共享 `budgetItemPayerSchema.amount` 等线字段仍是费用自身币种金额。新增计算遵守整数分和精确对账，但不能直接把响应 amount 改成 cents；必须显式记录每个边界的单位。服务端与客户端同名分摊镜像需同步验证。

## 验证

根目录：`npm run test --workspace=server -- tests/unit/db/migration-version-atomicity.test.ts`；金额改动选 `tests/unit/nest/budget.service.calc.test.ts` 与 `.db.test.ts`。迁移还需空库、升级、重复启动及失败回滚场景。命令会运行隔离测试，不是生产维护工具；规范文档修改不运行数据库操作。
