# Foregate Seamless Wallet 商户接入与接口文档

版本：1.2｜更新：2026-09-30



本文按实际接入顺序说明双方交付物、接口、鉴权、登录、资金流、联调与上线。适用于 商户 及后续 Seamless 商户；第 3.4 节列出的 Foregate 测试与生产域名为正式提供的环境地址；其他示例中的商户号、商户域名、用户和金额仅为说明用，不能当作已开通配置。



**接入完成的标准：用户从商户网站进入 Foregate，能查看商户账户余额、买入、撤单、卖出和领取收益；商户用户资金流水与 Foregate 订单、商户保证金结果能够核对，重试不重复动钱。**



本文为独立交付文档，包含接入流程、双方交付物、全部 13 个双向接口、鉴权、字段、请求响应示例、错误码、资金规则与验收清单，无需配合其他文档阅读。所有示例凭证和签名均为占位符，时间戳在实际调用时必须重新生成；示例费用仅用于说明，不代表固定费率。



## **阅读导航**



- 第 1. 先了解双方的分工
- 第 2. 接入总流程
- 第 3. 双方交换的信息和密钥
- 第 4. Foregate 提供的接口：商户后端调用
- 第 5. 商户提供的接口：Foregate 服务端调用
- 第 6. 双向鉴权：先完成这一关再联调资金
- 第 7. 跑通登录和余额
- 第 8. 跑通买入、成交和撤单
- 第 9. 跑通卖出和收益领取
- 第 10. 幂等、超时和状态同步
- 第 11. 用户限制、退出与再次进入
- 第 12. 订单、持仓与资金状态
- 第 13. 错误码与 HTTP 重试规则
- 第 14. 联调验收清单：按顺序逐项签字
- 第 15. 上线前的最终交付表
- 第 16. 常见问题定位

## **1. 先了解双方的分工**



| 参与方 | 负责什么 |
|-|-|
| 商户网站及后端 | 登录并识别自己的用户；向 Foregate 申请跳转链接；同步用户限制；查询订单和对账 |
| 商户资金服务 | 保存用户 法币 余额；提供余额、冻结、扣款、解冻、入账和状态接收接口 |
| Foregate | 提供交易页面、订单、持仓、成交及领取；将交易金额换算为 法币 后调用商户 |
| Foregate 商户保证金账户 | 独立记录商户 USDT 保证金及其冻结、划转；由 Foregate 开通和维护 |



商户不需要开发 Foregate 下单页面，也不需要通过商户接口代用户下单。用户进入 Foregate 页面后，页面调用 Foregate 自身的交易接口。商户后台只需实现本文所列服务端接口和跳转入口。



商户用户账户与 Foregate 商户保证金是两套账：用户有 法币 余额，不代表商户 USDT 保证金足够。两者均需在联调前准备好。用户无需向 Foregate 充值，也无需领取个人链上钱包地址。





## **2. 接入总流程**



| 步骤 | 商户操作 | Foregate 操作 | 本步完成标志 |
|-|-|-|-|
| 1. 交换信息 | 提交第 3.2 节资料 | 分配商户号，交付第 3.1 节资料 | 双方环境配置表填齐 |
| 2. 开通环境 | 配置商户接口域名、密钥和来源白名单 | 开通商户、保证金账户和测试市场，配置商户接口 | 双向网络可达，保证金已就绪 |
| 3. 开发商户接口 | 实现 6 个接口和幂等账务 | 提供请求样例并协助验证 | 商户接口独立自测通过 |
| 4. 跑通免登录 | 后端签名调用 launch，浏览器打开返回链接 | 自动创建/关联用户并建立会话 | 首次与再次进入均成功 |
| 5. 跑通资金闭环 | 检查商户资金动作及余额变化 | 提供可成交、可挂单、可领取的场景 | 买入、撤单、卖出、领取均正确 |
| 6. 异常与对账 | 验证超时、重复请求、状态限制及对账 | 验证资金恢复与保证金流水 | 无重复账、无遗留冻结或待入账 |
| 7. 上线 | 提供生产地址、独立密钥和监控联系人 | 配置生产并完成小额验证 | 双方验收通过后开放用户入口 |



## **3. 双方交换的信息和密钥**



### **3.1 Foregate 提供给商户**



| 交付项 | 内容及用途 | 必须 |
|-|-|-|
| 环境标识 | 测试/预发/生产，明确对应版本与启用的市场类型 | 是 |
| `merchantId` | Foregate 分配，所有双向请求必须一致 | 是 |
| `FG_API_BASE` | Foregate API 域名，见第 3.4 节；不包含服务路径前缀 | 是 |
| `PARTNER_TO_FG_SECRET` | 商户后端调用 Foregate 时签名用；由 Foregate 分配并安全交付 | 是 |
| Foregate 调用商户接口的出口 IP/CIDR | 商户将这些 IP 加入商户接口的入站白名单 | 是 |
| 交易页面域名及跳转说明 | 实际跳转用接口返回的完整 `launchUrl`，不自行拼接前端地址 | 是 |
| 商户保证金安排 | 账户已开通确认、币种 USDT、测试资金、生产资金到位方式、核对及告警联系渠道 | 是 |
| 联调场景 | 普通订单簿、周期盘、Poly 直连各自的市场/选项，以及可领取收益的测试条件；按实际开通范围提供 | 是 |
| 接口文档、联调联系人、上线确认 | 本接入与接口文档，异常反馈渠道和验收负责人 | 是 |



**\*\*商户调用 Foregate 当前没有必填 \`X-API-KEY\` 或独立 \`apiKey\` 字段。\*\***认证用 \`merchantId、requestTime、nonce、sign\`。若商务/管理台账另行登记了 API Key，不要自行把它当作本协议签名字段。



### **3.2 商户提供给 Foregate**



| 交付项 | 内容及用途 | 必须 |
|-|-|-|
| 商户名称及联系人 | 技术、测试、资金对账、生产故障联系人 | 是 |
| `MERCHANT_WALLET_BASE` | Foregate 可访问的 HTTPS 根地址；需能在其后拼接 `/wallet/balance` 等路径 | 是 |
| `FG_TO_MERCHANT_API_KEY` | 商户颁发给 Foregate，Foregate 放在请求头 `X-API-KEY` | 是 |
| `FG_TO_MERCHANT_SECRET` | Foregate 调商户的 HMAC Secret；由商户生成、安全交付，双方保存相同值 | 是 |
| 商户后端出口 IP/CIDR | Foregate 加入商户 API 入站白名单；不是终端用户 IP | 是 |
| `merchantUserId` 规则 | 商户内稳定唯一、不可分配给另一用户；在 launch 和商户接口中完全一致 | 是 |
| 测试用户与初始余额 | 正常余额、余额不足、受限用户；提供测试余额重置方式 | 是 |
| 商户资金流水查询方式 | 可按 requestId、orderId、fillId、transactionId 查询，协助验证超时后结果 | 是 |
| 用户入口和返回地址 | 商户网站入口，WEB/MOBILE 场景；如需返回按钮则提供 `returnUrl` | 返回地址可选 |
| 用户默认展示资料 | language、nickname、email，按需提供 | 否 |
| 限流与维护说明 | 商户接口承载能力、限流反馈、维护窗口、告警及密钥轮换联系人 | 是 |



域名包含前缀时需明确，例如 `https://wallet.example.com/foregate` 最终调用 `https://wallet.example.com/foregate/wallet/balance`。根地址不要已经包含 `/wallet/balance`，也不要重复附加 `/wallet`。



### **3.3 密钥方向：不要混用**



| 名称 | 谁生成并交付 | 谁签名 | 谁验签 | 传输位置 |
|-|-|-|-|-|
| `PARTNER_TO_FG_SECRET` | Foregate → 商户 | 商户后端 | Foregate | Secret 不传输，只传计算出的 sign |
| `FG_TO_MERCHANT_API_KEY` | 商户 → Foregate | 不参与 HMAC 拼接 | 商户校验 | Header `X-API-KEY` |
| `FG_TO_MERCHANT_SECRET` | 商户 → Foregate | Foregate | 商户 | Secret 不传输，只传计算出的 sign |



这三项按环境独立配置；两个方向不要使用同一个 Secret。Secret 按约定的 UTF-8 字符串使用，即使外观是十六进制，也不要先解码成二进制。密钥通过双方约定的安全渠道交付，不写入页面、App、URL、工单或公开仓库。轮换时双方确认切换时刻，不假设系统自动接受新旧双密钥。



### **3.4 Foregate 平台环境地址**



| 配置项 | 测试环境 | 生产环境 |
|-|-|-|
| `FG_API_BASE` | `https://devtssapis.foregate.com` | `https://apis.foregate.com` |



