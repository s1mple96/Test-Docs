# BUG-【任货行】清欠还款并发重复支付

## 【BUG描述】
清欠还款功能在并发场景下存在漏洞，同一钱包在短时间内可以创建多笔还款订单并成功支付，导致用户重复充值，资金损失。

**影响范围**：所有清欠还款用户  
**严重程度**：🔴 高危（涉及资金安全）  
**复现频率**：高（并发场景必现）

## 【前置条件】
- **系统**：任货行小程序
- **用户**：满足清欠还款条件的钱包
  - 余额 < 0（有欠款）
  - 冻结金额 < 欠款绝对值
  - 有 PURGE_Wallet 标签
  - 还款金额 > 0
- **场景**：多人同时点击"立即还款"按钮

## 【复现步骤】

### 方式1：实际并发复现
1. 准备测试钱包：蒙ZMM777
2. 两台设备同时登录该车辆账号
3. 同时点击"立即还款"按钮（间隔 < 1秒）
4. 观察支付结果和订单记录

### 方式2：代码层面分析
1. 查看 `TopupRecordService.java:1140-1207` 的 `debtRepaymentWechatPay` 方法
2. 分析并发控制逻辑：
   ```
   T1: 请求1 获取锁成功
   T2: 请求1 调用 isPayment() 返回 false（没有 SUCCESS/PAYING 订单）
   T3: 请求1 调用 mixedPay() 创建订单（状态=NOT_PAY）
   T4: 请求1 发起微信支付请求（耗时 1-3秒）
   T5: 请求1 释放锁
   ──────────────────────────────────────
   T6: 请求2 获取锁成功
   T7: 请求2 调用 isPayment() 返回 false ❌（订单状态仍为 NOT_PAY）
   T8: 请求2 调用 mixedPay() 创建第2个订单 ❌
   T9: 请求2 发起微信支付请求
   ```
3. 查看 `isPayment()` 方法（1215-1224行）
4. 发现只查询 `SUCCESS` 和 `PAYING` 状态，**漏掉了 `NOT_PAY` 状态**

## 【期望结果】
1. ✅ 同一钱包在10分钟内只能创建1笔清欠还款订单
2. ✅ 第二个并发请求应返回："您的支付已成功提交，系统正在处理中，请勿重复操作!"
3. ✅ 数据库中该钱包只有1笔清欠还款订单记录
4. ✅ 用户只支付1次

## 【实际结果】
1. ❌ 同一钱包在4秒内创建了2笔清欠还款订单
2. ❌ 两笔订单都支付成功
3. ❌ 用户被重复扣款

**实际订单记录**：

| 时间 | 车牌号 | 金额 | 充值类型 | 状态 | 订单号 |
|------|--------|------|---------|------|--------|
| 2026-01-23 10:12:55 | 蒙ZMM777-主动充值 | 0.02 | 清欠充值 | 微信-支付成功 | T20260123101259000009 |
| 2026-01-23 10:12:59 | 蒙ZMM777-主动充值 | 0.02 | 清欠充值 | 微信-支付成功 | T20260123101254000007 |

**时间差**：4秒  
**重复支付金额**：0.02元 × 2 = 0.04元

## 【附件】

### 1. 订单记录截图
```
2026-01-23    蒙ZMM777-主    0.02    清欠    T20260123101259000009    P20260123101259000010    徽商    441622199602075195    微信    支付    0.00    分    2026-01-23
10:12:59      动充值                充值                                                        敏                            成功            来    10:12:59

2026-01-23    蒙ZMM777-主    0.02    清欠    T20260123101254000007    P20260123101254000008    徽商    441622199602075195    微信    支付    0.00    分    2026-01-23
10:12:55      动充值                充值                                                        敏                            成功            来    10:12:55
```

### 2. 问题代码定位

**文件**：`java/hcb/hcbapi/src/main/java/com/hcb/api/service/information/topuprecord/impl/TopupRecordService.java`

**问题方法1**：`isPayment()` - 1215:1224行

