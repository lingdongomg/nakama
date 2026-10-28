# 04 · 运营数据模型与配置中心

`02-gap-matrix.md` 已判定能力 11（关卡/行为分析）与能力 12（配置中心）必须自建，但只到要点级。本文补两件事：**这两块的数据模型长什么样，以及哪些东西不要做**。

参考对象是 Heroic Labs 的 Satori（LiveOps 平台）。它把整个产品压成一条链，本文沿用这个模型：

```
事件  →  玩家属性  →  分群  →  个性化体验
```

链的价值在于：所有运营动作建立在同一份玩家状态上，不需要每个功能各自定义一次「谁是目标用户」。

## 0. 四个决策先定

在写表之前必须定死，事后改代价极高：

1. **属性在服务端派生，不在客户端计算。** 客户端只上报原始事件。否则每加一个指标都要发版。
2. **分群是「算出来的」，不是「配进去的」。** 运营只写规则，成员由系统推导。手工名单只作为例外覆盖。
3. **配置版本必须写进业务记录。** 战绩、订单、发奖记录都要带当时生效的配置版本，否则历史数据无法解释。
4. **灰度分桶必须稳定。** 同一玩家在同一配置下永远命中同一个桶，不能用「按百分比随机」。

## 1. 事件模型

统一宽表，所有分析都从这一张表出发，避免每来一个需求加一张表。

```sql
CREATE TABLE events (
  event_time  timestamptz NOT NULL,
  event_id    bigserial   NOT NULL,
  app_id      text        NOT NULL,
  user_id     uuid        NOT NULL,
  session_id  uuid,
  event_name  text        NOT NULL,
  props       jsonb       NOT NULL DEFAULT '{}',
  PRIMARY KEY (event_time, event_id)
) PARTITION BY RANGE (event_time);
```

三点说明：

- **主键必须包含分区键。** PostgreSQL 的分区表要求主键涵盖分区列，写成 `PRIMARY KEY (event_id)` 会直接建表失败。这是建分区表最常见的报错。
- **`app_id` 从一开始就有。** 多款游戏共用一套链路时，事后加这个维度等于重刷全部历史数据。
- **`session_id` 不为空。** 没有它就算不出会话时长、单次会话内的行为序列。

### 1.1 埋点字典先行

`02` 已指出这是同类系统 80% 的失败原因，这里给出落地形态：

```sql
CREATE TABLE event_dictionary (
  event_name  text NOT NULL,
  prop_key    text NOT NULL,
  prop_type   text NOT NULL,          -- int / float / string / bool / enum
  enum_values text[],                 -- prop_type = enum 时必填
  unit        text,
  owner       text NOT NULL,          -- 谁负责这个字段
  status      text NOT NULL DEFAULT 'active',   -- active / deprecated
  created_at  timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (event_name, prop_key)
);
```

**写入侧要校验。** 未注册的 `event_name` 或 `prop_key` 直接拒收，不要「先收下来以后再治理」。一旦允许脏数据进来，后续所有报表都要先做清洗。

宁可丢几个事件，也不要污染全表。

### 1.2 双链路分离

| 链路 | 时效 | 数据要求 | 用途 |
|---|---|---|---|
| 实时 | 秒级 | 可丢、可近似 | 大盘、告警、活动实时看板 |
| 离线 | T+1 | 全量、准确 | 漏斗、留存、归因、结算 |

混在一条链路上，分析查询会拖慢线上写入。实时链路建议走消息队列 + 列式存储，离线链路走每日全量落盘。

## 2. 玩家属性

事件进来之后派生属性。属性是分群的唯一输入。

```sql
CREATE TABLE player_properties (
  user_id    uuid        NOT NULL,
  key        text        NOT NULL,
  value_num  numeric,                -- 数值型属性
  value_str  text,                   -- 字符串型属性
  updated_at timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (user_id, key)
);
CREATE INDEX ON player_properties (key, value_num);
```