所有 Foregate 接口统一使用 `FG_API_BASE`。登录与用户状态接口路径包含 `/api/user`，订单与商户保证金查询路径包含 `/api/orderv2`；完整 URL = FG_API_BASE + 本文列出的完整接口路径。



测试和生产分别开通商户与凭证，商户应按目标环境配置 merchantId、双向密钥和来源白名单，不混用测试与生产配置。上述地址为服务端 API 地址；用户浏览器进入交易页面仍使用 launch 接口返回的完整 launchUrl。



## **4. Foregate 提供的接口：商户后端调用**



根地址以 Foregate 实际交付为准。下表包含服务路径前缀，不能将文档内部的 `/partner/...` 直接拼到裸域名。



| 方法 | 完整地址拼法 | action | 用途 |
|-|-|-|-|
| POST | `{FG_API_BASE}/api/user/partner/session/launch` | `launch` | 获取免登录跳转链接；必接 |
| POST | `{FG_API_BASE}/api/user/partner/users/status` | `userstatus` | 同步启用、禁止买入、禁止登录等状态；必接 |
| GET | `{FG_API_BASE}/api/orderv2/partner/orders/{orderId}` | `getorder` | 单笔订单概要 |
| GET | `{FG_API_BASE}/api/orderv2/partner/orders/{orderId}/detail` | `getorderdetail` | 订单、成交与资金处理详情；对账必接 |
| GET | `{FG_API_BASE}/api/orderv2/partner/orders` | `listorders` | 按更新时间分页拉取订单；对账必接 |
| GET | `{FG_API_BASE}/api/orderv2/partner/merchant/balance` | `getmerchantbalance` | 查询本商户 USDT 保证金余额 |
| GET | `{FG_API_BASE}/api/orderv2/partner/merchant/ledger` | `getmerchantledger` | 查询商户账务视图，辅助对账 |



测试环境 launch 完整 URL 为 `https://devtssapis.foregate.com/api/user/partner/session/launch`；生产环境为 `https://apis.foregate.com/api/user/partner/session/launch`。订单路径前缀为 `/api/orderv2`。



七个接口均使用服务端 HMAC，不使用浏览器登录 Token。成功为 HTTP 2xx 且 JSON `code=0`。订单只允许查询签名商户所属数据。



