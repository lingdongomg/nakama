# 03 · 账号与登录体系

评估 Nakama 的认证与会话设计，作为自建数据中心「账号层」的参照。基于上游源码 `7c94b8f`。

## 0. 结论

1. Nakama 提供的是**完整闭环**：认证 → 会话签发 → 绑定/解绑 → 封禁 → 会话撤销，不是半成品。
2. 内置 **9 个专用端点 + 1 个通用通道**，覆盖设备、邮箱、Apple、Google、Facebook、GameCenter、Steam 等欧美渠道。
3. **没有微信 / 抖音 / TikTok 的内建登录**，全仓检索 `wechat|weixin|douyin|tiktok|bytedance|toutiao` 零命中。
   2026 年新增的**通用 Provider 通道**即为此类平台设计的正式扩展点。
4. 同样**没有**手机号短信登录、实名认证。国内合规所需的手机号绑定、实名校验需要自建。

## 1. 内置登录方式

| 端点 | 校验方式 | 客户端传什么 |
|---|---|---|
| `/v2/account/authenticate/device` | **无校验** | 设备指纹（10–128 字节） |
| `/v2/account/authenticate/email` | bcrypt 比对，cost ≥ 13 | 邮箱 + 密码 |
| `/v2/account/authenticate/custom` | **无校验** | 自定义 ID（6–128 字节） |
| `/v2/account/authenticate/apple` | 服务端验 identity token | Apple token |
| `/v2/account/authenticate/google` | 服务端验 idToken | Google idToken |
| `/v2/account/authenticate/facebook` | 服务端调 Graph API 验 access token | access token |
| `/v2/account/authenticate/steam` | publisher key + ticket 验签 | Steam ticket |
| `/v2/account/authenticate/gamecenter` | 公钥验签 | playerID + salt + signature |
| `/v2/account/authenticate/facebookinstantgame` | signed player info 验签 | signedPlayerInfo |
| `/v2/account/authenticate` | **由自注册 provider 决定** | provider 名 + 任意 JSON payload |

路由定义：`apigrpc/apigrpc.proto:127-271`。

**关键认知：`device` 与 `custom` 属于「客户端自证」，不做任何密码学校验。**
谁持有该 ID 字符串，谁就是该账号。二者只适合游客态，不能作为真实身份。把 `custom` 当通用登录使用，等同于把账号判定权交给客户端。

## 2. 数据模型

### 2.1 主表

`users` 以 UUID 为主键，一个玩家一行。身份相关信息分散在两类载体中。

### 2.2 两套身份存储并存

**专用列**（历史设计），每列自带 UNIQUE 约束：
`email` + `password`、`apple_id`、`google_id`、`facebook_id`、`facebook_instant_game_id`、`gamecenter_id`、`steam_id`、`custom_id`。

**通用表 `user_device`**，`provider` 列为 2026-08 新增（`migrate/sql/20260819141500-add-user-provider.sql`）：

```sql
CREATE TABLE user_device (
    PRIMARY KEY (id),
    FOREIGN KEY (user_id) REFERENCES users (id) ON DELETE CASCADE,
    id       VARCHAR(128) NOT NULL,
    user_id  UUID         NOT NULL,
    provider VARCHAR(128) NOT NULL DEFAULT '',
    UNIQUE (user_id, id)
);
```

新式 provider 一律走 `user_device(provider, id)`，专用列仅服务内置渠道。两套并存是历史包袱，自建系统时应统一为一张身份表。

### 2.3 两个必须提前决策的陷阱

**陷阱一：`id` 是主键，全局唯一，跨 provider 共享同一命名空间。**
「设备 ID」与「微信 openid」落在同一个唯一约束里。若某 openid 恰好等于某个设备 ID，会互相顶掉。
做法：给 id 加平台前缀（`wx_mp1:oABC...`），不要裸存 openid。

