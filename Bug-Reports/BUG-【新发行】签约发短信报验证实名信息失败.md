# BUG-【新发行】签约发短信报验证实名信息失败

## 1. BUG描述

调用代扣签约发短信接口时，接口返回 500，提示「验证实名信息失败」，无法完成签约发短信，浙商银行测试数据（储蓄卡、二类卡）均报同样错误。

**影响**：
- ❌ 申办场景下无法完成绑卡发短信（注册会员失败）
- ❌ 浙商银行联调测试无法进行签约流程

---

## 2. 复现步骤

### 前置条件
- 测试环境: 192.168.1.60
- 系统: 新发行（分米ETC）
- 数据库: fenmi_etc、fenmi_pay
- 测试数据: 吕玲 / 511529196111020062 / 13173765674；银行卡 622309331002366579 或 6223091910000008252

### 操作步骤

**Step 1**: 调用签约发短信接口

```http
POST /fenmi/etc/app/withhold/signSendSms
Content-Type: application/json

{
  "holderIdCode": "511529196111020062",
  "holderName": "吕玲",
  "holderPhoneNumber": "13173765674",
  "bankCardNo": "622309331002366579",
  "bizSource": "2"
}
```

**Step 2**: 观察接口返回

```json
{
  "code": 500,
  "msg": "验证实名信息失败",
  "data": null,
  "success": false
}
```

**Step 3**: 查看 ETC 服务日志

```bash
tail -n 200 /home/fenmi/fenmi-etc/log.txt | grep -E "registerMember|511529196111020062|验证实名"
```

**日志片段**：
```
注册分米合单会员失败,原因,vo:EtcRegisterWithholdMemberVO(userMemberNo=null, registerStatus=FAIL, resMsg=验证实名信息失败), holderIdCode:511529196111020062, holderName:吕玲, holderPhoneNumber:13173765674, bizSource:2
```

---

## 3. 预期结果

- 注册会员成功，签约发短信接口返回 200，返回 signApplyId、verifyCodeNo，用户可收到短信验证码并继续签约验证。

**预期**：
- ✅ 接口返回 200，data 中包含 signApplyId、verifyCodeNo
- ✅ 用户收到银行短信验证码

---

## 4. 实际结果

- 接口返回 500，msg 为「验证实名信息失败」；注册会员未成功，未进入发短信环节。

**实际**：
- ❌ 接口返回 500，msg: "验证实名信息失败"
- ❌ ETC 日志：注册分米合单会员失败，resMsg=验证实名信息失败
- ❌ 储蓄卡（622309331002366579）、二类卡（6223091910000008252）均报相同错误

**ETC 错误日志**：
```
com.fenmi.common.core.exception.ServiceException: 验证实名信息失败
	at com.fenmi.etc.service.user.impl.UserWithholdServiceImpl.registerMember(UserWithholdServiceImpl.java:111)
```

---

**日志路径**: `/home/fenmi/fenmi-etc/log.txt`（ETC）；支付侧请求平安付的日志可查 MongoDB 或 fenmi-pay 日志（平安付 C0001.A5170 域账户注册）