`GET /partner/orders` 需要 \`start、end\`（Unix **毫秒**）、\`offset、size\`；按 \`[start,end)\` 查询，窗口最长 1 小时，size 默认 100、最大 500，按 \`updatedAt ASC, orderId ASC\` 排序。\`/partner/merchant/ledger\` 同样使用毫秒窗口和分页，可选 \`merchantUserId、orderId、type\`；type 支持 \`OMNIBUS_FREEZE / OMNIBUS_UNFREEZE / OMNIBUS_DEBIT / OMNIBUS_CREDIT\`。注意签名的 requestTime 仍是**秒**。



保证金余额来自商户 USDT 账户，不是终端用户的 法币 账户余额。当前 ledger 是已成功商户资金动作的 USDT 账务视图，`balanceAfter` 可能为空，不能仅凭该列表判断 WMS 实际划转已完成；保证金争议需由 Foregate 核对实际划转流水。不要用该字段自行推算未返回的历史余额。



### **本章通用调用要求**



所有接口携带第 6 章通用鉴权字段。下方 `{FG_API_BASE}` 使用第 3.4 节对应环境的 API 域名；请求路径中的 orderId 为字符串形式的 Foregate 订单号。只有 HTTP 2xx 且 code=0 才算成功。所有响应示例的金额均为 decimal string，时间为 ISO-8601 UTC，长 ID 按字符串处理；不要根据 ID 前缀生成业务关联。



### **4.1 创建免登录跳转会话**



```HTTP
POST {FG_API_BASE}/api/user/partner/session/launch
Content-Type: application/json
```



请求：



```JSON
{
  "merchantId": "10001",
  "merchantUserId": "player_10001",
  "nickname": "player01",
  "language": "en",
  "email": "player@example.com",
  "ip": "203.0.113.10",
  "platform": "WEB",
  "returnUrl": "https://merchant.example/game",
  "requestTime": 1789783200,
  "nonce": "17f7ee3c695747ef",
  "sign": "hex-hmac-sha256"
}
```



字段：



| 字段 | 必填 | 规则 |
|-|-|-|
| merchantId | 是 | Foregate 分配的商户标识 |
| merchantUserId | 是 | 商户 用户唯一标识，商户内不可复用 |
| nickname | 否 | 用户昵称 |
| language | 否 | 使用 Foregate 已有语言码值 |
| email | 否 | 合法邮箱；不作为账户归属键 |
| ip | 是 | 用户发起跳转时的公网 IP |
| platform | 是 | WEB 或 MOBILE |
| returnUrl | 否 | 可选的返回地址；未传时为空，服务端不校验白名单 |



响应：



```JSON
{
  "code": 0,
  "message": "success",
  "data": {
    "launchUrl": "https://dev-public.foregate.com/zh/third-party-sign-in?launchTicket=Z-6NFamaONCI9OpQkO48Ww_y6mooIBnMkwVk1wNMd9o",
    "expiresIn": 120
  }
}
```



规则：



- Foregate 以 `(merchantId, merchantUserId)` 查找或创建现有 users 表中的用户。
- Seamless 用户不创建 WMS 个人钱包地址。
- Ticket 有效期 120 秒，只能成功兑换一次。
- 商户 应通过浏览器顶层跳转打开 launchUrl，不得提前访问或记录 Ticket。

### **4.2 同步商户用户状态**



```HTTP
POST {FG_API_BASE}/api/user/partner/users/status
Content-Type: application/json
```



请求：



```JSON
{
  "merchantId": "10001",
  "merchantUserId": "player_10001",
  "status": "LOGIN_DISABLED",
  "reason": "risk_control",
  "occurredAt": "2026-09-19T02:00:00Z",
  "requestId": "pge-user-status-0001",
  "requestTime": 1789783200,
  "nonce": "ce72cb10a11148d0",
  "sign": "hex-hmac-sha256"
}
```



请求业务字段（另需第 6 章鉴权字段）：



| 字段 | 类型 | 必填 | 说明 |
|-|-|-|-|
| merchantUserId | string | 是 | 已关联的商户用户 ID |
| status | string | 是 | 下表中的用户状态 |
| reason | string | 否 | 状态调整原因 |
| occurredAt | string | 是 | 业务发生时间，ISO-8601 UTC，用于防止旧事件覆盖新状态 |
| requestId | string | 是 | 商户生成的状态变更幂等键；重试保持不变 |



status：



| 值 | 效果 |
|-|-|
| ACTIVE | 允许正常交易 |
| BUY_DISABLED | 禁止新买入；允许撤单、卖出和 Claim |
| LOGIN_DISABLED | 禁止获得完整交易会话；有未结订单或持仓时只允许撤单、卖出和 Claim |
| FULLY_DISABLED | 没有未结订单、持仓、冻结和待入账任务后完全禁用，已有 Token 失效 |



响应：



```JSON
{
  "code": 0,
  "message": "success",
  "data": {
    "merchantUserId": "player_10001",
    "status": "LOGIN_DISABLED",
    "updatedAt": "2026-09-19T02:00:01Z"
  }
}
```



requestId 用于状态请求幂等；旧 occurredAt 不覆盖新状态。



### **4.3 查询订单概要**



```HTTP
GET {FG_API_BASE}/api/orderv2/partner/orders/{orderId}?merchantId=10001&requestTime=1789783200&nonce=...&sign=...
```



响应：



```JSON
{
  "code": 0,
  "message": "success",
  "data": {
    "orderId": "100001",
    "merchantUserId": "player_10001",
    "marketId": "2531",
    "optionId": "9725",
    "marketTitle": "Example market",
    "orderStatus": "PARTIALLY_FILLED",
    "amount": "207.50",
    "feeAmount": "2.08",
    "currency": "USD",
    "createdAt": "2026-09-19T02:00:00Z",
    "updatedAt": "2026-09-19T02:01:00Z"
  }
}
```



该接口响应保持概要用途，不返回 merchantId。amount、feeAmount 当前来自原始买单冻结本金与冻结费用，币种 法币；不是已成交扣款累计值。逐笔成交及资金处理请查看订单详情。



### **4.4 查询订单详情**



```HTTP
GET {FG_API_BASE}/api/orderv2/partner/orders/{orderId}/detail?merchantId=10001&requestTime=1789783200&nonce=...&sign=...
```



响应：



```JSON
{
  "code": 0,
  "message": "success",
  "data": {
    "orderId": "100001",
    "merchantUserId": "player_10001",
    "marketId": "2531",
    "marketTitle": "Example market",
    "outcomeId": "9725",
    "side": "BUY",
    "orderType": "LIMIT",
    "orderStatus": "PARTIALLY_FILLED",
    "price": "0.50000000",
    "quantity": "20.00000000",
    "filledQuantity": "5.00000000",
    "remainingQuantity": "15.00000000",
    "grossAmountUsdt": "2.50000000",
    "feeAmountUsdt": "0.02500000",
    "netAmountUsdt": "2.52500000",
    "fxRate": "83.000000000000000000",
    "grossAmountFiat": "207.50",
    "feeAmountFiat": "2.08",
    "netAmountFiat": "209.58",
    "freezeRequestId": "fg-freeze-100001",
    "fundingStatus": "PROCESSING",
    "fills": [
      {
        "fillId": "900001",
        "quantity": "5.00000000",
        "price": "0.50000000",
        "grossAmountUsdt": "2.50000000",
        "feeAmountUsdt": "0.02500000",
        "netAmountUsdt": "2.52500000",
        "fxRate": "83.000000000000000000",
        "netAmountFiat": "209.58",
        "walletRequestId": "fg-debit-900001",
        "walletTransactionId": "pge-tx-700001",
        "fundingStatus": "COMPLETED",
        "filledAt": "2026-09-19T02:00:30Z"
      }
    ],
    "createdAt": "2026-09-19T02:00:00Z",
    "updatedAt": "2026-09-19T02:01:00Z"
  }
}
```



金额字段按实际订单方向返回：买入 net 为本金与外扣手续费之和，卖出 net 为扣费后的入账金额。本接口是订单详情，不是独立 Claim 查询接口。响应不返回 merchantId。



### **4.5 分页查询订单**



```HTTP
GET {FG_API_BASE}/api/orderv2/partner/orders?merchantId=10001&start=1789783200000&end=1789786800000&offset=0&size=100&requestTime=1789783200&nonce=...&sign=...
```



规则：



- start、end 使用 Unix 毫秒时间戳，按 `updatedAt` 的 `[start, end)` 查询。
- 单次时间窗口最大 1 小时。
- size 默认 100，最大 500。
- 固定按 `updatedAt ASC, orderId ASC` 排序。
- 仅返回签名 merchantId 所属订单。

响应：



```JSON
{
  "code": 0,
  "message": "success",
  "data": {
    "items": [],
    "offset": 0,
    "size": 100,
    "hasMore": false
  }
}
```



items 的单项结构与订单概要一致。



### **4.6 查询商户 Omnibus 保证金余额**



```HTTP
GET {FG_API_BASE}/api/orderv2/partner/merchant/balance?merchantId=10001&requestTime=1789783200&nonce=...&sign=...
```



响应：



```JSON
{
  "code": 0,
  "message": "success",
  "data": {
    "currency": "USDT",
    "balance": "100000.00000000",
    "totalBalance": "100000.00000000",
    "availableBalance": "90000.00000000",
    "frozenBalance": "10000.00000000",
    "updatedAt": "2026-09-19T02:00:00Z"
  }
}
```



响应不返回 merchantId。



`balance` 是兼容字段，与 `totalBalance` 相同；availableBalance 和 frozenBalance 是本期扩展字段。



### **4.7 查询商户账务视图**



```HTTP
GET {FG_API_BASE}/api/orderv2/partner/merchant/ledger?merchantId=10001&start=1789783200000&end=1789786800000&offset=0&size=100&requestTime=1789783200&nonce=...&sign=...
```



可选过滤参数：`merchantUserId`、`orderId`、`type`。时间窗口和分页限制与订单列表相同，但时间筛选使用商户资金动作创建时间，不是订单 updatedAt。type 支持 `OMNIBUS_FREEZE / OMNIBUS_UNFREEZE / OMNIBUS_DEBIT / OMNIBUS_CREDIT`。当前视图来自成功的商户资金动作，不能代替实际保证金划转凭证；balanceAfter 当前可为 null。



响应：



```JSON
{
  "code": 0,
  "message": "success",
  "data": {
    "items": [
      {
        "ledgerId": "ml-10001",
        "merchantUserId": "player_10001",
        "orderId": "100001",
        "fillId": "900001",
        "type": "OMNIBUS_DEBIT",
        "amount": "2.52500000",
        "currency": "USDT",
        "balanceAfter": null,
        "occurredAt": "2026-09-19T02:00:30Z"
      }
    ],
    "offset": 0,
    "size": 100,
    "hasMore": false
  }
}
```



### **4.8 查询参数和响应字段说明**



**查询参数：**所有 GET 都必须带通用鉴权字段。概要、详情的 orderId 位于路径中；余额查询不需要额外业务参数。列表和账务查询参数如下：



| 参数 | 类型 | 必填 | 约束 |
|-|-|-|-|
| start | integer | 是 | Unix 毫秒，窗口左闭 |
| end | integer | 是 | Unix 毫秒，窗口右开；end > start，差值不超过 3600000 |
| offset | integer | 否 | 从 0 开始，非负；后续页按已读取条数递增 |
| size | integer | 否 | 默认 100，范围 1～500 |
| merchantUserId | string | 否 | 仅账务视图支持，按商户用户过滤 |
| orderId | string | 否 | 仅账务视图支持，按订单过滤 |
| type | string | 否 | 仅账务视图支持，取第 4.7 节枚举 |



分页响应 items 为数组，offset/size 为整数，hasMore 为布尔值。hasMore=true 时在同一窗口继续拉取；订单可能继续更新，增量对账建议重叠回扫时间窗口并按 orderId、updatedAt 去重，不把一次分页当作不可变快照。



**订单概要与详情：**ID、枚举、时间、金额、价格和数量均以字符串返回。概要的 optionId 对应详情的 outcomeId。某些数据尚未生成时关联字段可能为 null；不得据此虚构商户资金处理成功结果。



| 字段 | 含义 |
|-|-|
| orderId / merchantUserId | Foregate 订单号 / 商户用户 ID |
| marketId / marketTitle | 市场 ID / 展示标题 |
| optionId / outcomeId | 订单选择的选项 ID |
| orderStatus | 订单业务状态，独立于资金状态 |
| amount / feeAmount / currency | 概要中的原冻结本金 / 费用 / 法币；不代表成交累计 |
| side / orderType | BUY 或 SELL；LIMIT 或 MARKET |
| price / quantity | 委托价格（USDT）/ 委托份额 |
| filledQuantity / remainingQuantity | 已成交份额 / 剩余份额 |
| grossAmountUsdt / feeAmountUsdt / netAmountUsdt | 成交本金 / 手续费 / 实际资金影响；买入净额相加，卖出净额相减 |
| fxRate | 换算汇率，1 USDT 对应的 法币 |
| grossAmountFiat / feeAmountFiat / netAmountFiat | 对应 法币 金额快照；以接口实际返回值对账 |
| freezeRequestId | 关联商户原冻结请求；无冻结的场景可为空 |
| fundingStatus | 订单资金处理状态，见第 12 章 |
| fills | 成交明细数组；无成交时为空数组 |
| createdAt / updatedAt | 创建时间 / 更新时间 |



**fills 单项：**fillId 为成交 ID；quantity、price 为本笔份额和价格；grossAmountUsdt、feeAmountUsdt、netAmountUsdt、fxRate、netAmountFiat 为本笔资金快照。walletRequestId 为对应 DEBIT 或 CREDIT 的幂等键；walletTransactionId 为商户确认成功的流水号，尚未成功时可为空。fundingStatus 为该笔资金处理状态，filledAt 为成交时间。



**保证金余额：**currency=USDT；balance 与 totalBalance 为兼容的总额字段，availableBalance 为可用，frozenBalance 为冻结，updatedAt 为更新时间。该接口不返回终端用户的 法币 账户余额。



**账务视图单项：**ledgerId 为记录 ID；merchantUserId、orderId、fillId 用于关联业务，其中 fillId 非成交场景可为空；type 为账务类型；amount 为 USDT 金额，currency=USDT；balanceAfter 当前可能为 null；occurredAt 为动作时间。该查询不替代 WMS 实际划转核对。



## **5. 商户提供的接口：Foregate 服务端调用**



六个接口均须实现并对 Foregate 开放。所有请求带 `X-API-KEY`，另带商户鉴权字段；不要要求 Foregate 携带商户浏览器 Cookie。



| 方法与路径 | action | 业务字段摘要，不含通用鉴权字段 | 商户应做的账务处理 |
|-|-|-|-|
| GET `/wallet/balance` | `getbalance` | merchantUserId、currency | 返回 balance、availableBalance、frozenBalance |
| POST `/wallet/freeze` | `freeze` | amount、feeAmount、orderId、requestId、holdActionType=ORDER_HOLD | 冻结本金+费用：可用减少，冻结增加，总余额不变 |
| POST `/wallet/debit` | `debit` | amount、feeAmount、orderId、fillId、freezeRequestId、requestId、debitActionType=ORDER_OPEN | 从原冻结扣本金+费用；不再扣一次可用余额 |
| POST `/wallet/unfreeze` | `unfreeze` | orderId、freezeRequestId、requestId、reason、holdActionType=ORDER_HOLD_RELEASE | 释放原冻结的全部剩余金额；请求不传 amount |
| POST `/wallet/credit` | `credit` | grossAmount、feeAmount、amount、orderId、positionId、requestId、creditActionType；卖出另有 fillId | 可用与总余额增加净额 amount；冻结不变 |
| POST `/wallet/transaction-sync` | `transactionsync` | eventId、sequence、schemaVersion、objectType、objectId 及状态快照 | 只记录状态，绝不改变余额 |



资金请求还包含 `merchantUserId、currency=法币`。完整请求、响应和逐字段要求见本章下方各接口定义。`marketTitle` 用于展示，接收端应允许为空或缺省，不能因此拒绝合法资金请求；各种长 ID 作为字符串存储，不用 JavaScript Number 截断。



`creditActionType`：卖出为 `SELL_PROCEEDS`；收益领取为 `CLAIM_PAYOUT`。卖出逐 Fill 入账，Claim 不要求 fillId。净额必须满足 `amount = grossAmount - feeAmount`，商户直接入账 amount，不再重复扣 feeAmount。0 净额 Claim 不调用 credit。



`unfreeze.reason`：`ORDER_CANCELLED、ORDER_FILLED、ORDER_EXPIRED、FREEZE_COMPENSATION`。原冻结已部分扣款时，只释放剩余金额；不能把原冻结总额全部退回。



### **本章通用调用要求**



完整 URL 为 `{MERCHANT_WALLET_BASE}` + 本章接口路径。每个请求均携带 `X-API-KEY: <FG_TO_MERCHANT_API_KEY>`；每个 POST 还须使用 `Content-Type: application/json`。action 取本章总览表固定值，签名使用 FG_TO_MERCHANT_SECRET。



每个资金 POST 都必填 `merchantId、merchantUserId、currency=法币、requestTime、nonce、sign`，接口字段表只列增量业务字段；这些通用字段不能省略。merchantId、merchantUserId 均为 string，currency 为 string，requestTime 为 integer，nonce 和 sign 为 string。状态同步同样携带 merchantId、merchantUserId 与鉴权字段。



资金成功响应必须返回商户生成的唯一 transactionId、currency 和该接口示例中的金额字段。transactionId 按 string 处理，金额为两位小数的 decimal string。同一业务 requestId 重试返回原 transactionId、原金额及原业务结果，不能用重试时的最新余额替换历史结果。示例中 freeze → debit → unfreeze → SELL_PROCEEDS 的余额变化为同一演示账本，Claim 请求为独立场景。



### **5.1 GET /wallet/balance**



#### **5.1.1 作用**



实时查询商户用户的 法币 总余额、可用余额和冻结余额。Foregate 会在页面查询余额和买单前调用该接口，不使用历史余额兜底。



#### **5.1.2 请求**



```HTTP
GET /wallet/balance?merchantId=201001&merchantUserId=player_10001&currency=USD&requestTime=1789783200&nonce=0123456789abcdef&sign=...
X-API-KEY: merchant-test-api-key
```



| 字段 | 类型 | 必填 | 说明 |
|-|-|-|-|
| merchantId | string | 是 | Foregate 分配的商户 ID |
| merchantUserId | string | 是 | 商户侧用户唯一 ID |
| currency | string | 是 | 商户 `法币` |
| requestTime | integer | 是 | Unix 秒级时间戳 |
| nonce | string | 是 | 本次请求随机值 |
| sign | string | 是 | action=`getbalance` 的签名 |



#### **5.1.3 成功响应**



```JSON
{
  "code": 0,
  "message": "success",
  "data": {
    "merchantUserId": "player_10001",
    "currency": "USD",
    "balance": "2000.00",
    "availableBalance": "1660.00",
    "frozenBalance": "340.00",
    "updatedAt": "2026-09-19T02:00:00Z"
  }
}
```



必须满足：



```Plain Text
balance = availableBalance + frozenBalance
```



三个余额字段必须存在、可解析、非负。用户不存在返回 `54005`，不要返回三个零余额来伪装用户存在。



### **5.2 POST /wallet/freeze**



#### **5.2.1 作用**



买单提交前冻结最大可能支出的资金。冻结成功后用户总余额不变，可用余额减少，冻结余额增加。



```Plain Text
freezeTotal = amount + feeAmount
availableBalance -= freezeTotal
frozenBalance += freezeTotal
balance 不变
```



#### **5.2.2 请求**



```JSON
{
  "merchantId": "201001",
  "merchantUserId": "player_10001",
  "currency": "法币",
  "amount": "830.00",
  "feeAmount": "6.64",
  "orderId": "100001",
  "marketTitle": "Example market",
  "holdActionType": "ORDER_HOLD",
  "requestId": "fg-freeze-100001",
  "requestTime": 1789783200,
  "nonce": "42b4fb4fece746fc",
  "sign": "hex-hmac-sha256"
}
```



| 字段 | 类型 | 必填 | 说明 |
|-|-|-|-|
| merchantUserId | string | 是 | 商户用户 ID |
| currency | string | 是 | 商户 `法币` |
| amount | decimal string | 是 | 冻结本金，不含手续费 |
| feeAmount | decimal string | 是 | 最大手续费 |
| orderId | string | 是 | Foregate 订单 ID |
| marketTitle | string | 否 | 展示字段，可为空或缺省；不得因此拒绝资金请求 |
| holdActionType | string | 是 | 固定 `ORDER_HOLD` |
| requestId | string | 是 | 本次冻结的长期幂等键 |



#### **5.2.3 商户必须实现**



- 校验用户存在且允许交易。
- 校验可用余额不小于 `amount + feeAmount`。
- 原子更新可用余额和冻结余额。
- 建立可由 `freezeRequestId` 引用的冻结记录。
- 相同 requestId 重试返回原 transactionId 和相同结果。
- 余额不足返回 `54006`，不得产生部分冻结。

#### **5.2.4 成功响应**



```JSON
{
  "code": 0,
  "message": "success",
  "data": {
    "transactionId": "merchant-freeze-700001",
    "currency": "USD",
    "frozenAmount": "836.64",
    "beforeAvailableBalance": "2000.00",
    "afterAvailableBalance": "1163.36"
  }
}
```



### **5.3 POST /wallet/debit**



#### **5.3.1 作用**



订单发生实际成交后，从原冻结中按每个 Fill 扣除资金。部分成交会调用多次，每个 Fill 使用独立 requestId。



```Plain Text
debitTotal = amount + feeAmount
frozenBalance -= debitTotal
balance -= debitTotal
availableBalance 不变
```



不得再次从 availableBalance 扣款。



#### **5.3.2 请求**



```JSON
{
  "merchantId": "201001",
  "merchantUserId": "player_10001",
  "currency": "法币",
  "amount": "207.50",
  "feeAmount": "2.08",
  "orderId": "100001",
  "fillId": "900001",
  "freezeRequestId": "fg-freeze-100001",
  "marketTitle": "Example market",
  "debitActionType": "ORDER_OPEN",
  "requestId": "fg-debit-900001",
  "requestTime": 1789783230,
  "nonce": "bb44c58e44db4173",
  "sign": "hex-hmac-sha256"
}
```



| 字段 | 类型 | 必填 | 说明 |
|-|-|-|-|
| amount | decimal string | 是 | 本 Fill 成交本金 |
| feeAmount | decimal string | 是 | 本 Fill 手续费 |
| orderId | string | 是 | Foregate 订单 ID |
| fillId | string | 是 | 成交 ID；同一 Fill 只能扣一次 |
| freezeRequestId | string | 是 | 引用原 `/wallet/freeze` 的 requestId |
| marketTitle | string | 否 | 展示字段，可为空或缺省；不得因此拒绝资金请求 |
| debitActionType | string | 是 | 固定 `ORDER_OPEN` |
| requestId | string | 是 | 本 Fill 扣款幂等键，不能与 freeze requestId 相同 |



#### **5.3.3 商户必须实现**



- 找到 `freezeRequestId` 对应的原冻结。
- 校验 merchantId、merchantUserId、orderId 与原冻结一致。
- 校验冻结剩余金额足够扣除 `amount + feeAmount`。
- 按 fillId 和 requestId 防止同一成交重复扣款。
- 原子扣减冻结余额和总余额。
- 原冻结不存在返回 `54008`；冻结余额不足返回 `54006`。

#### **5.3.4 成功响应**



```JSON
{
  "code": 0,
  "message": "success",
  "data": {
    "transactionId": "merchant-debit-700002",
    "currency": "USD",
    "debitedAmount": "209.58",
    "remainingFrozenAmount": "627.06",
    "balance": "1790.42"
  }
}
```



### **5.4 POST /wallet/unfreeze**



#### **5.4.1 作用**



订单取消、完全成交、过期或冻结补偿时，释放原冻结当前尚未 debit 的全部剩余金额。



请求不传 amount。商户必须根据 `freezeRequestId` 自行计算并一次性释放剩余冻结。



```Plain Text
releasedAmount = 原冻结金额 - 已 debit 金额 - 已释放金额
frozenBalance -= releasedAmount
availableBalance += releasedAmount
balance 不变
```



#### **5.4.2 请求**



```JSON
{
  "merchantId": "201001",
  "merchantUserId": "player_10001",
  "currency": "法币",
  "orderId": "100001",
  "holdActionType": "ORDER_HOLD_RELEASE",
  "freezeRequestId": "fg-freeze-100001",
  "reason": "ORDER_CANCELLED",
  "requestId": "fg-unfreeze-100001",
  "requestTime": 1789783260,
  "nonce": "8e2664683e394a9d",
  "sign": "hex-hmac-sha256"
}
```



| 字段 | 类型 | 必填 | 说明 |
|-|-|-|-|
| orderId | string | 是 | Foregate 订单 ID |
| holdActionType | string | 是 | 固定 `ORDER_HOLD_RELEASE` |
| freezeRequestId | string | 是 | 原冻结 requestId |
| reason | string | 是 | `ORDER_CANCELLED`、`ORDER_FILLED`、`ORDER_EXPIRED`、`FREEZE_COMPENSATION` |
| requestId | string | 是 | 解冻请求幂等键 |



#### **5.4.3 成功响应**



```JSON
{
  "code": 0,
  "message": "success",
  "data": {
    "transactionId": "merchant-unfreeze-700003",
    "currency": "USD",
    "releasedAmount": "627.06",
    "remainingFrozenAmount": "0.00",
    "availableBalance": "1790.42"
  }
}
```



如果第一次请求已经成功释放，再次以相同 requestId 请求必须返回第一次的 `releasedAmount` 和 transactionId，不能返回一条新的 0 元结果。



### **5.5 POST /wallet/credit**



#### **5.5.1 作用**



将用户卖出所得或 Claim 派奖入账到商户用户账户。



```Plain Text
amount = grossAmount - feeAmount
availableBalance += amount
balance += amount
frozenBalance 不变
```



#### **5.5.2 卖出入账请求**



```JSON
{
  "merchantId": "201001",
  "merchantUserId": "player_10001",
  "currency": "法币",
  "grossAmount": "249.00",
  "feeAmount": "2.49",
  "amount": "246.51",
  "orderId": "100001",
  "positionId": "80001",
  "fillId": "900010",
  "marketTitle": "Example market",
  "creditActionType": "SELL_PROCEEDS",
  "requestId": "fg-credit-900010",
  "requestTime": 1789783300,
  "nonce": "1b719db4eeec4e33",
  "sign": "hex-hmac-sha256"
}
```



#### **5.5.3 Claim 入账请求**



```JSON
{
  "merchantId": "201001",
  "merchantUserId": "player_10001",
  "currency": "USD",
  "grossAmount": "830.00",
  "feeAmount": "0.00",
  "amount": "830.00",
  "orderId": "100001",
  "positionId": "80001",
  "marketTitle": "Example market",
  "creditActionType": "CLAIM_PAYOUT",
  "requestId": "fg-claim-80001",
  "requestTime": 1789783400,
  "nonce": "a72dc0829a4a4c20",
  "sign": "hex-hmac-sha256"
}
```



| 字段 | 类型 | 必填 | 说明 |
|-|-|-|-|
| grossAmount | decimal string | 是 | 扣手续费前金额 |
| feeAmount | decimal string | 是 | 手续费 |
| amount | decimal string | 是 | 实际入账净额，必须等于 grossAmount - feeAmount |
| orderId | string | 是 | Foregate 订单 ID |
| positionId | string | 是 | Foregate 仓位 ID |
| fillId | string | 条件必填 | `SELL_PROCEEDS` 必填；`CLAIM_PAYOUT` 不传 |
| marketTitle | string | 否 | 展示字段，可为空或缺省；不得因此拒绝资金请求 |
| creditActionType | string | 是 | `SELL_PROCEEDS` 或 `CLAIM_PAYOUT` |
| requestId | string | 是 | 每个 Fill 或每次 Claim 的幂等键 |



Foregate 只在 Claim 实际净入账金额大于 0 时调用该接口。0 元 Claim 由 Foregate 内部完成仓位状态和 0 元操作记录，不调用商户 `/wallet/credit`，商户无需生成 0 元商户资金流水。



#### **5.5.4 成功响应**



```JSON
{
  "code": 0,
  "message": "success",
  "data": {
    "transactionId": "merchant-credit-700004",
    "currency": "法币",
    "creditedAmount": "246.51",
    "balance": "2036.93"
  }
}
```



### **5.6 POST /wallet/transaction-sync**



#### **5.6.1 作用**



同步订单、持仓和 Claim 的状态，方便商户展示、对账和补偿。该接口绝对不能改变商户账户余额。



#### **5.6.2 请求**



```JSON
{
  "merchantId": "201001",
  "merchantUserId": "player_10001",
  "eventId": "fg-event-01J7EXAMPLE",
  "sequence": 4,
  "schemaVersion": 1,
  "objectType": "ORDER",
  "objectId": "100001",
  "orderId": "100001",
  "fillId": "900001",
  "orderStatus": "PARTIALLY_FILLED",
  "currency": "USD",
  "grossAmount": "207.50",
  "feeAmount": "2.08",
  "netAmount": "209.58",
  "occurredAt": "2026-09-19T02:00:30Z",
  "requestTime": 1789783230,
  "nonce": "529d3c8db71e4228",
  "sign": "hex-hmac-sha256"
}
```



| 字段 | 类型 | 必填 | 说明 |
|-|-|-|-|
| eventId | string | 是 | 全局事件幂等键 |
| sequence | integer | 是 | 同一个对象单调递增的状态序号 |
| schemaVersion | integer | 是 | 当前固定为 `1` |
| objectType | string | 是 | `ORDER`、`POSITION`、`CLAIM` |
| objectId | string | 是 | 对应对象 ID |
| orderId | string | 是 | Foregate 订单 ID |
| fillId | string | 否 | 与成交相关时提供 |
| orderStatus | string | 是 | 当前业务状态 |
| currency | string | 是 | 固定 `法币` |
| grossAmount | decimal string | 视事件而定 | 毛金额 |
| feeAmount | decimal string | 视事件而定 | 手续费 |
| netAmount | decimal string | 视事件而定 | 净额或实际资金影响金额 |
| occurredAt | string | 是 | ISO-8601 UTC 时间 |



常用状态：



```Plain Text
ACCEPTED
PARTIALLY_FILLED
FILLED
CANCELLED
FAILED
SETTLED
CLAIMED
SOLD
```



#### **5.6.3 商户必须实现**



- 按 eventId 幂等去重。
- 为同一 merchantId 下的每个 `(objectType, objectId)` 保存已经接受的最大 sequence。
- 小于或等于最大 sequence 的旧事件不能覆盖新状态。
- sequence 跳号可以先接收并告警，不应回滚更高 sequence 的状态。
- 该接口不能创建任何余额、冻结、扣款或入账流水。

#### **5.6.4 成功响应**



```JSON
{
  "code": 0,
  "message": "success",
  "data": {
    "eventId": "fg-event-01J7EXAMPLE",
    "acceptedSequence": 4
  }
}
```



## **6. 双向鉴权：先完成这一关再联调资金**



```Plain Text
signContent = merchantId + requestTime + nonce + action
sign = lowercase_hex(HMAC-SHA256(secret 的 UTF-8 字节, signContent 的 UTF-8 字节))
```



- 拼接没有分隔符，不排序，不包含 URL、业务 JSON、API Key 或 Secret 本身。
- `requestTime`：Unix 秒，双方系统时间偏差不超过 120 秒。
- `nonce`：每次网络请求重新生成的 16～64 位随机字符串；建议使用 32 位十六进制随机值。
- `sign`：64 位小写十六进制。
- POST 的鉴权字段放 JSON Body；GET 放 Query String。action 用于计算签名，不是必传业务字段。
- 重试时更新 requestTime、nonce、sign；资金 requestId 和业务内容保持不变。
- POST 使用 `Content-Type: application/json`，字符编码 UTF-8。接口直接返回 JSON，不做 301/302 跳转。

### **通用鉴权字段与响应**



| 字段 | 类型 | 必填 | 说明 |
|-|-|-|-|
| merchantId | string | 是 | Foregate 分配的商户号 |
| requestTime | integer | 是 | Unix 秒级时间戳，允许时钟偏差 120 秒 |
| nonce | string | 是 | 本次网络请求唯一随机值，16～64 位 |
| sign | string | 是 | HMAC-SHA256 的 64 位小写十六进制结果 |



成功响应：



```JSON
{"code":0,"message":"success","data":{}}
```



失败响应（两个方向分开）：



```JSON
{"code":4002,"message":"invalid signature","data":null}
```



```JSON
{"code":54006,"message":"insufficient balance","data":null}
```



第一个失败示例由 Foregate 返回给商户；第二个由商户返回给 Foregate。code 为整数，完整错误码见第 13 章。不得混用两个方向的错误码，也不要使用面向用户页面的成功码替代 code=0。



金额使用十进制字符串，法币 保留 2 位小数；USDT 查询金额通常保留 8 位小数，汇率字段保留实际响应精度。转换使用 HALF_UP。订单概要的 amount/feeAmount 为 法币；订单详情通过 Usdt/法币 后缀明确币种；商户保证金余额与账务视图为 USDT。时间戳鉴权用秒，分页筛选用毫秒，业务时间字段用 ISO-8601 UTC。



### **可直接使用的 Python 3 签名和调用示例**



将下列代码保存为 `merchant_partner_demo.py`。仅使用标准库，从环境变量读取配置；运行只申请 launch 链接，不下单、不调用资金接口。`launchUrl` 是短时登录凭据，生产应用应直接交给当前用户跳转，不写入普通访问日志。



```Python
import hashlib
import hmac
import json
import os
import secrets
import time
import urllib.error
import urllib.parse
import urllib.request

