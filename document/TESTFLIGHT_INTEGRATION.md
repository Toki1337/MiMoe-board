# TestFlight 接入说明（V2board 品牌后端）

本文件说明基于 V2board 的品牌后端如何接入 `pop-cat-proxy` 来签发与管理 PopCat 的 TestFlight 测试资格。逐字段的请求与响应见 [API_REFERENCE.md 第二节](./API_REFERENCE.md#二代理对品牌后端开放的-testflight-内部接口prefix-apiv1internal)，通用接入背景见 [INTERNAL_INTEGRATION.md](./INTERNAL_INTEGRATION.md)。

## 职责划分

品牌后端是业务事实来源。`pop-cat-proxy` 只负责调用 App Store Connect 并执行被请求的 TestFlight 操作，**不会回调、也不会异步通知品牌后端**。每一次本地状态变更，都必须由品牌后端在自己同步发起的调用拿到响应后立即完成。

- 基础地址：`https://pop-cat.com/api/v1/internal`
- 鉴权头：`X-PopCat-Internal-Token: <brand internalToken>`
- 不要发送 `brand_id`，代理用令牌映射品牌。

## 品牌名额上限

当前各品牌可用测试员名额（硬编码）：

```text
sakanacloud: 7000
sunbear:     1000
wbyun:       1000
```

品牌后端必须在调用 PopCat 接口前自行核对名额。PopCat 不了解 V2board 的用户表，也不了解品牌本地的名额分配。

每个符合条件的 V2board 用户默认 1 个测试员名额。需要更多时走工单，管理员可手动增减该用户的名额上限。建议在 `v2_user` 上加字段：

```text
testflight_tester_limit int not null default 1
```

品牌后端按此上限统计该用户的本地 TestFlight 名额记录。

## 可用用户判定

任何 TestFlight 操作前，品牌后端都必须先过滤可用用户。用户可用至少意味着：

- 有生效的套餐/订阅
- 订阅未过期
- 仍有剩余流量
- 未被封禁/停用

V2board 订阅路由已有专门的 `available user` 判定函数，应复用它，而不是在 TestFlight 控制器里另写一套松散判断。用户不可用时，品牌后端不得调用 PopCat。

## 本地名额模型

每个测试员邮箱应由一条归属某个 V2board 用户的本地记录表示。不能只依赖苹果侧状态，因为 PopCat 的业务规则需要品牌本地名额、归属、停用与释放历史。建议表 `v2_popcat_testflight_slots`：

```text
id bigint primary key
user_id bigint not null
email varchar(255) not null
normalized_email varchar(255) not null
status varchar(32) not null
popcat_beta_tester_id varchar(128) null
last_invitation_id varchar(128) null
last_error text null
created_at int not null
updated_at int not null
invited_at int null
disabled_at int null
released_at int null
```

建议索引：`unique(normalized_email)`、`index(user_id, status)`。写库或调用 PopCat 前，邮箱先做小写加去空格的规范化。

## 本地状态机

本地使用四种状态：

| 状态 | 含义 | 是否占名额 |
| --- | --- | --- |
| `active` | 邮箱存在于 App Store Connect 且应在 `PopCat-01` Beta 组内 | 是 |
| `disabled` | 已移出 `PopCat-01`，但仍留在 All Testers，可用重新邀请恢复 | 是 |
| `released` | 已从 All Testers 删除，名额已释放，同邮箱可走首次邀请重新加入 | 否 |
| `failed` | 操作在到达稳定状态前失败，存 `last_error` 待管理员重试或手动释放 | 视实现而定 |

正常操作不要删除本地行，保留它们用于审计与名额历史。`released` 行可作为历史数据留存，但不再计入名额。

## 名额的占用与释放

- `DELETE /testflight/testers`（**停用**）：只把人移出 `PopCat-01`，测试员仍在 All Testers，按策略继续占用名额。**不释放名额。**
- `DELETE /testflight/testers/release`（**释放**）：从 All Testers 删除测试员，释放苹果名额。品牌后端应把本地标记为 `released` 并停止计入名额。

关于自动释放，需保守处理。V2board 集成的定时释放任务，只能在用户订阅确实过期时释放测试员（`expired_at` 非空且 `expired_at <= now`，包含 `expired_at = 0`）。不要仅因为用户流量耗尽、无套餐、被封禁或临时不可用就自动释放——那些状态可阻断新操作，但自动释放会撤销一封刚送达的苹果邀请邮件，使链接显示「邀请已被撤销或无效」。

## 品牌后端推荐操作流程

每个操作都遵循「先校验、先占位、再同步调 PopCat、拿到 2xx 后落本地状态」。PopCat 不发回调，不要等异步通知。

### 添加测试员邮箱

1. 校验用户可用。
2. 校验该用户 `active + disabled` 名额数未达 `testflight_tester_limit`。
3. 校验品牌 `active + disabled` 名额数未达品牌上限。
4. 调 PopCat 前先建本地占位记录（有 `pending` 状态用 `pending`，否则先建为 `failed`），防止并发请求超额。
5. 同步调 `POST /testflight/invitations`。
6. 该 HTTP 调用返回 2xx 后，立即更新同一条记录：`status=active`、`popcat_beta_tester_id=response.beta_tester_id`、`last_invitation_id=response.invitation_id`、`invited_at=now`。
7. 不要等待异步通知再置为 `active`。
8. `409` 时展示「您的邮箱已在PopCat的测试名单中。」并移除占位记录，除非管理员要手动核对归属。
9. 除 `409` 外的非 2xx，保留 `failed` 记录并存 `last_error`，或按实现移除占位记录。

### 停用测试员（保留名额）

1. 校验本地记录属于该 V2board 用户。
2. 同步调 `DELETE /testflight/testers`。
3. 返回 2xx 后置本地为 `disabled`，继续计入名额。

### 恢复已停用测试员

1. 校验用户可用。
2. 校验本地记录属于该用户且状态为 `disabled`。
3. 同步调 `POST /testflight/invitations/reinvite`。
4. 返回 2xx 后置本地为 `active`，并按响应更新 `last_invitation_id`。

### 释放名额

1. 校验本地记录属于该用户，或要求管理员权限。
2. 同步调 `DELETE /testflight/testers/release`。
3. 返回 2xx 后置本地为 `released`，停止计入名额。

### 查询状态

1. 先读本地记录。
2. 可选调 `GET /testflight/testers?email=...` 与苹果侧对账。
3. 若苹果返回 `exists=false` 而本地为 `active` / `disabled`，仅在符合管理员动作或显式对账流程时才置为 `released`。

## V2board 前端集成

V2board 前端必须调用品牌后端自己的用户接口，绝不能直接调 PopCat 内部接口，也绝不能拿到或保存 `X-PopCat-Internal-Token`。建议提供「TestFlight / App 测试资格」用户页面，仅登录后可见；后端用户中间件与可用判定仍是事实来源，前端要按 `403` 展示后端返回的提示。

### 品牌后端面向用户的接口

以下接口由品牌后端实现，使用与其他 `/api/v1/user/*` 一致的 V2board 用户鉴权。

**列出当前用户可见名额**——只返回 `active` / `disabled` / `pending`，`released` 与 `failed` 可留库审计但不展示给用户。

```http
GET /api/v1/user/testflight/slots
```

```json
{
  "data": {
    "slots": [
      {
        "id": 1,
        "email": "user@example.com",
        "status": "active",
        "status_text": "已邀请",
        "created_at": 1779926400,
        "updated_at": 1779926400,
        "invited_at": 1779926400,
        "disabled_at": null,
        "released_at": null
      }
    ],
    "limit": 1,
    "used": 1
  },
  "message": "TestFlight名额"
}
```

该响应不得包含 PopCat / App Store Connect 内部 ID、品牌全局名额字段，或 `last_error` 等后端诊断字段。

**创建首次邀请**：

```http
POST /api/v1/user/testflight/invitations
Content-Type: application/json

{ "email": "user@example.com" }
```

**停用但保留名额**（请求体支持 `{ "id": 1 }` 或 `{ "email": "user@example.com" }`）：

```http
DELETE /api/v1/user/testflight/testers
```

**恢复已停用测试员**：

```http
POST /api/v1/user/testflight/invitations/reinvite
Content-Type: application/json

{ "id": 1 }
```

**释放名额**：

```http
DELETE /api/v1/user/testflight/testers/release
Content-Type: application/json

{ "id": 1 }
```

**查询某条名额的苹果远端状态**（支持 `?id=1` 或 `?email=user@example.com`）：

```http
GET /api/v1/user/testflight/testers/status?id=1
```

```json
{
  "data": {
    "slot": {
      "id": 1,
      "email": "user@example.com",
      "status": "active",
      "status_text": "已邀请",
      "created_at": 1779926400,
      "updated_at": 1779926400,
      "invited_at": 1779926400,
      "disabled_at": null,
      "released_at": null
    },
    "testflight_status": {
      "exists": true,
      "in_beta_group": true,
      "state_text": "已邀请"
    }
  },
  "message": "TestFlight状态"
}
```

不要把苹果原始状态或 PopCat 响应对象直接暴露给前端。

### 前端状态映射

| 本地状态 | 展示 | 可用操作 |
| --- | --- | --- |
| `active` | 已邀请 / 使用中 | 停用、释放 |
| `disabled` | 已停用 / 名额保留 | 重新邀请、释放 |
| `pending` | 处理中 | 禁用所有行操作，请求结束后刷新列表 |
| `failed` | 处理失败 | 不展示后端诊断文本；允许释放或提示联系客服。`409` 不应在用户列表里产生 `failed` 行 |
| `released` | 已释放 | 不计入已用名额，无需行操作 |

前端用 `used / limit` 展示用户名额占用。品牌全局名额是后端/管理员的事，不得由面向用户的接口返回。

### 推荐交互流程

页面加载：调 `GET /api/v1/user/testflight/slots`，渲染名额与用量，`used >= limit` 时禁用添加表单。

添加邮箱：本地先校验邮箱格式做即时反馈；提交 `POST /api/v1/user/testflight/invitations`，请求期间禁用提交按钮；成功展示后端 `message` 并刷新列表；失败原样展示后端 `message`，多行要保留换行。

`409` 时直接展示后端消息。若 PopCat 响应含 `details.status_message`，品牌后端应在返回前把苹果原始状态转成面向用户的中文，前端不应直接展示 `INVITED` / `ACCEPTED` / `INSTALLED` / `REVOKED` 等原始状态。示例：

```text
您的邮箱已在PopCat的测试名单中。
您的测试状态为已邀请。
```

苹果状态到中文的建议映射：

```text
INVITED   -> 已邀请
ACCEPTED  -> 已接受
INSTALLED -> 已安装
REVOKED   -> 已撤销
null      -> 不展示状态行
unknown   -> 状态未知
```

`403` 展示后端消息并隐藏/禁用 TestFlight 操作，通常表示用户不可用（套餐过期、无剩余流量或被封禁）。`422` 在表单或行操作旁展示后端消息，通常表示达到名额上限或所选名额状态不对。`503` 作为临时容量错误展示。

### 前端文案

```text
页面标题：TestFlight 测试资格
添加表单标签：Apple ID 邮箱
添加按钮：发送邀请
用量标签：已使用 {used}/{limit}
停用操作：停用
恢复操作：重新邀请
释放操作：释放名额
状态操作：查看状态
```

面向用户的界面不要提到 PopCat 内部令牌、品牌 ID 或 App Store Connect 内部 ID。

## App Store Connect 凭据

代理调用苹果接口需配置以下生产环境变量，缺任一项时 TestFlight 接口返回 `500`：

```text
APP_STORE_CONNECT_ISSUER_ID=<issuer UUID>
APP_STORE_CONNECT_KEY_ID=<key id>
APP_STORE_CONNECT_PRIVATE_KEY="-----BEGIN PRIVATE KEY-----\n...\n-----END PRIVATE KEY-----"
APP_STORE_CONNECT_APP_ID=<App Store Connect 应用资源 ID>
APP_STORE_CONNECT_BETA_GROUP_ID=<TestFlight Beta 组资源 ID>
```

## 公开邀请链接

`GET /testflight/public-link` 返回 Beta 组的公开 TestFlight 链接。它是苹果 Beta 组链接，非一次性、不绑定品牌用户。该接口为 PopCat 内部保留，当前 V2board 集成应使用邮箱邀请管理，不要依赖公开链接。字段见 API 参考第 2.6 节。
