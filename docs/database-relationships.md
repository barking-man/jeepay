# Jeepay 数据库表关系详解

本文梳理 Jeepay 全部 **23 张表**之间的业务关系，按"配置层 → 交易主干 → 分账"的顺序展开，便于从零理解整个支付链路。

> 配套阅读：[payment-channel-model.md](./payment-channel-model.md)（讲支付方式、支付接口、收单机构与路由的业务概念）。

数据来源：

- `docs/sql/init.sql`：建表语句 + 初始化数据（23 张表全量）
- `docs/sql/patch.sql`：老版本升级用的增量脚本（全新安装不需要执行）

> **重要前提**：库内**没有任何物理外键**（SQL 中不存在 `FOREIGN KEY` 约束）。本文描述的关系来自字段命名与建表注释，并已用业务代码（Mapper 的 JOIN、Service 的读写逻辑）交叉验证。数据库层面不会帮你保证一致性，删数据、改商户号时需要自行判断影响范围。

---

## 0. 三条阅读前提

1. **没有物理外键**：关系全靠同名字段（`mch_no`、`app_id`、`if_code`…）维系。
2. **配置层是"策略"，订单层是"快照"**：订单一生成，费率、商户名、通道号就复制一份到自己身上，之后改配置不影响历史订单。
3. **两类多态引用**，读之前必须先看类型字段：
   - `t_pay_interface_config.info_id`：按 `info_type` 指向服务商 / 商户 / 应用；
   - `t_mch_notify_record.order_id` 与 `t_order_snapshot.order_id`：按 `order_type` 指向支付单 / 退款单 / 转账单。

## 1. 表清单（按业务分组）

| 分组 | 表名 | 说明 |
|---|---|---|
| ① 系统与权限 | `t_sys_user` | 系统用户（运营平台 + 商户中心共用一张表，按 `sys_type` 区分） |
| | `t_sys_user_auth` | 用户认证方式（账号 / 手机号 / 邮箱 / 第三方 openid） |
| | `t_sys_role` | 角色 |
| | `t_sys_user_role_rela` | 用户 ↔ 角色 关联 |
| | `t_sys_entitlement` | 权限（菜单、页面、按钮），`pid` 自关联成菜单树 |
| | `t_sys_role_ent_rela` | 角色 ↔ 权限 关联 |
| | `t_sys_config` | 系统参数 KV 配置（各平台访问地址等） |
| | `t_sys_log` | 操作日志 |
| ② 商户与服务商 | `t_isv_info` | 服务商 |
| | `t_mch_info` | 商户（普通 / 特约） |
| | `t_mch_app` | 商户应用（签名主体，持有 `app_secret`） |
| ③ 支付方式与通道配置 | `t_pay_way` | 支付方式字典（微信扫码、支付宝 APP…） |
| | `t_pay_interface_define` | 支付接口（上游通道）定义 |
| | `t_pay_interface_config` | 各主体在通道上的对接参数与上游费率 |
| | `t_mch_pay_passage` | 商户支付通道：路由 + 商户费率 |
| ④ 交易订单与通知 | `t_pay_order` | 支付订单 |
| | `t_refund_order` | 退款订单 |
| | `t_transfer_order` | 转账订单 |
| | `t_mch_notify_record` | 商户通知（回调）记录 |
| | `t_order_snapshot` | 订单接口报文快照（代码中未使用，预留表） |
| ⑤ 分账 | `t_mch_division_receiver_group` | 分账接收账号组 |
| | `t_mch_division_receiver` | 分账接收者账号 |
| | `t_pay_order_division_record` | 分账记录 |

---

## 2. ① 系统与权限：谁能登录、能点哪些菜单

