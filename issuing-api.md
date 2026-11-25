# 发卡 Open API 接口文档

API 版本：`v1` · 更新日期：2026-11-17 · 传输格式：HTTPS / JSON

[HTML 文档](index.html) · [兼容入口](partner-open-api-channel-spec.html) · [Markdown 下载](issuing-api.md) · [OpenAPI YAML](openapi.yaml) · [OpenAPI JSON](openapi.json)

## 1. 概述与接入准备 {#overview}

本接口供商户服务端接入产品查询、单卡余额开卡、卡充值、卡状态管理、余额、交易历史、钱包账本和事件通知。所有资源均受商户归属及已开通权限限制。

本文使用 `https://api.example.com` 作为**示例地址**，不是实际服务地址。请将示例中的 `{BASE}` 替换为接入开通资料提供的 API 根地址，不带末尾 `/`。文档站地址不等于 API 地址。

接入前请取得发卡类型 `ISSUING` 的 AccessKeyId、SecretKey、商户编码和服务地址，并确认可用产品、关联账户和主钱包资金。首次开卡报价及新申请要求商户**已配置唯一可用开卡账户**；账户配置缺失或不唯一时应联系服务支持，不能在开卡请求中指定另一个账户绕过。

商户开通、凭证管理、主钱包入金及提现不由本文接口提供，按已约定的服务流程办理。Webhook 使用独立 Secret，并需事先配置商户 HTTPS 接收地址及订阅事件。

### 1.1 资金与查询对象 {#money-model}

| 对象 | 用途 | 查询方式 |
| --- | --- | --- |
| 主钱包 `MAIN` | 支付开卡、充值本金及服务费 | `/wallets`、`/wallets/{walletId}` |
| 单卡余额 | 支付该卡消费并接收适用的卡内退款 | `/cards/{cardId}/balance` |
| 卡交易 | 消费、授权、退款、冲正等可见记录 | `/transactions` |
| 钱包账本 | 钱包入账、预占、释放等资金变动 | `/wallet-ledger` |
| 卡组钱包 `CARD_GROUP` | 已有卡组资金及兼容查询 | `/wallets`，按钱包区分账本 |

开卡或充值受理时，主钱包一次预占本金和服务费；成功后确认该预占，不重复扣款。后续卡内消费和退款不再次扣退主钱包本金。卡余额不足不会自动从主钱包补款。销卡完成与余额退回主钱包是两个需要分别确认的结果。

### 1.2 账户查询范围 {#account-scope}

`memberUserId` 是商户关联账户 ID，不是商户编码。仅在明确支持该参数的接口传入；指定值必须属于当前商户。

| 接口 | 省略账户参数时的范围 |
| --- | --- |
| 卡列表、卡详情 | 当前商户可见的卡；卡列表不接受 `memberUserId` |
| 钱包列表 | 当前商户范围内关联账户的可见钱包 |
| 交易、钱包账本、卡组列表 | 商户默认关联账户；查询指定账户时传 `memberUserId` |
| 开卡报价、新开卡 | 必须已配置唯一可用开卡账户，接口不提供账户选择参数 |
| 已有卡的余额、充值与状态操作 | 按卡 ID 校验商户归属并使用该卡所属账户 |

默认账户查询不等于所有账户汇总。导入多个账户的历史交易或账本时，应先确认账户映射，再分别查询。

## 2. 公共协议 {#protocol}

### 2.1 请求、金额与时间 {#formats}

- 基础路径为 `/open/v1`，业务请求使用 HTTPS；JSON 请求头为 `Content-Type: application/json`。
- 除免签健康检查外，所有请求需携带发卡凭证与签名。
- 新开卡和充值金额为 USD 最小单位的正整数，`1000` 表示 10.00 USD；JSON 不接受小数、科学计数法、数字字符串或超出 64 位有符号整数范围的金额。
- 兼容卡组划出接口的 `amount` 是准确的十进制字符串，例如 `"50.00"`；不得与整数金额接口混用。
- 返回 ID 的 JSON 类型按各模型说明保留；字符串 ID 不应经浮点转换。路径使用平台返回的资源 ID，不使用完整卡号或其他来源的标识代替。
- 请求签名时间戳为 Unix **秒**；Webhook 时间戳为 Unix **毫秒**。业务时间采用 ISO-8601，查询建议使用 UTC `Z`，如 `2026-11-01T00:00:00Z`。
- 未注明的请求参数不作为支持能力。新开卡、充值及状态操作拒绝未知 JSON 字段。
- 返回对象允许扩展字段；接入程序应忽略不认识的响应字段，同时保留不认识的状态值供核对。

### 2.2 公共响应 {#response-envelope}

成功响应：

```json
{"code":"0","message":"OK","data":{"status":"UP"},"traceId":"example-trace-id"}
```

错误响应示例：

```json
{"code":"40111","message":"请求签名不匹配","traceId":"example-trace-id"}
```

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `code` | string | `"0"` 表示本次 API 调用成功；其他值为业务错误码 |
| `message` | string | 结果说明，程序分支优先使用 `code` |
| `data` | object / array | 各接口的业务数据；错误时可能不出现 |
| `traceId` | string / null | 排查引用，可能为空；联系支持时提供 |

后文“响应 data”示例只展示 `data` 内容，实际 HTTP 响应包含上述外壳。HTTP 200 或 `code="0"` 不能代替异步操作的业务完成状态，开卡、充值及卡操作仅以 `status=SUCCEEDED` 确认完成。

## 3. 请求鉴权与签名 {#authentication}

### 3.1 请求头 {#request-headers}

