# 06 · storage 设计与使用规范

`storage` 是 Nakama 里唯一一张「用一张表承载所有业务域」的表。背包、卡牌、任务进度、玩家设置、任何自定义业务对象，全部落在这一张表上。

它的设计密度很高，用错一处就会把权限防线交给客户端。本文按「结构 → 机制 → 接口 → 判定 → 命名 → 规范」六层展开，所有结论标注源码位置（快照 `cbec6ab`）。

## 0. 一句话结论

**`storage` 用三元主键换来免 DDL 的建模自由度，用行级 `read`/`write` 换来声明式权限，用 `md5(value)` 换来无读的乐观锁。代价是：默认权限对客户端敞开，且 `collection` / `key` 都没有白名单，必须自己建立约束。**

## 1. 表结构与三元组

```sql
CREATE TABLE storage (
  collection  VARCHAR(128) NOT NULL,
  key         VARCHAR(128) NOT NULL,
  user_id     UUID         NOT NULL,
  value       JSONB        NOT NULL DEFAULT '{}',
  version     VARCHAR(32)  NOT NULL,
  read        SMALLINT     NOT NULL DEFAULT 1 CHECK (read >= 0),
  write       SMALLINT     NOT NULL DEFAULT 1 CHECK (write >= 0),
  create_time TIMESTAMPTZ  NOT NULL DEFAULT now(),
  update_time TIMESTAMPTZ  NOT NULL DEFAULT now(),
  PRIMARY KEY (collection, key, user_id)
);
```

### 1.1 字段职责

| 字段 | 作用 | 设计要点 |
|---|---|---|
| `collection` | 命名空间，等价「表名」 | 无需 DDL，写入时自动存在；**零预置，全部由业务定义** |
| `key` | 对象标识，等价「主键」 | 客户端可自行构造，省一次查询 |
| `user_id` | 归属者，等价「分区键」 | 全零 UUID 表示全局对象；**客户端写入时被强制为 token 中的 userID** |
| `value` | 业务数据 | JSONB，**必须是 JSON 对象** |
| `version` | 乐观锁 | `md5(value)`，32 位十六进制 |
| `read` / `write` | 行级权限 | 见 §3 |

三元组唯一确定一个对象。三者都参与主键，所以「同一 collection 同一 key」在不同用户下是两条独立记录。

### 1.2 三个索引的用途

| 索引 | 服务的查询 |
|---|---|
| `(collection, read, user_id, key)` | 查某人某集合的全部对象 |
| `(collection, read, key, user_id)` | 查某集合某 key 的所有人（跨用户） |
| `(collection, user_id, read, key)` | 带权限过滤的按人查 |

三个索引都把 `read` 放在中间位置，因为列表查询必须先按可见性过滤（`read = 2 or (read = 1 and user_id = $4)`，`core_storage.go:547`），再做分页。把 `read` 放进索引，过滤在索引层完成，不用回表。

### 1.3 三元组的角色分工

```
player_save                        ← collection：一类业务对象（表名）
├── user_A
│   ├── slot_1                     ← key：对象标识（主键）
│   └── slot_2
└── user_B
    └── slot_1
```

| 维度 | 决定什么 | 变更成本 | 例子 |
|---|---|---|---|
| `collection` | **schema**：权限、索引、查询入口 | 极高（要迁移全部数据） | `player_progress` |
| `key` | 实例：具体哪个对象 | 低（换一个 key 而已） | `slot_1`、`main` |
| `user_id` | 归属：谁的数据 / 是否全局 | 不可变 | 用户 UUID 或全零 |

这个区分是理解后续所有约束的前提：**collection 是 schema 级决策，key 是实例级决策，两者的归属权不应该一样。**

### 1.4 全局对象

`user_id` 为全零 UUID（`00000000-0000-0000-0000-000000000000`）时表示**系统对象**：不归属任何玩家，所有人在 `read=2` 下都能读。