**陷阱二：若 provider 返回的 ID 本身是 UUID 格式，Nakama 直接拿它当 `user_id`**（`core_authenticate.go:211-215`）。
抖音 / TikTok 的 openid 不是 UUID，不受影响；但接入任何返回 UUID 的平台时，玩家的 `user_id` 将等于平台 ID，后续迁移极难掰开。

### 2.4 认证的核心三步

以 `AuthenticateCustom` 为例（`core_authenticate.go:259`），通用 Provider 路径（`:124`）结构相同：

1. **查**：按身份查 `users` / `user_device`，命中则校验 `disable_time`（封禁判断）。
2. **建**：未命中且 `create=true` 时新建用户；通用路径下「建用户 + 建身份」在**同一事务**内完成（`:217-235`），避免留下无身份的孤儿账号。
3. **签发**：生成 token / refresh token，写入 SessionCache。

唯一键冲突的处理值得注意：`users_username_key` → 「用户名已被占用」；`users_pkey` → 「user_id 已被占用」；其余唯一冲突 → 视为并发写入，返回内部错误（`:237-248`）。

## 3. 会话设计

| 机制 | 实现 | 位置 |
|---|---|---|
| 令牌格式 | JWT，签名算法 **HS256** | `jwt.go:22` |
| 双令牌 | access 与 refresh **使用不同签名密钥** | `config.go:809-812` |
| 默认有效期 | access 60 秒 / refresh 3600 秒 | `config.go:807-813` |
| 会话白名单 | SessionCache 记录 tokenID，支持按 token 撤销 | `session_cache.go:156-201` |
| 单点登录 | `single_session` 登录时踢掉旧 token | `api_authenticate.go:190-192` |
| 封禁 | `Ban / Unban`，登录时检查 `disable_time` | `session_cache.go:201` |
| 续期 | `POST /v2/account/session/refresh` | `apigrpc.proto:103` |
| 登出 | `POST /v2/session/logout` | `apigrpc.proto:119` |

两个值得借鉴的设计细节：

- **access 默认仅 60 秒。** 不是笔误，客户端 SDK 会自动刷新，代价是令牌泄露窗口极小。
- **refresh token 用独立密钥签名**，天然无法被当作 access token 使用，省掉类型字段校验。并且 refresh 复用同一 `tokenID`（`api_authenticate.go:198`），一个会话一个 ID，撤销时一次清干净。

补充能力：邮箱密码支持 MFA（`mfa.go`），密码哈希使用 bcrypt，成本下限强制为 13（`core_authenticate.go:41`）。

## 4. 自定义 Provider 接入

### 4.1 注册（Go 示例）

```go
type wechatProvider struct {
    appID, appSecret string
}

func (p *wechatProvider) Authenticate(ctx context.Context, logger runtime.Logger,
    db *sql.DB, nk runtime.NakamaModule, payload map[string]any,
) (runtime.AuthenticateProviderResult, error) {

    code, _ := payload["code"].(string)
    if code == "" {
        return nil, runtime.NewError("code required", 3)
    }
    // AppSecret 仅服务端使用，永不下发客户端
    openid, err := code2Session(p.appID, p.appSecret, code)
    if err != nil {
        return nil, runtime.NewError("wechat auth failed", 16)
    }
    return &runtime.DefaultAuthenticateProviderResult{ProviderUserID: openid}, nil
}

func (p *wechatProvider) GetFriends(_ context.Context, _ runtime.Logger, _ *sql.DB,
    _ runtime.NakamaModule, _ map[string]any, _ runtime.AuthenticateProviderResult,
) ([]string, bool, error) {
    return nil, false, nil
}

// InitModule
initializer.RegisterAuthenticateProvider("wechat", &wechatProvider{...})
```

Lua 与 JS 同样支持（`nk.register_authenticate_provider`），不强制使用 Go。
接口定义：`runtime.go:1111`（`AuthenticateProvider`）、`:1129`（`DefaultAuthenticateProviderResult`）。