MERCHANT_ID = os.environ["FG_MERCHANT_ID"]
SECRET = os.environ["FG_PARTNER_SECRET"]  # PARTNER_TO_FG_SECRET
API_BASE = os.environ["FG_API_BASE"].rstrip("/")

def call(method, path, action, business):
    timestamp = int(time.time())
    nonce = secrets.token_hex(16)
    content = f"{MERCHANT_ID}{timestamp}{nonce}{action}"
    signature = hmac.new(SECRET.encode("utf-8"), content.encode("utf-8"),
                         hashlib.sha256).hexdigest()
    params = dict(business)
    params.update(merchantId=MERCHANT_ID, requestTime=timestamp,
                  nonce=nonce, sign=signature)
    url = API_BASE + path
    headers = {"Accept": "application/json"}
    body = None
    if method == "GET":
        url += "?" + urllib.parse.urlencode(params)
    else:
        body = json.dumps(params, separators=(",", ":")).encode("utf-8")
        headers["Content-Type"] = "application/json"
    req = urllib.request.Request(url, data=body, headers=headers, method=method)
    try:
        with urllib.request.urlopen(req, timeout=15) as response:
            result = json.load(response)
    except urllib.error.HTTPError as error:
        raise RuntimeError(f"HTTP {error.code}; check endpoint and authentication") from None
    if result.get("code") != 0:
        raise RuntimeError(f"code={result.get('code')}, message={result.get('message') or result.get('msg')}")
    return result["data"]

