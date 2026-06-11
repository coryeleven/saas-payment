# SaaS 会员支付系统规范文档

## 1. 项目概述

本项目为一个从零开始构建的 SaaS 会员支付系统，目标是实现完整的会员订阅支付闭环：

> 用户微信扫码登录 → 选择月付会员 → 创建订单 → 微信/支付宝支付 → 支付回调验签 → 开通或续费会员 → 后台管理订单、退款、发票和会员状态。

第一版重点面向 **桌面 Web 用户**，采用前后端分离架构。

---

## 2. 已确认需求

### 2.1 项目类型

- 当前项目为空项目，从零开始开发。
- 产品类型：SaaS 功能会员。
- 用户必须登录后才能购买会员。
- 长期支持多端，但第一版以桌面 Web 为主。

### 2.2 技术选型

| 模块 | 技术 |
|---|---|
| 前端用户端 | React |
| 管理后台 | React |
| 后端 | Node.js / NestJS |
| 数据库 | MySQL |
| 架构 | 前后端分离 |
| 部署 | 阿里云 / 腾讯云 |
| 包管理 | 推荐 pnpm workspace |

### 2.3 登录方式

- 用户登录：微信开放平台扫码登录。
- 用户唯一身份：微信 `UnionID`。
- 后台登录：单管理员账号密码登录。

### 2.4 支付方式

第一版支持：

- 微信支付：Native 扫码支付。
- 支付宝：电脑网站支付。
- 支付接入方式：直连官方 API。
- 开发测试：
  - 支付宝使用沙箱。
  - 微信支付可使用 Mock provider 辅助开发。
  - 上线前使用小额真实支付验证微信链路。

### 2.5 会员规则

- 第一版只有一个月付会员套餐。
- 套餐价格由管理后台维护。
- 会员周期：30 天。
- 用户未到期时再次购买：从当前到期时间顺延 30 天。
- 用户已过期后购买：从当前时间开始计算 30 天。
- 到期后有 7 天宽限期。
- 宽限期内仍可使用高级功能，但需要提示续费。

### 2.6 退款规则

- 第一版支持后台手动退款。
- 退款成功后，立即扣回该订单贡献的会员时长。
- 如果扣回后用户没有剩余有效权益，则会员立即失效。
- 第一版只支持整单退款。

### 2.7 订单规则

- 未支付订单 15 分钟后自动关闭。
- 支付状态以服务端验签后的支付回调为准。
- 前端返回页不能直接确认支付成功。
- 支付回调必须幂等，防止重复开通会员。

### 2.8 发票规则

- 第一版支持用户提交发票申请。
- 后台人工处理发票申请。
- 不接电子发票 API。
- 用户可提交：
  - 发票类型
  - 抬头
  - 税号
  - 邮箱
  - 手机号
  - 开票金额

### 2.9 通知规则

- 第一版使用微信公众号模板消息或订阅消息。
- 通知类型包括：
  - 支付成功
  - 会员即将到期
  - 进入宽限期
  - 退款成功
  - 发票状态变更

---

## 3. 系统架构规范

推荐采用 monorepo 结构：

```text
pay_project/
  apps/
    api/        # NestJS 后端 API
    web/        # React 用户端
    admin/      # React 管理后台
  packages/
    shared/     # 共享类型、枚举、DTO
  infra/
    docker/     # 本地 MySQL、Redis 等开发环境
    deploy/     # 阿里云/腾讯云部署配置
```

---

## 4. 后端模块规范

后端位于：

```text
apps/api/
```

核心模块如下：

| 模块 | 职责 |
|---|---|
| `config` | 环境变量、微信、支付宝、JWT、数据库、Redis 配置 |
| `database` | MySQL 连接、迁移、事务工具 |
| `auth` | 微信扫码登录、UnionID 用户创建、用户 session |
| `admin-auth` | 管理员账号密码登录 |
| `users` | 用户资料、会员状态查询 |
| `plans` | 会员套餐管理 |
| `orders` | 创建订单、订单查询、订单过期 |
| `payments` | 微信支付、支付宝支付、支付回调验签 |
| `membership` | 会员开通、续费、宽限期、权限判断 |
| `refunds` | 后台退款、会员时长扣回 |
| `invoices` | 发票申请和后台处理 |
| `notifications` | 微信通知 |
| `scheduler` | 定时任务：订单过期、到期提醒、对账 |
| `audit` | 后台操作和支付相关审计日志 |

---

## 5. 支付 Provider 规范

统一定义支付接口：

