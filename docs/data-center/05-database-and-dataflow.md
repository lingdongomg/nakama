# 05 · 库表结构与数据流

回答四个问题：有哪些表、读写走什么策略、缓存在哪、数据怎么上报。所有结论标注源码位置，基于仓库快照 `cbec6ab`。

## 0. 三个结论先说

1. **19 张表，全部以 `users` 为中心。** 没有任何一张表是为「对战」或「事件」准备的。
2. **没有 Redis，没有外部缓存。** 11 处缓存全部是进程内结构，多节点部署时各自独立、互不可见。
3. **对战不落库、事件不落库。** 这两块是自建数据中心真正的起点，也是 `02-gap-matrix.md` 判为「自建」的根因。

## 1. 表清单

建表总入口 `migrate/sql/20180103142001_initial_schema.sql`，后续 19 个迁移增量改列。

| 域 | 表 | 主键 | 说明 |
|---|---|---|---|
| 身份 | `users` | `id` | 账号主表，含 `metadata` / `wallet` 两个 JSONB |
| | `user_device` | `id`(设备号) | 一用户多设备；`UNIQUE(user_id, id)` |
| | `user_tombstone` | `(create_time, user_id)` | 已删账号墓碑，防重注册冲突 |
| | `console_user` | `id` | 后台账号，与玩家账号完全隔离 |
| 社交 | `user_edge` | `(source_id, state, position)` | 好友关系，`state` 0-3（好友/已发邀请/收到邀请/拉黑） |
| | `groups` | `(disable_time, lang_tag, edge_count, id)` | 群组，`max_count` 默认 100 |
| | `group_edge` | `(source_id, state, position)` | 群成员，`state` 0-4（超管/管理/成员/申请/封禁） |
| 存档 | `storage` | `(collection, key, user_id)` | 通用文档存储，全库最核心的表 |
| 资产 | `wallet_ledger` | `(user_id, create_time, id)` | 钱包流水；余额本身在 `users.wallet` |
| 竞技 | `leaderboard` | `id` | 榜单定义，兼锦标赛字段 |
| | `leaderboard_record` | `(leaderboard_id, expiry_time, score, subscore, owner_id)` | 分数记录 |
| 通信 | `message` | `(stream_mode, stream_subject, stream_descriptor, stream_label, create_time, id)` | 聊天与群事件流 |
| | `notification` | `(user_id, create_time, id)` | 站内通知 |
| 交易 | `purchase` | `transaction_id` | IAP 订单，天然幂等 |
| | `subscription` | `original_transaction_id` | 订阅 |
| 运维 | `setting` | `key` | 全局配置 KV |
| | `users_notes` | `(user_id, create_time, id)` | 客服备注 |
| | `console_audit_log` | `(create_time, console_username, action, resource, id)` | 后台操作审计 |
| | `console_acl_template` | `id` | 后台权限模板 |

**没有的三张表：** `match`（对战）、`event`（埋点）、`inventory`（背包）。前两个是能力缺失，第三个是刻意用 `storage` 承载。

## 2. 五张核心表的字段设计

### 2.1 `users`：强类型列 + 两个 JSONB

```sql
metadata  JSONB NOT NULL DEFAULT '{}'    -- 自定义档案
wallet    JSONB NOT NULL DEFAULT '{}'    -- 资产余额，map[string]int64
```

各平台 ID 是**独立的强类型列**（`facebook_id` / `google_id` / `gamecenter_id` / `steam_id` / `custom_id` / `email`），每列自带 `UNIQUE`。不是关联表。

这个设计换来两个特性：

- 一个账号在每个平台只能绑一个 ID —— 数据库层面就锁死了，不需要应用层校验
- 换绑必须先把旧值置空，天然对应「先 Unlink 再 Link」

代价是新增登录方式要 `ALTER TABLE`。Nakama 至今支持六七种，够用；如果要做成可插拔的，就得改成关联表。

**没有 `level` / `exp` 列。** 等级经验必须自己设计，放 `metadata` 或 `storage`。

**系统用户固定为 `00000000-0000-0000-0000-000000000000`**，`user_edge` 和 `group_edge` 都有 `CHECK (source_id <> 全零)` 把它排除在外。所有「服务端发的」记录都用这个 ID 作为 sender。

### 2.2 `storage`：全库最值得抄的表

> 本节只讲它在整体数据流里的位置。字段、权限判定、客户端写入规则与使用规范见 `06-storage-design.md`。

