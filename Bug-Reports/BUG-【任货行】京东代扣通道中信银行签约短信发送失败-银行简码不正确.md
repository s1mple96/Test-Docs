# BUG-【任货行】京东代扣通道中信银行签约短信发送失败-银行简码不正确

## 1. BUG描述

任货行小程序绑卡页，选择已配置京东代扣通道（`JD_QUICK_PAY`）的中信银行，点击发送签约短信时，返回失败提示"签约短信发送失败：银行简码不正确"，无法完成代扣签约流程。

**影响**：
- ❌ 中信银行信用卡（卡 BIN：`622688`）通过京东代扣通道发送签约短信失败
- ✅ 其他已正常配置银行简码的银行卡签约短信不受影响

---

## 2. 复现步骤

### 前置条件
- 测试环境: 192.168.1.60
- 系统: 任货行小程序（chefuAPP）
- 数据库: hcb、fenmi_admin
- 测试账号: truckUserId=`23744c9f063a401b8a9ffea003ca02d3`
- 测试数据:
  - 银行卡号: `6226881012353932`（中信银行信用卡）
  - 身份证: `441424199202126952`
  - 手机号: `18929382525`
  - 银行组织: id=`10`（中信银行，`bank_channel_code=JD_QUICK_PAY`）

### 操作步骤

**Step 1**: 进入任货行小程序绑卡页，选择中信银行，填写持卡人信息，点击"发送验证码"

```json
{
  "bankCardNo": "6226881012353932",
  "bankCardName": "中信银行",
  "bankChannelCode": "JD_QUICK_PAY",
  "bankOrganizeId": "10",
  "bankCardType": "02",
  "holderName": "江东威",
  "holderIdCard": "441424199202126952",
  "phoneNumber": "18929382525",
  "sceneType": "1",
  "relativeurl": "com.hcb.bank.sendSmsV2"
}
```

**Step 2**: 查询 `hcb.hcb_bank_organize` 中信银行配置

```sql
SELECT id, bank_name, bank_code, bank_channel_code, bank_channel_name, status
FROM hcb_bank_organize WHERE bank_code='CNCB';
```

**数据查询结果**:
```
id: 10
bank_name: 中信银行
bank_code: CNCB
bank_channel_code: JD_QUICK_PAY
bank_channel_name: 京东代扣
status: 0（启用）
```

**Step 3**: 查询 `fenmi_admin.bank_code_info` 中 BIN `622688` 的配置记录

```sql
SELECT id, bank_name, bank_code, card_no_prefix, card_type
FROM bank_code_info WHERE card_no_prefix LIKE '622688%';
```

**数据查询结果**:
```
id: 1440969040049868800  bank_name: 不支持银行  bank_code: CEB   card_no_prefix: 622688  card_type: 02
id: 1440969040456716289  bank_name: 中信银行    bank_code: CNCB  card_no_prefix: 622688  card_type: 02
```

**Step 4**: 观察任货行日志

```
2026-03-04 10:40:18 [http-nio-8080-exec-1] ERROR 代扣签约短信发送异常：银行简码不正确
2026-03-04 10:41:33 [http-nio-8080-exec-6] ERROR 代扣签约短信发送异常：银行简码不正确
```

---

## 3. 预期结果

```
[INFO] - 签约短信发送成功
[INFO] - 手机 18929382525 收到京东代扣签约验证码短信
```

**预期**：
- ✅ 调用分米支付签约短信接口成功
- ✅ `fenmi_pay.bank_quick_sign_apply` 新增一条记录，`pay_channel_code=JING_DONG_QUICK_PAY`，`sign_apply_status=1`
- ✅ 前端返回 `ret=1`，用户手机收到京东签约短信

---

## 4. 实际结果

```
[ERROR] - 代扣签约短信发送异常：银行简码不正确
com.hcb.api.util.exception.CustomException: 银行简码不正确
    at com.hcb.api.service.truck.bank.impl.BankQuickPayV2ServiceImpl.sendSms(BankQuickPayV2ServiceImpl.java:112)
    at com.hcb.api.service.truck.bank.impl.BankUserQuickPayServiceImpl.signSendSmsV2(BankUserQuickPayServiceImpl.java:659)
    at com.hcb.api.service.truck.bank.impl.BankUserQuickPayServiceImpl.addNewCardSendSmsV2(BankUserQuickPayServiceImpl.java:257)
    at com.hcb.api.service.truck.bank.impl.BankUserQuickPayServiceImpl.sendSmsV2(BankUserQuickPayServiceImpl.java:131)
```

**前端返回**：
```json
{
  "ret": "0",
  "msg": "签约短信发送失败：银行简码不正确",
  "data": null
}
```

**实际**：
- ❌ 签约短信发送失败，用户无法通过中信银行信用卡开通京东代扣
- ❌ `fenmi_pay.bank_quick_sign_apply` 无新增记录（分米侧未收到请求）
- ❌ `fenmi_admin.bank_code_info` 中 BIN `622688` 存在两条记录，`bank_code` 分别为 `CEB`（不支持银行）和 `CNCB`（中信银行）

---

**日志路径**:
- 任货行: `/home/tomcat/tomcat-hcbapi/logs/catalina.out`
- 分米支付: `/home/fenmi/fenmi-pay/log.txt`