| 关系 | 连接字段 | 说明 |
|---|---|---|
| 用户 → 认证方式 | `t_sys_user_auth.user_id` → `t_sys_user.sys_user_id` | 一个用户可以有多条认证记录，**每条 = 一种登录方式**（`identity_type`：1 登录用户名 / 2 手机号；3 邮箱、10 微信、20 QQ 为预留）；密码存 `credential`（BCrypt 散列）+ `salt` |
| 用户 ↔ 角色 | 通过 `t_sys_user_role_rela`（`user_id` + `role_id` 联合主键） | 多对多 |
| 角色 ↔ 权限 | 通过 `t_sys_role_ent_rela`（`role_id` + `ent_id` 联合主键） | 多对多 |
| 权限自关联 | `t_sys_entitlement.pid` → 自身 `ent_id` | 菜单树；`ent_type`：`ML` 左侧菜单 / `MO` 其他菜单 / `PB` 页面按钮 |
| 用户 / 角色 → 商户 | `belong_info_id` → `t_mch_info.mch_no` | 商户中心的用户和角色绑定到具体商户；运营平台固定为 `'0'` |
| 商户 → 初始用户 | `t_mch_info.init_user_id` → `t_sys_user.sys_user_id` | 创建商户时自动生成的管理员账号 |

要点：

- `sys_type`（`MGR` 运营平台 / `MCH` 商户中心）在用户、角色、权限三张表上都存在，**两个平台是两套完全独立的用户、角色、菜单**，靠该字段隔离。
- 创建用户时默认写入**两条**认证记录：登录用户名（`identity_type=1`）和手机号（`identity_type=2`），共用同一个密码。
  参见 `jeepay-service/src/main/java/com/jeequan/jeepay/service/impl/SysUserAuthService.java:48`
- 登录时不需要用户选择方式：输入内容匹配手机号格式就走手机号认证，否则按用户名认证（`JeeUserDetailsServiceImpl.loadUserByUsername()`）。
  查询条件是 `(identity_type, identifier, sys_type)` 三件套，所以同一个手机号在运营平台和商户中心可以是两个不同用户。
- 修改密码会遍历该用户的**全部**认证记录逐条更新（`SysUserAuthService.resetAuthInfo()`）；没有 `salt` 的记录会被跳过（预留给无密码的第三方登录）。
- 目前代码里只用到 `1` 和 `2` 两种类型，`CS.AUTH_TYPE` 中的邮箱 / 微信 / QQ 常量没有任何写入或查询点，属于**预留未启用**。
- 建商户是一次事务完成的多表写入：`MchInfoService.addMch()` 会依次写 `t_mch_info`、`t_sys_user`（带 `belong_info_id`）、`t_mch_app`（默认应用），最后把 `init_user_id` 回填到商户表。
  参见 `jeepay-service/src/main/java/com/jeequan/jeepay/service/impl/MchInfoService.java:61`
- `t_sys_config`（全局参数）与 `t_sys_log`（操作日志，`user_id` 指向用户）属于旁支，不参与交易链路。

## 3. ② 商户与服务商：钱最终进谁的口袋

```text
t_isv_info (服务商) 1 ──── N t_mch_info (商户) 1 ──── N t_mch_app (应用)
            isv_no ─────────┘          mch_no ─────────┘
```

| 关系 | 连接字段 | 说明 |
|---|---|---|
| 服务商 → 商户 | `t_mch_info.isv_no` → `t_isv_info.isv_no` | `t_mch_info.type` = 1 普通商户（`isv_no` 为空，自己对接渠道）/ 2 特约商户（服务商模式，`isv_no` 必填） |
| 商户 → 应用 | `t_mch_app.mch_no` → `t_mch_info.mch_no` | 一个商户可以有多个应用，用于拆分业务线或回调地址 |

要点：

- 建特约商户时会校验服务商是否可用（`MchInfoService.addMch()`）。
- `t_mch_app` 是**技术主体**：`app_id` + `app_secret` 是商户请求签名、平台回调签名用的凭证。
- 这三张表是所有交易表的"归属三件套"：订单与记录表都冗余 `mch_no`、`isv_no`、`app_id`，本质是指向这三张表的逻辑外键。

## 4. ③ 支付方式与通道配置：走哪条通道、费率多少

四张表分工完全不同，这是最容易混淆的地方：

| 表 | 它回答的问题 | 关键字段 |
|---|---|---|
| `t_pay_way` | 用户能用哪些**支付方式** | `way_code`（如 `WX_NATIVE`、`ALI_APP`）、`way_name` |
| `t_pay_interface_define` | 平台接入了哪些**上游通道**、通道支持哪些支付方式与商户模式 | `if_code`、`if_name`、`is_mch_mode`、`is_isv_mode`、`way_codes`(JSON)、`state` |
| `t_pay_interface_config` | 某个主体在某条通道上的**对接参数**（密钥、证书、子商户号） | `info_type` + `info_id`、`if_code`、`if_params`(JSON)、`if_rate`（上游费率）、`state` |
| `t_mch_pay_passage` | 某商户应用用某支付方式时**实际走哪条通道、商户费率多少** | `mch_no`、`app_id`、`if_code`、`way_code`、`rate`、`risk_config`、`state` |