```java
private boolean isPayment(String truckUserWalletId) {
    Date endDate = new Date();
    TradeOrder queryTradeOrder = new TradeOrder();
    queryTradeOrder.setUserWalletId(truckUserWalletId);
    queryTradeOrder.setStartCreateTime(DateUtil.offsetMinute(endDate, -10));
    queryTradeOrder.setEndCreateTime(endDate);
    
    // ❌ 问题：只查询 SUCCESS 和 PAYING 状态，漏掉 NOT_PAY
    queryTradeOrder.setOrderStatusList(Lists.newArrayList(
        OrderStatusState.SUCCESS.name(), 
        OrderStatusState.PAYING.name()
    ));
    
    queryTradeOrder.setBizType(OrderBizType.DEBT_REPAYMENT_RECHARGE.name());
    return tradeOrderService.countTradeOrderByCondition(queryTradeOrder) > 0;
}
```

**问题方法2**：`mixedPay()` - TradeOrderPayManagerServiceImpl.java:156行

```java
// 创建订单时，初始状态为 NOT_PAY
tradeOrder = assembleTradeOrder(dto, idCode, userName);
tradeOrderBizService.createOrder(tradeOrder);  // 此时订单状态 = NOT_PAY

// 组装订单参数时设置状态
order.setOrderStatus(OrderStatusState.NOT_PAY.name());  // 568-574行
```

## 【原因定位】

### 根本原因
**并发防重校验逻辑不完整**，`isPayment()` 方法只查询 `SUCCESS` 和 `PAYING` 状态的订单，**遗漏了 `NOT_PAY` 状态**。

### 漏洞利用时间窗口
```
订单生命周期：NOT_PAY → PAYING → SUCCESS/FAIL
              ↑
              ▼
        【漏洞窗口】：1-3秒
        在此期间并发请求可绕过校验
```

### 时间线分析

| 时间点 | 请求1 | 请求2 | 订单状态 | isPayment()结果 |
|--------|-------|-------|---------|----------------|
| T0 | 获取锁 | - | 无订单 | false |
| T1 | 创建订单 | - | NOT_PAY | - |
| T2 | 发起支付中... | - | NOT_PAY | - |
| T3 | 释放锁 | 获取锁 | NOT_PAY | **false ❌** |
| T4 | - | 创建订单 | NOT_PAY×2 | - |
| T5 | - | 发起支付 | NOT_PAY×2 | - |
| T6 | 支付成功 | 支付成功 | SUCCESS×2 | - |

**问题关键**：T3时刻，请求2的 `isPayment()` 查询时，订单状态还是 `NOT_PAY`，因此返回 `false`，导致重复创建订单。

### 代码逻辑缺陷

```java
// 1. Redis分布式锁（正确）
rLock = redisLockUtil.lock(RedisConst.TRADE_ORDER_LOCK_KEY + truckUser.getTruckUserWalletId());
boolean isLock = rLock.tryLock(2, TimeUnit.MINUTES);

// 2. 检查是否有正在进行的订单（❌ 逻辑不完整）
if (this.isPayment(truckUser.getTruckUserWalletId())) {
    return "您的支付已成功提交，系统正在处理中，请勿重复操作!";
}

// 3. isPayment 只查询 SUCCESS/PAYING，漏掉 NOT_PAY（❌ 根本原因）
queryTradeOrder.setOrderStatusList(Lists.newArrayList(
    OrderStatusState.SUCCESS.name(), 
    OrderStatusState.PAYING.name()
    // ❌ 缺少：OrderStatusState.NOT_PAY.name()
));
```

## 【修复建议】

### 方案1：修改 isPayment() 方法（推荐）⭐

**文件**：`TopupRecordService.java:1221行`

```java
// ❌ 修改前
queryTradeOrder.setOrderStatusList(Lists.newArrayList(
    OrderStatusState.SUCCESS.name(), 
    OrderStatusState.PAYING.name()
));

// ✅ 修改后
queryTradeOrder.setOrderStatusList(Lists.newArrayList(
    OrderStatusState.SUCCESS.name(), 
    OrderStatusState.PAYING.name(),
    OrderStatusState.NOT_PAY.name()  // 新增：拦截未支付状态的订单
));
```

