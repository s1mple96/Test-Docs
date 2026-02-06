# BUG报告：【任货行】清欠还款回调钱包回正失败导致分表路由异常

## 1. BUG描述

清欠还款支付回调成功后，在执行"钱包回正"逻辑时，查询 `HCB_HLJ_HEIDEDUCRECORD` 分表报错：

```
java.lang.IllegalStateException: no table route info
### SQL: select count(1) from HCB_HLJ_HEIDEDUCRECORD 
where TRUCKUSER_ID = ? and OPERATOR_CODE = ? and STATUS != '1'
```

**影响**：
- ✅ 支付回调正常，钱包余额已加钱、冻结金额已解冻
- ❌ 钱包回正失败，无法自动触发"一键注销"流程
- ❌ 用户需手动登录小程序进行后续操作

---

## 2. 复现步骤

### 前置条件
- 车辆满足清欠还款条件
- 车牌号：蒙ZMM777
- 钱包ID：`7b4bfaa4a8de11f084f1141877512abb`

### 操作步骤

**Step 1**: 发起清欠还款支付

```bash
POST /hcbapi/app.do
{
  "truckUserId": "6457f83940ad4501bc2f01474093c8c4",
  "openId": "oTpKf4vchzE1-cTZtKxbEB_gKJ_A",
  "relativeurl": "com.hcb.debtRepaymentWechatPay",
  "caller": "chefuAPP",
  "timestamp": "1737690000000",
  "hashcode": "test123"
}
```

**Step 2**: 支付成功后，分米回调通知

```bash
POST /hcbapi/pay/notify/FEN_MI_APPLET_V2_PAY/DEBT_REPAYMENT_RECHARGE
Content-Type: application/json

{
  "merchantOrderNo": "P202601241013400000030",
  "orderAmount": 1.01,
  "orderStatus": "SUCCESS",
  "tradeOrderNo": "PAY202601241013310004",
  "paySuccessTime": "2026-01-24 10:13:31",
  "bankChannelCode": "YUN_ZHUO_APPLET_GZ_PAY",
  "actualPayee": "FM"
}
```

**Step 3**: 观察日志

```bash
tail -f /home/tomcat/tomcat-hcbapi/logs/catalina.out | grep "executionSueRepaymentCancel"
```

---

## 3. 预期结果

```
2026-01-24 10:26:02 - 执行<walletOperateBalanceAdd>开始  ✅
2026-01-24 10:26:02 - 方法:<walletOperateBalanceAdd>释放锁成功  ✅
2026-01-24 10:26:02 - 执行<walletOperateBalanceUnFreez>开始  ✅
2026-01-24 10:26:02 - 方法:<walletOperateBalanceUnFreez>释放锁成功  ✅
2026-01-24 10:26:02 - 清欠还款，钱包余额为正，触发一键注销  ✅
2026-01-24 10:26:02 - 查询未支付账单成功  ✅
2026-01-24 10:26:02 - 发送延迟消息成功，walletId: 7b4bfaa4a8de11f084f1141877512abb  ✅
```

**预期**：
- ✅ 钱包余额增加
- ✅ 冻结金额解冻
- ✅ 查询未支付账单成功
- ✅ 发送延迟MQ，1分钟后触发一键注销

---

## 4. 实际结果

```
2026-01-24 10:26:02 -1794767 [http-nio-8080-exec-4] - INFO    - 执行<walletOperateBalanceAdd>开始,申请加锁  ✅
2026-01-24 10:26:02 -1794792 [http-nio-8080-exec-4] - INFO    - 方法:<walletOperateBalanceAdd>释放锁成功  ✅
2026-01-24 10:26:02 -1794804 [http-nio-8080-exec-4] - INFO    - 执行<walletOperateBalanceUnFreez>开始,申请加锁  ✅
2026-01-24 10:26:02 -1794825 [http-nio-8080-exec-4] - INFO    - 方法:<walletOperateBalanceUnFreez>释放锁成功  ✅
2026-01-24 10:26:02 -1795035 [http-nio-8080-exec-4] - ERROR   - 执行注销逻辑异常，用户钱包id为:7b4bfaa4a8de11f084f1141877512abb  ❌

org.mybatis.spring.MyBatisSystemException: nested exception is org.apache.ibatis.exceptions.PersistenceException: 
### Error querying database.  Cause: java.lang.IllegalStateException: no table route info
### The error may exist in file [/home/tomcat/tomcat-hcbapi/webapps/hcbapi/WEB-INF/classes/mybatis/truck/HeiDeducRecordMapper.xml]
### The error may involve HeiDeducRecordMapper.countUnpaidBill-Inline
### The error occurred while setting parameters
### SQL: select count(1) as num from HCB_HLJ_HEIDEDUCRECORD where TRUCKUSER_ID = ? and OPERATOR_CODE = ? and STATUS != '1'
### Cause: java.lang.IllegalStateException: no table route info
	at com.hcb.api.service.truck.heideducrecord.impl.HeiDeducRecordService.checkUnpaidBill(HeiDeducRecordService.java:615)
	at com.hcb.api.service.truck.truckuser.impl.TruckUserService.executionSueRepaymentCancel(TruckUserService.java:5793)
	at com.hcb.api.strategy.impl.orderComplete.DebtRepaymentRechargeCompleteImpl.completeSuccessOrder(DebtRepaymentRechargeCompleteImpl.java:133)
```

**实际**：
- ✅ 钱包余额已增加
- ✅ 冻结金额已解冻
- ❌ 查询 `HCB_HLJ_HEIDEDUCRECORD` 分表失败，报错 `no table route info`
- ❌ 钱包回正流程中断，未发送延迟MQ

---

**日志路径**: `/home/tomcat/tomcat-hcbapi/logs/catalina.out`
