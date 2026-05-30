# 品牌后端接入说明（Internal）

本文件说明各品牌后端如何接入 `pop-cat-proxy`，覆盖登录、用户信息、节点、设备记录、订阅登录、路由配置、版本策略与引导代理等通用功能。TestFlight 相关接入单独见 [TESTFLIGHT_INTEGRATION.md](./TESTFLIGHT_INTEGRATION.md)，逐字段的请求与响应见 [API_REFERENCE.md](./API_REFERENCE.md)。

## 设计前提

`pop-cat-proxy` 是 `pop-cat.com` 的统一入口。PopCat 客户端只与代理通信，各品牌后端作为内部提供方藏在一份稳定契约之后。客户端拿到的是代理签发的 `pop-cat.com` 令牌，而非品牌令牌；该令牌内部绑定了 `brand_id` 与品牌私有令牌（`upstream_token`），后续请求据此直接路由到对应品牌，无需逐个探测。代理不保存任何登录态，可在 Vercel 无状态实例上运行。

品牌后端只需做两件事：

1. 实现一组契约接口（`/api/v1/internal/apple/*`），供代理回源调用。
2. 在品牌注册表中登记 `originUrl` 与 `internalToken`，让代理能找到并鉴权。

## 品牌注册表

通过 Vercel 环境变量 `BRAND_BACKENDS` 配置一个品牌数组。每个品牌字段：

| 字段 | 必填 | 说明 |
| --- | --- | --- |
| `id` | 是 | 品牌标识，全局唯一 |
| `name` | 是 | 品牌全称 |
| `shortName` | 否 | 简称，省略时取 `name` |
| `originUrl` | 是 | 品牌后端回源地址，末尾斜杠会被去掉 |
| `hostHeader` | 否 | 回源时覆盖 `Host` 请求头 |
| `internalToken` | 是 | 代理与品牌之间的共享密钥 |
| `logoUrl` | 否 | 品牌图标 |
| `subscriptionHosts` | 否 | 订阅登录允许的主机名数组，用于按订阅链接匹配品牌 |
| `links` | 否 | `website` / `register` / `telegram` 三个外链 |
| `routingConfig` | 否 | 该品牌的路由模板覆盖 |

`originUrl`、`hostHeader`、`internalToken` 仅留在服务端，绝不下发给客户端；客户端只看到 `id`、`name`、`shortName`、`logoUrl`、`links`（见 API 参考的 publicBrand）。注册表有 60 秒缓存（`BRAND_REGISTRY_CACHE_SECONDS`）。

当前默认对 `sakanacloud`、`wbyun`、`sunbear` 下发内置路由模板。

## 鉴权两条链路

接入时要区分两个方向的鉴权，二者都用 `X-PopCat-Internal-Token`，但谁发给谁不同：

- **代理回源到品牌**：代理调用品牌契约接口时，在请求头放入该品牌的 `internalToken`。品牌后端应校验这个头，确认请求来自代理。
- **品牌后端调用代理的 TestFlight 接口**：品牌后端在请求头放入自己的 `internalToken`，代理据此把调用方匹配到品牌。