```sql
PRIMARY KEY (collection, key, user_id)
version  VARCHAR(32)   -- md5(value)
read     SMALLINT NOT NULL DEFAULT 1 CHECK (read >= 0)
write    SMALLINT NOT NULL DEFAULT 1 CHECK (write >= 0)
```

**`read` / `write` 的默认值是 1，不是 0。** 意味着「建了就能自己写」。要收紧权威数据，必须显式传 0。这是 01 第 5 节点出的那个坑的数据库层根源。

三个索引，对应三种列表查询：

| 索引 | 服务什么查询 |
|---|---|
| `(collection, read, user_id, key)` | 查某人某集合的全部对象 |
| `(collection, read, key, user_id)` | 查某集合某 key 的所有人（跨用户） |
| `(collection, user_id, read, key)` | 带权限过滤的按人查 |

`version = md5(value)` 有个副作用：**相同内容写入产生相同版本号**。所以「改回原样」的重试不会被识别为变更，天然幂等。

### 2.3 `leaderboard_record`：主键里带分数

```sql
PRIMARY KEY (leaderboard_id, expiry_time, score, subscore, owner_id)
UNIQUE (owner_id, leaderboard_id, expiry_time)
```

这是个**用写换读**的设计：

- 主键顺序即排名顺序，取 TOP N 是一次索引范围扫描，不需要 `ORDER BY` + 排序
- 代价是分数变化会改变主键值，PostgreSQL 需要移动行（等价于 delete + insert）

写入用的是 `ON CONFLICT (owner_id, leaderboard_id, expiry_time) DO UPDATE`（`core_leaderboard.go:492`），冲突目标走的是 `UNIQUE` 约束而不是主键，所以能正确落到「同一个 owner 同一次赛季」这一行。

**`expiry_time` 在 `UNIQUE` 里**，这是赛季隔离的关键：同一玩家在不同赛季是两条独立记录，重置只需换 `expiry_time`，不动历史数据。

### 2.4 `wallet_ledger`：只记变化量，不记余额

```sql
changeset  JSONB  -- {"gold": 100}
metadata   JSONB
```

**没有 `balance_after`。** 与 `03-design-patterns.md` P4 的建议不同——Nakama 不存余额快照。

后果：查「某笔交易后余额是多少」必须把该用户之前的全部流水累加。数据量小时无感，几年后单用户几千条流水时，客服查询会很难受。

自建时建议补上这一列，成本极低。

### 2.5 `message`：按流聚簇

主键前四位是流的标识（`stream_mode` / `subject` / `descriptor` / `label`），后两位是 `create_time` 和 `id`。

这意味着同一频道的消息在物理上连续存放，读历史是一次范围扫描，不需要额外索引。

代价是消息只能按流查，做不了「查某人发过的所有消息」。要做内容审核得另建索引。

**没有自动清理。** 消息会无限增长，只能靠 Console 手动按时间删（`console_channel.go:52`）。线上必须自己加定时任务。

## 3. 缓存全景

11 处，全部进程内。

| 缓存 | 内容 | 填充时机 | 淘汰 | 跨节点 |
|---|---|---|---|---|
| `SessionCache` | 有效会话/刷新令牌 | 签发时 | 到期 | 不需要（JWT 自校验） |
| `SessionRegistry` | 在线会话与连接 | 连接时 | 断开 | 否 |
| `StatusRegistry` | 在线状态、关注关系 | 连接时 | 断开 | 否 |
| `MatchRegistry` | 运行中对局 + bluge 索引 | 创建对局时 | 对局结束 | 否 |
| `PartyRegistry` | 队伍 | 创建时 | 全员离开 | 否 |
| `LeaderboardCache` | 榜单定义 | 懒加载 + 启动 | 不淘汰 | 否（各节点各一份） |
| `LeaderboardRankCache` | 排名（skiplist） | 启动 `Fill` + 写入时 `Insert` | 按 `expiry` 裁剪 | 否 |
| `StorageIndex` | 存储索引（bluge） | 启动全量 `Load` + 写后同步 | 超 `maxEntries` 淘汰最旧 | 否 |
| `LoginAttemptCache` | 登录失败锁定 | 失败时 | TTL | 否 |
| `RuntimeLocalCache` | 脚本侧 KV | 脚本调用 | TTL | 否 |
| Satori 响应缓存 | HTTP 响应 | 首次请求 | `CacheTTLSec` | 否 |

