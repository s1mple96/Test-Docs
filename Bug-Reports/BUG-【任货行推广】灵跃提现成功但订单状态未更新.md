# BUG-【任货行推广】灵跃提现成功但订单状态未更新

## 1. BUG描述

用户发起灵跃渠道提现，实际打款已成功（钱已到账），但系统订单状态停留在"打款中"，未更新为"打款成功"。

**影响**：
- ✅ 提现功能正常，用户资金正常到账
- ✅ 前端可以正常发起提现申请
- ❌ 订单状态未自动更新，停留在PAYING状态
- ❌ 灵跃回调通知处理失败（连续3次回调都失败）

---

## 2. 复现步骤

### 前置条件
- 测试环境：http://192.168.1.60:9205/fenmi-pay
- 测试账号：15818376788（骆志敏）
- 身份证号：441622199602075195
- 测试订单号：`WITHDRAWRHXTG202602021631380001`
- 灵跃批次号：`110202602028240885768978455`
- 提现渠道：`LING_YUE_ACCOUNT_PAY_test`（灵跃测试环境）

### 操作步骤

**Step 1**: 用户发起提现

```json
// 请求接口：com.hcb.withdrawApply
{
  "userId": "f4cbdc75703240449b51fee8d4dc5576",
  "openId": "o25r74s_Tz0pRYDuInlpyPSCurN4",
  "type": "2",
  "amount": "1",
  "verifyCode": "4191",
  "bankCardId": "5451a39046364a6c83534d1043235055"
}

// 响应
{
  "ret": "1",
  "msg": "账户转帐下单成功"
}
```

**Step 2**: 等待约2分钟后，用户确认钱已到账（银行卡到账0.92元）

**Step 3**: 查询订单状态

```bash
# 查询 fenmi_pay 订单表
mysql -h192.168.1.60 -uroot -pfm123456 fenmi_pay -e "
SELECT trade_order_no, order_status, pay_success_time, whithdraw_amount, create_time 
FROM withdraw_trade_order 
WHERE trade_order_no='WITHDRAWRHXTG202602021631380001'"

# 查询 hcb 提现记录表
mysql -h192.168.1.60 -uroot -pfm123456 hcb -e "
SELECT remit_trx_no, withdraw_status, remit_success_time, remit_amount 
FROM hcb_bank_withdraw_record 
WHERE remit_trx_no='CHANNEL202602021631360000003'"
```

**数据查询结果**：
```
# fenmi_pay.withdraw_trade_order
trade_order_no: WITHDRAWRHXTG202602021631380001
order_status: PAYING  ❌（应该是SUCCESS）
pay_success_time: NULL  ❌（应该有时间）
whithdraw_amount: 0.92
create_time: 2026-02-02 16:31:38

# hcb.hcb_bank_withdraw_record
remit_trx_no: CHANNEL202602021631360000003
withdraw_status: REMITTING  ❌（应该是REMIT_SUCCESS）
remit_success_time: NULL  ❌（应该有时间）
remit_amount: 0.92
```

**Step 4**: 查看日志

```bash
# 查看灵跃回调日志
grep '110202602028240885768978455' /home/fenmi/fenmi-pay/log.txt | tail -20
```

---

## 3. 预期结果

**数据库状态**：
```
# fenmi_pay.withdraw_trade_order
order_status: SUCCESS  ✅
pay_success_time: 2026-02-02 16:33:11  ✅

# hcb.hcb_bank_withdraw_record
withdraw_status: REMIT_SUCCESS  ✅
remit_success_time: 2026-02-02 16:33:11  ✅
```

**日志输出**：
```
[INFO] - 灵跃账户转账回调处理成功
[INFO] - 订单状态更新为SUCCESS
[INFO] - 打款成功时间: 2026-02-02 16:33:11
```

**预期**：
- ✅ 灵跃回调通知正常处理
- ✅ 订单状态自动更新为SUCCESS/REMIT_SUCCESS
- ✅ 记录打款成功时间

---

## 4. 实际结果