```ts
interface PaymentProvider {
  provider: 'WECHAT_PAY' | 'ALIPAY';

  createPayment(input: CreatePaymentInput): Promise<CreatePaymentResult>;

  verifyNotify(input: RawNotifyInput): Promise<VerifiedPaymentNotify>;

  queryPayment(input: QueryPaymentInput): Promise<QueryPaymentResult>;

  refund(input: RefundInput): Promise<RefundResult>;

  queryRefund(input: QueryRefundInput): Promise<QueryRefundResult>;
}
```

### 5.1 微信支付

文件建议：

```text
apps/api/src/modules/payments/providers/wechat-pay.provider.ts
```

要求：

- 使用微信支付 API v3。
- 使用 Native 支付生成二维码链接 `code_url`。
- 回调必须：
  - 验证签名。
  - 验证平台证书。
  - 解密回调资源。
  - 校验订单号、金额、商户号、AppID。
  - 幂等处理支付成功事件。

### 5.2 支付宝支付

文件建议：

```text
apps/api/src/modules/payments/providers/alipay.provider.ts
```

要求：

- 使用支付宝电脑网站支付。
- 后端生成支付跳转表单或支付 URL。
- `return_url` 仅用于前端展示。
- `notify_url` 才是确认支付成功的依据。
- 回调必须：
  - 验证支付宝签名。
  - 校验订单号、金额、商户身份。
  - 判断 `trade_status`。
  - 幂等处理支付成功事件。

### 5.3 Mock 支付

文件建议：

```text
apps/api/src/modules/payments/providers/mock.provider.ts
```

用途：

- 本地开发。
- 自动化测试。
- 微信支付沙箱能力不足时模拟支付成功/失败/退款。

---

## 6. 数据库规范

### 6.1 用户表 `users`

字段：

- `id`
- `wechat_unionid`
- `wechat_openid_web`
- `nickname`
- `avatar_url`
- `status`
- `created_at`
- `updated_at`

要求：

- `wechat_unionid` 必须唯一。
- 不允许仅通过 `openid` 创建重复用户。

### 6.2 管理员表 `admin_users`

字段：

- `id`
- `username`
- `password_hash`
- `status`
- `last_login_at`
- `created_at`
- `updated_at`

要求：

- 密码必须使用 argon2 或 bcrypt 哈希。
- 第一版只需要单管理员。

### 6.3 会员套餐表 `membership_plans`

字段：

- `id`
- `code`
- `name`
- `price_cents`
- `currency`
- `duration_days`
- `is_active`
- `sort_order`
- `created_at`
- `updated_at`

要求：

- 第一版只有一个月付套餐。
- `duration_days = 30`。
- 套餐价格由后台维护。
- 订单创建时必须保存价格快照，避免改价影响历史订单。

### 6.4 订单表 `orders`

字段：

- `id`
- `order_no`
- `user_id`
- `plan_id`
- `provider`
- `amount_cents`
- `currency`
- `status`
- `expires_at`
- `paid_at`
- `provider_trade_no`
- `provider_payload_json`
- `created_at`
- `updated_at`

订单状态：

```text
PENDING
PAYING
PAID
EXPIRED
CLOSED
REFUNDING
REFUNDED
FAILED
```

要求：

- `order_no` 必须唯一。
- 未支付订单 15 分钟后过期。
- 支付成功必须由后端回调验签确认。
- 支付成功后必须在事务内更新订单和会员状态。

### 6.5 支付流水表 `payment_transactions`

用途：

- 保存支付和退款流水。
- 保存第三方交易号。
- 记录金额、状态和原始通知摘要。

字段：

- `id`
- `order_id`
- `provider`
- `provider_trade_no`
- `transaction_type`
- `status`
- `amount_cents`
- `raw_notify_json`
- `verified_at`
- `created_at`

### 6.6 支付回调日志表 `payment_notify_logs`

用途：

- 记录微信/支付宝的原始回调。
- 便于排查问题、重放和审计。

字段：

- `id`
- `provider`
- `notify_id`
- `order_no`
- `event_type`
- `raw_body`
- `headers_json`
- `verified`
- `processed`
- `error_message`
- `created_at`

### 6.7 用户会员表 `user_memberships`

字段：

- `id`
- `user_id`
- `paid_until`
- `grace_until`
- `status`
- `last_order_id`
- `updated_at`

会员状态：

```text
NONE
ACTIVE
GRACE
EXPIRED
CANCELED
```

访问规则：

```text
允许访问高级功能，当：
1. now <= paid_until
或
2. paid_until < now <= grace_until
```

宽限期内接口需要返回：