**关键推论：多节点部署时，排名缓存和存储索引在每个节点各算一份。** 节点 A 写入的分数进了 A 的 skiplist，节点 B 的缓存里没有。这就是官方文档说「索引最终一致」的真实原因——不是延迟，是**物理上就是两套**。

Nakama 的应对是：所有列表接口都回表查库（`LeaderboardRecordsList` 直接 `SELECT`），缓存只用来加速「查单个 owner 的名次」这一种场景（`core_leaderboard.go:553`）。

### 3.1 哪些地方刻意没有缓存

| 操作 | 实现 | 位置 |
|---|---|---|
| 读存档 | 直接 `db.QueryContext` | `core_storage.go:324` |
| 读账号 | 直接查 `users` | `core_account.go` |
| 拉榜单列表 | 直接 `SELECT` 全表序 | `core_leaderboard.go:127` |
| 读聊天历史 | 直接查 `message` | `core_channel.go` |

**Nakama 几乎没做读缓存。** 它靠 PostgreSQL 的 shared buffer 和索引硬抗，把复杂度留给了数据库。

这个选择对自建有参考价值：如果你的 QPS 没到单库瓶颈，就不需要引入 Redis。过早加一层缓存，只会多一处不一致的来源。

## 4. 读写策略

### 4.1 写：全部同步写穿

没有 write-behind，没有消息队列缓冲，所有业务写都是同步 SQL。

事务边界只有两处：

| 场景 | 是否事务 | 位置 |
|---|---|---|
| `MultiUpdate`（改账户 + 写存档 + 改钱包） | **是**，单 pgx 事务 | `core_multi.go:27` |
| 普通 `StorageWrite` | 否，逐条语句 | `core_storage.go:785/811/832` |

所以「扣钱 + 发道具 + 加经验」要么走 `MultiUpdate` 一次搞定，要么自己承担部分成功的风险。这是 01 第 6 节说「比任何单表能力都重要」的那一点。

### 4.2 排行榜写入：一个值得单独看的 SQL

`core_leaderboard.go:415` 的 `LeaderboardRecordWrite`，核心是一条 UPSERT：

```sql
INSERT INTO leaderboard_record (...)
VALUES (...)
ON CONFLICT (owner_id, leaderboard_id, expiry_time)
DO UPDATE SET score = GREATEST(leaderboard_record.score, $4),
              num_score = leaderboard_record.num_score + 1,
              ...
WHERE leaderboard_record.score < $4 OR leaderboard_record.subscore < $5
RETURNING ...
```

三个细节：

1. **`best` 算子在 SQL 层实现**，用 `LEAST` / `GREATEST`，不在应用层比较。省一次读。
2. **`WHERE` 子句挡住「更差的分数」**。新分数不更优时不执行更新，`RETURNING` 返回空行，Go 侧检测到 `unchanged`，**连内存缓存都不动**。等于用 SQL 的谓词做了一次前置校验。
3. **先写库、后写缓存**（`:556` 的 `rankCache.Insert` 在 SQL 之后）。缓存永远是派生的，不是真相。

这个模式可以直接搬到自建系统：**用 `ON CONFLICT ... WHERE` 把「要不要更新」的判断推给数据库**，避免读-判断-写三步。

### 4.3 对局索引：批量异步刷新

`MatchRegistry` 维护运行中对局的 bluge 索引，供 match listing 查询。但**不在每次对局状态变化时更新索引**，而是写进 `pendingUpdates` map（`match_registry.go:145`），由一个按 `LabelUpdateIntervalMs` 触发的 ticker 批量刷入（`:184`）。

这是全库唯一一处主动的异步写。理由是对局标签变化频繁但查询不频繁，实时同步索引是浪费。

## 5. 数据上报的三条通路

三条路，三种命运。

### 5.1 存档上报 → `storage` 表（落库）

客户端 `PUT /v2/storage`，直接写库，受 `read` / `write` 约束，`version` 做乐观锁。

这是唯一一条「客户端数据进库」的标准通路。特征：

- 同步落库，无缓存
- 有权限模型
- 有版本冲突检测
- 原子性：批量写在单语句内（多值 INSERT），但**跨表不保证**

### 5.2 事件上报 → `Event` 接口（不落库）

```go
// server/api_event.go
func (s *ApiServer) Event(ctx context.Context, in *api.Event) (*emptypb.Empty, error)
```

`api.Event` 只有四个字段：

```protobuf
string name
map<string,string> properties
Timestamp timestamp
bool external          // 是否来自客户端直调
```

**整个文件没有任何 db 调用**（`grep -c "db\." api_event.go` = 0）。事件只有两个去处：

