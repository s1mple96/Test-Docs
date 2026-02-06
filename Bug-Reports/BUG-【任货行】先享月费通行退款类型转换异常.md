# BUG-【任货行】先享月费通行退款类型转换异常

## 1. BUG描述

账单计费任务执行后，触发先享月费通行退款时抛出类型转换异常，导致先享月费账单无法自动退款。

**影响**：
- ✅ 账单计费任务正常完成
- ✅ 账单扣款任务正常执行
- ❌ 先享月费通行自动退款失败
- ❌ 所有运营商的先享月费退款均受影响

---

## 2. 复现步骤

### 前置条件
- 测试环境：http://192.168.1.60:8087/hcbTimer-bill
- 测试运营商：MTK（蒙通卡）
- 测试账单：蒙ZMM777，账单日期2026-01-28，金额1.00元
- 测试账号：骆志敏（441622199602075195）
- 钱包ID：`7b4bfaa4a8de11f084f1141877512abb`
- 先享月费账单ID：`1991417185075343361`, `1991417185075343362`

### 操作步骤

**Step 1**: 执行账单计费任务

```bash
curl -X GET "http://192.168.1.60:8087/hcbTimer-bill/test/deductBillCalculatTask"
```

**Step 2**: 系统自动触发先享月费通行退款MQ消息

```
MQ队列：monthlyPayLaterWalletBillRefundQueue
消息内容：{"walletId":"7b4bfaa4a8de11f084f1141877512abb","deductId":"1223640129399169025"}
```

**Step 3**: 查看日志

```bash
tail -n 100 /home/tomcat/tomcat-hcbTimer-bill/logs/catalina.out | grep -A 10 "先享月费"
```

---

## 3. 预期结果

```
[INFO] - 先享月费账单通行退款监听 收到消息 msg:{"walletId":"xxx","deductId":"xxx"}
[INFO] - 开始处理先享月费退款 id:1991417185075343361
[INFO] - 查询运营商开关配置成功 operatorCode:MTK
[INFO] - 先享月费退款成功 id:1991417185075343361
[INFO] - 更新账单状态为已退款
```

**预期**：
- ✅ 查询运营商配置成功
- ✅ 判断退款开关状态
- ✅ 先享月费账单自动退款成功
- ✅ 钱包余额减少，账单状态更新为已退款

---

## 4. 实际结果