if __name__ == "__main__":
    data = call("POST", "/api/user/partner/session/launch", "launch", {
        "merchantUserId": os.environ["MERCHANT_USER_ID"],
        "ip": os.environ["USER_PUBLIC_IP"],  # 当前终端用户公网 IP
        "platform": "WEB",
        "language": "en",
    })
    print(data["launchUrl"])  # 联调时立即在浏览器打开；不要使用脚本提前兑换
```



测试环境配置后运行（API 地址已填入；商户号、用户 ID、用户 IP 和密钥需使用实际测试配置）：



```Bash
export FG_MERCHANT_ID='<Foregate 分配的商户号>'
export FG_API_BASE='https://devtssapis.foregate.com'
export MERCHANT_USER_ID='<商户已存在的测试用户 ID>'
export USER_PUBLIC_IP='<测试用户公网 IP>'
read -r -s -p 'Foregate 提供的 PARTNER_TO_FG_SECRET: ' FG_PARTNER_SECRET
export FG_PARTNER_SECRET
python3 merchant_partner_demo.py
```



生产环境将 FG_API_BASE 改为 `https://apis.foregate.com`，并同时替换为生产商户号、用户和密钥。



用同一 call 函数查询（每次调用都会生成新鉴权字段）：



```Python
# 下列代码在已加载上述函数的 Python 程序内运行。
merchant_balance = call("GET", "/api/orderv2/partner/merchant/balance",
                        "getmerchantbalance", {})
end = int(time.time() * 1000)
orders = call("GET", "/api/orderv2/partner/orders", "listorders",
              {"start": end - 3600000, "end": end, "offset": 0, "size": 100})
# 用实际订单号代替占位符；不要把 fillId 或 positionId 当作 orderId。
# detail = call("GET", "/api/orderv2/partner/orders/<orderId>/detail", "getorderdetail", {})
```