### 4.2 客户端调用

```
POST /v2/account/authenticate
{"account": {"provider": "wechat", "payload": {"code": "..."}}}
```

约束（`core_authenticate.go:145-168`）：
provider 名统一转小写；`providerUserID` 上限 128 字节且禁止空格与控制字符；provider 返回的 username 若含非法字符会被拒绝。

### 4.3 绑定已有账号

游客转正必须用 **link**，不能用 authenticate 新建账号：

```
POST /v2/account/link
{"provider": "wechat", "payload": {"code": "..."}}
```

`core_link.go:113` 的 `Link` 同样调用 provider 的 `Authenticate` 做校验，**绑定与登录安全等级一致**。
写入语句为 `ON CONFLICT (id) DO UPDATE ... WHERE user_device.user_id = EXCLUDED.user_id`：重复绑定同一身份幂等，抢绑他人身份则报 `AlreadyExists`。

## 5. 微信 / 抖音 / TikTok 落地要点

### 5.1 Provider 命名规范（一次定死）

| 场景 | provider 名 | id 格式 |
|---|---|---|
| 微信小游戏 | `wechat_mp1` | `wx<appid>:<openid>` |
| 微信公众号 / H5 | `wechat_h5` | `wx<appid>:<unionid>` |
| 抖音小游戏 | `douyin_prod` | `dy<appid>:<openid>` |
| TikTok（海外） | `tiktok_intl` | `tt<appid>:<openid>` |

同一 appid 对应一个 provider 名；id 带 appid 前缀，规避 `user_device.id` 全局唯一带来的跨平台碰撞。

### 5.2 五个必须提前处理的问题

1. **code 一次性且 5 分钟过期。** Nakama 的 refresh token 只能续 Nakama 自身令牌，**无法续平台会话**。客户端必须实现「refresh 失败 → 重新 `wx.login()` → 重新 authenticate」的降级链路。
2. **unionid 才是跨端身份。** 需在开放平台绑定后将多端打通；若用 openid，同一玩家在不同端会成为多个账号。
3. **多 appid 必须隔离。** 同一套代码部署多个小程序时 openid 各不相同，见 5.1 命名规范。
4. **抖音与 TikTok 是两套体系。** 前者走 `developer.toutiao.com` 的 code2session，后者走海外 Login Kit / 小游戏体系，接口与字段均不同，不可共用一个 provider 名。
5. **AppSecret 绝不下发。** 「客户端拿 code 自己换 openid 再传给服务端」等于把身份判定权交给客户端，是本类接入最常见的致命错误。

### 5.3 客户端登录状态机

```
未登录 ──device 登录──▶ 游客态 ──link(wechat code)──▶ 正式账号
   ▲                        │                              │
   └──── refresh 失败 ──────┴────── 重新 code 登录 ◀────────┘
```

要点：游客态与正式账号使用**同一个 user_id**（通过 link），不做账号合并。

## 6. 解绑与安全边界

`Unlink` 系列均有「不得解绑最后一个登录方式」的保护，返回 `Cannot unlink last account identifier`（`core_unlink.go:97-125`）。这一点自建系统必须照做，否则用户解绑后会永久失去账号。

## 7. 对本项目的决策建议

1. **一期只开两条通道**：`device`（游客）+ `wechat_mp1`（正式）。通道越多，账号合并逻辑越复杂，而合并出错是不可逆的数据事故。
2. **禁用 `custom` 登录**：`custom_id` 缺乏校验，若未来用作任何形式的身份凭据，等同于永久后门。
3. **身份表统一**：自建系统参照 `user_device` 设计，但只保留一张表（`provider` + `external_id` + `user_id`），不重复 Nakama 的「专用列 + 通用表」双轨。
4. **手机号与实名**：Nakama 不提供，需自建，且应在账号层预留 `phone` 身份类型的扩展位。
