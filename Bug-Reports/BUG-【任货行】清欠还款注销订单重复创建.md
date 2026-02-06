# BUG-【任货行】清欠还款注销订单重复创建

## 1. BUG描述

清欠还款成功后，系统自动创建诉讼还款注销订单。驳回注销订单后，过一段时间系统会再次自动创建新的注销订单，导致同一辆车存在多个注销订单。

**影响**：
- ✅ 清欠还款流程正常
- ✅ 第一次创建注销订单正常
- ✅ 驳回注销订单功能正常
- ❌ 驳回后会重复创建新的注销订单
- ❌ 同一辆车可能存在多个注销订单

---

## 2. 复现步骤

### 前置条件
- 测试环境：http://788360p9o5.yicp.fun
- 测试账号：蒙ZMM777（身份证号：441622199602075195）
- 生产环境配置：`sue.repayment.delayed = 20`（延迟20分钟）

### 操作步骤

**Step 1**: 清欠还款充值

```bash
curl -X POST 'http://788360p9o5.yicp.fun/hcbapi/information/topuprecord/debtRepaymentWechatPay.do' \
  -H 'Content-Type: application/json' \
  -d '{
    "carNum": "蒙ZMM777",
    "vehicleColor": "1",
    "amount": "0.01",
    "rechargeWay": "WEI_XIN",
    "relativeurl": "com.hcb.debtRepaymentWechatPay"
  }'
```

**支付成功时间**: 2026-01-28 17:00:00

**Step 2**: 等待20分钟后，查看注销订单

20分钟后（17:20），查询注销订单：

```bash
# 查询注销订单
SELECT CANCELBUSSINE_ID, PLATE_NUM, STATUS, CANCEL_ORDER_TYPE, CREATE_TIME 
FROM hcb_hlj_cancelbussine 
WHERE IDCODE='441622199602075195' 
ORDER BY CREATE_TIME DESC
```

**数据查询结果**：
```
# 第一次自动创建的注销订单（20分钟后创建）
CANCELBUSSINE_ID: xxx-订单1-A
PLATE_NUM: 蒙ZMM777
STATUS: 1（静置期）
CANCEL_ORDER_TYPE: 8（诉讼还款注销）
CREATE_TIME: 2026-01-28 17:20:00

CANCELBUSSINE_ID: xxx-订单1-B
PLATE_NUM: 蒙ZMM888
STATUS: 1（静置期）
CANCEL_ORDER_TYPE: 8（诉讼还款注销）
CREATE_TIME: 2026-01-28 17:20:00
```

**Step 3**: 驳回注销订单

在后台驳回车辆A的注销订单（17:21）：

```
访问：http://788360p9o5.yicp.fun/hcbadmin/newCancelBussine/rejectCancel.do
参数：CANCELBUSSINE_ID=xxx-订单1-A
```

驳回后查询车辆和订单状态：

```bash
# 查询车辆状态
SELECT CAR_NUM, STATUS, BLACK_RESOURE 
FROM hcb_truckuser 
WHERE CAR_NUM='蒙ZMM777'

# 查询注销订单状态
SELECT CANCELBUSSINE_ID, STATUS 
FROM hcb_hlj_cancelbussine 
WHERE CANCELBUSSINE_ID='xxx-订单1-A'
```

**数据查询结果**：
```
# 车辆状态
CAR_NUM: 蒙ZMM777
STATUS: 1（正常）
BLACK_RESOURE: NULL

# 注销订单状态
CANCELBUSSINE_ID: xxx-订单1-A
STATUS: 3（已驳回）
```

**Step 4**: 等待10分钟后，再次查询注销订单

30分钟后（17:30，距离支付成功30分钟），再次查询注销订单：

```bash
SELECT CANCELBUSSINE_ID, PLATE_NUM, STATUS, CANCEL_ORDER_TYPE, CREATE_TIME 
FROM hcb_hlj_cancelbussine 
WHERE IDCODE='441622199602075195' 
ORDER BY CREATE_TIME DESC
```

**Step 5**: 查看MQ消费日志

```bash
# 查看hcbapi日志
tail -f /home/java-server/hcbapi/log.txt | grep "清欠还款完成方法,起诉注销监听"
```

---

## 3. 预期结果

```
[2026-01-28 17:20:00] - 进入清欠还款完成方法,起诉注销监听,msg:{"tuckUserWalletId":"xxx","tradeOrderNo":"xxx"}
[2026-01-28 17:20:01] - 创建注销订单成功：订单1-A
[2026-01-28 17:20:01] - 创建注销订单成功：订单1-B
```

**预期**：
- ✅ 20分钟后MQ到达，创建注销订单
- ✅ 驳回订单后，车辆状态恢复正常
- ✅ 不应该再次创建新的注销订单
- ✅ 数据库中每辆车只有一个注销订单记录

---

## 4. 实际结果

```
[2026-01-28 17:20:00] - 进入清欠还款完成方法,起诉注销监听,msg:{"tuckUserWalletId":"xxx","tradeOrderNo":"xxx"}
[2026-01-28 17:20:01] - 创建注销订单成功：订单1-A
[2026-01-28 17:20:01] - 创建注销订单成功：订单1-B

[2026-01-28 17:30:00] - 进入清欠还款完成方法,起诉注销监听,msg:{"tuckUserWalletId":"xxx","tradeOrderNo":"xxx"}
[2026-01-28 17:30:01] - 创建注销订单成功：订单2-A
```

**数据查询结果**：
```
# 第二次创建的注销订单（30分钟后又创建了）
CANCELBUSSINE_ID: xxx-订单2-A
PLATE_NUM: 蒙ZMM777
STATUS: 1（静置期）
CANCEL_ORDER_TYPE: 8（诉讼还款注销）
CREATE_TIME: 2026-01-28 17:30:00

# 第一次创建的注销订单（已驳回）
CANCELBUSSINE_ID: xxx-订单1-A
PLATE_NUM: 蒙ZMM777
STATUS: 3（已驳回）
CANCEL_ORDER_TYPE: 8（诉讼还款注销）
CREATE_TIME: 2026-01-28 17:20:00
```

**实际**：
- ❌ 30分钟后（距离支付成功30分钟），系统再次消费了同一条MQ消息
- ❌ 再次创建了新的注销订单（订单2-A）
- ❌ 同一辆车（蒙ZMM777）存在2个注销订单
- ❌ 车辆状态再次变为"注销中"

---

**日志路径**: `/home/java-server/hcbapi/log.txt`