**为什么拆两列而不是一个 jsonb.** 分群规则要做范围比较（`max_stage > 12`）和枚举匹配，拆列才能走索引。放 jsonb 里只能全表扫。

### 2.1 派生规则

| 类型 | 例子 | 计算方式 |
|---|---|---|
| 累计型 | 总付费额、总看广告次数 | 事件累加 |
| 极值型 | 最高关卡、最高段位 | 取 max |
| 状态型 | 最后活跃时间、连续登录天数 | 覆盖 / 日期差 |
| 派生型 | 是否付费用户、付费档位 | 由前几类推出 |

**派生规则本身要版本化。** 改了「活跃用户」的定义（从 7 天改成 14 天），历史数据口径就变了。规则表带 `version` 与 `effective_from`，报表要标注用的是哪一版。

## 3. 分群

```sql
CREATE TABLE audiences (
  id          text PRIMARY KEY,
  app_id      text NOT NULL,
  name        text NOT NULL,
  rule        jsonb NOT NULL,        -- 规则表达式，见下
  refresh     text NOT NULL DEFAULT 'daily',   -- daily / hourly / realtime
  updated_at  timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE audience_members (
  audience_id text NOT NULL,
  user_id     uuid NOT NULL,
  computed_at timestamptz NOT NULL,
  PRIMARY KEY (audience_id, user_id)
);
```

规则表达式用 JSON，不要用字符串拼接 SQL（注入风险 + 无法校验）：

```json
{
  "all": [
    { "prop": "max_stage",  "op": "eq",  "value": 12 },
    { "prop": "days_since_last_session", "op": "gte", "value": 3 },
    { "prop": "total_pay",  "op": "lt",  "value": 1 }
  ],
  "none": [
    { "prop": "is_banned", "op": "eq", "value": true }
  ]
}
```

### 3.1 重算策略

| 策略 | 频率 | 适用 | 成本 |
|---|---|---|---|
| 全量重算 | 每日 / 每小时 | 绝大多数运营场景 | 随用户数线性 |
| 增量更新 | 事件触发，只算本人 | 「刚发生某事」的即时响应 | 低 |
| 请求时评估 | 每次拉配置时实时算 | 强实时个性化 | 每次请求都有计算 |

**建议：先只做全量重算。** 日更对 IAA 的运营动作（发 push、调难度、发补偿）完全够用，实现成本最低。

Satori 是「默认 10 分钟全量重算 + 事件触发立即更新本人 + 请求时评估资格」三档并存，那是为了支撑实时个性化。自建早期不需要，加了只会让链路变复杂、结果难复现。

**分群成员要留快照。** 发奖、发 push 时必须记录「当时这批人是按哪次计算的结果」，否则事后无法解释「为什么这个人也在名单里」。

## 4. 配置中心

`02` 列了五项必须支持的能力，下面给出对应实现。

### 4.1 版本化

```sql
CREATE TABLE configs (
  name        text        NOT NULL,
  version     integer     NOT NULL,
  payload     jsonb       NOT NULL,
  schema_ref  text,
  status      text        NOT NULL DEFAULT 'draft',   -- draft / published / archived
  created_by  text        NOT NULL,
  created_at  timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (name, version)
);

-- 指针表：当前生效版本
CREATE TABLE config_releases (
  env        text        NOT NULL,
  name       text        NOT NULL,
  version    integer     NOT NULL,
  updated_at timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (env, name)
);
```

**发布 = 新增版本 + 移动指针，不是覆盖。** 回滚就是把指针指回上一版，秒级生效且不留脏数据。

主键写成 `(name, version)` 而不是 `name`，这一条决定了能不能回滚。

### 4.2 灰度

```sql
CREATE TABLE config_rollouts (
  env      text     NOT NULL,
  name     text     NOT NULL,
  version  integer  NOT NULL,
  percent  smallint NOT NULL,        -- 0-100
  salt     text     NOT NULL,        -- 分桶盐
  PRIMARY KEY (env, name)
);
```