关系连线：

| 关系 | 连接字段 | 说明 |
|---|---|---|
| 接口定义 → 接口参数 | `t_pay_interface_config.if_code` → `t_pay_interface_define.if_code` | 1 对 N：同一条通道，服务商 / 商户 / 应用各配一份参数；唯一键 `(info_type, info_id, if_code)` |
| 接口参数 → 主体（多态） | `info_id` 按 `info_type` 指向上表 | `info_type` = 1 指 `t_isv_info.isv_no`，2 指 `t_mch_info.mch_no`，3 指 `t_mch_app.app_id` |
| 商户通道 → 商户 / 应用 / 通道定义 / 支付方式 | `mch_no`、`app_id`、`if_code`、`way_code` | 唯一键 `(app_id, if_code, way_code)`，说明同一应用同一种支付方式可配置多条通道，多通道时按 `if_code` 选一条可用的 |

要点：

- 通道参数的读取按商户类型区分（`ConfigContextQueryService`，`jeepay-payment/src/main/java/com/jeequan/jeepay/pay/service/ConfigContextQueryService.java:96`）：
  - 普通商户：只读应用级参数（`INFO_TYPE_MCH_APP`）；
  - 特约商户：读**服务商参数（`INFO_TYPE_ISV`）+ 应用级子商户参数（`INFO_TYPE_MCH_APP`）**两套。
- "谁能收钱、走哪条通道、费率多少" = ②的商户 / 应用 + ③的定义 / 参数 / 通道表共同决定。

## 5. ④ 交易主干：从 t_pay_order 出发

`t_pay_order` 的 5 个外键分别指向：

| 字段 | 指向 | 含义 |
|---|---|---|
| `mch_no` | `t_mch_info` | 收款商户（必填） |
| `isv_no` | `t_isv_info` | 服务商（普通商户为空） |
| `app_id` | `t_mch_app` | 用哪个应用下的单，决定验签密钥 |
| `if_code` | `t_pay_interface_define` | 这笔单实际走的通道（预下单、通道未定时为空） |
| `way_code` | `t_pay_way` | 支付方式 |

下单时的"取值 → 快照"过程（`AbstractPayOrderController`，`jeepay-payment/src/main/java/com/jeequan/jeepay/pay/ctrl/payorder/AbstractPayOrderController.java:166`）：

1. 用 `(mchNo, appId, wayCode)` 查 `t_mch_pay_passage`，命中后拿到该通道行（含 `if_code` 与 `rate`）；查不到直接报"商户应用不支持该支付方式"。
2. 生成订单时写入 `mch_fee_rate = passage.rate`（**费率快照**）、`mch_fee_amount = 金额 × 费率`（**手续费快照**），同时复制 `mch_name`、`mch_type`。
3. 订单状态 `INIT(0)` 先落库，随后才与渠道交互：`ING(1)` → `SUCCESS(2)` / `FAIL(3)`；此外还有 `CANCEL(4)`、`REFUND(5)`、`CLOSED(6)`。
4. 订单上冗余三组状态：退款组（`refund_state`、`refund_amount`、`refund_times`）、分账组（`division_mode`、`division_state`、`division_last_time`）、通知组（`notify_state`）。

挂在这张订单上的四张表：

### 5.1 退款单 `t_refund_order`

- `pay_order_id` → `t_pay_order.pay_order_id`：一笔支付单可对应多张退款单（支持部分退款）。
- 唯一键 `(mch_no, mch_refund_no)` 防止商户重复提交同一笔退款。
- 退款成功时（`RefundOrderService.updateIng2Success()`，`jeepay-service/src/main/java/com/jeequan/jeepay/service/impl/RefundOrderService.java:73`）先更新退款单状态，再通过 `updateRefundAmountAndCount` 回写支付单：`refund_times + 1`、`refund_amount` 累加、按订单金额判定 `refund_state`，全额退款时把支付单状态置为 `5`（已退款）。
- 该 SQL（`jeepay-service/src/main/java/com/jeequan/jeepay/service/mapper/PayOrderMapper.xml:86`）自带条件 `refund_amount + 本次退款金额 <= amount`，**从数据库层面防止超额退款**。
- 冗余 `channel_pay_order_no`（渠道支付单号）便于对账。