首次只验证一个正确签名，再验证错误 Secret、过期时间、重复 nonce 和错误来源 IP，确认双方鉴权行为一致。商户验证 Foregate 请求时使用 `FG_TO_MERCHANT_SECRET`，不要误用示例中的 `FG_PARTNER_SECRET`。



## **7. 跑通登录和余额**



1. 商户先在自己的系统创建测试用户和 法币 账户，保证 merchantUserId 可被 `/wallet/balance` 查询。Foregate 自动创建映射用户，不替商户创建其本地账户。
2. 用户登录商户网站后，商户后端读取真实用户 ID 和公网 IP，签名调用 launch。不要让前端上传一个任意 merchantUserId 后就替其签名。
3. launch 成功返回 `data.launchUrl`、`expiresIn=120`。
4. 商户让当前用户的浏览器做顶层跳转至完整 launchUrl；不由商户服务器访问该链接，不预抓取，不多人共用。
5. Foregate 页面兑换一次性 Ticket 并建立会话；此操作由 Foregate 前端负责，商户无需实现 `/auth/seamless/exchange`。
6. Ticket 过期或已使用时，从第 2 步重新申请；不要反复打开旧链接。
7. 检查 Foregate 顶部、头像菜单和个人中心的余额，并核对商户收到的余额请求。





当前商户结算币种为 法币，交易与展示基准为 USDT（币种编号 2）。Foregate 做汇率换算；商户收到 法币 后不要再次换算。法币 金额使用十进制字符串并保留 2 位小数，数据库使用 DECIMAL。示例汇率 `1 USDT=83 法币` 时，830.00 法币 对应 10 USDT；实际以环境有效汇率和各笔资金快照为准，不把示例汇率写死。



## **8. 跑通买入、成交和撤单**





先测一笔不会立即成交的限价买单：freeze 成功后应出现挂单；撤销后商户收到 unfreeze，原冻结余额归零，可用余额恢复。再测立即成交和分笔成交。



买单每个 Fill 独立扣款；订单被标为 FILLED 不等于资金已全部完成，需同时核对逐笔 DEBIT、剩余冻结及保证金确认。发生部分成交时，已经 debit 的金额不能因撤单退回。资金暂时处理中时保留原请求，等待系统重试和查询收敛。



**可人工核对的单笔示例，全部为 法币：**



| 动作 | 本次变化 | 总余额 | 可用余额 | 冻结余额 |
|-|-|-|-|-|
| 初始 | — | 2000.00 | 2000.00 | 0.00 |
| freeze | 本金 830.00 + 费用 16.60 | 2000.00 | 1153.40 | 846.60 |
| 第一个 Fill debit | 本金 207.50 + 费用 4.15 | 1788.35 | 1153.40 | 634.95 |
| 第二个 Fill debit | 本金 600.00 + 费用 12.00 | 1176.35 | 1153.40 | 22.95 |
| 全成后 unfreeze | 剩余 22.95；请求不传金额 | 1176.35 | 1176.35 | 0.00 |



若只完成第一个 Fill 就撤单，则应释放 634.95，最后总余额与可用余额均为 1788.35。所有时刻满足 `balance = availableBalance + frozenBalance`。



## **9. 跑通卖出和收益领取**