| 请求头 | 必填 | 说明 |
| --- | --- | --- |
| `X-Access-Key-Id` | 是 | 已开通的 ISSUING AccessKeyId |
| `X-Merchant-Code` | 是 | 与凭证绑定的商户编码 |
| `X-Vcc-Timestamp` | 是 | 当前 Unix 秒时间戳的十进制字符串 |
| `X-Vcc-Signature` | 是 | HMAC-SHA256 的小写十六进制结果 |
| `Idempotency-Key` | 写操作是 | 稳定业务幂等键，详见[幂等与恢复](#idempotency) |
| `Content-Type` | JSON 请求是 | `application/json` |

`GET /open/v1/health` 不要求签名。本文标准发卡请求采用四行签名，**不发送 `X-Vcc-Nonce`**；通知头中的 nonce 属于另一个验签协议。

### 3.2 四行 Canonical Request {#canonical-request}

按以下顺序生成四行内容，各行以单个 LF（`\n`）连接，末尾不额外添加换行：

```text
大写HTTP方法
不含域名和query的请求路径
规范化query
原始body字节的SHA256小写hex
```

规范化 query 的步骤：取全部键值对，包括重复参数；分别对键和值作 RFC3986 百分号编码；按**编码后的键**、再按**编码后的值**升序排列；以 `key=value` 形式用 `&` 拼接。空格是 `%20`，不能用 `+`；百分号编码使用大写 hex。没有 query 时第三行为空。

GET 没有 body 时，对空字节求 SHA256，不对 `{}` 求散列。POST 应先固定 UTF-8 JSON 字节，再用同一份字节签名和发送。

```text
signature = hex_lower(HMAC_SHA256(UTF8(SecretKey), UTF8(CanonicalRequest)))
```

SecretKey 按明文 UTF-8 字节使用，不做 Base64 解码。Secret 仅保存在商户服务端，不放入浏览器、移动客户端、URL 或日志。建议同步服务器时钟；默认允许偏差为前后 10 分钟，实际以开通配置为准，收到 `40112` 时校时后重新签名。

### 3.3 Python 签名示例 {#signing-example}

以下代码只生成签名和请求头，不发送请求。请求参数、body 与实际发出的内容必须一致。

```python
import hashlib
import hmac
import time
from urllib.parse import quote, urlsplit, parse_qsl

def canonical_query(pairs):
    encoded = [(quote(str(k), safe="-._~"), quote(str(v), safe="-._~"))
               for k, v in pairs]
    return "&".join(k + "=" + v for k, v in sorted(encoded))

def signed_headers(access_key, secret, merchant_code, method, url, body=b""):
    parts = urlsplit(url)
    query = canonical_query(parse_qsl(parts.query, keep_blank_values=True))
    canonical = "\n".join([
        method.upper(), parts.path, query, hashlib.sha256(body).hexdigest()
    ])
    signature = hmac.new(secret.encode("utf-8"), canonical.encode("utf-8"),
                         hashlib.sha256).hexdigest()
    return {
        "X-Access-Key-Id": access_key,
        "X-Merchant-Code": merchant_code,
        "X-Vcc-Timestamp": str(int(time.time())),
        "X-Vcc-Signature": signature,
    }

url = "https://api.example.com/open/v1/wallets"
headers = signed_headers("<ACCESS_KEY_ID>", "<SECRET_KEY>", "<MERCHANT_CODE>",
                         "GET", url)
```

## 4. 幂等、限流与结果恢复 {#idempotency}

写入前保存商户业务单号、稳定幂等键、HTTP 方法、路径、query 和**原始请求体字节**。每个新的业务动作使用独立 key；同一业务动作的重试继续使用原 key。新卡操作 key 最多 128 字符，不含控制字符，不以 `batch:` 开头。

| 情况 | 处理方式 |
| --- | --- |
| 已知 `operationId` | 查询原操作，不创建替代单 |
| POST 超时、断网、一般 409 或 5xx | 保留原 key 和原始请求，以原方法、路径、query、body 原样重放；每次 HTTP 重新生成时间戳和签名 |
| 同 key 修改产品、金额、昵称、quoteId 或 body 序列化 | 可能返回 `40904`，不能视为新业务单受理 |
| `PENDING` / `PROCESSING` | 有界轮询并退避，或等待通知；停止后续资金/状态动作 |
| `recoveryRequired=true` | 保留原单并联系支持核对，不自动释放预占或换 key 补单 |
| 明确 HTTP 409 且 `code="40931"` | 本次报价不匹配、失效或变化，未受理、未预占；重新报价和确认后提交新的请求，使用新业务幂等键 |

不能把任意 409 或未知结果都当成“未受理”。`40931` 的响应可能被缓存，不能在原 key 下换报价后直接重试。原已受理请求按原 key、原字节恢复，不要求旧报价仍有效。

HTTP 重放响应的默认缓存期为 24 小时，可按服务配置调整；**缓存到期不表示业务 key 可以重复用于新操作**。已经关联资金操作的 key 始终用于恢复该原单。

默认访问限额为每商户每分钟 100 次，实际额度以商户配置为准。遇到 HTTP 429 / `42902` 应减少并发并退避重试，不依赖固定的重试周期或必然存在的 `Retry-After` 头。轮询采用有上限的退避策略，不持续高频查询。

## 5. 快速开始 {#quick-start}

1. 查询主钱包和当前产品，确认目标账户及 USD 可用余额。
2. 用产品 ID 和初始入卡本金获取报价，向业务方展示本金、服务费、合计和有效期。
3. 记录原订单与请求后，在报价有效期内提交一次开卡。
4. 保存返回的 `operationId`，查询或等待通知，只有 `SUCCEEDED` 才保存操作返回的 `cardId`。
5. 用该 `cardId` 查询卡资料和卡余额，再接入交易历史与账本同步。

```http
GET {BASE}/open/v1/wallets
GET {BASE}/open/v1/card-products
GET {BASE}/open/v1/card-operations/quote?cardProductId=123&amountMinor=1000
POST {BASE}/open/v1/card-operations
GET {BASE}/open/v1/card-operations/{operationId}
GET {BASE}/open/v1/cards/{cardId}/balance
```

以上各请求分别签名；POST 另带幂等键和 JSON body。不要根据卡列表中“最新一张卡”猜测某次订单的发卡结果。

## 6. 接口速查 {#endpoint-index}

| 方法 | 完整路径 | 用途 |
| --- | --- | --- |
| GET | `/open/v1/health` | 健康检查，免签 |
| GET | `/open/v1/card-products` | 可用产品 |
| GET | `/open/v1/card-operations/quote` | 开卡报价 |
| POST | `/open/v1/card-operations` | 提交开卡 |
| GET | `/open/v1/card-operations/{operationId}` | 查询卡操作 |
| GET | `/open/v1/cards` | 卡列表 |
| GET | `/open/v1/cards/{cardId}` | 脱敏卡详情 |
| GET | `/open/v1/cards/{cardId}/sensitive` | 按需获取卡敏感资料 |
| GET | `/open/v1/cards/{cardId}/balance` | 卡内余额 |
| GET | `/open/v1/cards/{cardId}/recharge-quote` | 充值报价 |
| POST | `/open/v1/cards/{cardId}/recharge` | 提交充值 |
| POST | `/open/v1/cards/{cardId}/operations` | 冻结、解冻、销卡 |
| GET | `/open/v1/wallets` | 钱包列表 |
| GET | `/open/v1/wallets/{walletId}` | 钱包详情 |
| GET | `/open/v1/wallet-ledger` | 钱包账本 |
| GET | `/open/v1/transactions` | 卡交易历史 |
| GET | `/open/v1/card-groups` | 兼容卡组列表 |
| POST | `/open/v1/card-groups/{cardGroupId}/release` | 卡组资金划回主钱包 |
| GET | `/open/v1/card-issuance-tasks/{taskId}` | 兼容发卡任务查询 |
| GET | `/open/v1/merchant-fee-rules/effective` | 生效手续费规则与试算 |

## 7. 健康与产品 {#products}

### 7.1 健康检查 {#health}

`GET /open/v1/health`，无参数、免签。响应 data：

```json
{"status":"UP"}
```

`status` 为字符串服务状态。该检查用于连通性，不证明产品权限、发卡处理或资金链路已可用。

### 7.2 查询可用产品 {#card-products}

`GET /open/v1/card-products`，无参数，需签名。返回当前商户可见、已发布的产品数组。

```json
[{"id":123,"code":"USD_SINGLE_CARD","name":"USD单卡余额产品","currency":"USD","processor":"<服务标识>","fundingMode":"SINGLE_CARD_BALANCE","initialAmountRequired":true,"supportsSpendingConstraints":false,"minimumInitialAmountMinor":1000,"minimumRechargeAmountMinor":0,"rechargeFeeMinor":20,"published":true,"memberVisibility":"ALL_MEMBERS","merchantCode":"M_EXAMPLE"}]
```

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | integer(int64) | 产品 ID，作为 `cardProductId` |
| `code` | string | 产品编码 |
| `name` | string / null | 产品名称 |
| `currency` | string | 币种，当前单卡余额产品为 USD |
| `processor` | string | 服务标识，不透明值；不根据其名称推导能力 |
| `fundingMode` | string | `SINGLE_CARD_BALANCE` 表示单卡余额 |
| `initialAmountRequired` | boolean | 是否必须提供初始本金 |
| `supportsSpendingConstraints` | boolean | 当前为 false，不提供消费约束设置 |
| `minimumInitialAmountMinor` | integer(int64) | 最低初始本金，USD 分 |
| `minimumRechargeAmountMinor` | integer(int64) | 最低充值本金；即使返回 0，充值仍需大于 0 |
| `rechargeFeeMinor` | integer(int64) | 产品充值服务费参考值，确认时以当次报价为准 |
| `published` | boolean | 是否已发布 |
| `memberVisibility` | string | 产品可见范围标识 |
| `merchantCode` | string | 当前商户编码 |

示例金额和费用仅用于说明字段，不能作为固定费率。开卡来源由选定产品决定，请使用实际返回的产品 ID。

## 8. 开卡、充值与卡操作 {#operations}

### 8.1 开卡报价 {#create-quote}

`GET /open/v1/card-operations/quote`，需签名。

| Query 参数 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `cardProductId` | integer(int64) | 是 | 正整数产品 ID |
| `amountMinor` | integer(int64) | 是 | 正整数初始本金，USD 分，满足产品最低额 |

```http
GET {BASE}/open/v1/card-operations/quote?cardProductId=123&amountMinor=1000
```

响应 data 使用报价模型：

```json
{"amountMinor":1000,"feeMinor":20,"totalMinor":1020,"currency":"USD","minInitialAmountMinor":1000,"quoteId":"11111111-2222-4333-8444-555555555555","expiresAt":"2026-11-17T08:05:00Z"}
```

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `amountMinor` | integer(int64) | 本次本金 |
| `feeMinor` | integer(int64) | 本次服务费 |
| `totalMinor` | integer(int64) | 需确认的主钱包预占合计，即本金加服务费 |
| `currency` | string | USD |
| `minInitialAmountMinor` | integer(int64) | 本产品最低初始金额参考值 |
| `quoteId` | string | 本次报价 ID，最长 36 字符 |
| `expiresAt` | string(date-time) | 报价到期时间 |

报价有效 **300 秒**，仅对对应账户、产品、操作和本金有效。示例 quoteId 不可直接提交。主钱包可用余额需覆盖 `totalMinor`；金额、费用或产品变化时重新报价并确认。

### 8.2 提交开卡 {#create-card}

`POST /open/v1/card-operations`，需签名及 `Idempotency-Key`。

| JSON 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `cardProductId` | integer(int64) | 是 | 正整数产品 ID |
| `initialAmountMinor` | integer(int64) | 是 | 正整数初始本金，USD 分 |
| `displayName` | string / null | 否 | 卡片展示名，最多 128 字符；恢复请求保持原值 |
| `quoteId` | string | 是 | 对应本次申请的有效报价 ID，最长 36 字符 |

```http
POST {BASE}/open/v1/card-operations
Idempotency-Key: create-order-example-001
Content-Type: application/json

{"cardProductId":123,"initialAmountMinor":1000,"displayName":"采购卡","quoteId":"11111111-2222-4333-8444-555555555555"}
```

请求不支持 `memberUserId`、`channelCode`、卡组资金池、消费约束或账单地址。响应 data 为[卡操作模型](#operation-model)，初始可能为：

```json
{"operationId":123,"issuanceTaskId":456,"cardId":null,"status":"PENDING","stage":"READY","operationType":"CREATE","amountMinor":1000,"feeMinor":20,"currency":"USD","errorCode":null,"terminal":false,"recoveryRequired":false,"createdAt":"2026-11-17T08:00:00Z","updatedAt":"2026-11-17T08:00:00Z"}
```

保存 `operationId` 后查询原操作。受理时 `cardId` 为空不表示失败；不因暂未拿到卡 ID 而重复开卡。

### 8.3 查询卡操作 {#get-operation}

`GET /open/v1/card-operations/{operationId}`，需签名。路径参数 `operationId` 为正整数卡操作 ID，不是兼容任务 ID 或钱包操作 ID。

```http
GET {BASE}/open/v1/card-operations/123
```

响应 data 为卡操作模型。例如：

```json
{"operationId":123,"issuanceTaskId":456,"cardId":789,"status":"SUCCEEDED","stage":"SUCCEEDED","operationType":"CREATE","amountMinor":1000,"feeMinor":20,"currency":"USD","errorCode":"CARD_CREATED_AND_PAID","terminal":true,"recoveryRequired":false,"createdAt":"2026-11-17T08:00:00Z","updatedAt":"2026-11-17T08:03:00Z"}
```

### 8.4 卡操作模型与状态 {#operation-model}

开卡、充值、冻结、解冻、销卡及其查询共用该模型。

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `operationId` | integer(int64) | 原卡操作 ID |
| `issuanceTaskId` | integer(int64) / null | 关联发卡任务 ID，可能尚未取得或不适用 |
| `cardId` | integer(int64) / null | 关联卡账户 ID；开卡成功后使用该值 |
| `status` | string | `PENDING`、`PROCESSING`、`SUCCEEDED`、`FAILED` |
| `stage` | string | 当前处理阶段，见下表；不替代最终状态 |
| `operationType` | string | `CREATE`、`RECHARGE`、`FREEZE`、`UNFREEZE`、`CLOSE` |
| `amountMinor` | integer(int64) | 本次本金，生命周期动作可为 0 |
| `feeMinor` | integer(int64) | 本次服务费 |
| `currency` | string | 币种 |
| `errorCode` | string / null | 业务结果或核对标识；成功也可能有值，不以非空判断失败 |
| `terminal` | boolean | 是否已到 SUCCEEDED / FAILED 终态 |
| `recoveryRequired` | boolean | 是否需要进一步核对；为 true 时保留原单并联系支持 |
| `createdAt`、`updatedAt` | string(date-time) | 创建、最近更新时间 |

| `stage` 常见值 | 含义 |
| --- | --- |
| `READY`、`SUBMITTING` | 待处理或正在提交 |
| `ACCEPTED`、`PENDING` | 已受理或仍在处理 |
| `UNKNOWN`、`MANUAL_REVIEW` | 结果待确认或需要核对 |
| `SUCCEEDED`、`FAILED` | 对应完成或失败阶段 |

仅 `status=SUCCEEDED` 表示本次操作完成。失败后的资金返还以实际账本和钱包结果核对；未知结果不按超时自动退款。该查询模型不返回 `fundingState`，资金状态字段仅见[操作通知](#operation-webhook)。

### 8.5 充值报价 {#recharge-quote}

`GET /open/v1/cards/{cardId}/recharge-quote`，需签名。

| 参数 | 位置 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- | --- |
| `cardId` | path | string | 是 | 当前商户可见的卡账户 ID |
| `amountMinor` | query | integer(int64) | 是 | 正整数充值本金，USD 分 |

```http
GET {BASE}/open/v1/cards/789/recharge-quote?amountMinor=1000
```

响应 data 与[报价模型](#create-quote)相同，含 `quoteId/expiresAt/amountMinor/feeMinor/totalMinor/currency/minInitialAmountMinor`。保存本次充值报价；其中 `minInitialAmountMinor` 是兼容字段名，最低充值本金应结合产品的 `minimumRechargeAmountMinor` 及实际校验结果判断。

### 8.6 提交充值 {#recharge-card}

`POST /open/v1/cards/{cardId}/recharge`，需签名及独立稳定 `Idempotency-Key`。路径 `cardId` 同上。

| JSON 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `amountMinor` | integer(int64) | 是 | 正整数本金，USD 分 |
| `quoteId` | string | 是 | 本次有效充值报价，最长 36 字符 |

```http
POST {BASE}/open/v1/cards/789/recharge
Idempotency-Key: recharge-order-example-001
Content-Type: application/json

{"amountMinor":1000,"quoteId":"11111111-2222-4333-8444-555555555555"}
```

响应 data 为[卡操作模型](#operation-model)。按返回 `operationId` 确认 SUCCEEDED 后，再刷新主钱包、卡余额和相关账本；不能仅根据 POST 受理响应增加卡余额。恢复规则与开卡相同。

### 8.7 冻结、解冻、销卡 {#card-actions}

`POST /open/v1/cards/{cardId}/operations`，需签名及每个独立动作自己的 `Idempotency-Key`。

| JSON 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `operationType` | string | 是 | 严格大写 `FREEZE`、`UNFREEZE` 或 `CLOSE` |

```http
POST {BASE}/open/v1/cards/789/operations
Idempotency-Key: freeze-action-example-001
Content-Type: application/json

{"operationType":"FREEZE"}
```

响应 data 为[卡操作模型](#operation-model)。同一卡充值和状态变更互斥；前一操作未完成时不并发提交下一动作。不提供 `ACTIVATE` 操作。销卡成功后仍需查询卡的 `closeRefundStatus` 及钱包账本，不能直接把卡内余额记成已收到的回款。

## 9. 卡片查询与余额 {#cards}

### 9.1 卡列表 {#list-cards}

`GET /open/v1/cards`，需签名，按当前商户范围查询。

| Query 参数 | 类型 | 必填 | 默认 / 范围 |
| --- | --- | --- | --- |
| `page` | integer | 否 | 0，从 0 开始 |
| `size` | integer | 否 | 20，有效范围 1–100 |
| `cardGroupId` | integer(int64) | 否 | 省略查全部；0 查未归属卡组；正数需为默认关联账户可见卡组 |

```http
GET {BASE}/open/v1/cards?page=0&size=100
```

响应 data：

```json
{"total":1,"page":0,"size":100,"merchantCode":"M_EXAMPLE","items":[{"id":"789","displayName":"采购卡","panMasked":"**** **** **** 1234","panLast4":"1234","status":"ACTIVE","cardProductId":123,"cardProductName":"USD单卡余额产品","platformCardGroupId":null,"cardGroupName":null,"cardGroupCode":null,"cardGroupMemberOwned":false,"cardGroupLabel":"单卡余额","channelDisplayName":"<服务展示标识>","issuingEnvironment":null,"fundingMode":"SINGLE_CARD_BALANCE","supportsSpendingConstraints":false,"balanceMinor":1000,"balanceAsOf":"2026-11-17T08:03:00Z","balanceStale":false,"closeRefundStatus":null,"pendingAction":null,"merchantCode":"M_EXAMPLE"}]}
```

`total` 为当前匹配总数，`page` 为页号，`size` 为有效请求页长，`items` 为卡片数组。导入全部历史卡时省略 `cardGroupId`。本接口不支持账户、状态或渠道查询参数。

### 9.2 脱敏卡详情 {#get-card}

`GET /open/v1/cards/{cardId}`，需签名。`cardId` 为卡列表返回的字符串 `id`，或成功卡操作的 `cardId`。

```http
GET {BASE}/open/v1/cards/789
```

响应 data 是单个卡对象，字段与列表项相同：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | string | 卡账户 ID |
| `displayName` | string / null | 卡片展示名 |
| `panMasked`、`panLast4` | string / null | 掩码卡号及后四位 |
| `status` | string | 卡状态，如 ACTIVE / FROZEN / CLOSED；与卡操作状态分开处理 |
| `cardProductId` | integer(int64) | 产品 ID |
| `cardProductName` | string / null | 产品名称 |
| `platformCardGroupId` | integer(int64) / null | 关联卡组 ID，无卡组时可空 |
| `cardGroupName`、`cardGroupCode` | string / null | 卡组名称和编码 |
| `cardGroupMemberOwned` | boolean | 是否为所属账户自建卡组 |
| `cardGroupLabel` | string | 用于展示的卡组/资金模式标签 |
| `channelCode` | string，可不出现 | 不透明服务标识 |
| `channelDisplayName` | string | 不透明服务展示标识 |
| `issuingEnvironment` | string / null | 环境标识，按实际返回处理 |
| `merchantCode` | string | 商户编码 |
| `fundingMode` | string，适用产品返回 | 单卡余额为 SINGLE_CARD_BALANCE |
| `supportsSpendingConstraints` | boolean，适用产品返回 | 是否支持消费约束 |
| `balanceMinor` | integer(int64) / null，适用产品返回 | 最近确认卡余额，可能未知 |
| `balanceAsOf` | string(date-time) / null，适用产品返回 | 最近确认时间 |
| `balanceStale` | boolean，适用产品返回 | 余额是否已过期、需刷新 |
| `closeRefundStatus` | string / null，适用产品返回 | AWAITING_CONFIRMATION 待核对回款；RETURNED 已登记回款 |
| `pendingAction` | string / null，适用产品返回 | 正在处理的动作，例如 RECHARGE / FREEZE / UNFREEZE / CLOSE |

卡列表和详情不承诺返回 `currency`、`memberUserId`；单卡余额的币种从余额接口取得。列表中的余额是快照，需要当前余额时调用下述余额接口。

### 9.3 卡敏感资料 {#sensitive-card}

`GET /open/v1/cards/{cardId}/sensitive`，需签名，并受卡归属、产品支持范围和可用开卡资料限制。

| 参数 | 位置 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- | --- |
| `cardId` | path | string | 是 | 卡账户 ID |
| `memberUserId` | query | integer(int64) | 否 | 如传入，须与卡片所属关联账户一致 |

```http
GET {BASE}/open/v1/cards/789/sensitive
```

响应 data 字段如下。敏感值使用占位文本，不能直接用于支付。

```json
{"cardId":"789","channelCode":"<服务标识>","pan":"<完整卡号>","cvv":"<安全码>","expMonth":12,"expYear":2030,"merchantCode":"M_EXAMPLE"}
```

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `cardId` | string | 卡账户 ID |
| `channelCode` | string | 不透明服务标识 |
| `pan`、`cvv` | string | 完整卡号与安全码 |
| `expMonth`、`expYear` | integer | 到期月、四位年份 |
| `merchantCode` | string | 商户编码 |

响应包含 `Cache-Control: no-store`、`Pragma: no-cache`。仅在业务必需时获取，限制展示权限，不记入日志或普通数据导入，也不持久缓存。历史卡或不支持该能力的卡可能拒绝读取，不以列表中存在该卡推定敏感资料可用。

### 9.4 查询单卡余额 {#card-balance}

`GET /open/v1/cards/{cardId}/balance`，需签名，适用于已开通的单卡余额产品。

```http
GET {BASE}/open/v1/cards/789/balance
```

响应 data：

```json
{"cardId":789,"balanceMinor":1000,"currency":"USD","asOf":"2026-11-17T08:03:00Z","stale":false,"errorCode":null}
```

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `cardId` | integer(int64) | 卡账户 ID |
| `balanceMinor` | integer(int64) / null | 卡内余额，最小货币单位；null 表示未知，不能转为 0 |
| `currency` | string | 币种 |
| `asOf` | string(date-time) / null | 最近可靠余额确认时间 |
| `stale` | boolean | true 表示刷新失败或快照已过期，可能保留旧余额 |
| `errorCode` | string / null | 余额刷新或核对标识 |

展示余额时同时保留时效信息，不把旧余额当成刚确认的可用资金。

## 10. 钱包与账本 {#wallets}

### 10.1 钱包列表 {#list-wallets}

`GET /open/v1/wallets`，需签名。

| Query 参数 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `memberUserId` | integer(int64) | 否 | 指定则查该关联账户；省略查商户可见钱包 |

```http
GET {BASE}/open/v1/wallets?memberUserId=456
```

响应 data 为钱包数组：

```json
[{"walletId":"54","currency":"USD","availableAmountMinor":100000,"availableAmount":"1000.00","cardGroupId":0,"walletRole":"MAIN","cardGroupName":null,"merchantCode":"M_EXAMPLE"}]
```

### 10.2 钱包详情 {#get-wallet}

`GET /open/v1/wallets/{walletId}`，需签名。路径 `walletId` 为钱包列表返回的字符串 ID。

```http
GET {BASE}/open/v1/wallets/54
```

响应 data 为单个钱包对象：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `walletId` | string | 钱包 ID |
| `currency` | string | 币种 |
| `availableAmountMinor` | integer(int64) | 可用余额，最小货币单位 |
| `availableAmount` | string | 可用余额展示值，两位小数 |
| `cardGroupId` | integer(int64) | 主钱包为 0，卡组钱包为对应卡组 ID |
| `walletRole` | string | MAIN / CARD_GROUP |
| `cardGroupName` | string / null | 卡组名称，主钱包可空 |
| `merchantCode` | string | 商户编码 |

选定实际开卡关联账户的 USD MAIN 钱包，再比较可用余额与报价合计；多个账户存在时不任意取第一个 MAIN。接口不提供冻结余额或总资产字段，卡组钱包也不等于单卡余额。

### 10.3 钱包账本分页 {#wallet-ledger}

`GET /open/v1/wallet-ledger`，需签名。合并所选关联账户的 **USD 主钱包和所有卡组钱包** 账本。

| Query 参数 | 类型 | 必填 | 默认 / 范围 |
| --- | --- | --- | --- |
| `memberUserId` | integer(int64) | 否 | 默认关联账户 |
| `page` | integer | 否 | 0，从 0 开始 |
| `size` | integer | 否 | 20，上限 200；建议固定 100，不超过上限 |
| `sort` | string | 否 | desc；支持 asc / desc |

```http
GET {BASE}/open/v1/wallet-ledger?memberUserId=456&page=0&size=100&sort=asc
```

响应 data：

```json
{"total":1,"page":0,"size":1,"sort":"asc","merchantCode":"M_EXAMPLE","items":[{"id":"901","occurredAt":"2026-11-17T08:00:00Z","amountMinor":-1020,"amount":"10.20","direction":"DEBIT","balanceAfterMinor":98980,"balanceAfter":"989.80","entryType":"ISSUING_RESERVE","referenceType":"ISSUING_OPERATION","referenceId":"123","businessRef":"issuing:123","walletAccountId":"54","cardGroupId":"0"}]}
```

`data.size` 是**本页实际条数**，不是请求页长；下一页继续使用原来的 `size=100`，只递增 `page`。`total` 为匹配总数，`items` 为账本记录。

| 记录字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | string | 账本 ID，导入去重键 |
| `occurredAt` | string(date-time) | 业务发生时间 |
| `amountMinor` | integer(int64) | 带符号变动金额 |
| `amount` | string | 绝对值展示金额 |
| `direction` | string | CREDIT / DEBIT；金额 0 归 CREDIT |
| `balanceAfterMinor`、`balanceAfter` | integer(int64)、string | 本条所属钱包变动后的余额 |
| `entryType` | string / null | 记账类型 |
| `referenceType`、`referenceId` | string / null | 业务关联类型和 ID |
| `businessRef` | string / null | 业务引用 |
| `walletAccountId` | string | 所属钱包 ID |
| `cardGroupId` | string | 本条卡组 ID；注意本接口为字符串 |

本接口没有 `walletId/cardId/from/to/cursor` 过滤参数。只展示主钱包时，根据 `walletAccountId` 在结果中区分；`balanceAfterMinor` 属于该条钱包，不能把多个钱包当作一条余额序列。

### 10.4 开卡与充值账本核对 {#ledger-reconciliation}

| `entryType` | 含义 |
| --- | --- |
| `ISSUING_RESERVE` | 一次预占本金和服务费合计，负向变动 |
| `ISSUING_FEE_COMMITTED` | 金额为 0 的费用确认记录，不是再次扣费，也不表示零手续费 |
| `ISSUING_RELEASE` | 失败后实际释放预占的正向记录 |
| `ISSUING_CLOSE_RETURN` | 已确认销卡回款的记录，不根据卡已关闭自行推算 |

当 `referenceType=ISSUING_OPERATION` 时，`referenceId` 可关联原 `operationId`。例如本金 1000、服务费 20 的申请，预占 -1020；成功确认时不能在商户业务系统重复扣 20。卡消费与卡内退款通过交易和卡余额核对，不应期待每一笔都出现在主钱包账本。

## 11. 交易历史与数据同步 {#transactions}

### 11.1 交易分页查询 {#list-transactions}

`GET /open/v1/transactions`，需签名。读取当前商户及所选账户范围内已经可见的交易记录。

| Query 参数 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `memberUserId` | integer(int64) | 否 | 默认关联账户；指定时校验商户归属 |
| `cardId` | integer(int64) | 否 | 平台卡账户 ID，不能覆盖账户与商户范围 |
| `txnType` | string | 否 | 交易类型过滤 |
| `from`、`to` | string(date-time) | 否 | 按发生时间过滤，带时区偏移，两端均包含 |
| `page` | integer | 否 | 默认 0，从 0 开始 |
| `size` | integer | 否 | 默认 20，有效范围 1–200，建议 100 |

```http
GET {BASE}/open/v1/transactions?memberUserId=456&cardId=789&from=2026-11-01T00:00:00Z&to=2026-11-17T08:00:00Z&page=0&size=100
```

日期和其他参数按标准 URL 编码后发送并参与签名。结果按发生时间倒序、再按 ID 倒序排列。接口没有 `status/channelCode/updatedSince/cursor` 参数；需要渠道筛选时根据返回的不透明标识在本地处理。

响应 data：

```json
{"total":1,"page":0,"pageSize":100,"itemCount":1,"merchantCode":"M_EXAMPLE","items":[{"id":"902","memberUserId":"456","cardAccountId":"789","txnType":"PURCHASE","status":"POSTED","amountMinor":-500,"amount":"5.00","currency":"USD","occurredAt":"2026-11-16T09:00:00Z","merchantReference":null,"channelReference":"example-transaction-ref","narrative":null,"merchantDescription":"Example Store","channelStatus":null,"channelDetailedStatus":null,"feeTotalMinorSnapshot":null,"feeRuleSourceSnapshot":null,"feeRuleIdSnapshot":null,"feeRuleTableSnapshot":null,"feeSummarySnapshot":null,"feeSnapshotAvailable":false,"feeEstimateAvailable":false,"merchantCode":"M_EXAMPLE"}]}
```

| 分页字段 | 类型 | 说明 |
| --- | --- | --- |
| `total` | integer(int64) | 匹配总数 |
| `page` | integer | 当前页号 |
| `pageSize` | integer | 有效请求页长 |
| `itemCount` | integer | 本页实际条数 |
| `items` | array | 交易对象数组 |
| `merchantCode` | string | 商户编码 |

### 11.2 交易记录字段 {#transaction-model}

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | string | 平台交易记录 ID，用于去重及状态更新 |
| `memberUserId` | string | 关联账户 ID |
| `cardAccountId` | string / null | 卡账户 ID |
| `txnType`、`status` | string | 交易类型及状态，按实际值处理，不把所有记录当成最终消费 |
| `amountMinor` | integer(int64) | 带符号金额 |
| `amount` | string | 绝对值展示金额 |
| `currency` | string | 币种 |
| `occurredAt` | string(date-time) | 发生时间 |
| `merchantReference`、`channelReference` | string / null | 商户引用与交易服务引用 |
| `merchantDescription`、`narrative` | string / null | 消费商户描述与摘要 |
| `channelCode` | string / null | 不透明服务标识 |
| `channelStatus`、`channelDetailedStatus` | string / null | 服务状态与细状态，按原值保存 |
| `merchantCode` | string | 商户编码 |
| `feeSnapshotAvailable`、`feeEstimateAvailable` | boolean | 是否提供手续费快照或当前估算 |
| `feeTotalMinorSnapshot` | integer(int64) / null | 已记录的手续费最小单位金额 |
| `feeRuleSourceSnapshot`、`feeRuleIdSnapshot`、`feeRuleTableSnapshot` | string / null | 不透明规则关联标识，不用于商户程序判断收费方式 |
| `feeSummarySnapshot` | string / null | 快照说明 |
| `feeSnapshotFormatted` | string，条件返回 | 快照展示金额 |

兼容交易可能按数据情况额外返回下列字段；不出现时不按 0 补齐：

| 条件字段 | 类型 | 说明 |
| --- | --- | --- |
| `feeSnapshotMinor`、`feeEstimateMinor` | integer(int64) | 快照、估算最小单位金额 |
| `feeSnapshot`、`feeEstimate` | string | 快照、估算展示金额 |
| `feeSnapshotSummary`、`feeSnapshotNote` | string / null | 历史快照说明 |
| `feeSnapshotRuleSource`、`feeRuleSource` | string / null | 不透明规则来源标识 |
| `feeSummary`、`feeEstimateNote`、`feeEstimateVsSnapshotNote` | string / null | 当前估算与差异说明 |

当前单卡余额产品的交易可能没有可用消费手续费快照或估算，可用标记为 false **不表示零手续费**。开卡与充值服务费按当次报价和原操作的 `feeMinor` 核对，不套用交易估算。

交易类型和服务状态可扩展，消费、退款、冲正和授权应分别处理。REST 可以返回归属匹配且已可见的授权记录，但交易通知不提供实时授权事件。

### 11.3 首次历史导入 {#history-import}

1. 确认商户、关联账户和卡 ID 范围。卡列表分页导入时省略 `cardGroupId` 以覆盖全部卡。
2. 固定本轮交易截止时间 `T0`，按日或周切分 `from/to` 时间窗，固定 `size=100`，每窗从第 0 页向后读取。
3. 按平台交易 `id` 做 upsert；已存在 ID 的状态变化更新原行，不重复增加交易。
4. 两端包含边界，窗口交接处需要去重；扫描期间数据可能增加，完成后重复核对重叠窗口。
5. 钱包账本首次使用 `sort=asc`、固定 `size=100` 逐页导入，以账本 ID 去重，并按 `walletAccountId` 分账。

这些查询提供的是当前商户及账户可见记录，不自动补齐全部历史。历史不在当前归属范围或尚不可见时可能查不到；空列表不证明没有交易。全量迁移前请确认服务开通范围、历史覆盖日期和账户映射。

### 11.4 增量同步与余额对账 {#incremental-sync}

通过 Webhook 触发查询，并定期重扫最近交易窗口。`from/to` 按发生时间而不是更新时间筛选，较早交易的后续更新及迟到记录可能落在近期窗口之外，因此还需定期补扫较早窗口或全量核对。

账本没有时间或 ID 游标查询，增量可用 `sort=desc` 读取最近页，按 ID 去重并保留重叠；不能把“遇到第一个已见 ID 就永久停止”作为严格完整性保证。卡、交易和账本均为分页查询，扫描期间不承诺强一致快照。

对账按钱包分别核对可用余额与相关预占、释放、入账记录，再核对卡余额及卡交易。余额查询和历史扫描可能不是同一观察时点。不要通过简单相加所有授权、消费和退款来替代当前卡余额查询。

## 12. 兼容查询、卡组与费用 {#compatibility}

本章用于已有资源。新单卡余额开卡和充值使用第 8 章流程，不先创建卡组或通过卡组划款为卡充值。

### 12.1 卡组列表 {#list-card-groups}

`GET /open/v1/card-groups`，需签名。可选 query `memberUserId`（integer(int64)），省略时使用默认关联账户。返回有效的账户自建卡组数组，不含余额。

```http
GET {BASE}/open/v1/card-groups?memberUserId=456
```

```json
[{"id":9,"code":"GROUP_EXAMPLE","name":"采购组","memberOwned":true,"status":"ACTIVE","merchantCode":"M_EXAMPLE"}]
```

字段：`id` 为 integer(int64)；`code/name/status/merchantCode` 为 string；`memberOwned` 为 boolean。余额通过钱包列表的 CARD_GROUP 对象查询。

### 12.2 卡组资金划回主钱包 {#release-card-group}

`POST /open/v1/card-groups/{cardGroupId}/release`，需签名及稳定 `Idempotency-Key`。

| 参数 | 位置 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- | --- |
| `cardGroupId` | path | integer(int64) | 是 | 所选关联账户有效自建卡组 ID |
| `memberUserId` | query | integer(int64) | 否 | 省略时默认关联账户 |
| `currency` | JSON | string | 是 | USD |
| `amount` | JSON | string | 是 | 正十进制金额；建议按 USD 两位小数发送，超出分精度时四舍五入，换算后须大于 0 |

```http
POST {BASE}/open/v1/card-groups/9/release?memberUserId=456
Idempotency-Key: release-order-example-001
Content-Type: application/json

{"currency":"USD","amount":"50.00"}
```

响应 data 为钱包操作，不是卡操作：

```json
{"id":"903","reference":"wo_903","status":"SUCCEEDED","type":"CARD_GROUP_RELEASE","currency":"USD","amountMinor":5000,"amount":"50.00","merchantCode":"M_EXAMPLE"}
```

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id`、`reference` | string | 钱包操作 ID 及业务引用 |
| `status` | string | 钱包操作结果，按实际状态处理 |
| `type` | string | CARD_GROUP_RELEASE |
| `currency` | string | USD |
| `amountMinor`、`amount` | integer(int64)、string | 申请金额及展示值 |
| `failureReason` | string，条件返回 | 失败原因 |
| `reversesOperationId` | string，条件返回 | 关联撤销操作，存在时返回 |
| `feeDisplayMode` | string，条件返回 | 费用展示方式标识 |
| `feeRateBps` | integer，条件返回 | 费率基点，100 基点为 1% |
| `feeFlatMinor`、`feeTotalMinor` | integer(int64)，条件返回 | 固定费及费用合计 |
| `feeSnapshotMissing` | boolean，条件返回 | 是否缺少历史费用快照 |
| `feeSummary`、`feeTotalDisplay` | string，条件返回 | 费用说明及展示金额 |
| `feeRuleSource` | string，条件返回 | 不透明规则来源标识 |
| `merchantCode` | string | 商户编码 |

划出仍受卡组归属、余额、可用性和风险校验限制，不保证所有余额均可划出。该操作不是卡充值，也不是销卡回款。其 `id` 不可用卡操作查询接口查询；结果不明时按原 key 和原字节恢复，并核对两个钱包和账本。

### 12.3 兼容发卡任务查询 {#historical-task}

`GET /open/v1/card-issuance-tasks/{taskId}`，需签名。`taskId` 是此前取得的发卡任务 ID。

```http
GET {BASE}/open/v1/card-issuance-tasks/456
```

```json
{"id":"456","status":"SUCCEEDED","channelRequestId":null,"merchantCode":"M_EXAMPLE"}
```

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | string | 任务 ID |
| `status` | string | 任务状态 |
| `channelRequestId` | string / null | 不透明请求引用 |
| `merchantCode` | string | 商户编码 |

此模型不含 `cardId` 或错误详情。新流程应保存并查询 `operationId`，不能用本接口代替新卡操作查询。

### 12.4 查询生效手续费规则及试算 {#effective-fees}

`GET /open/v1/merchant-fee-rules/effective`，需签名。

| Query 参数 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `feeType` | string | 是 | CARD_ISSUANCE / TRANSACTION / REFUND / MANUAL_TOP_UP / WITHDRAWAL |
| `baseAmountMinor` | integer(int64) | 否 | 试算基数，最小货币单位；省略按 0 处理 |

```http
GET {BASE}/open/v1/merchant-fee-rules/effective?feeType=TRANSACTION&baseAmountMinor=1000
```

```json
{"feeType":"TRANSACTION","source":"MERCHANT","ruleId":"12","ruleTable":"<规则标识>","calculationMode":"FIXED","currency":"USD","fixedFeeMinor":"20","minFeeMinor":"0","ratePercent":null,"tierRef":null,"appliesTo":null,"baseAmountMinor":"1000","foreignTransaction":true,"quotedFeeMinor":"20","summary":"固定服务费","merchantCode":"M_EXAMPLE"}
```

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `feeType` | string | 本次费用类型 |
| `source` | string | 规则来源标识，例如 MERCHANT / GLOBAL |
| `ruleId`、`ruleTable` | string | 不透明兼容规则标识，不解析其内容 |
| `calculationMode` | string | FIXED / PERCENT / FIXED_PLUS_PERCENT |
| `currency` | string | 规则币种 |
| `fixedFeeMinor`、`minFeeMinor` | string | 最小货币单位整数的字符串 |
| `ratePercent` | string / null | 百分比字符串 |
| `tierRef`、`appliesTo` | string / null | 规则适用及档位标识 |
| `baseAmountMinor`、`quotedFeeMinor` | string | 试算基数及费用，均为最小单位整数的字符串 |
| `foreignTransaction` | boolean | 本次试算属性；不是本接口请求参数 |
| `summary` | string | 规则摘要 |
| `merchantCode` | string | 商户编码 |

开卡和充值的实际收费确认以对应报价及原操作为准。本兼容试算不能等同于单卡消费的实际服务费；没有快照或试算结果也不能推断零手续费。

## 13. Webhook 通知 {#webhooks}

### 13.1 配置与接收规则 {#webhook-delivery}

使用接入服务提供的商户通知配置流程，设置 HTTPS 接收地址、订阅事件及独立 Webhook Secret。事件只有在已开通适用能力时产生；订阅某事件不代表全部产品都会触发它。

| `eventType` | 用途 |
| --- | --- |
| `merchant.webhook.issuing.operation.updated` | 本接口发起的卡操作更新 |
| `merchant.webhook.transaction.updated` | 交易清算、入账等后续更新，不含实时授权 |
| `merchant.webhook.wallet.deposit.posted` | 已开通数币自动充值的到账通知 |
| `merchant.webhook.wallet.card_binding.verification_code` | 适用卡产品的绑卡验证码 |

收到通知后先验签，再核对事件 ID、可靠保存或入队，成功后返回 HTTP 2xx。通知可能重试、重复、乱序；以 `eventId` 去重，不用收到时间覆盖较新业务状态。没有承诺固定的重试次数或时间表。开卡和充值应使用通知中的 `operationId` 查询最新结果。

### 13.2 通知请求头与签名 {#webhook-signature}

| 请求头 | 说明 |
| --- | --- |
| `Content-Type` | application/json;charset=utf-8 |
| `X-Vcc-Timestamp` | Unix 毫秒字符串 |
| `X-Vcc-Nonce` | 32 位小写十六进制随机值 |
| `X-Vcc-Event-Id` | 与 JSON `eventId` 一致的事件 ID |
| `X-Vcc-Signature` | 小写 hex HMAC-SHA256 |

用独立 Webhook Secret 对下面内容验签，**没有分隔符**：

```text
payload = UTF8(timestamp + nonce) || 原始body字节
signature = hex_lower(HMAC_SHA256(UTF8(WebhookSecret), payload))
```

通知协议与请求四行签名不同。不得先解析 JSON、重排或重新序列化后再验签。商户应限制可接受时间偏差、恒时比较签名，并校验头部事件 ID 与 body 一致；在保证验签的前提下，持久去重处理可避免重复业务动作。

```python
import hashlib
import hmac
import json
import re
import time

def verify_notification(secret, headers, raw_body, allowed_skew_ms):
    # headers 由 HTTP 框架转换为小写键；raw_body 为未修改的 bytes。
    stamp = headers.get("x-vcc-timestamp", "")
    nonce = headers.get("x-vcc-nonce", "")
    signature = headers.get("x-vcc-signature", "")
    if not re.fullmatch(r"[0-9]+", stamp):
        return None
    if not re.fullmatch(r"[0-9a-f]{32}", nonce):
        return None
    if not re.fullmatch(r"[0-9a-f]{64}", signature):
        return None
    if abs(int(time.time() * 1000) - int(stamp)) > allowed_skew_ms:
        return None
    payload = (stamp + nonce).encode("utf-8") + raw_body
    expected = hmac.new(secret.encode("utf-8"), payload, hashlib.sha256).hexdigest()
    if not hmac.compare_digest(expected, signature):
        return None
    try:
        event = json.loads(raw_body)
    except (ValueError, UnicodeDecodeError):
        return None
    if not isinstance(event, dict) or not event.get("eventId"):
        return None
    if event["eventId"] != headers.get("x-vcc-event-id"):
        return None
    return event  # 接着按 eventId 持久去重；可靠入队后再返回 2xx。
```

`allowed_skew_ms` 由商户接收策略指定。该函数不替代持久去重；返回事件后还需校验 `eventType`、业务资源归属及字段类型。验证码不记入普通日志。

### 13.3 通知公共信封 {#webhook-envelope}

```json
{"eventType":"merchant.webhook.issuing.operation.updated","eventId":"event-example-001","occurredAt":"2026-11-17T08:03:00Z","merchantCode":"M_EXAMPLE","data":{}}
```

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `eventType` | string | 事件类型 |
| `eventId` | string | 事件去重 ID，与请求头一致 |
| `occurredAt` | string(date-time) | 事件发生时间 |
| `merchantCode` | string / null | 商户编码，部分事件可空；不能作为唯一验真依据 |
| `data` | object | 事件数据，见下文 |

### 13.4 卡操作更新通知 {#operation-webhook}

事件：`merchant.webhook.issuing.operation.updated`。

```json
{"operationId":123,"issuanceTaskId":456,"cardId":789,"operationType":"CREATE","status":"SUCCEEDED","stage":"SUCCEEDED","amountMinor":1000,"feeMinor":20,"currency":"USD","fundingState":"COMMITTED","errorCode":"CARD_CREATED_AND_PAID","terminal":true,"recoveryRequired":false,"createdAt":"2026-11-17T08:00:00Z","updatedAt":"2026-11-17T08:03:00Z"}
```

`data` 包含[卡操作模型](#operation-model)字段，并增加 `fundingState`（string）：

| 值 | 说明 |
| --- | --- |
| `HELD` | 原本金及服务费处于预占状态 |
| `COMMITTED` | 原预占已确认 |
| `RELEASED` | 原预占已释放 |
| `NONE` | 本操作无对应预占 |

操作查询接口不返回 `fundingState`。通知用于推动查询；`PENDING`、待核对或暂时没有通知都不表示已退款。

### 13.5 交易更新通知 {#transaction-webhook}

事件：`merchant.webhook.transaction.updated`。响应 `data` 示例：

```json
{"transactionId":"txn_example-transaction-ref","cardAccountId":789,"txnType":"PURCHASE","status":"POSTED","amountMinor":-500,"currency":"USD","occurredAt":"2026-11-16T09:00:00Z","merchantReference":null,"channelReference":"example-transaction-ref"}
```

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `transactionId` | string | 通知交易标识，形式为 `txn_{channelReference}` |
| `cardAccountId` | integer(int64) / null | 卡账户 ID |
| `txnType`、`status` | string | 交易类型和状态，可扩展 |
| `amountMinor` | integer(int64) | 带符号最小单位金额 |
| `currency` | string | 币种 |
| `occurredAt` | string(date-time) | 业务发生时间 |
| `merchantReference` | string / null | 商户引用 |
| `channelReference` | string | 交易服务引用 |

通知 `transactionId` 与 REST 交易记录 `id` **不是同一套 ID**。分别保存通知标识、渠道引用和卡归属，收到通知后补查 REST，再用 REST `id` 更新本地交易。通知不包含实时授权；不能只靠此事件实现全量或实时授权查询。

### 13.6 数币充值到账通知 {#deposit-webhook}

事件：`merchant.webhook.wallet.deposit.posted`，仅适用于已开通的自动充值到账，不是所有钱包变动的通用通知。

```json
{"depositId":1001,"txHash":"<交易哈希>","chainId":1,"fromAddress":"<转出地址>","toAddress":"<转入地址>","amountMinor":10000,"amount":"100.00","currency":"USD","amountRaw":"100","payCurrencyId":1,"payRechargeId":2001,"channelCode":null,"walletOperationId":"904","walletId":"54","occurredAt":"2026-11-17T07:00:00Z"}
```

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `depositId`、`payRechargeId` | integer(int64) | 到账记录和充值引用 |
| `txHash` | string / null | 交易哈希 |
| `chainId`、`payCurrencyId` | integer | 网络及支付币种标识，以实际值为准 |
| `chainName`、`chainSymbol` | string，条件返回 | 网络名称及符号 |
| `fromAddress`、`toAddress` | string / null | 转出、转入地址 |
| `amountMinor`、`amount` | integer(int64)、string | 钱包入账金额及展示值 |
| `currency` | string | 钱包入账币种 USD |
| `amountRaw` | string / null | 原始资产金额字符串，不替代钱包入账金额 |
| `channelCode` | string / null | 不透明服务标识 |
| `walletOperationId` | string | 钱包入账操作 ID |
| `walletId` | string，条件返回 | 钱包 ID |
| `occurredAt` | string(date-time) | 到账时间 |

示例网络和币种 ID 仅说明类型，不构成支持列表。收到事件后核对目标钱包及账本，不把重复通知再次记为入账。

### 13.7 绑卡验证码通知 {#binding-code-webhook}

事件：`merchant.webhook.wallet.card_binding.verification_code`，只用于适用并已开通的卡产品，不承诺所有新卡均会触发。

```json
{"memberUserId":456,"inboundId":1002,"walletId":54,"cardAccountId":789,"bindingCode":"000000","expiresAt":"2026-11-17T08:05:00Z","channelCode":"<服务标识>"}
```

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `memberUserId` | integer(int64) | 关联账户 ID |
| `inboundId` | integer(int64) | 本次通知关联记录 ID |
| `walletId` | integer(int64)，条件返回 | 钱包 ID；注意本事件为数值，到账事件中为字符串 |
| `cardAccountId` | integer(int64)，条件返回 | 关联卡账户 ID |
| `bindingCode` | string | 绑卡验证码，示例为合成值 |
| `expiresAt` | string(date-time) | 验证码到期时间 |
| `channelCode` | string | 不透明服务标识 |

保持实际字段名 `bindingCode`；不要改用其他名称解析。仅向有权限的业务对象展示，在到期前使用，不写入普通日志或长期存储。

## 14. 错误码与排查 {#errors}

根据 HTTP 状态和字符串 `code` 一起处理。以下为发卡接口常用错误；`message` 可能携带本次请求的具体说明。

| HTTP | code | 含义与建议 |
| --- | --- | --- |
| 400 | `40001` | 参数校验失败；检查类型、金额、时间、未知字段及必填项 |
| 400 | `40003` | 缺少 Idempotency-Key；使用原业务动作的稳定 key |
| 400 | `40012` | 时间戳格式无效；API 请求使用 Unix 秒 |
| 400 | `40020` | 开卡资格未满足；按接入要求完善资格 |
| 400 | `40021` | 产品不可用；重新查询可用产品，不替代未知原单 |
| 400 | `40030` | 钱包余额不足；核对目标主钱包可用余额与报价总额 |
| 400 | `40031` | 费用规则与操作币种不一致；联系支持 |
| 400 | `40032` / `40033` | 手续费规则未配置或无效；联系支持 |
| 401 | `40110` | 凭证无效或已吊销；核对凭证状态和环境 |
| 401 | `40111` | 签名不匹配；核对原始 body、query 编码排序、路径和 Secret |
| 401 | `40112` | 超出时间窗口；校时后重新签名 |
| 401 | `40113` | 接口未开放或路径不允许；使用本文列明的方法与路径 |
| 403 | `40301` | 归属或权限不符；核对商户、账户和资源 ID |
| 403 | `40302` | 风险规则拒绝；联系支持核对，不盲目换 key |
| 403 | `40314` | 凭证类型不符；使用已开通的 ISSUING 凭证 |
| 404 | `40401` | 资源不存在或不在可见范围 |
| 404 | `40402` | 卡片不存在或不可见 |
| 404 | `40403` | 钱包或钱包资源不存在 |
| 409 | `40901` | 资源冲突或同请求仍在处理中；查询原资源并退避 |
| 409 | `40904` | 同幂等键请求不一致；检查原方法、路径、query 和 body 字节 |
| 409 | `40905` | 卡状态不允许当前操作；先确认原操作与卡状态 |
| 409 | `40931` | 本次报价未通过、未受理；重新报价确认后作为新请求提交 |
| 429 | `42902` | 请求频繁；减少并发并退避 |
| 500 / 503 | `50001` | 服务暂时异常；写请求先恢复原单，不换 key 重建 |

签名失败时优先检查：GET 是否散列空字节；请求路径是否包含 `/open/v1`；query 是否编码后排序；body 是否在签名后被重新序列化；时间戳是否误用了毫秒；Secret 是否错误解码。

历史数据为空时依次核对环境与凭证、默认或指定账户、卡归属、时间范围和历史覆盖。余额或手续费缺省时保留“未知”状态，不补成零。联系支持时提供商户编码、请求时间、`traceId`、资源或原操作 ID，不发送 Secret、完整卡号、安全码或验证码。

## 15. 接入检查清单 {#integration-checklist}

- 服务端正确保存凭证，并通过只读查询验证签名。
- 开卡前确认唯一关联账户、实际产品、本金、服务费和主钱包余额。
- 业务请求先持久保存稳定 key 和原始字节，支持未知结果恢复。
- 异步操作仅按 SUCCEEDED 进入后续流程，待核对状态可转人工支持。
- 卡余额展示处理 null、stale 和观察时间，销卡与回款分别核对。
- 历史导入按 ID 更新，处理窗口边界、重复、迟到和较早交易的状态变化。
- 钱包账本按钱包区分，正确处理预占、零金额确认记录及实际释放。
- Webhook 使用独立 Secret、原始 body 验签、事件 ID 去重并允许乱序。
- 敏感卡资料和验证码不进入日志、普通导入和持久缓存。
- 正式使用前按已开通范围完成业务验收，保留定期查询和对账机制。