### 5.2 转账单 `t_transfer_order`

- 与支付单**没有直接关系**，只共享归属字段 `mch_no` / `isv_no` / `app_id` / `if_code`。
- 它是平台主动出款，不是用户付款：`entry_type` 决定入账方式（微信零钱 / 支付宝 / 银行卡），`account_no` / `account_name` / `bank_name` 是收款方信息。
- 唯一键 `(mch_no, mch_order_no)`；状态：0 订单生成、1 转账中、2 转账成功、3 转账失败、4 订单关闭。

### 5.3 通知记录 `t_mch_notify_record`（多态）

- `order_id` + `order_type`：`1` 支付单（`pay_order_id`）、`2` 退款单（`refund_order_id`）、`3` 转账单（`transfer_id`）。
- 创建规则见 `PayMchNotifyService`（`jeepay-payment/src/main/java/com/jeequan/jeepay/pay/service/PayMchNotifyService.java:76`）。
- 唯一键 `(order_id, order_type)`：一笔订单一种类型最多一条通知记录。
- `notify_count` / `notify_count_limit` 控制重试次数，`res_result` 存商户响应，`state`：1 通知中、2 通知成功、3 通知失败。
- 还冗余 `mch_no` / `isv_no` / `app_id`：通知需要拿应用的 `app_secret` 签名。

### 5.4 订单快照 `t_order_snapshot`

- 主键 `(order_id, order_type)`，同样按 `order_type` 指向支付单 / 退款单。
- 存下游请求/响应、上游请求/响应的报文与时间。
- **代码中检索不到任何引用**（仅 `init.sql` 建表），属于预留表，实际运行时不会有数据。

## 6. ⑤ 分账：账号组 → 接收者 → 分账记录

```text
t_mch_division_receiver_group ──1:N──> t_mch_division_receiver
        (auto_division_flag)                     (division_profit)
              │                                          │
              └────────► t_pay_order_division_record ◄───┘
                              (pay_order_id → 支付单)
```

| 表 | 关键字段 | 说明 |
|---|---|---|
| `t_mch_division_receiver_group` | `receiver_group_id`、`mch_no`、`auto_division_flag` | 账号组；`auto_division_flag = 1` 的组即"自动分账组"（商户中心设置自动组时会把其它组置否，逻辑上只有一个） |
| `t_mch_division_receiver` | `receiver_id`、`receiver_group_id`、`mch_no`、`app_id`、`if_code`、`acc_type`、`acc_no`、`relation_type`、`division_profit`、`state` | 分账接收者；`acc_type` 0 个人 / 1 商户，`relation_type` 为上游（如微信）分账关系类型，`division_profit` 是比例（小数，`0.3` 表示 30%），`state` 1 正常 / 0 暂停 |
| `t_pay_order_division_record` | `record_id`、`pay_order_id`、`receiver_id`、`receiver_group_id`、`batch_order_id`、`state`、`cal_division_amount` | 分账记录；`batch_order_id` 为一次分账的批次号（同批次多条记录共用），`state` 0 待分账 / 1 成功 / 2 失败 / 3 上游已受理 |

资金如何计算、何时执行（`PayOrderDivisionProcessService`，`jeepay-payment/src/main/java/com/jeequan/jeepay/pay/service/PayOrderDivisionProcessService.java:75`）：

1. **触发**：订单 `division_mode = 1`（自动）时，支付成功后延迟 80 秒发 MQ 执行分账；`division_mode = 2`（手动）时由商户调用分账接口，可指定 `receiverId` / `receiverGroupId` 列表。
2. **选接收者**：先按 `mchNo + appId + ifCode + state=可用` 过滤；自动模式再要求属于 `auto_division_flag = 1` 的组；手动模式按入参过滤。
3. **可分配金额** = 支付金额 − 商户手续费 − 已退款金额 − 已成功分账金额（`PayOrderService.calMchIncomeAmount()`），结果为负时取 0。
4. **单条金额** = 可分配金额 × 该接收者的 `division_profit`；若所有比例之和超过 100%，最后一条按剩余金额兜底并向下取整，避免超分。
5. 记录写入 `t_pay_order_division_record` 后调用渠道分账接口，回写 `state` 与上游批次号；失败记录可重试（状态改回待分账）。
6. 记录表保存的是完整的**账号快照**（`receiver_alias` / `acc_no` / `division_profit` 等），事后修改接收者账号或比例，不影响历史分账记录。