**日志输出**：
```
2026-02-02 16:33:11.597 [http-nio-9205-exec-3] ERROR c.f.c.b.w.b.LingYueAccountPaidImpl - [accountTransPaidVerify,291] - [4ea4db22de251025,6a3c4d97cc30fa8f] - 灵跃账户转帐回调解密异常：OrgDataStr:{"biz_content":"{\"recvType\":\"BANK\",\"custBatchNo\":\"BWITHDRAWRHXTG202602021631380001\",\"totalDeduction\":0.98,\"batchTotalAgentFeeAmt\":0.00,\"batchServFeeAmt\":0.06,\"batchAmt\":0.92,\"batchStatus\":1,\"platBatchNo\":\"110202602028240885768978455\"}","method":"callBack","sign":"W7x3M4sPX24slrShJrES1Dr1Gs8MCJJYhZfEEAIe+VTKTQ7v63iFvO0G+w/untvHOt3IsT8mfjBEyH5yQpoa2pqsCso6S7FXGgSfpTFHazJii10YzlVmQaUNmYWXo87MLfEJHwWzSGcufUhNf41CiWqv6ufBhJeGbF/Qu048bzOYKoNjTNf7djOI1MFNr4ede3rAi8TDTaOuZQ8m3NjHzUxDRnPwqMCOsl2YgjiH2p3qUeg+QFUdntz6yKi/Lveq9lI2HFmnOxD/h06JplozzndYte31Bf8GLqTD6DxXlkFwOUs2lh8wzzVOR8WKyUGLXV2bvQmQxhyucsu2SiLxWA==","merchant_request_no":"add3ba5622fd40b68297e314544df67f","sign_type":"RSA2","version":"1.0","timestamp":"2026-02-02"},灵鹊云灵活用工回调数据为空！

2026-02-02 16:36:26.969 [http-nio-9205-exec-7] ERROR c.f.c.b.w.b.LingYueAccountPaidImpl - [accountTransPaidVerify,291] - [60d429c901172847,d5e8ed4d13b43d55] - 灵跃账户转帐回调解密异常：OrgDataStr:{"biz_content":"{\"recvType\":\"BANK\",\"custBatchNo\":\"BWITHDRAWRHXTG202602021631380001\",\"totalDeduction\":0.98,\"batchTotalAgentFeeAmt\":0.00,\"batchServFeeAmt\":0.06,\"batchAmt\":0.92,\"batchStatus\":1,\"platBatchNo\":\"110202602028240885768978455\"}","method":"callBack","sign":"ErHOux+WNG9+i/++Mjjaz0A3VtAGpLuKS8/xh6c8YNxqnolo+UrYPUKsOXKgXMpTFyjlvLiW5Pjv5ASO+9nSsI3gUeGFT3pl3eM40GxP+TgBpDvN+nzwJEBqxDMzJGN7QypKIcJ+VggldvFBFptoGqRnvGQlqTrrWDESxHI0lnKW6ZkxEsLE4kEHCPhd5SPTIx15fUy9oJ6Fz7AioWX2+OM6bEy28UyiO4ZpfHmCevBtDo8MoX0AwTqS8wJJvaoIqJqgpC1osrESXG+m9Dw2htko2h+OA/wDclJ/7q4sJg10KjsVNoKkvprTmwQbr6TwR8lizsp/ziGzEFgue+fMgA==","merchant_request_no":"ab6311bb9c5b40c5b47f11518b0a4b0a","sign_type":"RSA2","version":"1.0","timestamp":"2026-02-02"},灵鹊云灵活用工回调数据为空！

2026-02-02 16:40:28.513 [http-nio-9205-exec-5] ERROR c.f.c.b.w.b.LingYueAccountPaidImpl - [accountTransPaidVerify,291] - [d5c595d46334ad93,6dffde8c6eabf506] - 灵跃账户转帐回调解密异常：OrgDataStr:{"biz_content":"{\"recvType\":\"BANK\",\"custBatchNo\":\"BWITHDRAWRHXTG202602021631380001\",\"totalDeduction\":0.98,\"batchTotalAgentFeeAmt\":0.00,\"batchServFeeAmt\":0.06,\"batchAmt\":0.92,\"batchStatus\":1,\"platBatchNo\":\"110202602028240885768978455\"}","method":"callBack","sign":"GurnyFDr9VFhM7YJbpuASakNPSOg/mi+gGlAy5zjvi7SrTH3VNMFhoKRGqXAiWpCIIgbo9954VJitDzM6NPvjkZOtu8uXMfF0GQmL9lMOKHz67KLILZrAbEyivjfIJCL3rOOy470eLHx8yzyo9bMhHBHc9wOWEtvjBiIYIZXooevRRrZKK67RrXFT8awHgVrsSyI4y/pkekB/FKT1J1J1TnbQ5xV3RWs2wm+4OrerqSpgcR0Xk2nXwJdszjc5EoTAfkfAL7RxIZTv/V/J8nsif92NoU/RJL41OMlD7xHdiHXdoDEhMylA7iMpHbbpHJjOmYiQX4ABAvRB+NFeImHbw==","merchant_request_no":"84963f8f68fe45528c973f35d74ae36a","sign_type":"RSA2","version":"1.0","timestamp":"2026-02-02"},灵鹊云灵活用工回调数据为空！
```

**回调数据内容**（从日志中可见）：
```json
{
  "batchStatus": 1,  // 1=成功
  "custBatchNo": "BWITHDRAWRHXTG202602021631380001",
  "platBatchNo": "110202602028240885768978455",
  "batchAmt": 0.92,
  "batchServFeeAmt": 0.06,
  "totalDeduction": 0.98
}
```

**实际**：
- ❌ 灵跃发送了3次回调通知（16:33, 16:36, 16:40）
- ❌ 所有回调都处理失败，报错"灵鹊云灵活用工回调数据为空"
- ❌ 订单状态停留在PAYING/REMITTING
- ❌ pay_success_time/remit_success_time 为NULL
- ✅ 但用户资金实际已到账（0.92元）

---

**日志路径**: `/home/fenmi/fenmi-pay/log.txt`