适合放**配置、公告、赛季定义、公共素材清单**这类「所有玩家看到同一份」的数据。

```go
// 服务端写入全局对象
nk.StorageWrite(ctx, []*runtime.StorageWrite{{
    Collection:      "global_config",
    Key:             "season_01",
    UserID:          "",   // 空字符串 = 全零 UUID = 全局对象
    Value:           json,
    PermissionRead:  2,    // 所有人可读
    PermissionWrite: 0,    // 只有服务端能写
}})
```

`UserID` 传空字符串会被转成全零 UUID（`core_storage.go:728-730`）。客户端读不到别人的对象，但读得到全局对象——这正是它存在的意义。

**注意**：全局对象的 `write` 必须显式设 `0`。虽然 owner 是全零 UUID、没有真实用户能写，但配合默认值 `1` 在语义上是错的，不要留下这个歧义。

## 2. version：内容指纹，不是修订号

`version = md5(value)` 而不是自增序号，带来三个衍生行为：

**① 内容相同 → 版本相同。** 写入相同内容不被识别为变更，天然幂等。这让「重试」变安全：客户端超时后原样重发，不会产生额外的更新。

**② 不需要读就能算。** 服务端算完 md5 直接带进 `WHERE version = $n`，省掉「先读再写」的一次往返。这是它比自增序号更省的地方。

**③ 32 位定长。** 比 UUID 或纳秒时间戳都省，作为索引列也更快。

**限制：无法从 version 判断先后。** 它是「内容指纹」，不是「修订号」。不能用它做「谁的更新」这类判断，也排不出时间顺序。要判断先后必须看 `update_time`。

## 3. 权限模型

`read` 三档、`write` 两档。

| | `read=0` | `read=1` | `read=2` |
|---|---|---|---|
| 所有者本人 | 不可读 | 可读 | 可读 |
| 其他玩家 | 不可读 | 不可读 | 可读 |
| 服务端 | 可读 | 可读 | 可读 |

| | `write=0` | `write=1` |
|---|---|---|
| 所有者本人 | 不可写、不可删 | 可写、可删 |
| 服务端 | 可写 | 可写 |

### 3.1 默认值是 1，不是 0

```sql
read   SMALLINT NOT NULL DEFAULT 1
write  SMALLINT NOT NULL DEFAULT 1
```

意味着：**新建 collection 时如果不显式传权限，玩家对写进去的数据有完整读写权。**

这是 `01-nakama-baseline.md` 第 5 节说的「把权威数值放进 `write=1` 的 collection 等于自毁防线」的数据库层根源。框架提供了能力，是否收紧完全由使用者决定。

**建 collection 时的第一条规则：所有 collection 一律显式声明权限，不依赖默认值。**

## 4. 接口契约

在讨论「谁写什么」之前，先看清接口是怎么组织的，否则会误以为「能调」等于「该调」。

### 4.1 一份 proto，两种协议

所有对外接口定义在 `apigrpc/apigrpc.proto`，由 `grpc-gateway` 自动生成 HTTP 映射：

```proto
rpc WriteStorageObjects (api.WriteStorageObjectsRequest) returns (api.StorageObjectAcks) {
  option (google.api.http) = { put: "/v2/storage" body: "*" };
}
```

gRPC 和 REST **不是两套接口**，是同一份 contract 的两种编码。改 proto 一处，两边同时生效，不会出现两套接口行为不一致。

### 4.2 三种鉴权身份

| 方式 | 标识什么 | 用在哪 |
|---|---|---|
| `Authorization: Basic base64(serverKey:)` | **应用身份**（哪个游戏） | 客户端初始化连接 |
| `Authorization: Bearer <session>` | **用户身份** | 绝大多数 API |
| `?http_key=<runtime.http_key>` | **服务端身份**（无用户上下文） | 服务端调 RPC |