**修复原理**：
- 在并发请求获取锁后，查询最近10分钟内的订单时，包含 `NOT_PAY` 状态
- 即使第1个请求还在发起支付中（订单状态=NOT_PAY），第2个请求也能检测到并拦截

**影响范围**：
- ✅ 只影响清欠还款功能（`bizType=DEBT_REPAYMENT_RECHARGE`）
- ✅ 不影响其他充值业务

### 方案2：数据库唯一索引（辅助）

**表**：`hcb_trade_order`

```sql
-- 添加唯一索引，防止同一钱包10分钟内创建多笔清欠订单
ALTER TABLE hcb_trade_order 
ADD UNIQUE INDEX uk_wallet_biztype_time (
    user_wallet_id, 
    biz_type, 
    DATE_FORMAT(create_time, '%Y-%m-%d %H:%i')
) WHERE biz_type = 'DEBT_REPAYMENT_RECHARGE';
```

**注意**：MySQL不支持条件唯一索引，此方案需要改用应用层控制或触发器。

### 方案3：延长锁持有时间（不推荐）

保持锁到支付完成再释放，但会降低并发性能，不推荐。

## 【验证方案】

### 1. 单元测试
```java
@Test
public void testConcurrentDebtRepayment() throws Exception {
    String walletId = "test-wallet-id";
    
    // 模拟并发请求
    ExecutorService executor = Executors.newFixedThreadPool(2);
    CountDownLatch latch = new CountDownLatch(2);
    
    AtomicInteger successCount = new AtomicInteger(0);
    
    for (int i = 0; i < 2; i++) {
        executor.submit(() -> {
            try {
                ResponsePkg result = topupRecordService.debtRepaymentWechatPay(paramMap);
                if ("1".equals(result.getReturnJson().getString("ret"))) {
                    successCount.incrementAndGet();
                }
            } finally {
                latch.countDown();
            }
        });
    }
    
    latch.await();
    
    // 断言：只有1个请求成功
    Assert.assertEquals(1, successCount.get());
    
    // 断言：数据库只有1笔订单
    int orderCount = tradeOrderService.countTradeOrderByWalletIdAndBizType(
        walletId, OrderBizType.DEBT_REPAYMENT_RECHARGE.name()
    );
    Assert.assertEquals(1, orderCount);
}
```

### 2. 回归测试
- TC-050：并发场景-多人同时支付同一车辆
- TC-051：短时间内连续点击支付按钮
- TC-052：支付中再次点击支付

### 3. 生产数据验证
```sql
-- 查询最近7天内同一钱包有多笔清欠订单的记录
SELECT 
    user_wallet_id,
    COUNT(*) as order_count,
    GROUP_CONCAT(trade_order_no) as order_nos,
    GROUP_CONCAT(create_time) as create_times
FROM hcb_trade_order
WHERE biz_type = 'DEBT_REPAYMENT_RECHARGE'
  AND create_time >= DATE_SUB(NOW(), INTERVAL 7 DAY)
  AND order_status IN ('SUCCESS', 'PAYING')
GROUP BY user_wallet_id
HAVING COUNT(*) > 1;
```

## 【相关用例】
- **TC-050**：二次校验-并发场景-多人同时查询同一车辆（需修改为"并发重复支付被拦截"）
- **TC-051**：短时间内连续点击支付按钮
- **TC-052**：支付处理中再次点击支付

## 【风险评估】
- **资金风险**：🔴 高（用户重复支付，可能引发投诉和退款）
- **影响用户数**：所有清欠还款用户
- **修复优先级**：🔴 P0（必须立即修复）
- **修复难度**：🟢 低（仅需修改1行代码）
- **测试成本**：🟡 中（需并发测试）

## 【备注】
1. 建议立即修复此问题，避免用户资金损失
2. 修复后需进行压力测试，验证高并发场景
3. 需排查是否有其他业务存在类似问题（如普通充值、退款等）
4. 建议对历史重复支付订单进行补偿处理