## 7. 完整链路串起来

1. 运营平台创建服务商、商户（②），创建商户时会自动生成管理员用户与默认应用（①）。
2. 商户在商户中心创建应用，拿到 `appId` / `appSecret`（②）。
3. 运营平台为服务商 / 商户 / 应用配置通道参数（③：`t_pay_interface_define` + `t_pay_interface_config`）。
4. 商户把"应用 + 支付方式"绑定到具体通道并设置商户费率（③：`t_mch_pay_passage`）。
5. 用户下单：按 `(appId, wayCode)` 命中通道，`if_code` 与费率写入支付订单快照（④）。
6. 支付成功后按 `division_mode` 自动或手动分账，接收者按 `mchNo + appId + ifCode` 匹配（⑤）。
7. 退款、转账、商户通知分别挂在订单上并回写状态（④）。
8. 全流程的操作由 ① 中的用户发起，并留有 `t_sys_log` 日志。

## 8. 容易踩的坑

- **金额单位是分**（`BIGINT`），**费率是小数**（`DECIMAL(20,6)`，`0.0055` 表示 0.55%）；界面上展示时会乘 100。
- **两个费率不是一回事**：`t_mch_pay_passage.rate` 是向商户收取的费率；`t_pay_interface_config.if_rate` 是上游通道的成本费率。
- **三个订单号**：`mch_order_no`（商户系统单号）、`pay_order_id`（平台单号）、`channel_order_no`（渠道单号）；对账主要用渠道单号。
- 订单表冗余了 `mch_name` / `mch_type`，查询历史订单**不要** join 当前商户信息，否则商户改名后历史数据会变。
- `t_order_snapshot` 目前是空转的预留表；`t_sys_config`、`t_sys_log` 与交易链路无关。
- 数据库没有外键约束，跨表删除（如删商户）必须在业务代码里判断是否存在交易数据。

## 附录 A：关键枚举速查

| 表 / 字段 | 取值 |
|---|---|
| `t_pay_order.state` | 0 订单生成、1 支付中、2 支付成功、3 支付失败、4 已撤销、5 已退款、6 订单关闭 |
| `t_pay_order.refund_state` | 0 未发生实际退款、1 部分退款、2 全额退款 |
| `t_pay_order.division_mode` | 0 不允许分账、1 支付成功按配置自动分账、2 商户手动分账 |
| `t_pay_order.division_state` | 0 未发生分账、1 等待分账任务处理、2 分账处理中、3 分账任务已结束 |
| `t_refund_order.state` | 0 订单生成、1 退款中、2 退款成功、3 退款失败、4 退款任务关闭 |
| `t_transfer_order.state` | 0 订单生成、1 转账中、2 转账成功、3 转账失败、4 订单关闭 |
| `t_mch_notify_record.order_type` | 1 支付、2 退款、3 转账 |
| `t_mch_notify_record.state` | 1 通知中、2 通知成功、3 通知失败 |
| `t_pay_order_division_record.state` | 0 待分账、1 分账成功、2 分账失败、3 分账已受理（上游受理） |
| `t_pay_interface_config.info_type` | 1 服务商、2 商户、3 商户应用 |
| `t_mch_info.type` | 1 普通商户、2 特约商户（服务商模式） |
| `t_mch_division_receiver.acc_type` | 0 个人（对私）、1 商户（对公） |
| `t_sys_entitlement.ent_type` | `ML` 左侧菜单、`MO` 其他菜单、`PB` 页面 / 按钮 |
| `t_sys_user.sys_type` | `MGR` 运营平台、`MCH` 商户中心 |

## 附录 B：ER 图（Mermaid）

> GitHub / Gitee / Typora 等支持 Mermaid 的 Markdown 渲染器会直接画出下图。