```ts
{
  isGrace: true,
  shouldPromptRenewal: true
}
```

### 6.8 会员流水表 `membership_ledger`

用途：

- 记录会员时长增加和扣回。
- 支持退款后准确扣回权益。
- 提供审计能力。

字段：

- `id`
- `user_id`
- `order_id`
- `refund_id`
- `entry_type`
- `duration_seconds`
- `effective_from`
- `effective_until`
- `reason`
- `created_at`

流水类型：

```text
GRANT
CLAWBACK
ADMIN_ADJUSTMENT
```

要求：

- 支付成功写入 `GRANT`。
- 退款成功写入 `CLAWBACK`。
- 同一个订单的退款扣回必须幂等。

### 6.9 退款表 `refunds`

字段：

- `id`
- `refund_no`
- `order_id`
- `user_id`
- `provider`
- `amount_cents`
- `status`
- `provider_refund_no`
- `reason`
- `admin_user_id`
- `requested_at`
- `succeeded_at`
- `provider_payload_json`
- `created_at`
- `updated_at`

退款状态：

```text
REQUESTED
PROCESSING
SUCCEEDED
FAILED
CANCELED
```

要求：

- 第一版只支持整单退款。
- 一个订单只能成功退款一次。
- 退款成功后必须扣回会员权益。

### 6.10 发票申请表 `invoice_applications`

字段：

- `id`
- `user_id`
- `order_id`
- `invoice_type`
- `title`
- `tax_no`
- `email`
- `phone`
- `amount_cents`
- `status`
- `admin_note`
- `processed_by_admin_id`
- `processed_at`
- `created_at`
- `updated_at`

发票状态：

```text
SUBMITTED
PROCESSING
ISSUED
REJECTED
```

### 6.11 通知日志表 `notification_logs`

字段：

- `id`
- `user_id`
- `channel`
- `template_code`
- `template_id`
- `status`
- `payload_json`
- `provider_response_json`
- `sent_at`
- `created_at`

### 6.12 审计日志表 `audit_logs`

字段：

- `id`
- `actor_type`
- `actor_id`
- `action`
- `resource_type`
- `resource_id`
- `metadata_json`
- `ip`
- `user_agent`
- `created_at`

需要记录：

- 套餐改价。
- 发起退款。
- 发票状态修改。
- 会员手动调整。
- 管理员登录。

---

## 7. API 规范

### 7.1 用户 API

```http
GET  /api/auth/wechat/login-url
GET  /api/auth/wechat/callback
POST /api/auth/logout

GET  /api/me
GET  /api/me/membership

GET  /api/plans

POST /api/orders
GET  /api/orders/:id
GET  /api/orders/:id/payment

POST /api/invoices
GET  /api/invoices
```

### 7.2 支付回调 API

```http
POST /api/payments/wechat/notify
POST /api/payments/alipay/notify
GET  /api/payments/alipay/return
```

要求：

- `notify` 接口必须使用原始 body 验签。
- 支付宝 `return` 接口不能改变订单状态，只能跳转展示结果页。
- 所有回调必须记录到 `payment_notify_logs`。

### 7.3 管理后台 API

```http
POST /api/admin/auth/login
POST /api/admin/auth/logout
GET  /api/admin/me

GET   /api/admin/plans
POST  /api/admin/plans
PATCH /api/admin/plans/:id

GET /api/admin/users
GET /api/admin/users/:id

GET  /api/admin/orders
GET  /api/admin/orders/:id
POST /api/admin/orders/:id/refund

GET /api/admin/refunds

GET   /api/admin/invoices
PATCH /api/admin/invoices/:id

GET /api/admin/audit-logs
```

---

## 8. 前端页面规范

### 8.1 用户端

路径：

```text
apps/web/
```

页面：

| 路由 | 功能 |
|---|---|
| `/login` | 微信扫码登录 |
| `/` | 用户首页 / SaaS 仪表盘 |
| `/membership` | 查看会员状态、到期时间、宽限期提示、续费入口 |
| `/checkout/:orderId` | 支付页面 |
| `/payment/result` | 支付结果页 |
| `/invoices` | 发票申请和记录 |

支付页要求：

- 微信支付展示二维码。
- 展示 15 分钟倒计时。
- 前端轮询订单状态。
- 支付成功后跳转会员页。
- 支付超时后提示重新下单。
- 支付宝支付跳转到支付宝电脑网站支付。
- 支付宝返回后必须重新查询后端订单状态。

### 8.2 管理后台

路径：

```text
apps/admin/
```

页面：

