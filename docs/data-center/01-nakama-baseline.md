# 01 · Nakama 数据层基线

记录 Nakama 现有数据能力的实测结论，作为自建数据中心的参照。所有结论均标注源码位置，便于版本升级后复核（本文基于仓库快照 `cbec6ab`）。

## 1. 存储分层

一句话概括：**强类型字段兜住身份与安全，JSONB 兜住业务，Runtime 云函数是唯一权威写入方。**

| 层 | 载体 | 承载内容 |
|---|---|---|
| 强类型列 | `users` 表 | 身份、档案、各平台 ID、计数、时间戳 |
| 半结构化列 | `users.metadata`、`users.wallet` | 自定义档案、资产余额 |
| 文档型 | `storage` 表 | 背包、卡牌、任务进度、玩家设置 |
| 流水型 | `wallet_ledger`、`purchase` | 资产变动审计、IAP 订单 |
| 内存态 | session / status / match registries | 在线状态、对局，不持久化 |

建表语句：`migrate/sql/20180103142001_initial_schema.sql`。

## 2. 账号主表 `users`

关键列：`id`(UUID) / `username`(唯一) / `display_name`(昵称) / `avatar_url` / `lang_tag` / `location` / `timezone` / `metadata`(JSONB) / `wallet`(JSONB) / 各平台 ID / `edge_count` / 四个时间戳。

两点结论：

- **没有 level / exp 字段。** Nakama 不承载任何具体玩法数值，等级经验必须自行设计。
- **`metadata` 是整体覆盖，不是合并。** `server/core_account.go:373-376` 生成的是 `metadata = $N`。并发改不同字段会互相覆盖（last-write-wins）。对比 `server/core_wallet.go:272` 用了 `metadata = metadata || $2` 合并，说明作者知情且刻意区别对待。

## 3. 钱包 `wallet`

- 运行时类型是 `map[string]int64`（`server/core_wallet.go:145`），**key 自由 → 天然多货币**，新增货币不需要 DDL。
- 更新是增量语义：`newValue = walletMap[k] + v`（`:188-201`）。
- 唯一业务校验：`newValue < 0` 抛 `WalletNegativeError`，整个事务回滚（`:191-199`）。**无上限约束**。
- 并发安全：`SELECT ... FOR UPDATE`（`:126`）锁整行；写回前 `sort.Strings(updateOrder)` 按 userID 排序（`:231`）避免批量更新死锁。
- 局限：只有 int64，无法承载「一件带强化等级、词条、耐久的装备」。

## 4. 通用文档存储 `storage`

```
PRIMARY KEY (collection, key, user_id)
value    JSONB
version  VARCHAR(32)   -- md5(value)
read     SMALLINT      -- 0 无 / 1 仅本人 / 2 公开
write    SMALLINT      -- 0 禁写 / 1 本人可写
```

三元主键的语义：`collection` = 命名空间（表名），`key` = 主键，`user_id` = 归属/分区键。配合 `version`（乐观锁）与行级 `read/write`，等价于「一张表里的无数张表」，新增业务域无需 DDL。

关键机制：

| 机制 | 实现 | 位置 |
|---|---|---|
| 乐观锁 OCC | `version = md5(value)`，写入时 `WHERE version = $8` | `core_storage.go:76`、`:785` |
| 读权限过滤 | `read = 2 or (read = 1 and user_id = $4)` | `core_storage.go:547` |
| 归属强制 | 客户端写入时 `OwnerID` 取 Token，不取请求体 | `api_storage.go` |
| 字段索引 | bluge 内存索引，按 JSON 字段过滤排序 | `storage_index.go` |

索引注册 API：`initializer.RegisterStorageIndex(name, collection, key, fields, sortableFields, maxEntries, indexOnly)`；查询用 `nk.StorageIndexList(...)`。

## 5. 写入权限边界（最重要的设计）

| 数据 | 谁可写 | 机制 |
|---|---|---|
| `wallet`、`metadata` | 仅服务端 | 对外 API 契约 `apigrpc/apigrpc.proto` 中无 wallet 方法；`api_account.go:160` 把 metadata 硬编码为 `nil` |
| 等级、战力、进度 | 仅服务端（**需自行设 `write=0`，默认不是 0**） | storage 行级权限 |
| UI 偏好、按键布局 | 客户端 | `write=1` |
| 他人数据 | 均不可 | `OwnerID` 由 Token 决定 |

结论：**「权威写入」是按字段成立的，不是全局成立的。** 框架提供能力，是否收紧由使用者决定。把权威数值放进 `write=1` 的 collection，等于自毁防线——这是同类项目最常见的漏洞来源。

## 6. 事务边界

`server/core_multi.go:27` 的 `MultiUpdate` 把账户更新、storage 写入/删除、钱包更新放在同一个 pgx 事务中：

```
updateAccounts → storageWriteObjects → storageDeleteObjects → updateWallets
```

因此「扣钱 + 发道具 + 加经验」可用一次调用完成，失败整体回滚，无需业务侧补偿逻辑。这是 Nakama 最值得借鉴的一处设计——**它比任何单表能力都重要**。

## 7. 对外接口形态

| 入口 | 端口 | 说明 |
|---|---|---|
| gRPC | `7349`（= `socket.port - 1`） | `api.go:159-170` |
| HTTP/JSON REST | `7350` | grpc-gateway，本地回环 `127.0.0.1:7349` 转 gRPC（`api.go:203-225`） |
| WebSocket | `7350/ws` | 实时对战、组队、聊天、推送（`api.go:235`） |
| Console | `7351` | 运营后台 |

自定义业务逻辑通过 `/v2/rpc/{id}` 暴露（`api.go:244`），鉴权双通道：`?http_key=` 表示「服务端调用，无用户身份」，`Authorization: Bearer <session_token>` 表示「有用户上下文」（`api_rpc.go:47-77`）。

## 8. 数据库约束

- 仅支持 PostgreSQL 系（CockroachDB 或其他 Postgres wire-compatible），见 `README.md:34`。
- 代码深度绑定 Postgres：157 个 Go 文件中 27 个 import `jackc/pgx`；190 处 `::UUID` 强制转换、15 处 `unnest(...)`、23 处 `ON CONFLICT`。
- **结论：迁移到 MySQL 等于重写整个 SQL 访问层，不可作为配置项处理。** 若公司技术栈强制 MySQL，需在新系统中从零设计 SQL 层，仅可借鉴其数据模型与权限模型，不能复用其代码。

## 9. 可直接复用的部分 vs 需重建的部分

可直接复用（概念与模型）：

- `users` 强类型 + `metadata` 半结构化的分层
- `wallet` 的「key 自由 + 增量 + 非负校验 + 行锁 + 排序防死锁」
- `storage` 的三元主键 + md5 乐观锁 + 行级读写权限
- `MultiUpdate` 的单事务语义
- `purchase` 以 `transaction_id` 为主键实现幂等

必须重建（Nakama 无此能力）：

- 背包/卡牌/装备（无内建，只能基于 storage 自建）
- 对战结果上报与结算
- 关卡分析与埋点分析链路
- 配置中心
