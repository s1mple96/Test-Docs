# BUG-【任货行】申办后广州工行直接支付开户失败，分米支付返回"服务未找到"

## 1. BUG描述

用户在任货行小程序完成 ETC 申办（支付成功）后，系统应自动触发广州工行直接支付开户流程，但 hcbapi 向分米支付新系统发起开户请求后，分米支付返回"服务未找到"，导致 `hcb_bank_account` 记录的 `account_status` 被更新为 `5`（失败），用户无法完成广州工行直接支付开户，小程序无任何提示。

**影响**：
- ✅ 申办主流程（提交资料、支付预存款）正常完成
- ❌ 广州工行直接支付开户接口调用失败，返回"服务未找到"
- ❌ `hcb_bank_account` 开户状态更新为失败（`account_status=5`）
- ❌ 小程序侧用户无感知，未收到任何广州工行签约相关提示

---

## 2. 复现步骤

### 前置条件
- 测试环境：`192.168.1.60`
- 系统：任货行小程序
- 数据库：`hcb`
- 测试账号手机号：`15818376788`
- 测试身份证号：`441622199602075195`
- 测试车牌：`苏ZSB678`（订单时间：`2026-03-19 17:21~17:24`）
- 前置数据状态：`hcb_bank_account` 中该身份证下不存在 `account_status` 为 `1`（正常）或 `4`（开户中）的广州工行账户记录

### 操作步骤

**Step 1**：使用任货行小程序，以身份证号 `441622199602075195` 完成一辆新车 `苏ZSB678` 的 ETC 申办并支付预存款（`0.02元`，支付方式：云卓微信广州银行）

**Step 2**：申办支付成功后，查询 `hcb_bank_account` 确认开户状态

```sql
SELECT id, bank_code, account_status, reason, member_no, open_account_order_no, mch_order_no, create_time, update_time
FROM hcb_bank_account
WHERE id_code = '441622199602075195'
ORDER BY id DESC
LIMIT 5;
```

**数据查询结果**：
```
id:                   2034560945413640192
bank_code:            GZ_ICBC_ZJ_ACCOUNT
account_status:       5（失败）
reason:               服务未找到
member_no:            NULL
open_account_order_no: NULL
mch_order_no:         05FC999404E745B099DB372A42173340
create_time:          2026-03-19 17:21:44
update_time:          2026-03-19 17:22:36
```

**Step 3**：查看 hcbapi 日志

```bash
grep -an 'openGzAccount\|GZ_ICBC_ZJ\|05FC999404E745B099DB372A42173340' \
  /home/tomcat/tomcat-hcbapi/logs/catalina.out | grep '2026-03-19'
```

---

## 3. 预期结果

**预期**：
- ✅ 申办完成后 hcbapi 调用分米支付新系统开户接口，分米支付返回开户成功（或开户中）响应
- ✅ `hcb_bank_account` 中 `account_status` 更新为 `4`（开户中）或 `1`（正常），`member_no` 和 `open_account_order_no` 有值
- ✅ 小程序展示广州工行直接支付签约状态

---

## 4. 实际结果

hcbapi 日志（`catalina.out` 第 `4606733~4606750` 行）显示：

```
2026-03-19 17:21:44 [http-nio-8080-exec-4] ERROR
货车广州工行直接支付开户打印开户请求参数:
{
  "bankAccountCode": "GZ_ICBC_ZJ_ACCOUNT",
  "mchOrderNo": "20BB46A861DF428C8FBDB5D6076BAB71",
  "idCode": "441622199602075195",
  "accountName": "骆志敏",
  "bankCardNo": "6222034000046037473",
  "bankCode": "STKLYpro57",
  "bankName": "中国工商银行",
  "phoneNumber": "15818376788",
  "notifyUrl": "http://788360p9o5.yicp.fun/hcbapi/bank/newSystemOpenAccountNotify",
  "merchantNo": "RHX00001",
  "fenMiAppCode": "RHX",
  ...
}

2026-03-19 17:21:55 [http-nio-8080-exec-4] ERROR
货车广州工行直接支付开户打印开户返回参数：
（返回内容与请求参数完全一致，无 code/success/data 字段，无开户结果）

2026-03-19 17:22:35 [http-nio-8080-exec-4] ERROR  （第二次请求，mchOrderNo 更新为 05FC999404...）
货车广州工行直接支付开户打印开户返回参数：
（同上，仍无有效响应）
```

**实际**：
- ❌ 分米支付新系统接口返回"服务未找到"（接口未就绪或路由不通），未返回标准 `code/success/data` 响应体
- ❌ hcbapi 连续发起两次开户请求，均失败，最终将 `account_status` 更新为 `5`，`reason` 写入"服务未找到"
- ❌ 小程序无任何广州工行签约提示，用户无感知

---

**日志路径**：`/home/tomcat/tomcat-hcbapi/logs/catalina.out`（关键行号：`4606719` ~ `4606750`，时间段 `2026-03-19 17:21~17:22`）