| 路由 | 功能 |
|---|---|
| `/login` | 管理员登录 |
| `/dashboard` | 总览 |
| `/plans` | 套餐管理 |
| `/users` | 用户列表 |
| `/orders` | 订单列表 |
| `/orders/:id` | 订单详情和退款 |
| `/refunds` | 退款记录 |
| `/invoices` | 发票申请处理 |
| `/audit-logs` | 审计日志 |

后台要求：

- 只有管理员登录后可访问。
- 退款操作需要二次确认。
- 套餐改价需要记录审计日志。
- 发票状态变更需要记录操作人和处理时间。

---

## 9. 关键状态机

### 9.1 订单状态机

```text
PENDING
  -> PAYING
  -> PAID
  -> EXPIRED
  -> CLOSED
  -> FAILED

PAID
  -> REFUNDING
  -> REFUNDED
```

规则：

- `PENDING` / `PAYING` 超过 15 分钟未支付，变为 `EXPIRED`。
- 只有支付回调验签成功后才能变为 `PAID`。
- `PAID` 后才能退款。
- 重复回调不能重复开通会员。

### 9.2 会员状态机

```text
NONE
  -> ACTIVE

ACTIVE
  -> GRACE
  -> EXPIRED

GRACE
  -> ACTIVE
  -> EXPIRED

ACTIVE / GRACE
  -> CANCELED
```

规则：

- 支付成功进入 `ACTIVE`。
- 到期后进入 `GRACE`。
- 宽限期 7 天后进入 `EXPIRED`。
- 宽限期续费后重新进入 `ACTIVE`。
- 退款扣回后无有效权益，进入 `CANCELED` 或 `EXPIRED`。

### 9.3 退款状态机

```text
REQUESTED
  -> PROCESSING
  -> SUCCEEDED
  -> FAILED
  -> CANCELED
```

规则：

- 管理员发起退款后进入 `REQUESTED`。
- 调用支付平台退款接口后进入 `PROCESSING`。
- 确认退款成功后进入 `SUCCEEDED`。
- 退款成功后必须扣回会员权益。

### 9.4 发票状态机

```text
SUBMITTED
  -> PROCESSING
  -> ISSUED
  -> REJECTED
```

---

## 10. 安全规范

### 10.1 支付安全

必须做到：

- 不信任前端支付结果。
- 不信任支付宝 return URL。
- 微信和支付宝回调必须验签。
- 必须校验：
  - 订单号
  - 金额
  - 商户号
  - AppID
  - 支付渠道
  - 第三方交易号
- 支付成功处理必须幂等。
- 支付回调必须记录日志。
- 支付密钥、证书不得提交到代码库。

### 10.2 登录安全

用户端：

- 微信 UnionID 作为唯一身份。
- 不通过 openid 创建重复用户。
- session 建议使用 HTTP-only Cookie。

管理端：

- 管理员密码必须哈希存储。
- 管理员登录接口必须限流。
- 后台接口必须鉴权。
- 管理端 token 生命周期应短于用户端。
- 退款、改价等敏感操作必须记录审计日志。

### 10.3 数据安全

- 发票信息属于敏感信息，只有管理员可查看。
- 日志中不得明文输出私钥、证书、access token。
- 支付回调 raw body 可以保存，但敏感字段需要脱敏。
- 生产环境必须启用 HTTPS。
- MySQL 需要开启备份。

---

## 11. 实施顺序

### 阶段一：项目初始化

- 初始化 pnpm monorepo。
- 创建 NestJS API。
- 创建 React 用户端。
- 创建 React 管理后台。
- 创建 shared package。
- 配置 TypeScript。
- 配置 `.env.example`。
- 配置本地 MySQL / Redis。

### 阶段二：后端基础设施

- 配置模块。
- 数据库连接。
- Prisma / TypeORM schema。
- 全局 DTO validation。
- 全局异常处理。
- 日志格式。
- 用户 session。
- 管理员 session。
- 审计日志基础能力。

### 阶段三：登录

- 微信扫码登录 URL。
- 微信 callback。
- 获取 UnionID。
- 创建或读取用户。
- 签发用户 session。
- 管理员账号密码登录。
- 初始化管理员账号。

### 阶段四：套餐、订单、会员模型

- 创建数据库表。
- 后台维护月付套餐。
- 用户获取套餐。
- 创建订单。
- 订单保存价格快照。
- 订单 15 分钟过期。
- 会员状态查询接口。

### 阶段五：支付集成