前两个通常同时出现：Basic 说明「属于哪个游戏」，Bearer 说明「是哪个玩家」。第三种用于机器对机器调用，在 RPC 里靠它区分：

```go
// server/api_rpc.go:52
if httpKey := queryParams.Get("http_key"); httpKey != "" {
    // 服务端调用，ctx 里没有 userId
} else if auth := r.Header["Authorization"]; len(auth) >= 1 {
    // 用户调用，ctx 里有 userId
}
```

因此 RPC 里可以用 `ctx.Value(runtime.RUNTIME_CTX_USER_ID) == nil` 判断「这是不是服务端调用」——这是收口外部访问的常用手段。

### 4.3 两类接口

| 类型 | 例子 | 契约 | 谁定义 |
|---|---|---|---|
| **内建 CRUD** | `/v2/storage`、`/v2/leaderboard/{id}`、`/v2/account` | 固定 schema | 框架 |
| **自定义 RPC** | `/v2/rpc/{id}` | `payload` 是字符串，内容随你定 | 你 |

RPC 的 payload 是原始字符串（不是结构化 message），意味着入参格式完全由你定义。这是绕过固定 schema 的唯一出口。

### 4.4 写入请求的字段

```go
// vendor/.../api/api.pb.go:8345
type WriteStorageObject struct {
    Collection      string
    Key             string
    Value           string
    Version         string
    PermissionRead  *wrapperspb.Int32Value
    PermissionWrite *wrapperspb.Int32Value
}
```

六个字段全部由请求体提供。服务端的校验只有一条：

```go
// server/api_storage.go
if object.GetCollection() == "" || object.GetKey() == "" || object.GetValue() == "" {
    return nil, status.Error(codes.InvalidArgument, "Invalid collection or key value supplied.")
}
```

**只校验非空——不校验格式、不校验白名单、不校验长度。** 这直接引出 §6 的话题。

## 5. 客户端写入的判定路径

这是最容易误解的部分。源码的判定比文档更精确。

### 5.1 三条规则

**规则一：归属者不可伪造。**

```go
// server/api_storage.go
for _, object := range in.GetObjects() {
    ops = append(ops, &StorageOpWrite{
        OwnerID: userID,        // 来自 token，不用请求体的值
        Object:  object,
    })
}
```

请求体里传的 `userId` 直接被忽略。「写别人的存档」在协议层就堵死了。

**规则二：新建对象时不做权限检查。**

```go
// server/core_storage.go:768
writeCheck := ""
if !authoritativeWrite {              // 客户端写入
    writeCheck = " AND storage.write = 1"
}
```

这个 `writeCheck` 只拼到 **UPDATE** 语句上。`version="*"` 的插入分支没有它，注释原话：

> Existing permission checks are not applicable for new storage objects.

结论：**客户端可以创建任意权限的新对象，包括把自己锁死（`write=0`）。**

这个特性可以反着用——需要「创建即锁定」的对象，让客户端首次写入时传 `write=0`，之后它自己都改不了。

**规则三：已有对象的写权限才是真闸门。**

一旦对象存在且 `write=0`，客户端的所有写入和删除都会被拒。删除的 SQL 对比更直观：

```sql
-- 服务端（权威删除）
DELETE FROM storage WHERE collection=$1 AND key=$2 AND user_id=$3

-- 客户端（非权威删除）
DELETE FROM storage WHERE collection=$1 AND key=$2 AND user_id=$3 AND write > 0
```

### 5.2 判定分支汇总

| 场景 | 客户端能否操作 | 依据 |
|---|---|---|
| 新建对象（`version="*"`） | 能，可自设 `read`/`write` | 插入分支无权限检查 |
| 新建对象（`version=""`，last-write-wins） | 能，可自设 `read`/`write` | `ON CONFLICT` 的 INSERT 分支无检查 |
| 更新已有对象，`write=1` | 能 | `AND storage.write = 1` 通过 |
| 更新已有对象，`write=0` | **不能** | UPDATE 影响 0 行 |
| 删除已有对象，`write=0` | **不能** | `AND write > 0` 不满足 |
| 任何 `user_id` 不是自己的对象 | **不能** | `OwnerID` 被强制覆盖 |

