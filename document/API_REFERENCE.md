# PopCat Proxy 接口参考

本文件列出 `pop-cat-proxy` 暴露的全部接口，以及品牌后端必须自行实现、供代理回源调用的契约接口。每个接口给出请求方法、路径、鉴权方式、请求字段、响应字段与错误码。功能背景与接入流程见同目录下的 [INTERNAL_INTEGRATION.md](./INTERNAL_INTEGRATION.md) 与 [TESTFLIGHT_INTEGRATION.md](./TESTFLIGHT_INTEGRATION.md)。

## 目录

- [接口分层](#接口分层)
- [鉴权与加密机制](#鉴权与加密机制)
- [一、公开客户端接口](#一公开客户端接口prefix-apiv1apple)
- [二、代理对品牌后端开放的 TestFlight 内部接口](#二代理对品牌后端开放的-testflight-内部接口prefix-apiv1internal)
- [三、品牌后端必须实现的契约接口](#三品牌后端必须实现的契约接口)
- [四、运维接口](#四运维接口)
- [公共数据结构](#公共数据结构)

---

## 接口分层

代理共有三层接口，调用方与鉴权方式各不相同。

| 层 | 路径前缀 | 调用方 | 鉴权 | 响应形态 |
| --- | --- | --- | --- | --- |
| 公开客户端接口 | `/api/v1/apple` | PopCat App（iOS / macOS） | Apple 请求签名 + 代理令牌 | 加密二进制（部分明文 JSON） |
| TestFlight 内部接口 | `/api/v1/internal` | 各品牌后端服务器 | `X-PopCat-Internal-Token` | 明文 JSON |
| 品牌后端契约接口 | `/api/v1/internal/apple` | 由代理回源调用 | `X-PopCat-Internal-Token`（代理下发）+ `Bearer upstream_token` | 明文 JSON |
| 运维接口 | 根路径 | 监控 | 无 | 明文 JSON |

前缀可由环境变量改写：公开接口前缀为 `PUBLIC_API_PREFIX`（默认 `/api/v1/apple`），内部接口前缀为 `INTERNAL_API_PREFIX`（默认 `/api/v1/internal`）。本文件按默认前缀书写。

---

## 鉴权与加密机制

### Apple 请求签名

公开客户端接口的每个请求都要携带以下请求头，由 `verifyAppleRequest` 校验：

| 请求头 | 含义 |
| --- | --- |
| `X-Client-Pubkey` | 客户端临时 P-256 公钥，未压缩格式 65 字节，base64url 编码（首字节 `0x04`） |
| `X-Timestamp` | 客户端毫秒时间戳，与服务器时间偏差需在 300 秒内 |
| `X-Nonce` | 一次性随机串 |
| `X-Signature` | 对签名串的 ECDSA（P-256 / SHA-256）签名，base64url 编码 |

签名串拼接规则（换行符分隔）：

```text
<timestamp>\n<nonce>\n<METHOD 大写>\n<去掉查询串的路径>\n<请求体 SHA-256 十六进制>
```

校验失败返回：缺少签名头 `400`；时间戳超窗 `401`；公钥非法 `400`；签名不匹配 `498`。

### 响应加密

凡用 `encryptedJson` / `encryptForApple` 返回的接口，响应体为 `application/octet-stream` 二进制，`cache-control: no-store`，结构为：

```text
[临时公钥 65 字节][密文][GCM 认证标签 16 字节]
```

派生方式：代理用一对临时 P-256 密钥与客户端公钥做 ECDH，得到共享密钥后用 X9.63 KDF（SHA-256，sharedInfo = 代理临时公钥）派生 32 字节，前 16 字节为 AES-128-GCM 密钥，后 16 字节为 IV。客户端用同一套规则解出明文 JSON。`/health` 与 TestFlight 内部接口不加密。

### 代理令牌（Proxy Token）

`/login` 与 `/subscription/login` 成功后返回一个由代理签发的 JWT（HS256，密钥 `POPCAT_PROXY_JWT_SECRET`），后续 `/user`、`/nodes`、`/routing-config` 用它鉴权。

请求头形式（`Bearer` 前缀可省略）：

```text
Authorization: Bearer <proxy token>
```

载荷字段：

| 字段 | 含义 |
| --- | --- |
| `iss` | 固定 `pop-cat.com` |
| `aud` | 固定 `popcat-client` |
| `iat` / `exp` | 签发与过期时间，默认有效期 30 天（`TOKEN_TTL_SECONDS`） |
| `brand_id` | 绑定的品牌标识 |
| `upstream_token` | 品牌后端返回的私有令牌，客户端不可见明文（在加密响应内） |
| `sub` | 用户主体标识 |
| `auth_type` | `password` 或 `subscription` |

令牌缺失或非法返回 `401`，过期返回 `401`（`Token expired`）。

### 品牌内部令牌（Internal Token）

TestFlight 内部接口与品牌契约接口都用 `X-PopCat-Internal-Token` 鉴权，取值为品牌注册表中该品牌的 `internalToken`。请求体可附带 `brand_id`，或用 `X-PopCat-Brand-Id` 请求头指定品牌；省略时代理用常量时间比较把令牌匹配到对应品牌。匹配失败返回 `401`。

### 通用错误体

所有 JSON 错误统一为：

```json
{
  "error": "错误信息",
  "details": { }
}
```

`details` 仅在存在附加信息时出现（如 TestFlight 冲突时的 Apple 状态）。

---

## 一、公开客户端接口（prefix `/api/v1/apple`）

以下接口均需 Apple 请求签名。客户端来源 IP 落在临时封锁网段 `39.144.` 时，整层返回 `403`：

```json
{
  "error": "BOOTSTRAP_DIRECT_BLOCKED",
  "message": "Unable to connect directly. Trying the built-in proxy."
}
```

### 1.1 登录

```http
POST /api/v1/apple/login
```

请求体：

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `email` | string | 是 | 登录邮箱；也接受 `login` 字段。自动小写并去空格 |
| `password` | string | 是 | 登录密码 |
| `device` | string | 否 | 设备描述，透传给品牌后端 |
| `brand_id` | string | 否 | 指定品牌；省略时代理自动探测 |

行为：未指定 `brand_id` 时，代理先对各品牌调 `accounts/may-exist` 过滤候选，再并发尝试登录。

成功响应（加密 JSON）：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `token` | string | 代理令牌 |
| `auth_type` | string | `password` |
| `brand` | object | 见 [publicBrand](#publicbrand-品牌公开信息) |
| `user` | object | 品牌后端返回的用户对象 |

多品牌命中同一账号密码时返回 `409`（明文 JSON，不加密），客户端需让用户选择后带 `brand_id` 重试：

```json
{
  "error": "BRAND_SELECTION_REQUIRED",
  "message": "Choose a provider. Retry /login with brand_id.",
  "brands": [ { "id": "...", "name": "...", "short_name": "...", "logo_url": null, "app_links": { } } ]
}
```

错误码：缺少邮箱或密码 `400`；品牌未知 `400`；无品牌匹配该邮箱 `422`（`No brand account matched this email`）；邮箱或密码错误 `422`（`Invalid email or password`）。

### 1.2 订阅地址登录

```http
POST /api/v1/apple/subscription/login
```

请求体：

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `url` | string | 是 | 订阅链接；也接受 `subscription_url`。仅允许 `http`/`https` |
| `device` | string | 否 | 设备描述 |

代理按订阅链接的主机名匹配品牌（品牌注册表 `subscriptionHosts`），命中后回源到品牌的 `subscription/login`。

成功响应（加密 JSON）：`auth_type` 为 `subscription`，且**不返回 `brand` 字段**，其余同登录。

错误码：链接缺失或非法 `400`；主机名不在任何品牌的订阅域名内 `422`（`Unsupported subscription host`）。

### 1.3 获取用户信息

```http
GET /api/v1/apple/user
Authorization: Bearer <proxy token>
```

成功响应（加密 JSON）：品牌后端返回的用户对象，附加 `brand` 字段。其中：

- 订阅登录（`auth_type=subscription`）会把 `name`、`email`、`app_links` 置空，且不附加 `brand`。
- 内部账号（`popcat@pop-cat.com`）会被改写为统一的内部品牌与内部套餐信息。

用户对象建议结构见 [user 用户对象](#user-用户对象)。该接口同时触发一次设备记录（回源 `devices/seen`）。

### 1.4 获取节点

```http
GET /api/v1/apple/nodes
Authorization: Bearer <proxy token>
```

成功响应（加密）：节点配置。品牌后端返回 JSON 时为加密 JSON；返回二进制时代理直接加密透传，响应类型 `application/octet-stream`。内部账号返回内置节点集合。结构见 [节点配置](#节点配置)。

### 1.5 路由配置

```http
GET /api/v1/apple/routing-config
Authorization: Bearer <proxy token>
X-Routing-Config-Revision: <已缓存版本号>
```

代理按令牌绑定的品牌返回路由模板。若请求头携带的版本号与当前版本一致，返回 `204`（`cache-control: no-store`，无响应体）；否则返回加密 JSON。结构见 [路由配置](#路由配置)。

### 1.6 版本策略

```http
GET /api/v1/apple/version-policy
```

无需代理令牌，仅需 Apple 签名。成功响应（加密 JSON）：

```json
{
  "generated_at": "2026-05-30T00:00:00.000Z",
  "policy": {
    "ios":   { "latestVersion": "0.0.0", "minSupportedVersion": "0.0.0", "minSupportedBuild": null, "updateUrl": null, "releaseNotes": null },
    "macos": { "latestVersion": "0.0.0", "minSupportedVersion": "0.0.0", "minSupportedBuild": null, "updateUrl": null, "releaseNotes": null }
  }
}
```

字段语义与强制更新规则见 [INTERNAL_INTEGRATION.md](./INTERNAL_INTEGRATION.md#版本策略) 与仓库根目录 `VERSION_CONTROL.md`。

### 1.7 引导代理配置（Bootstrap Proxy）

```http
GET /api/v1/apple/bootstrap-proxy
X-Bootstrap-Proxy-Revision: <已缓存版本号>
```

供客户端在直连受阻时获取内置代理出口。版本号一致返回 `204`，否则返回加密 JSON：

```json
{
  "revision": "<sha256>",
  "api_base_urls": ["https://..."],
  "proxies": [
    {
      "id": "bp1",
      "type": "shadowtls",
      "server": "1.2.3.4",
      "server_port": 443,
      "password": "...",
      "sni": "example.com",
      "allow_insecure": false,
      "priority": 100
    }
  ],
  "updated_at": "2026-05-30T00:00:00.000Z"
}
```

`proxies` 按 `priority` 升序、再按 `id` 字典序排列；缺少 `id` / `type` / `server` 或端口非正的条目会被剔除。

---

## 二、代理对品牌后端开放的 TestFlight 内部接口（prefix `/api/v1/internal`）

调用方为品牌后端，鉴权用 `X-PopCat-Internal-Token`。代理在内部调用 App Store Connect 完成操作。完整接入流程见 [TESTFLIGHT_INTEGRATION.md](./TESTFLIGHT_INTEGRATION.md)。

> 名额规则：`DELETE /testflight/testers` 只是把人移出 Beta 组（停用，名额保留）；`DELETE /testflight/testers/release` 才会从 App Store Connect 删除测试员、释放苹果名额。

### 2.1 首次邀请

```http
POST /api/v1/internal/testflight/invitations
Content-Type: application/json
X-PopCat-Internal-Token: <brand internalToken>
```

请求体：`{ "email": "user@example.com" }`（邮箱会被规范化为小写去空格，并校验格式）。

成功 `201`：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `email` | string | 规范化后的邮箱 |
| `beta_tester_id` | string | App Store Connect 测试员资源 ID |
| `beta_tester_created` | boolean | 固定 `true` |
| `invitation_id` | string \| null | 当前实现固定 `null`（通过加入 Beta 组下发邀请） |
| `invited` | boolean | `true` |
| `invitation_delivery` | string | `beta_group_assignment` |

邮箱已存在于 PopCat 的测试名单时返回 `409`：

```json
{
  "error": "您的邮箱已在PopCat的测试名单中。",
  "details": {
    "beta_tester_id": "12345678-...",
    "state": "INVITED",
    "status_message": "您的测试状态为已邀请。"
  }
}
```

`details.state` 为苹果原始状态（`INVITED` / `ACCEPTED` / `INSTALLED` / `REVOKED`），仅供后端逻辑与日志使用。

### 2.2 重新邀请（恢复已停用测试员）

```http
POST /api/v1/internal/testflight/invitations/reinvite
Content-Type: application/json
X-PopCat-Internal-Token: <brand internalToken>
```

请求体：`{ "email": "user@example.com" }`。

成功 `200`：在 [测试员状态](#testflight-测试员状态) 字段基础上追加：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `invitation_id` | null | 固定 `null` |
| `invited` | boolean | `true` |
| `invitation_delivery` | string | `beta_group_assignment` |
| `beta_tester_created` | boolean | `false` |

测试员不存在返回 `404`（`Tester is not in PopCat test list`）。

### 2.3 查询测试员状态

```http
GET /api/v1/internal/testflight/testers?email=user@example.com
X-PopCat-Internal-Token: <brand internalToken>
```

成功 `200`：[测试员状态](#testflight-测试员状态) 对象。

### 2.4 停用测试员（保留名额）

```http
DELETE /api/v1/internal/testflight/testers
Content-Type: application/json
X-PopCat-Internal-Token: <brand internalToken>
```

请求体：`{ "email": "user@example.com" }`（也接受查询串 `?email=`）。

成功 `200`：在 [测试员状态](#testflight-测试员状态) 基础上追加 `disabled: true`、`slot_retained: true`（此时 `in_beta_group` 为 `false`）。测试员不存在返回 `404`。

### 2.5 释放名额

```http
DELETE /api/v1/internal/testflight/testers/release
Content-Type: application/json
X-PopCat-Internal-Token: <brand internalToken>
```

请求体：`{ "email": "user@example.com" }`。

成功 `200`（已删除）：

```json
{ "email": "user@example.com", "exists": false, "beta_tester_id": "12345678-...", "released": true, "slot_released": true }
```

邮箱本就不在名单中时同样 `200`，但 `released` 与 `slot_released` 为 `false`、`beta_tester_id` 为 `null`。

### 2.6 公开邀请链接

```http
GET /api/v1/internal/testflight/public-link
X-PopCat-Internal-Token: <brand internalToken>
```

成功 `200`（`cache-control: private, max-age=300`）：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `app_id` | string | App Store Connect 应用资源 ID |
| `beta_group_id` | string | Beta 组资源 ID |
| `name` | string \| null | Beta 组名称 |
| `public_link` | string \| null | 公开 TestFlight 链接 |
| `public_link_enabled` | boolean | 是否启用公开链接 |
| `public_link_limit_enabled` | boolean | 是否启用人数上限 |
| `public_link_limit` | number \| null | 人数上限 |
| `public_link_id` | string \| null | 链接短码 |
| `is_internal_group` | boolean | 是否内部组 |
| `feedback_enabled` | boolean | 是否开启反馈 |
| `generated_at` | string | 生成时间 |
| `cache_expires_at` | string | 缓存过期时间（生成后 5 分钟） |

该接口为 PopCat 内部保留，当前 V2board 集成不应依赖。

> App Store Connect 未配置（缺 `APP_STORE_CONNECT_*` 任一项）时，TestFlight 接口返回 `500`，`details.missing` 列出缺失的环境变量名。苹果接口 5xx 会被转成 `502`，`details` 含 `app_store_connect_status` 与原始 `errors`。

---

## 三、品牌后端必须实现的契约接口

以下接口由**品牌后端自行实现**，代理在处理客户端请求时回源调用。代理发起时携带该品牌的 `X-PopCat-Internal-Token`，并透传客户端的 `User-Agent`、`X-Client-Pubkey`、`X-Device-Meta`、`X-Nonce`、`X-Signature`、`X-Timestamp`、`X-Forwarded-For` 等请求头。响应均为明文 JSON。回源超时 `504`，连接失败 `502`。

### 3.1 账号是否可能存在

```http
POST /api/v1/internal/apple/accounts/may-exist
```

请求体：`{ "login": "user@example.com" }`。

响应：`{ "may_exist": true }`。

用于登录前过滤候选品牌，建议用 Bloom filter 等服务端手段，避免把账号存在性数据下发到客户端。代理对该接口的 4xx 容错（视为 `may_exist=false`），仅 5xx 向上抛错。

### 3.2 账号密码登录

```http
POST /api/v1/internal/apple/auth/login
```

请求体：`{ "email": "user@example.com", "password": "secret", "device": "MacBookPro/..." }`。

响应：

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `upstream_token` | string | 是 | 品牌私有令牌；缺失时代理返回 `502` |
| `subject` | string | 否 | 稳定用户标识；省略时回退为登录邮箱 |
| `user` | object | 否 | 用户对象，见 [user 用户对象](#user-用户对象) |

### 3.3 订阅地址登录

```http
POST /api/v1/internal/apple/subscription/login
```

请求体：`{ "url": "https://订阅链接", "device": "..." }`。

响应字段同 `auth/login`（`upstream_token` 必填，`subject`、`user` 可选）。

### 3.4 用户信息

```http
GET /api/v1/internal/apple/user
Authorization: Bearer <upstream_token>
```

响应：保持 Apple 既有用户结构，见 [user 用户对象](#user-用户对象)。

### 3.5 节点

```http
GET /api/v1/internal/apple/nodes
Authorization: Bearer <upstream_token>
```

响应：节点配置。可返回 JSON，也可返回二进制（代理会原样加密透传）。结构见 [节点配置](#节点配置)。

### 3.6 设备记录

```http
POST /api/v1/internal/apple/devices/seen
Authorization: Bearer <upstream_token>
```

代理在客户端携带 `X-Client-Pubkey` 时调用，请求体：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `user_id` | string \| null | 令牌 `sub` |
| `client_pubkey` | string | 客户端公钥 |
| `device_meta` | string | 设备描述（`X-Device-Meta`） |
| `ip` | string | 客户端 IP（`X-Forwarded-For`） |

代理对该接口的失败仅记录日志、不影响主流程，因此返回任何 2xx 即可。

---

## 四、运维接口

### 4.1 健康检查

```http
GET /health
```

无鉴权，明文 JSON：

```json
{ "ok": true, "brands": ["sakanacloud", "sunbear", "wbyun"] }
```

代理还提供 `/`、`/support`、`/privacy` 三个 HTML 落地页（`cache-control: public, max-age=300`），与接口集成无关。未匹配任何路由返回 `404`：`{ "error": "Not found" }`。

---

## 公共数据结构

### publicBrand（品牌公开信息）

仅品牌注册表中的 `id`、`name`、`shortName`、`logoUrl`、`links` 会下发给客户端，`originUrl` / `hostHeader` / `internalToken` 永不外泄。

```json
{
  "id": "sakanacloud",
  "name": "SakanaCloud鱼云",
  "short_name": "鱼云",
  "logo_url": "https://pop-cat.com/brands/sakanacloud.png",
  "app_links": {
    "website": "https://iplc.homes/",
    "register": "https://iplc.homes/#/register",
    "telegram": "https://t.me/+L2ngMruEQkY5ZmY1"
  }
}
```

### user 用户对象

品牌后端返回、并经 `/user` 透传的建议结构：

```json
{
  "name": "Alice",
  "email": "user@example.com",
  "subscriptions": [
    {
      "id": "...",
      "plan": "...",
      "cycle_reset_at": null,
      "cycles_left": 1,
      "used_traffic": 0,
      "total_traffic": 0
    }
  ],
  "app_links": {
    "website": "https://brand.example/",
    "register": "https://brand.example/#/register",
    "telegram": "https://t.me/brand"
  },
  "log_redact": {}
}
```

代理会在非订阅登录时追加 `brand`（publicBrand）；订阅登录时把 `name` / `email` / `app_links` 置空。

### TestFlight 测试员状态

`GET /testflight/testers` 与停用、重新邀请响应的基础结构：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `email` | string | 规范化邮箱 |
| `exists` | boolean | 是否存在于 App Store Connect 测试员列表 |
| `beta_tester_id` | string \| null | 测试员资源 ID |
| `in_beta_group` | boolean | 是否在配置的 `PopCat-01` Beta 组内 |
| `state` | string \| null | 苹果原始状态：`INVITED` / `ACCEPTED` / `INSTALLED` / `REVOKED`，可能为 `null` |

不存在时：`exists=false`、`beta_tester_id=null`、`in_beta_group=false`、`state=null`。

### 节点配置

`/nodes` 与品牌契约 `apple/nodes` 的 JSON 结构（内置节点集合同形）：

```json
{
  "entry_groups": {
    "hk": {
      "entries": [
        { "name": "香港入口", "domain": "hk-xxx.example", "port": 8443, "protocol": "anytls", "tier": 0, "weight": 110, "stack": "dual" }
      ]
    }
  },
  "default_entries": { "hk": "hk-xxx.example", "jp": "jp-xxx.example", "sg": "sg-xxx.example" },
  "selection_policy": { "switch_margin": 0.2, "reeval_interval_sec": 600 },
  "nodes": [
    {
      "type": "anytls",
      "name": "🇭🇰 HK1",
      "password": "...",
      "sni": "pic.example.com",
      "alpn": ["h2", "http/1.1"],
      "certificate_public_key_sha256": ["base64..."],
      "fingerprint": "apple",
      "entry_group": "hk",
      "sort": 0
    }
  ]
}
```

`entry_groups` 为多入口选路的入口候选，`default_entries` 为每个区域的默认入口域名，`selection_policy` 控制切换迟滞与重评估周期，`nodes` 为出口节点列表。

### 路由配置

```json
{
  "revision": "<sha256>",
  "brand_id": "sakanacloud",
  "default":  { "outbounds": "...JSON 数组字符串...", "route": "...JSON 对象字符串..." },
  "global":   { "outbounds": "...", "route": "..." },
  "headless": { "outbounds": "...", "route": "..." },
  "updated_at": "2026-05-30T00:00:00.000Z"
}
```

`default` / `global` / `headless` 三种模式各含一段 `outbounds` 与 `route`。支持的占位符：`{{PROBE_OUTBOUNDS_JSON}}`、`{{FB_GEOSITE_CN}}`、`{{FB_GEOSITE_GEOLOCATION_CN}}`、`{{FB_GEOIP_CN}}`。默认对 `sakanacloud` / `wbyun` / `sunbear` 下发内置模板，可在品牌对象的 `routingConfig` 或全局 `POPCAT_ROUTING_CONFIG_JSON` 中覆盖。

### 版本策略

每个平台（`ios` / `macos`）字段：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `latestVersion` | string | 最新版本号 |
| `minSupportedVersion` | string | 最低支持版本，低于此版本应强制更新 |
| `minSupportedBuild` | number \| null | 同一产品版本内的最低构建号，用于屏蔽问题构建 |
| `updateUrl` | string \| null | 更新跳转地址 |
| `releaseNotes` | string \| null | 更新说明 |