- 定义 payment provider interface。
- 实现支付宝电脑网站支付。
- 实现支付宝沙箱回调。
- 实现微信 Native 支付。
- 实现微信 Mock provider。
- 实现支付回调验签。
- 实现支付成功后的订单和会员事务处理。

### 阶段六：会员权益

- 支付成功写入会员流水。
- 新会员开通 30 天。
- 未到期续费顺延 30 天。
- 过期后续费从当前时间开始。
- 宽限期计算。
- 前端展示会员状态和续费提醒。

### 阶段七：退款

- 后台订单详情页增加退款按钮。
- 管理员填写退款原因。
- 调用支付平台退款接口。
- 退款成功后写退款记录。
- 扣回会员时长。
- 写审计日志。
- 发送微信通知。

### 阶段八：发票

- 用户提交发票申请。
- 后台查看申请。
- 后台更新状态：
  - 处理中
  - 已开票
  - 已驳回
- 发送发票状态通知。

### 阶段九：通知

- 接入公众号模板消息或订阅消息。
- 支付成功通知。
- 退款成功通知。
- 会员即将到期通知。
- 宽限期提醒。
- 发票状态通知。
- 记录通知日志。

### 阶段十：部署上线

- 配置阿里云 / 腾讯云部署。
- 配置 HTTPS。
- 配置支付回调域名。
- 配置微信开放平台回调域名。
- 配置支付宝 notify URL 和 return URL。
- 配置生产密钥和证书。
- 开启数据库备份。
- 配置日志和健康检查。
- staging 环境完成小额真实支付测试。

---

## 12. 验证规范

### 12.1 单元测试

必须覆盖：

- 订单状态流转。
- 未支付订单过期。
- 支付回调幂等。
- 金额不匹配时拒绝支付成功。
- 会员未到期续费顺延。
- 会员过期后续费重新计算。
- 宽限期访问权限。
- 退款后会员时长扣回。
- 重复退款不重复扣回。
- 套餐改价不影响历史订单。
- 发票申请状态流转。
- 管理后台鉴权。

### 12.2 集成测试

必须覆盖：

- 用户登录。
- 创建订单。
- 支付宝沙箱支付。
- 微信 Mock 支付。
- 支付成功后开通会员。
- 用户续费。
- 订单超时关闭。
- 后台退款。
- 发票申请和后台处理。

### 12.3 手工端到端测试

流程：

1. 用户打开桌面 Web。
2. 微信扫码登录。
3. 查看会员套餐。
4. 创建微信支付订单。
5. 扫码付款。
6. 确认会员开通。
7. 创建支付宝订单。
8. 跳转支付宝支付。
9. 返回后查询支付状态。
10. 再次购买会员，确认到期时间顺延。
11. 模拟会员过期，进入 7 天宽限期。
12. 确认宽限期仍可使用高级功能。
13. 管理员登录后台。
14. 查看订单详情。
15. 发起退款。
16. 确认会员时长被扣回。
17. 用户提交发票申请。
18. 管理员处理发票申请。
19. 检查审计日志、支付日志、通知日志。

---

## 13. 上线检查清单

上线前必须确认：

- 微信开放平台扫码登录回调域名配置正确。
- 微信支付 Native notify URL 可公网 HTTPS 访问。
- 支付宝 notify URL 配置正确。
- 支付宝 return URL 配置正确。
- 服务器时间同步正常。
- 微信支付证书和私钥未提交代码库。
- 支付宝私钥未提交代码库。
- 生产环境使用正式商户号和正式 AppID。
- 支付回调日志可查询。
- 支付回调重复发送不会重复开通会员。
- MySQL 自动备份已开启。
- Redis 可用。
- 管理员默认密码已修改。
- 退款操作有二次确认。
- 套餐改价有审计日志。
- 发票信息只有管理员可访问。
- 微信通知失败不会影响支付主流程。
- 支付成功、退款、会员变更均有日志可查。

---

## 14. 第一版范围边界

### 第一版必须做

- 微信扫码登录。
- 单一月付会员。
- 后台维护价格。
- 微信 Native 扫码支付。
- 支付宝电脑网站支付。
- 支付回调验签。
- 会员开通和续费。
- 7 天宽限期。
- 后台订单查询。
- 后台手动退款。
- 退款后扣回会员权益。
- 发票申请记录。
- 微信通知。
- 审计日志。

### 第一版暂不做

- 自动续费。
- 多会员等级。
- 手机 App 支付。
- 微信小程序支付。
- 电子发票 API 自动开票。
- 多管理员角色权限。
- 企业级审批流。
- 优惠券。
- 余额/积分体系。
- 部分退款。
- 团队/组织账号。