### 5.3 两种拒绝要用错误码区分

| 情况 | gRPC 码 | HTTP | 消息 |
|---|---|---|---|
| 权限不足 | `InvalidArgument` | 400 | `Storage write rejected.` |
| 版本冲突 | `FailedPrecondition` | 412 | `Storage write rejected.` |

```go
// server/core_storage.go:592-597
if errors.Is(writeErr, runtime.ErrStorageRejectedVersion) {
    return StatusError(codes.FailedPrecondition, "Storage write rejected.", writeErr)
}
if errors.Is(writeErr, runtime.ErrStorageRejectedPermission) {
    return StatusError(codes.InvalidArgument, "Storage write rejected.", writeErr)
}
```

**消息完全一样，只能靠错误码区分。** 客户端如果统一按「重试」处理，遇到权限问题会无限重试。

## 6. collection 与 key 的归属

### 6.1 协议上谁都能传，工程上归属不同

| | collection | key |
|---|---|---|
| 协议层 | 客户端传 | 客户端传 |
| 服务端默认校验 | 非空 | 非空 |
| 决定什么 | **schema**：权限、索引、查询入口 | 实例：具体哪个对象 |
| 变更成本 | 极高（迁移全部数据） | 低 |
| 应该谁定 | **服务端** | 客户端可参与，但要白名单 |

**collection 是 schema 级决策，不该由客户端决定。** 客户端只是把它「带上来」，不是「发明它」。

### 6.2 为什么 collection 必须服务端定

**① 索引绑在 collection 上。**

```go
initializer.registerStorageIndex(
  "progress_index",     // 索引名
  "player_progress",    // ← 绑定的 collection
  "", "max_stage", ["max_stage"], 10000, false
)
```

索引配置在启动时注册，绑定确定的 collection 名。如果客户端造了 `player_progress_v2`，它永远不会被索引——跨用户查询查不到，而且不报错，只静默漏数据。

**② 权限只能按 collection 预设。**

`read`/`write` 是行级字段，但工程上你是按 collection 统一规划的（「`player_settings` 允许客户端写，`player_progress` 不允许」）。如果 collection 名由客户端随便起，这套规划落不了地——新 collection 一律吃默认 `write=1`，全开放。

**③ collection 是查询入口。**

```proto
rpc ListStorageObjects (...) {
  option (google.api.http) = { get: "/v2/storage/{collection}" };
}
```

列表接口按 collection 查。collection 失控，等于查询入口失控。

**④ 运维看不到全貌。**

`SELECT collection, count(*) FROM storage GROUP BY collection` 是唯一能看清「库里有什么」的手段。如果 collection 名带日期、带版本、带随机后缀，这个清单会变成几百行垃圾。

### 6.3 客户端乱造 collection 的后果

| 后果 | 表现 |
|---|---|
| 索引失效 | 新 collection 没配索引，跨用户查不到 |
| 权限失效 | 新 collection 吃默认 `write=1`，玩家可写 |
| 存储放大 | 每换一个 collection 就是一个新「表」，无法统一清理 |
| 数据孤岛 | 同一业务散在 `save`、`save_v2`、`save_old` 里，迁移时没人知道该动哪个 |

四条都不会报错，只会在半年后让数据不可维护。

### 6.4 三档约束强度

「约定」是纸面的，「拒绝」是代码里的。没有 hook 或 RPC，约定对客户端没有任何约束力。

| 档 | 做法 | 客户端传错时 | 成本 |
|---|---|---|---|
| **一** | 只写文档约定 | 写入成功，产生垃圾数据 | 0 |
| **二** | before hook 白名单 | `400 InvalidArgument` | 一个函数 |
| **三** | 走 RPC，collection 不暴露 | 客户端根本没这个概念 | 多一层接口 |