分桶函数：

```
bucket  = crc32(user_id || ':' || name || ':' || salt) % 100
enabled = bucket < percent
```

**为什么不能用随机数.** `rand() < 0.1` 会让同一玩家这次命中、下次不命中，表现为「功能一会儿有一会儿没有」。

**`salt` 的作用.** 扩容灰度时（10% → 50%），用同一 salt 能保证原来那 10% 仍然命中，是增量扩大而不是重新洗牌。反过来，如果灰度出问题要重新抽一批人，改 salt 即可。

这个字段看似多余，实际决定了灰度能不能安全地分阶段推进。

### 4.3 热更与缓存

| 环节 | 策略 |
|---|---|
| 客户端启动 | 拉一次全量，带 `If-None-Match`，命中 304 就用本地缓存 |
| 运行中 | 每 N 分钟或每次回前台时校验版本号 |
| 关键操作前 | 校验配置版本，不一致则先拉取（发奖、下单这类不可逆操作） |
| 服务端 | 内存缓存 + TTL，配置变更时主动失效 |

**不要每次请求都查库。** 配置是读放大最严重的表，一个玩家一局可能读几十次。必须有服务端缓存层。

### 4.4 发布前校验

三道闸，缺一不可：

1. **Schema 校验.** 字段类型、必填项、数值范围。
2. **引用完整性.** 配置里引用的道具 ID、关卡 ID、奖励 ID 必须在目标环境存在。引用了不存在的道具直接拒绝发布。
3. **影响面预估.** 发布前算出「这次变更影响多少玩家、哪些分群」，超过阈值要求人工二次确认。

第 2 条是最容易漏的。跨环境的引用校验尤其重要：开发环境有、生产环境没有的道具，是上线事故的常见来源。

### 4.5 审计

```sql
CREATE TABLE config_audits (
  id          bigserial PRIMARY KEY,
  env         text NOT NULL,
  name        text NOT NULL,
  version     integer NOT NULL,
  action      text NOT NULL,        -- create / publish / rollback
  actor       text NOT NULL,
  diff        jsonb NOT NULL,       -- 字段级差异
  affected    integer,              -- 预估影响人数
  created_at  timestamptz NOT NULL DEFAULT now()
);
```

`diff` 要存字段级的差异，不是整份配置。出问题时第一个问题永远是「改了什么」，存全量配置等于让人自己 diff。

### 4.6 与业务数据的耦合

**配置版本必须写进业务记录。** `02` 已提到这一点，具体落法：

| 记录 | 需要落什么 |
|---|---|
| 对战战绩 | 结算时用的配置版本号 |
| 订单 / 发货 | 商品配置版本、价格版本 |
| 发奖记录 | 奖励配置版本（或直接存快照，见 `03` P4） |
| 埋点事件 | 客户端配置版本（用于分析时还原上下文） |

漏了这一步，线上出现「同一玩家两次战斗用不同配置」的争议时，无法自证。

## 5. 实验

灰度解决「逐步放量」，实验解决「哪个方案更好」。两者共用分桶基础设施，但实验多一个硬要求：**分组必须粘性**。

```sql
CREATE TABLE experiments (
  id        text PRIMARY KEY,
  app_id    text NOT NULL,
  name      text NOT NULL,
  variants  jsonb NOT NULL,      -- [{"key":"control","weight":50},{"key":"b","weight":50}]
  start_at  timestamptz,
  end_at    timestamptz,
  status    text NOT NULL DEFAULT 'draft'
);

CREATE TABLE experiment_assignments (
  experiment_id text NOT NULL,
  user_id       uuid NOT NULL,
  variant       text NOT NULL,
  assigned_at   timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (experiment_id, user_id)     -- 粘性由主键保证
);
```