| 场景 | 用户动作 | 商户应收到 | 验收重点 |
|-|-|-|-|
| 卖出挂单 | 已有持仓，提交限价卖单 | 未成交时不发生用户 法币 freeze/debit | Foregate 冻结持仓份额，不冻结用户账户资金 |
| 卖出成交 | 市价卖出或限价卖出被成交 | 每个 Fill 一次 credit，SELL_PROCEEDS | amount 净额一次入账；有对应保证金结算 |
| 取消未成交卖单 | 取消卖出挂单 | 不要求出现 法币 unfreeze | Foregate 恢复可售份额；已有卖出成交不能回滚 |
| Claim | 市场结算后，用户领取获胜持仓收益 | 正净额 credit，CLAIM_PAYOUT | 用户实际入账、领取资金处理完成、保证金结果一致 |
| 0 元 Claim | 领取净额为 0 | 不调用 credit | 无需商户生成零金额入账流水 |



卖出例：grossAmount=249.00、feeAmount=4.98、amount=244.02，则用户可用和总余额各增加 244.02。Claim 同样按净额入账，不因页面已显示 CLAIMED 就认定商户到账完成。



卖出和 Claim 的保证金与商户入账顺序可能不同，商户只需处理实际收到的幂等资金请求，不能自行依据页面状态再补发一笔 credit。



## **10. 幂等、超时和状态同步**



### **10.1 商户资金接口必须做到**



| 情况 | 商户应返回/执行 |
|-|-|
| 首次 requestId | 在同一数据库事务内提交余额、冻结记录、流水和幂等结果 |
| 同 requestId、相同业务内容 | 不再动账，返回首次的业务结果及 transactionId |
| 同 requestId、业务内容不同 | 返回 54007，不能覆盖历史记录或再次处理 |
| 已提交资金但 HTTP 响应丢失 | 后续原 requestId 重试返回首次成功结果 |
| 仍在处理/结果尚不能确定 | 54012 或 55001；保留原请求的处理状态 |
| 可用/冻结余额不足 | 54006；不能部分执行 |
| 原冻结不存在 | 54008；不新建一个替代冻结 |



幂等键为 `(merchantId, action, requestId)`，必须长期保存。比较业务内容时排除 requestTime、nonce、sign。nonce 的短期防重放与 requestId 的资金幂等是两种不同机制。



HTTP 超时、429、5xx 不代表资金没有发生。Foregate 重试原 requestId，不应要求其换新 ID。明确业务失败与未知结果要区分，不能将超时统一当作余额不足。



推荐数据库唯一键：



```SQL
UNIQUE KEY uk_wallet_idempotency (merchant_id, action, request_id)
```



业务内容比较时应排除 `requestTime`、`nonce`、`sign`，其他业务字段必须一致。



推荐事务顺序：



```Plain Text
开始数据库事务
  → 插入/锁定幂等记录
  → 如果已完成，返回第一次保存的响应
  → 如果 requestId 内容冲突，返回 54007
  → 锁定商户用户账户和原冻结记录
  → 校验余额和业务关联
  → 更新余额/冻结/流水
  → 保存最终响应
提交数据库事务
```



不得先返回成功再异步记账。如果系统只能异步处理，应返回 `54012` 或 `55001`，之后原 requestId 重试必须能够收敛到最终结果。



### **10.2 状态通知不记钱**



`transaction-sync` 按 eventId 去重，按同一商户下 `(objectType, objectId)` 的 sequence 单调更新；低序号不能覆盖高序号。objectType 为 ORDER / POSITION / CLAIM，schemaVersion 当前为 1。部分字段随事件类型可为空。



该接口只更新商户展示/对账视图，**不能再次冻结、扣款或入账**。同步事件与资金请求可能延迟或乱序到达，应保留事件和资金记录分别核对，并使用订单详情补查。



### **10.3 商户账务不变量**



每次操作完成后必须满足：



```Plain Text
balance = availableBalance + frozenBalance
availableBalance >= 0
frozenBalance >= 0
balance >= 0
```



操作影响：



| 操作 | availableBalance | frozenBalance | balance |
|-|-|-|-|
| freeze | 减少 | 增加 | 不变 |
| debit | 不变 | 减少 | 减少 |
| unfreeze | 增加 | 减少 | 不变 |
| credit | 增加 | 不变 | 增加 |
| transaction-sync | 不变 | 不变 | 不变 |



所有商户账户余额更新、幂等记录和商户资金流水必须在同一个数据库事务内完成。



## **11. 用户限制、退出与再次进入**



商户后端调用用户状态接口，包含 `merchantUserId、status、requestId、occurredAt` 及通用鉴权字段；occurredAt 使用 ISO-8601 UTC，例如 `2026-09-29T08:00:00Z`。旧事件不能覆盖新状态，重试保持业务 requestId。



| status | 用途 |
|-|-|
| ACTIVE | 正常使用 |
| BUY_DISABLED | 禁止新买入，允许撤单、卖出、Claim |
| LOGIN_DISABLED | 限制完整交易会话；有未结业务时仍保留必要的撤单、卖出、Claim 通道 |
| FULLY_DISABLED | 待未结订单、持仓、冻结和待入账业务清理后完全禁用 |



不要在仍有冻结或待入账时，直接让商户所有接口都返回“用户不存在”，否则会阻止正常资金恢复。商户需保留对原合法请求的查询/幂等重放能力。



returnUrl 为可选导航地址，不是商户回调地址，也不是支付结果通知地址；当前 launch 不承诺对其执行白名单拦截，商户应在自己后端限制为可信站内地址。再次进入 Foregate 必须重新申请 launchUrl，不复用旧 Ticket。



## **12. 订单、持仓与资金状态**



订单和持仓业务状态（不同对象使用各自适用的状态）：



| 状态 | 说明 |
|-|-|
| FREEZING | 买单资金冻结处理中 |
| ACCEPTED | 订单已接受 |
| PENDING | 等待成交 |
| PARTIALLY_FILLED | 部分成交 |
| FILLED | 全部成交 |
| CANCELED / CANCELLED | 已撤单；订单查询可能返回 CANCELED，状态同步可使用 CANCELLED，接收端兼容两种写法 |
| FAILED | 订单失败 |
| SETTLED | 仓位已结算、可 Claim |
| CLAIMING | Claim 资金处理中 |
| CLAIMED | 已完成 Claim；可包含 0 元 Claim |
| SOLD | 仓位已全部卖出 |



资金状态：



| 状态 | 说明 |
|-|-|
| PREPARING | 准备资金请求 |
| READY | 下单资金准备完成 |
| PROCESSING | 资金接口调用中 |
| RETRYING | 远端结果不明确，Foregate 正在恢复 |
| COMPLETED | 资金动作完成 |
| RELEASED | 剩余冻结已释放 |
| MANUAL_REVIEW | 需要人工核查 |
| FAILED | 资金动作明确失败；不代表冻结已释放，不允许据此以新 requestId 重复支付 |



订单状态与资金状态独立。资金失败不代表冻结已释放，订单 FILLED、仓位 CLAIMED 也不能替代商户入账核对。



汇率业务日采用 UTC；更新失败沿用最近一次有效汇率并在后台告警，允许跨日继续使用。只有没有有效汇率可用时才按汇率不可用处理。保留原 fxRateId、rateTime、businessDate，不能把旧汇率伪装成当天更新；已冻结订单使用其锁定汇率。



## **13. 错误码与 HTTP 重试规则**



### **13.1 Foregate 返回给商户**



本表仅适用于第 4 章 Foregate 提供的接口。无业务 requestId 的 GET 查询和 launch 重试时，仅刷新鉴权字段；包含业务 requestId 的请求遵循下表。



| Code | 含义 | 是否可按原 requestId 重试 |
|-|-|-|
| 0 | 成功 | 不需要；重复请求应返回原结果 |
| 4001 | 参数错误 | 否，修正请求后使用新 requestId |
| 4002 | 签名错误 | 可修正签名；业务内容和 requestId 不变 |
| 4003 | requestTime 超窗或 nonce 重复 | 是，仅刷新鉴权字段 |
| 4004 | 商户不存在或未授权 | 否，停用并告警 |
| 4005 | 用户不存在 | 否，停买并告警 |
| 4006 | 可用余额或冻结余额不足 | 否 |
| 4007 | 同一 requestId 的业务内容冲突 | 否，必须人工排查 |
| 4008 | 订单或原冻结不存在 | 否，人工复核 |
| 4009 | 用户状态限制 | 否，至少停止新买入 |
| 4010 | Seamless 商户服务、汇率或外部依赖暂时不可用 | 是，稍后使用原 requestId 重试 |
| 4011 | Foregate 当前有效汇率不可用 | 否，禁止新买入 |
| 4012 | 请求处理中 | 是，延迟后使用原 requestId |
| 4290 | 请求过于频繁 | 是，退避后使用原 requestId |
| 5000 | 系统内部错误 | 是，使用原 requestId |
| 5001 | 结果处理中或未知 | 是，使用原 requestId 查询/重试 |