第二档是最低成本的硬约束：

```go
var allowed = map[string]map[string]bool{
    "player_settings": {"settings": true},
    "player_tutorial": {"flags": true},
}

initializer.RegisterBeforeWriteStorageObjects(func(
    ctx context.Context, logger runtime.Logger, db *sql.DB, nk runtime.NakamaModule,
    in *api.WriteStorageObjectsRequest,
) (*api.WriteStorageObjectsRequest, error) {
    for _, obj := range in.Objects {
        keys, ok := allowed[obj.Collection]
        if !ok {
            return nil, runtime.NewError("collection not allowed: "+obj.Collection, 3)
        }
        if !keys[obj.Key] {
            return nil, runtime.NewError("key not allowed: "+obj.Key, 3)
        }
        if len(obj.Collection) > 128 || len(obj.Key) > 128 {
            return nil, runtime.NewError("collection or key too long", 3)
        }
    }
    return in, nil
})
```

签名来自 `runtime.go:740`，可以返回修改后的请求，也能直接拒绝。它跑在写入前，除了拦非法名，还能改权限、替换 value。

### 6.5 更彻底的做法：把命名收进 SDK

如果客户端是自研的，最省心的做法是把 collection 和 key 收进 SDK 常量，不暴露给业务代码：

```csharp
// 不要这样：业务代码里裸写字符串
await client.WriteStorageObjectsAsync(session, new[] {
    new StorageObjectWrite("player_settings", "settings", json)
});

// 这样：命名封在 SDK 内部
await NakamaSync.SaveSettingsAsync(session, settings);
```

好处是**业务开发者没有机会传错**，约定从「要求遵守」变成「想违反都做不到」。代价是每加一类数据要改一次 SDK，但对固定几类业务数据的游戏来说，这个成本很低。

### 6.6 collection 零预置

Nakama 本身**不定义任何 collection**，全部由使用者决定。验证：

```bash
grep -rn 'Collection:\s*"' server/ | grep -v _test     # 0 条
grep -in "insert into storage" migrate/sql/*.sql        # 0 条
grep -i "collection" data/*.yml                         # 0 条
```

源码无硬编码、建库脚本无初始数据、配置文件无相关项。

这是刻意的取舍：Nakama 把 `storage` 定位成**通用文档存储**，而不是业务框架。有预置的是 Hiro——它作为元游戏框架，已经替你把业务模型定好了。

所以问题从「Nakama 定义了哪几类」变成「你该怎么分几类」。分类的轴不是数据内容，而是**「谁写 + 更新频率 + 要不要跨用户查」**：

| 类别 | 典型 collection | 谁写 | read / write | 索引 | 判断依据 |
|---|---|---|---|---|---|
| 客户端偏好 | `player_settings` | 客户端 | 1 / 1 | 无 | 改了只影响自己观感 |
| 引导状态 | `player_tutorial` | 客户端 | 1 / 1 | 无 | 改了无收益 |
| 玩法进度 | `player_progress` | 服务端 | 1 / **0** | 按需 | 驱动解锁与发奖 |
| 收集/背包 | `player_inventory` | 服务端 | 1 / **0** | 可选 | 有交易价值 |
| 任务成就 | `player_quests` | 服务端 | 1 / **0** | 可选 | 驱动发奖 |
| 社交扩展 | `guild_extra` | 服务端 | **2** / **0** | 按需 | 需要跨用户查 |
| 全局配置 | `global_config` | 服务端 | **2** / **0** | 无 | 所有客户端可读 |

七类能覆盖绝大多数休闲游戏。做得再细也只是在这七类里拆，不会跳出这个结构。

这份清单应当在设计阶段定死，写进文档，代码里只使用清单里的名字。

## 7. 数据归属决策