1. `s.runtime.Event()` —— 你在 runtime 里 `RegisterEvent` 的回调（`api_event.go:57`）
2. Satori —— 如果配置了 `satori.url` 等参数（`runtime.go:710-718`）

**结论：Nakama 没有埋点存储能力。** 这个接口是为了对接 Satori 而存在的转发口。不接 Satori 也不写 handler 的话，事件进来就消失了。

这正是 `02` 能力 11 判为「自建」的源码依据。

### 5.3 对战上报 → WebSocket `MatchDataSend`（不落库）

客户端发消息 → 服务端 `matchLoop` 处理 → 广播给对局参与者。全程在 `MatchHandler` 的内存结构里。

`match_handler.go` 与 `match_common.go` **零 `sql.DB` 引用**。对局引擎在设计上就不认识数据库。

## 6. 对战数据：Nakama 存了什么

**什么都没存。**

| 阶段 | 位置 | 持久化 |
|---|---|---|
| 创建对局 | `MatchRegistry.matches`（`MapOf[uuid, *MatchHandler]`） | 否 |
| 运行中状态 | `MatchHandler` 内部字段 | 否 |
| 对局结束 | 从 map 移除，GC | 否 |
| 节点重启 | 全部消失 | 否 |

`matches` 是 `MapOf` 而不是普通 map，因为要支持并发读写。`matchCount` 是 `atomic.Int64`，说明「当前对局数」是被高频读取的指标。

### 6.1 自己存的两条路

| 方式 | 时机 | 写入量 | 崩溃影响 | 适用 |
|---|---|---|---|---|
| **每 tick 快照** | `matchLoop` 内定时 | 极大（10 tick × 10 人局 = 每秒百次写） | 可恢复到最近快照 | 长对局、可重连 |
| **结算落库** | `matchTerminate` 或结算 RPC | 极小（一局一次） | 崩溃丢这一局 | 短对局、休闲玩法 |

**推荐组合**：结算落库为主，关键节点（回合结束、掉线时）补快照。

`matchTerminate` 会被调用的前提是 `grace_period_sec > 0`，否则进程关闭时直接丢弃。这一点在部署配置里必须设。

### 6.2 落库的正确姿势

不要在 `matchLoop` 里直接 `nk.storageWrite`。理由：

- `matchLoop` 是 tick 驱动，写库会阻塞 tick，导致同局所有玩家卡顿
- storage 写入是同步 SQL，一次慢查询能拖垮整局的帧率

正确做法是**把结算数据通过 `nk.httpRequest` 或事件队列送到外部队列，由消费者落库**。Nakama 的 `RuntimeEventQueue`（`runtime_event.go:23`）提供了进程内队列，但队列满了会直接丢弃（`:64` 的 `CountDroppedEvents`）——需要监控这个指标。

## 7. 对自建的启示

### 可以直接照搬

| 设计 | 理由 |
|---|---|
| `storage` 三元主键 + md5 版本 + 行级读写 | 一张表承载所有业务域，免 DDL |
| `users` 强类型 ID 列 + 两个 JSONB | 平台唯一性由约束保证，不靠代码 |
| `leaderboard_record` 主键含分数 | 取 TOP N 零排序 |
| `purchase` 以 `transaction_id` 为主键 | 天然幂等 |
| 排行榜 UPSERT + `WHERE` 谓词 | 把判断推给数据库，省一次读 |
| 所有缓存进程内、不做分布式缓存 | 少一处不一致来源 |

### 必须自己补

| 缺口 | 说明 |
|---|---|
| 流水缺 `balance_after` | 补一列，否则客服查询要累加全历史 |
| `storage` 默认 `write=1` | 建表时就改成默认 0，或强制显式传 |
| 消息无自动清理 | 线上必须加定时任务，否则单表无限增长 |
| 无对战持久化 | 见 §6，自己设计快照与结算 |
| 无事件存储 | 见 §5.2，`Event` 只是转发口 |
| 无 TTL / 归档机制 | 除排行榜 `expiry_time` 外，所有表只增不减 |

### 一个反面提醒

Nakama 不在读路径上加缓存，是因为它假设「PostgreSQL 扛得住」。这个假设在单库 QPS 未打满时成立。但如果自建系统要做多节点、且榜单/索引类查询量大，就会出现「每个节点各算一份、结果不一致」的问题——这正是 Nakama 在多节点下只能回表查询的原因。

**要么接受回表，要么一开始就把共享缓存设计进去，不要中途加。**