相同 requestId、相同业务内容不是错误，不返回“重复提交”；必须返回第一次的相同结果。



| HTTP | 场景 | 处理 |
|-|-|-|
| 200 | 请求已被应用层处理 | 继续检查响应体 code |
| 400 | JSON、字段或路径参数不合法 | 修正请求，不盲目重试 |
| 401 | 签名或时间校验失败 | 修正鉴权字段 |
| 403 | 商户、IP 或资源归属不允许 | 不重试 |
| 404 | 资源不存在 | 不重试 |
| 429 | 限流 | 指数退避并保持业务幂等键 |
| 500/502/503/504 | 临时服务异常 | 原 requestId 指数退避重试 |



连接超时、读取超时或响应无法解析时，调用方不得推断失败，也不得更换 requestId。资金结果必须按“可能已成功”处理。



### **13.2 商户返回给 Foregate**



本表适用于第 5 章全部商户接口。成功为 0，失败统一使用以下五位码；不能用 Foregate 四位码替代。



| Code | 含义 | 商户处理要求 | Foregate 是否重试 |
|-|-|-|-|
| 0 | 成功 | 返回完整成功 data | 否 |
| 54001 | 参数错误 | 不执行资金操作 | 否 |
| 54002 | 签名错误 | 不执行资金操作 | 修正鉴权后可重试 |
| 54003 | 时间超窗或 nonce 重复 | 不执行资金操作 | 是，仅刷新鉴权字段 |
| 54004 | 商户不存在或未授权 | 不执行资金操作 | 否 |
| 54005 | 用户不存在 | 不执行资金操作 | 否 |
| 54006 | 可用余额或冻结余额不足 | 不得部分处理 | 否 |
| 54007 | requestId 业务内容冲突 | 不执行第二次资金操作并告警 | 否 |
| 54008 | 订单或原冻结不存在 | 不执行资金操作 | 否，人工复核 |
| 54009 | 用户状态限制 | 不执行受限操作 | 否 |
| 54010 | 商户临时不可用 | 保留幂等记录 | 是，原 requestId |
| 54012 | 请求处理中 | 保留原 requestId 处理状态 | 是，原 requestId |
| 54290 | 请求过于频繁 | 不丢失幂等状态 | 是，指数退避 |
| 55000 | 商户系统内部临时异常 | 结果应可通过原 requestId 收敛 | 是，原 requestId |
| 55001 | 结果处理中或未知 | 不能声明失败或回滚已成功资金 | 是，原 requestId |



HTTP 建议：



| HTTP | 使用场景 |
|-|-|
| 200 | 已得到明确业务结果，包括余额不足等业务拒绝 |
| 400 | JSON 或字段格式错误 |
| 401 | API Key、签名、时间或 nonce 校验失败 |
| 403 | 商户或来源 IP 未授权 |
| 404 | 明确不存在且不会异步出现的资源 |
| 429 | 服务限流 |
| 500/502/503/504 | 临时系统异常；Foregate 会按原 requestId 重试 |



对于 freeze/debit/unfreeze/credit，商户绝不能因为 HTTP 连接断开就回滚或重新创建 requestId。可能已成功的结果必须通过原 requestId 再次查询式重试并返回第一次结果。



## **14. 联调验收清单**



| 编号 | 场景 | 通过条件 |
|-|-|-|
| A1 | 双向签名、API Key、白名单 | 正确请求成功；错误密钥/时间/nonce/IP 被拒绝，不动账 |
| A2 | 首次和再次免登录 | 同 merchantUserId 始终关联同一用户；过期/重放 Ticket 不可用 |
| A3 | 余额展示 | 个人中心、顶部、交易弹窗与商户 法币 余额换算一致 |
| B1 | 未成交限价买单撤销 | freeze → unfreeze；原冻结归零，无 DEBIT |
| B2 | 限价买单更优价全部成交 | 按实际 Fill 扣款，预冻差额释放，无余额残留 |
| B3 | 市价买入、多 Fill | 每 Fill 一笔 DEBIT；预算余量最终释放，无漏扣或重复扣 |
| B4 | 部分成交后取消 | 已成交金额保留扣款，仅释放未用冻结；异常时可恢复 |
| B5 | 卖出挂单、撤单及成交 | 未成交只占份额；成交 SELL_PROCEEDS 净额到账且只入一次 |
| B6 | 正金额与0金额 Claim | 正金额净额入账；0金额不调用商户 credit |
| C1 | 响应丢失及重复请求 | 所有资金接口原请求重试，余额仅改变一次 |
| C2 | 同 ID 改金额 | 54007，账务完全不变 |
| C3 | 通知重放与乱序 | 旧状态不覆盖新状态，transaction-sync 不动余额 |
| C4 | 用户限制与恢复 | 新买入被按规则限制，存量资金退出路径可用 |
| D1 | 商户订单和账务查询 | 跨商户查询被拒；分页能完整拉取测试窗口 |
| D2 | 三方对账 | 用户 法币 动作、Foregate 成交/Claim、商户 USDT 保证金处理逐笔一致 |



B 组按已开通的普通盘、周期盘、Poly 直连分别执行；不能只测一种市场即认定所有市场通过。Claim 测试由 Foregate 提供已结算且获胜的测试持仓。每项保留商户用户 ID、订单号、成交号、请求号、响应、时间和前后余额，不保存 Secret 或完整登录 Ticket。



对账时按订单聚合 freeze/debit/unfreeze，按 Fill 检查 DEBIT/SELL_PROCEEDS，按 positionId 和 requestId 检查 CLAIM_PAYOUT。所有最终结束的测试买单剩余冻结应归零；同步通知成功不能替代资金流水成功。



## **15. 上线前的最终交付表**



复制此表，测试与生产分别填写；Secret/API Key 仅填写安全存储引用或交付确认，不填明文。



| 项目 | Foregate 填写 | 商户填写 | 确认结果 |
|-|-|-|-|
| 环境及上线版本 |  |  |  |
| merchantId |  | — |  |
| FG_API_BASE |  | — |  |
| PARTNER_TO_FG_SECRET 安全交付确认 |  | 已接收确认 |  |
| MERCHANT_WALLET_BASE | — |  |  |
| FG_TO_MERCHANT_API_KEY / SECRET 安全交付确认 | 已接收确认 |  |  |
| 双向出口 IP 白名单 | Foregate 出口 | 商户出口 |  |
| 用户入口、returnUrl、WEB/MOBILE | 前端域名 | 商户入口与返回地址 |  |
| 用户 ID 规则与测试账户 | 已关联确认 |  |  |
| 商户保证金准备 | 账户与可用额已核实 | 资金安排确认 |  |
| 开通市场及联调案例 |  |  |  |
| 重试恢复及状态通知 | 相关处理任务已启用 | 幂等/限流/通知可用 |  |
| 异常联系、监控与值班 |  |  |  |



生产使用独立地址和凭证，由 Foregate 确认该环境包含必要的结算、撤单、全部成交尾差释放及 Claim 恢复能力；文档存在不代表某个环境已部署。先完成小额闭环，再开放商户入口。暂停新交易时由双方协调保留存量扣款、解冻和入账恢复，不直接关掉整个商户资金服务。



## **16. 常见问题定位**



| 现象 | 优先检查 |
|-|-|
| launch 404 | 是否漏 `/api/user` 前缀，是否用错环境域名 |
| 订单查询404 | 是否使用 `/api/orderv2`；订单是否属于当前签名商户 |
| 签名不通过 | Secret 方向、秒级时间戳、action、无分隔符拼接、小写 hex、Secret 未二次解码 |
| 时间/nonce 错误 | 时钟同步；每次请求是否生成新 nonce；是否把毫秒当作秒 |
| launch 成功但余额查不到 | 商户本地用户是否已存在；merchantUserId 是否完全一致；商户六个接口配置及另一方向凭证 |
| 用户有余额但不能买入 | 同时检查用户可用 法币、商户保证金可用 USDT、用户状态与服务可用性 |
| FILLED 或 CLAIMED 但钱没到 | 查对应商户资金 requestId 和资金状态，再由 Foregate 查保证金确认；不要再次创建资金请求 |
| 全成/撤单后仍冻结 | 按 freezeRequestId 对账扣款与解冻，确认释放请求是否成功；保持原请求等待恢复 |
| 重复通知后余额变化 | 商户错误地把 transaction-sync 当成资金接口，需修正 |



提交问题时提供：环境、merchantId、merchantUserId、orderId/fillId/positionId、requestId、merchant transactionId、发生时间及时区、脱敏请求/响应和余额变化。不要提供 Secret、API Key 明文、登录 Cookie 或 launch Ticket。