客户端侧用的是代理令牌（JWT），与上面两条都无关。三者的字段细节见 [API_REFERENCE.md 的鉴权与加密机制](./API_REFERENCE.md#鉴权与加密机制)。

## 品牌后端必须实现的契约接口

代理回源调用以下接口（请求与响应字段见 API 参考第三节）：

| 接口 | 用途 | 鉴权 |
| --- | --- | --- |
| `POST /api/v1/internal/apple/accounts/may-exist` | 登录前过滤候选品牌 | `X-PopCat-Internal-Token` |
| `POST /api/v1/internal/apple/auth/login` | 账号密码登录 | `X-PopCat-Internal-Token` |
| `POST /api/v1/internal/apple/subscription/login` | 订阅地址登录 | `X-PopCat-Internal-Token` |
| `GET /api/v1/internal/apple/user` | 用户信息 | `Bearer upstream_token` |
| `GET /api/v1/internal/apple/nodes` | 节点配置 | `Bearer upstream_token` |
| `POST /api/v1/internal/apple/devices/seen` | 设备记录 | `Bearer upstream_token` |

代理回源时会透传客户端的若干请求头（`User-Agent`、`X-Client-Pubkey`、`X-Device-Meta`、`X-Nonce`、`X-Signature`、`X-Timestamp`、`X-Forwarded-For`），品牌后端可据此做风控或设备绑定。

### 登录流程

1. 客户端 `POST /api/v1/apple/login` 带邮箱、密码。
2. 未指定 `brand_id` 时，代理先对各品牌调 `accounts/may-exist`，过滤出可能存在该账号的品牌。
3. 代理并发对候选品牌调 `auth/login`。
4. 唯一命中：直接返回代理令牌。多个命中：返回 `409 BRAND_SELECTION_REQUIRED`，客户端让用户选品牌后带 `brand_id` 重试。
5. 品牌后端在 `auth/login` 响应里返回 `upstream_token`（必填）、`subject`、`user`。代理把 `upstream_token` 与 `brand_id` 封进代理令牌。

`may-exist` 应避免把账号存在性数据下发到客户端，建议用 Bloom filter 等服务端手段。代理对其 4xx 容错（当作不存在），仅 5xx 向上抛错。

### 用户信息与设备记录

客户端 `GET /api/v1/apple/user` 时，代理用令牌里的 `upstream_token` 回源 `apple/user`，并在客户端携带公钥时附带调用一次 `devices/seen`。`user` 建议保持 Apple 既有结构（`name`、`email`、`subscriptions`、`app_links`、`log_redact`），代理会按登录方式补充或剥离字段。

### 节点

`GET /api/v1/apple/nodes` 回源到品牌 `apple/nodes`。品牌可返回 JSON，也可返回二进制——代理对二进制原样加密透传，不解析。节点结构（入口分组、默认入口、选路策略、出口列表）见 API 参考的[节点配置](./API_REFERENCE.md#节点配置)。

### 订阅登录

客户端 `POST /api/v1/apple/subscription/login` 带订阅链接。代理按链接主机名在各品牌的 `subscriptionHosts` 中匹配，命中后回源到该品牌的 `subscription/login`。订阅登录的用户信息会被代理剥离 `name` / `email` / `app_links`，且不下发品牌信息。

## 代理侧统一下发的配置

以下配置由代理直接生成或读取环境变量，不经品牌后端，但接入时需要了解。

### 路由配置

`GET /api/v1/apple/routing-config` 按令牌绑定的品牌返回 `default` / `global` / `headless` 三套模板。客户端用 `X-Routing-Config-Revision` 携带已缓存版本号，版本一致时代理返回 `204` 省流量。

覆盖方式有两种：品牌对象内的 `routingConfig`，或全局环境变量 `POPCAT_ROUTING_CONFIG_JSON`（按 `brands.<id>` 或顶层 `<id>` 取覆盖）。模板里 `outbounds` 可为 JSON 数组或其字符串形式，`route` 可为 JSON 对象或其字符串形式。支持占位符 `{{PROBE_OUTBOUNDS_JSON}}`、`{{FB_GEOSITE_CN}}`、`{{FB_GEOSITE_GEOLOCATION_CN}}`、`{{FB_GEOIP_CN}}`。

### 版本策略

`GET /api/v1/apple/version-policy` 返回 `ios` / `macos` 两个平台的版本要求，读取环境变量 `APP_VERSION_POLICY`。客户端应采用语义化版本比较：

```text
若 当前版本 < minSupportedVersion：阻断主流程，要求更新
否则若 当前版本 < latestVersion：提示可选更新
否则：正常进入
```

`minSupportedBuild` 仅用于屏蔽同一产品版本内的问题构建（例如 `1.0.2 build 105` 损坏、`build 106` 修复时设为 `106`）。首个 TestFlight 构建发布前可不配置，接口会返回 `0.0.0` 占位策略，等于不强制更新。完整规则与 Vercel 更新流程见仓库根目录 `VERSION_CONTROL.md`。

### 引导代理配置

`GET /api/v1/apple/bootstrap-proxy` 在客户端直连受阻时提供内置代理出口，读取 `BOOTSTRAP_PROXY_CONFIG_JSON`，同样支持 `X-Bootstrap-Proxy-Revision` 版本协商返回 `204`。出口按 `priority` 升序排列。此外，来源 IP 落在临时封锁网段 `39.144.` 的客户端会被整层拦截，返回 `403 BOOTSTRAP_DIRECT_BLOCKED`，提示客户端改走内置代理。

## 关键环境变量

| 变量 | 说明 |
| --- | --- |
| `BRAND_BACKENDS` | 品牌注册表 JSON，必填 |
| `POPCAT_PROXY_JWT_SECRET` | 代理令牌签名密钥，生产环境用长随机值 |
| `PUBLIC_API_PREFIX` | 公开接口前缀，默认 `/api/v1/apple` |
| `INTERNAL_API_PREFIX` | 内部接口前缀，默认 `/api/v1/internal` |
| `TOKEN_TTL_SECONDS` | 代理令牌有效期，默认 30 天 |
| `REQUEST_TIMEOUT_MS` | 回源超时，默认 15000 |
| `BRAND_REGISTRY_CACHE_SECONDS` | 品牌注册表缓存，默认 60 |
| `POPCAT_ROUTING_CONFIG_JSON` | 全局路由模板覆盖 |
| `APP_VERSION_POLICY` | 版本策略 |
| `BOOTSTRAP_PROXY_CONFIG_JSON` | 引导代理配置 |
| `APP_STORE_CONNECT_*` | TestFlight 用的 App Store Connect 凭据，见 TestFlight 文档 |

## 接入自检清单

- [ ] 在 `BRAND_BACKENDS` 登记品牌，确认 `id`、`originUrl`、`internalToken` 正确。
- [ ] 实现六个契约接口并校验 `X-PopCat-Internal-Token` 与 `Bearer upstream_token`。
- [ ] `auth/login` 返回的 `upstream_token` 非空。
- [ ] `may-exist` 不向客户端泄露账号存在性。
- [ ] 需要订阅登录时配置 `subscriptionHosts` 并实现 `subscription/login`。
- [ ] 确认 `originUrl` / `hostHeader` / `internalToken` 不出现在任何下发给客户端的响应里。
- [ ] `GET /health` 返回的 `brands` 列表包含本品牌。