```
2026-01-29 14:30:51 [monthlyPayLaterBillRefund-1] ERROR   - 先享月费账单通行退款监听 收到消息 msg:{"walletId":"7b4bfaa4a8de11f084f1141877512abb","deductId":"1223640129399169025"}
2026-01-29 14:30:51 [monthlyPayLaterBillRefund-1] ERROR   - MonthlyPaylaterWalletServiceImpl_monthlyPaylaterBillRefund_先享月费退款异常_id:1991417185075343361
java.lang.IllegalArgumentException: Can not set java.lang.Integer field com.etc.api.entity.operator.Operator.profitSharingProportion to java.lang.String
	at sun.reflect.UnsafeFieldAccessorImpl.throwSetIllegalArgumentException(UnsafeFieldAccessorImpl.java:167)
	at sun.reflect.UnsafeFieldAccessorImpl.throwSetIllegalArgumentException(UnsafeFieldAccessorImpl.java:171)
	at sun.reflect.UnsafeObjectFieldAccessorImpl.set(UnsafeObjectFieldAccessorImpl.java:81)
	at java.lang.reflect.Field.set(Field.java:764)
	at com.etc.api.util.PageDataUtil.convertTo(PageDataUtil.java:77)
	at com.etc.api.service.truck.operator.impl.OperatorService.findBy(OperatorService.java:152)
	at com.etc.api.service.truck.operator.impl.OperatorService.isMonthlyPaylaterRefund(OperatorService.java:171)
	at com.etc.api.bizService.monthly.impl.MonthlyPaylaterWalletBillBizServiceImpl.monthlyPayLaterBillRefund(MonthlyPaylaterWalletBillBizServiceImpl.java:162)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at org.springframework.messaging.handler.invocation.InvocableHandlerMethod.doInvoke(InvocableHandlerMethod.java:181)
	at org.springframework.messaging.handler.invocation.InvocableHandlerMethod.invoke(InvocableHandlerMethod.java:114)
	at org.springframework.amqp.rabbit.listener.adapter.HandlerAdapter.invoke(HandlerAdapter.java:53)
	at org.springframework.amqp.rabbit.listener.adapter.MessagingMessageListenerAdapter.invokeHandler(MessagingMessageListenerAdapter.java:189)
	at org.springframework.amqp.rabbit.listener.adapter.MessagingMessageListenerAdapter.onMessage(MessagingMessageListenerAdapter.java:125)
	at org.springframework.amqp.rabbit.listener.AbstractMessageListenerContainer.doInvokeListener(AbstractMessageListenerContainer.java:1552)
	at org.springframework.amqp.rabbit.listener.AbstractMessageListenerContainer.actualInvokeListener(AbstractMessageListenerContainer.java:1478)
	at org.springframework.amqp.rabbit.listener.AbstractMessageListenerContainer.invokeListener(AbstractMessageListenerContainer.java:1466)
	at org.springframework.amqp.rabbit.listener.AbstractMessageListenerContainer.doExecuteListener(AbstractMessageListenerContainer.java:1461)
	at org.springframework.amqp.rabbit.listener.AbstractMessageListenerContainer.executeListener(AbstractMessageListenerContainer.java:1405)
	at org.springframework.amqp.rabbit.listener.SimpleMessageListenerContainer.doReceiveAndExecute(SimpleMessageListenerContainer.java:958)
	at org.springframework.amqp.rabbit.listener.SimpleMessageListenerContainer.receiveAndExecute(SimpleMessageListenerContainer.java:915)
	at org.springframework.amqp.rabbit.listener.SimpleMessageListenerContainer.access$1600(SimpleMessageListenerContainer.java:83)
	at org.springframework.amqp.rabbit.listener.SimpleMessageListenerContainer$AsyncMessageProcessingConsumer.mainLoop(SimpleMessageListenerContainer.java:1290)
	at org.springframework.amqp.rabbit.listener.SimpleMessageListenerContainer$AsyncMessageProcessingConsumer.run(SimpleMessageListenerContainer.java:1196)
	at java.lang.Thread.run(Thread.java:748)

2026-01-29 14:30:51 [monthlyPayLaterBillRefund-1] ERROR   - MonthlyPaylaterWalletServiceImpl_monthlyPaylaterBillRefund_先享月费退款异常_id:1991417185075343362
java.lang.IllegalArgumentException: Can not set java.lang.Integer field com.etc.api.entity.operator.Operator.profitSharingProportion to java.lang.String
	at sun.reflect.UnsafeFieldAccessorImpl.throwSetIllegalArgumentException(UnsafeFieldAccessorImpl.java:167)
	at sun.reflect.UnsafeFieldAccessorImpl.throwSetIllegalArgumentException(UnsafeFieldAccessorImpl.java:171)
	at sun.reflect.UnsafeObjectFieldAccessorImpl.set(UnsafeObjectFieldAccessorImpl.java:81)
	at java.lang.reflect.Field.set(Field.java:764)
	at com.etc.api.util.PageDataUtil.convertTo(PageDataUtil.java:77)
	at com.etc.api.service.truck.operator.impl.OperatorService.findBy(OperatorService.java:152)
	at com.etc.api.service.truck.operator.impl.OperatorService.isMonthlyPaylaterRefund(OperatorService.java:171)
```

**实际**：
- ❌ 查询运营商配置时抛出 `IllegalArgumentException: Can not set java.lang.Integer field to java.lang.String`
- ❌ 先享月费退款中断，未执行退款操作
- ❌ 受影响账单：`1991417185075343361`, `1991417185075343362`
- ❌ 所有运营商的先享月费退款均受影响

---

**日志路径**: `/home/tomcat/tomcat-hcbTimer-bill/logs/catalina.out`