```mermaid
erDiagram
    %% ================== 系统与权限 ==================
    t_sys_entitlement ||--o{ t_sys_role_ent_rela : "ent_id"
    t_sys_role ||--o{ t_sys_role_ent_rela : "role_id"
    t_sys_user ||--o{ t_sys_user_role_rela : "user_id"
    t_sys_role ||--o{ t_sys_user_role_rela : "role_id"
    t_sys_user ||--o{ t_sys_user_auth : "user_id"
    t_sys_user |o--o{ t_sys_log : "user_id"
    t_mch_info |o--o{ t_sys_user : "belong_info_id"
    t_mch_info |o--o{ t_sys_role : "belong_info_id"
    t_sys_user |o--o{ t_mch_info : "init_user_id"

    %% ================== 商户 / 服务商 ==================
    t_isv_info |o--o{ t_mch_info : "isv_no"
    t_mch_info ||--o{ t_mch_app : "mch_no"

    %% ================== 支付方式 / 接口 / 通道 ==================
    t_pay_interface_define ||--o{ t_pay_interface_config : "if_code"
    t_isv_info |o--o{ t_pay_interface_config : "info_id"
    t_mch_info |o--o{ t_pay_interface_config : "info_id"
    t_mch_app |o--o{ t_pay_interface_config : "info_id"
    t_mch_info ||--o{ t_mch_pay_passage : "mch_no"
    t_mch_app ||--o{ t_mch_pay_passage : "app_id"
    t_pay_interface_define ||--o{ t_mch_pay_passage : "if_code"
    t_pay_way ||--o{ t_mch_pay_passage : "way_code"

    %% ================== 交易订单 / 通知 / 快照 ==================
    t_mch_info ||--o{ t_pay_order : "mch_no"
    t_isv_info |o--o{ t_pay_order : "isv_no"
    t_mch_app ||--o{ t_pay_order : "app_id"
    t_pay_interface_define |o--o{ t_pay_order : "if_code"
    t_pay_way ||--o{ t_pay_order : "way_code"
    t_pay_order ||--o{ t_refund_order : "pay_order_id"
    t_pay_order ||--o{ t_mch_notify_record : "order_id"
    t_refund_order ||--o{ t_mch_notify_record : "order_id"
    t_transfer_order ||--o{ t_mch_notify_record : "order_id"
    t_pay_order |o--o{ t_order_snapshot : "order_id"
    t_refund_order |o--o{ t_order_snapshot : "order_id"
    t_mch_info ||--o{ t_transfer_order : "mch_no"
    t_isv_info |o--o{ t_transfer_order : "isv_no"
    t_mch_app ||--o{ t_transfer_order : "app_id"
    t_pay_interface_define ||--o{ t_transfer_order : "if_code"

    %% ================== 分账 ==================
    t_mch_info ||--o{ t_mch_division_receiver_group : "mch_no"
    t_mch_division_receiver_group |o--o{ t_mch_division_receiver : "receiver_group_id"
    t_mch_info ||--o{ t_mch_division_receiver : "mch_no"
    t_isv_info |o--o{ t_mch_division_receiver : "isv_no"
    t_mch_app ||--o{ t_mch_division_receiver : "app_id"
    t_pay_interface_define ||--o{ t_mch_division_receiver : "if_code"
    t_pay_order ||--o{ t_pay_order_division_record : "pay_order_id"
    t_mch_division_receiver ||--o{ t_pay_order_division_record : "receiver_id"
    t_mch_division_receiver_group |o--o{ t_pay_order_division_record : "receiver_group_id"

    %% ================== 实体定义 ==================
    t_sys_entitlement {
        varchar(64) ent_id PK
        varchar(8) sys_type PK
        varchar(32) pid FK
        varchar(32) ent_name
        char(2) ent_type
        tinyint state
    }
    t_sys_role {
        varchar(32) role_id PK
        varchar(32) role_name
        varchar(8) sys_type
        varchar(64) belong_info_id FK
    }
    t_sys_role_ent_rela {
        varchar(32) role_id PK, FK
        varchar(64) ent_id PK, FK
    }
    t_sys_user {
        bigint(20) sys_user_id PK
        varchar(32) login_username
        varchar(32) telphone
        varchar(64) belong_info_id FK
        varchar(8) sys_type
        tinyint is_admin
        tinyint state
    }
    t_sys_user_auth {
        bigint(20) auth_id PK
        bigint(20) user_id FK
        tinyint identity_type
        varchar(128) identifier
        varchar(128) credential
    }
    t_sys_user_role_rela {
        bigint(20) user_id PK, FK
        varchar(32) role_id PK, FK
    }
    t_sys_config {
        varchar(50) config_key PK
        varchar(50) config_name
        varchar(50) group_key
        text config_val
    }
    t_sys_log {
        int(11) sys_log_id PK
        bigint(20) user_id FK
        varchar(8) sys_type
        varchar(128) method_name
        varchar(256) req_url
    }
    t_isv_info {
        varchar(64) isv_no PK
        varchar(64) isv_name
        varchar(32) isv_short_name
        tinyint state
    }
    t_mch_info {
        varchar(64) mch_no PK
        varchar(64) isv_no FK
        bigint(20) init_user_id FK
        varchar(64) mch_name
        tinyint type
        tinyint state
    }
    t_mch_app {
        varchar(64) app_id PK
        varchar(64) mch_no FK
        varchar(64) app_name
        varchar(128) app_secret
        tinyint state
    }
    t_pay_way {
        varchar(20) way_code PK
        varchar(20) way_name
    }
    t_pay_interface_define {
        varchar(20) if_code PK
        varchar(20) if_name
        tinyint is_mch_mode
        tinyint is_isv_mode
        json way_codes
        tinyint state
    }
    t_pay_interface_config {
        bigint(20) id PK
        tinyint info_type
        varchar(64) info_id FK
        varchar(20) if_code FK
        decimal(20,6) if_rate
        tinyint state
    }
    t_mch_pay_passage {
        bigint(20) id PK
        varchar(64) mch_no FK
        varchar(64) app_id FK
        varchar(20) if_code FK
        varchar(20) way_code FK
        decimal(20,6) rate
        tinyint state
    }
    t_pay_order {
        varchar(30) pay_order_id PK
        varchar(64) mch_no FK
        varchar(64) isv_no FK
        varchar(64) app_id FK
        varchar(20) if_code FK
        varchar(20) way_code FK
        varchar(64) mch_order_no
        bigint(20) amount
        decimal(20,6) mch_fee_rate
        bigint(20) mch_fee_amount
        tinyint state
        tinyint refund_state
        bigint(20) refund_amount
        tinyint division_mode
        tinyint division_state
    }
    t_refund_order {
        varchar(30) refund_order_id PK
        varchar(30) pay_order_id FK
        varchar(64) mch_no FK
        varchar(64) app_id FK
        varchar(20) if_code FK
        varchar(64) mch_refund_no
        bigint(20) refund_amount
        tinyint state
    }
    t_transfer_order {
        varchar(32) transfer_id PK
        varchar(64) mch_no FK
        varchar(64) app_id FK
        varchar(20) if_code FK
        varchar(64) mch_order_no
        varchar(20) entry_type
        bigint(20) amount
        tinyint state
    }
    t_mch_notify_record {
        bigint(20) notify_id PK
        varchar(64) order_id FK
        tinyint order_type
        varchar(64) mch_no FK
        varchar(64) app_id FK
        tinyint state
        int(11) notify_count
    }
    t_order_snapshot {
        varchar(64) order_id PK, FK
        tinyint order_type PK
        text mch_req_data
        text channel_resp_data
    }
    t_mch_division_receiver_group {
        bigint(20) receiver_group_id PK
        varchar(64) mch_no FK
        varchar(64) receiver_group_name
        tinyint auto_division_flag
    }
    t_mch_division_receiver {
        bigint(20) receiver_id PK
        bigint(20) receiver_group_id FK
        varchar(64) mch_no FK
        varchar(64) app_id FK
        varchar(20) if_code FK
        tinyint acc_type
        varchar(50) acc_no
        decimal(20,6) division_profit
        tinyint state
    }
    t_pay_order_division_record {
        bigint(20) record_id PK
        varchar(30) pay_order_id FK
        bigint(20) receiver_id FK
        bigint(20) receiver_group_id FK
        varchar(64) mch_no FK
        varchar(30) batch_order_id
        decimal(20,6) division_profit
        bigint(20) cal_division_amount
        tinyint state
    }
```