分配逻辑：先查表，没有则按 hash 分组后 `INSERT ... ON CONFLICT DO NOTHING`，再以库里的结果为准。

**三条纪律：**

1. **必须有对照组。** 只上 B 方案不做 A/B，无法区分「方案有效」和「这周本来就涨」。
2. **分组一旦确定永不改变。** 玩家中途换组，数据直接作废。
3. **看指标要看两个。** 只盯收入，会出现「收入涨了但活跃跌了」的假胜利。收入 + 活跃一起看。

## 6. 明确不做的部分

Satori 有四块能力，自建时建议放弃其中两块，理由是成本与收益不匹配。

| 能力 | 是否自建 | 理由 |
|---|---|---|
| 事件 + 属性 + 分群 | 做 | 是整条链的地基，无法外购替代 |
| 配置中心（版本/灰度/审计） | 做 | `02` 已定为自建，且与存档强耦合 |
| 实验（A/B） | 做，但简化 | 只需要分桶 + 粘性 + 指标对比，不需要可视化 |
| Journey 可视化编排 | **不做** | 本质是带状态的工作流引擎，开发量远超收益。早期用「配置表 + 人工运营」替代 |
| 实时个性化（请求时评估） | **不做** | 每次请求都要算分群，延迟与复杂度不值。日更分群足够 |

**Journey 的替代方案.** 需要「玩家 3 天没回来就发 push」这类逻辑时，用一张任务表 + 每日批处理扫描即可：

```sql
CREATE TABLE ops_tasks (
  id          bigserial PRIMARY KEY,
  audience_id text NOT NULL,
  action      text NOT NULL,        -- push / mail / grant
  delay_days  integer NOT NULL DEFAULT 0,
  scheduled_at timestamptz NOT NULL,
  status      text NOT NULL DEFAULT 'pending'
);
```

等真的需要多步、带分支、按玩家各自节奏推进的流程时，再考虑上工作流引擎。绝大多数 IAA 运营用不到那一步。

## 7. 与 Satori 的对照

| 维度 | Satori | 本文方案 | 差异原因 |
|---|---|---|---|
| 分群更新 | 10 分钟全量 + 事件触发 + 请求时评估 | 日更全量（可提到小时级） | 无实时个性化需求 |
| 配置下发 | Feature flag 带 schema，按分群下发不同变体 | 同一套（版本表 + 灰度表） | 核心模型直接沿用 |
| 实验 | 完整实验平台，含显著性检验 | 分桶 + 粘性 + 指标对比 | 早期样本量不足以谈显著性 |
| 编排 | Journey 画布 | 任务表 + 批处理 | 开发成本 |
| 分析 | 内置留存/漏斗/RoAS | 接入现成 BI 读数仓 | 自建分析前端性价比低 |
| 归因 | 接 Adjust，出 RoAS 报表 | 接 MMP 自有报表 | 买量平台已提供 |

**值得照搬的三个设计：**

1. **flag 是带 schema 的配置块，不是布尔开关。** 一个 flag 可以承载整套奖励配置或商店布局。布尔开关只能控制「开/关」，带 schema 的能控制「长什么样」。
2. **事件与属性分离。** 客户端只报原始事件，属性服务端派生。加指标不发版。
3. **粘性分组。** 实验和灰度共用分桶，且结果落库固定。

## 8. 与 02 的对应关系

| `02` 能力编号 | 本文对应章节 | 结论 |
|---|---|---|
| 11 关卡 / 行为分析 | §1 事件模型、§2 属性、§3 分群 | 自建，事件模型与埋点字典先行 |
| 12 配置中心 | §4 | 自建，五项能力全部落地 |
| 13 定时调度 | §3.1、§6 | 不做通用调度，用「分群批处理 + 时间戳结算（`03` P7）」替代 |
| 14 数据导出 | §1.2 双链路 | 离线链路直接落数仓，导出即查询 |