技术上客户端能写自己所有 `write=1` 的对象。业务上要按「**改了这个数据，玩家能不能获得收益**」来划分。

### 7.1 按收益判断

| 数据类型 | 建议归属 | 理由 |
|---|---|---|
| UI 设置、音量、语言、按键布局 | 客户端 `write=1` | 改了只影响自己观感 |
| 新手引导完成标记 | 客户端 | 无收益 |
| 昵称、头像 URL | 客户端（需内容审核） | 内容型，但要防违规 |
| 本地进度缓存 | 客户端 | 仅作离线缓存，真值在服务端 |
| 关卡进度、解锁状态 | **服务端 `write=0`** | 驱动发奖与解锁 |
| 等级、经验、战力 | **服务端** | 驱动匹配、发奖 |
| 货币、道具、背包 | **服务端** | 有交易价值 |
| 任务 / 成就进度 | **服务端** | 驱动发奖 |
| 排行榜分数 | **服务端**（`authoritative=true`） | 竞技公平 |
| 对局统计、战绩 | **服务端** | 结算依据 |

**中间态：客户端上报 + 服务端校验。** 步数、滑行距离、通关时长这类数据，可以放 `write=1` 让客户端写，但服务端在发奖前做阈值校验与异常检测。准确性要求低于货币，高于设置。

**判断公式：玩家改了这个数据，能不能换到现实收益？** 能，就必须服务端写。

### 7.2 不该进 storage 的数据

`storage` 只承载**没有专表**的业务数据。Nakama 已有专表的不要重复造：

| 数据 | 应该在哪 | 为什么不用 storage |
|---|---|---|
| 货币余额 | `users.wallet` | 有非负校验、行锁、`\|\|` 合并语义 |
| 货币流水 | `wallet_ledger` | 只增不改的账本 |
| 排行榜分数 | `leaderboard_record` | 主键含分数，取 TOP N 零排序 |
| 聊天消息 | `message` | 按流聚簇，历史查询快 |
| 站内通知 | `notification` | 有持久化 / 已读语义 |
| 好友关系 | `user_edge` | 图结构 + 计数维护 |
| 群组成员 | `group_edge` | 同上 |
| 内购订单 | `purchase` | 以 `transaction_id` 主键保证幂等 |
| 高频事件 / 埋点 | 外部数仓 | 无聚合能力，写放大严重 |

**判断公式：Nakama 已经有专表 → 用专表；没有 → 才考虑 storage。**

## 8. 上线前自查

三条 SQL，跑完就知道权限与命名收没收紧。

### 8.1 找出所有对玩家开放写的 collection

```sql
SELECT collection, count(*) AS writable_objects
FROM storage
WHERE write = 1
GROUP BY collection
ORDER BY 2 DESC;
```

清单里的每个 collection 都应该能说出「为什么允许玩家写」。说不出来的，改成 `write=0`。

### 8.2 找出所有公开可读的对象

```sql
SELECT collection, count(*) AS public_objects
FROM storage
WHERE read = 2
GROUP BY collection
ORDER BY 2 DESC;
```

`read=2` 意味着**任何已登录玩家都能读**。手机号、设备指纹、内购记录放进去等于公开泄露。

### 8.3 找出没有预置在清单里的 collection

```sql
SELECT collection, count(DISTINCT key) AS key_variants, count(*) AS objects
FROM storage
GROUP BY collection
ORDER BY 2 DESC;
```

对照 §6.6 的分类清单。出现清单外的 collection，或某个 collection 的 `key_variants` 异常大（说明客户端在乱造 key），都要立即排查。

## 9. 实操约束

| 约束 | 值 | 说明 |
|---|---|---|
| `value` 类型 | **必须是 JSON 对象** | `api.go:62` 定义 `byteBracket = '{'`，数组、字符串、数字一律被拒 |
| `collection` / `key` 长度 | 128 字符 | `VARCHAR(128)`，超长返回 500 而非 400 |
| `value` 大小 | 无软限制 | 只有 PostgreSQL 单行 1GB 上限，但别真写大对象 |
| 批量写入 | 单事务 | 任一失败全批拒绝（`ExecuteInTxPgx`） |
| 索引更新 | 同步，在 DB 写之后 | `core_storage.go:919`，**不在同一事务内** |

`value` 必须是对象这条经常绊人：存数组（比如已解锁关卡列表）要包一层 `{"items": [...]}`。

`collection` / `key` 超长返回 500 而不是 400，因为长度约束在数据库层。客户端会误以为服务端故障而重试，实际是参数错误——用 §6.4 的 hook 提前拦住。

索引更新在事务外这一点要注意：事务提交成功但索引写入失败，会出现「数据在库里、索引里没有」的情况，需要靠 `Load` 重建。

## 10. 三个常见误用

### 10.1 改 value 不带权限，权限被重置

写入会**整体覆盖对象，包括权限**。如果只改 `value` 而没传 `permissionWrite`：

- 走 Go runtime → 权限被重置为 `0`
- 走客户端 SDK → 权限被重置为 `1`

结果是「改了一次存档，权限莫名其妙变了」。

**规范：所有更新操作先读、带着原权限写回，或者每次都显式传一遍。**

### 10.2 用 storage 做跨用户聚合查询

storage 是键值模型，没有聚合能力。要查「30 级以上的玩家有多少」，只能靠存储索引（`RegisterStorageIndex`），而索引有 `maxEntries` 上限、最终一致、且不是完整视图（见 `03-design-patterns.md` P8）。

需要真正的分析查询时，应把数据同步到数仓，不要在 storage 上硬查。

### 10.3 把 storage 当队列或事件表用

每个对象都要走一次 SQL 往返、参与版本冲突检测、触发索引更新。用它存高频追加的事件，写放大和冲突率都会失控。

事件应该走独立链路（见 `04-liveops-model.md` §1）。

## 11. 对自建的结论

### 直接照搬

| 设计 | 理由 |
|---|---|
| 三元主键 `(collection, key, owner)` | 一张表免 DDL 承载所有业务域 |
| `version = md5(value)` | 内容指纹，写入前无需读取 |
| 行级 `read` / `write` 两字段 | 权限声明在数据上，不散在代码里 |
| `writeCheck` 只作用于 UPDATE | 新建对象不做权限检查，符合逻辑 |
| 客户端删除带 `AND write > 0` | 与写入权限保持一致 |
| `OwnerID` 强制取 token | 归属不可伪造，从协议层堵死 |
| 用 `user_id = 全零` 表达全局对象 | 一套模型同时支持「每人一份」和「全局一份」 |

### 必须改

| 问题 | 改法 |
|---|---|
| 默认 `write=1` | 建表时把默认值改成 `0`，或强制所有写入显式传权限 |
| 权限整体覆盖 | 提供「只更新 value」的专用接口，内部保留原权限 |
| `collection` / `key` 无白名单 | 内置命名注册表，未注册的 collection 直接拒写 |
| 索引在事务外更新 | 要么进事务（牺牲性能），要么加对账任务修复 |
| 无内容大小上限 | 加 `CHECK (length(value::text) < N)`，防止单行过大 |
| 超长名返回 500 | 在 API 层先校验长度，返回 400 |
| 无 TTL / 归档 | 按 collection 设保留期，定期清理 |

### 一条原则

**这张表的设计价值不在「能存任意 JSON」，而在「权限和命名都跟着数据走」。**

结构很容易抄，约束需要纪律：

- 每个新建的 collection 都要在 code review 时被问一句「为什么是 `write=1`」
- 每次新增数据类别都要先决定「进哪个 collection、谁写、要不要索引」
- 每季度跑一遍 §8 的三条 SQL，看有没有长出清单外的名字
