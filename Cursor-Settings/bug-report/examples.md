# BUG 报告示例

**⚠️ 重要**: 以下示例展示了正确和错误的BUG报告写法对比。

---

## ❌ 错误示例（啰嗦、越界）

### 问题：添加了过多不必要的章节

```markdown
# BUG-【任货行】清欠还款失败

## 【BUG描述】
清欠用户还款时，钱包扣款失败，订单状态未更新。

## 【根因分析】  ❌ 这是开发的工作，不要写！
调用链路：
PayNotifyController.notify() 
→ PayService.processPayment()
→ WalletService.deduct()

问题代码：
```java
// java/hcb/hcb-api/src/.../WalletService.java:123  ❌ 不要定位代码
if (wallet.getBalance() < 0) {
    throw new Exception("余额不足");
}
```

根因：查询钱包时使用了错误的ID字段...  ❌ 不要分析根因

## 【修复建议】  ❌ 这是开发的工作，不要写！
1. 修改 WalletService.java 第123行
2. 将 user_wallet_id 改为 id_code
3. 添加唯一索引防止重复

## 【数据库表结构】  ❌ 不要写，开发自己会查
```sql
DESC hcb_wallet;
```

## 【影响评估】  ❌ 不要写
可能导致资金损失，影响xxx用户...

## 【回归测试】  ❌ 测试计划自己管理，不写在BUG中
- TC-001: 验证修复后正常扣款
- TC-002: 验证边界场景
...

## 【生产环境排查】  ❌ 不要写
生产环境查询SQL：
```sql
SELECT * FROM hcb_wallet WHERE ...
```
```

**问题**:
- 太啰嗦（6个章节，应该只有4个）
- 越界（做了开发的工作）
- 添加了代码分析、修复建议等禁止内容

---

## ✅ 正确示例1：空指针异常

### 简洁、专注、只描述问题

```markdown
# BUG-【任货行】注销流程静止期处理定时任务空指针异常

## 1. BUG描述

注销流程静止期处理定时任务执行时抛出空指针异常。

**影响**：
- ❌ 定时任务无法正常完成
- ❌ 部分注销订单的静止期状态无法自动更新

---

## 2. 复现步骤

### 前置条件
- 测试环境: 192.168.1.60
- 系统: 任货行定时任务 (hcb-Timer-bill)
- 数据库: hcb
- 问题订单ID: `4f70e7e94eda4cf5a01599d66331ffc4`
- 身份证号: `142229198411114612`

### 操作步骤

**Step 1**: 调用定时任务接口

```bash
curl http://192.168.1.60:8087/hcbTimer-bill/test/cancelBussineOverRestingStageHandle
```

**Step 2**: 查询问题订单的数据

```bash
# 查询注销订单
ssh_db_query(
  server="test",
  type="mysql",
  database="hcb",
  query="SELECT CANCELBUSSINE_ID, IDCODE, STATUS FROM hcb_hlj_cancelbussine WHERE CANCELBUSSINE_ID='4f70e7e94eda4cf5a01599d66331ffc4'",
  dbUser="root",
  dbPassword="fm123456",
  dbHost="192.168.1.60",
  dbPort=3306
)

# 查询对应的钱包
ssh_db_query(
  server="test",
  type="mysql",
  database="hcb",
  query="SELECT * FROM HCB_HLJ_TRUCKUSERWALLET WHERE ID_CODE='142229198411114612'",
  dbUser="root",
  dbPassword="fm123456",
  dbHost="192.168.1.60",
  dbPort=3306
)
```

**数据查询结果**（只客观描述）:
```
# 注销订单存在
CANCELBUSSINE_ID: 4f70e7e94eda4cf5a01599d66331ffc4
IDCODE: 142229198411114612
STATUS: 1
CREATE_TIME: 2025-11-17 11:24:11

# 钱包不存在
Empty set

# 车辆不存在
Empty set
```

**Step 3**: 观察日志

```bash
ssh_execute(server="test", command="tail -n 100 /home/tomcat/tomcat-hcbTimer-bill/logs/catalina.out")
```

---

## 3. 预期结果

```
[INFO] - 查询钱包余额成功
[INFO] - 更新订单状态成功
[INFO] - 定时任务执行完成
```

**预期**：
- ✅ 定时任务正常执行完成
- ✅ 所有订单的静止期状态更新成功

---

## 4. 实际结果

```
[ERROR] - 注销流程静止期处理定时任务失败null
java.lang.NullPointerException
    at com.etc.api.task.CancelBussineTimerTask.overRestingStageHandle(CancelBussineTimerTask.java:140)
    at com.etc.api.task.CancelBussineTimerTask.execute(CancelBussineTimerTask.java:89)
    at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
    ...
```

**实际**：
- ❌ 定时任务执行中抛出 `NullPointerException`
- ❌ 定时任务中断，后续订单未处理

---

**日志路径**: `/home/tomcat/tomcat-hcbTimer-bill/logs/catalina.out`
```

**优点**:
- 只有4个核心部分
- 复现步骤详细且可操作
- 数据查询结果客观，没有分析为什么
- 没有代码分析、没有修复建议
- 错误日志完整

---

## ✅ 正确示例2：接口返回错误

```markdown
# BUG-【任通行】快速缴费充值接口返回500错误

## 1. BUG描述

快速缴费充值时，接口返回500错误，充值失败。

**影响**：
- ❌ 用户无法充值
- ✅ 查询账单功能正常
- ✅ 其他功能正常

---

## 2. 复现步骤

### 前置条件
- 测试环境: 192.168.1.60
- 系统: 任通行APP
- 数据库: rtx
- 测试账号: 车牌号 黑A113355
- 充值金额: 100元

### 操作步骤

**Step 1**: 登录任通行APP

**Step 2**: 进入快速缴费页面，输入充值金额100元，点击充值

**Step 3**: 选择支付宝支付

**Step 4**: 观察接口响应

```bash
# 接口请求
POST http://192.168.1.60/rtx-app/api/recharge
Content-Type: application/json

{
  "carNum": "黑A113355",
  "amount": 100,
  "payType": "alipay"
}
```

**Step 5**: 查看日志

```bash
ssh_execute(server="test", command="tail -n 100 /home/java-server/rtx-app/log.txt")
```

---

## 3. 预期结果

```json
{
  "code": 200,
  "message": "充值成功",
  "data": {
    "orderId": "xxx",
    "amount": 100
  }
}
```

**预期**：
- ✅ 接口返回200
- ✅ 充值订单创建成功
- ✅ 跳转到支付页面

---

## 4. 实际结果

```
HTTP 500 Internal Server Error

{
  "code": 500,
  "message": "系统异常"
}
```

**日志**:
```
[ERROR] 2026-02-06 14:30:25 - 充值失败
java.lang.IllegalArgumentException: amount cannot be null
    at com.rtx.service.RechargeService.createOrder(RechargeService.java:45)
    ...
```

**实际**：
- ❌ 接口返回500错误
- ❌ 充值失败，订单未创建

---

**日志路径**: `/home/java-server/rtx-app/log.txt`
```

**优点**:
- 接口请求参数完整
- 预期和实际对比清晰
- 没有分析为什么amount会为null
- 没有建议修改代码

---

## ✅ 正确示例3：数据异常问题

```markdown
# BUG-【任货行】黑名单导出缺少起诉状态字段

## 1. BUG描述

黑名单导出功能缺少"起诉状态"字段，导出的Excel文件中没有该字段。

**影响**：
- ❌ 导出的黑名单缺少起诉状态信息
- ✅ 黑名单查询列表正常显示
- ✅ 其他导出功能正常

---

## 2. 复现步骤

### 前置条件
- 测试环境: 192.168.1.60
- 系统: 任货行管理后台
- 数据库: hcb
- 测试账号: admin
- 测试数据: 存在起诉状态的黑名单记录

### 操作步骤

**Step 1**: 登录任货行管理后台

**Step 2**: 进入黑名单管理页面

**Step 3**: 点击"导出"按钮

**Step 4**: 查看导出的Excel文件

**Step 5**: 查询数据库中的字段

```bash
# 查询黑名单记录
ssh_db_query(
  server="test",
  type="mysql",
  database="hcb",
  query="SELECT car_num, down_reason, sue_status FROM hcb_blacklist LIMIT 5",
  dbUser="root",
  dbPassword="fm123456",
  dbHost="192.168.1.60",
  dbPort=3306
)
```

**数据查询结果**:
```
car_num       down_reason  sue_status
贵H86206      欠费         1
黑A113355     欠费         0
...
```

---

## 3. 预期结果

导出的Excel文件应包含以下字段：
- 车牌号
- 下黑原因
- 起诉状态（0=未起诉, 1=已起诉）← 应该有但缺失
- 创建时间
- ...

**预期**：
- ✅ Excel包含所有字段
- ✅ 起诉状态字段正常显示

---

## 4. 实际结果

导出的Excel文件只包含：
- 车牌号
- 下黑原因
- 创建时间
- ...

**实际**：
- ❌ 缺少"起诉状态"字段
- ❌ 无法从导出文件中看到起诉信息

---

**日志路径**: `/home/tomcat/tomcat-hcbadmin/logs/catalina.out`
```

**优点**:
- 清晰说明缺少哪个字段
- 提供了数据库查询验证字段存在
- 没有分析为什么会缺少
- 没有建议修改代码添加字段

---

## 对比总结

| 维度 | ❌ 错误做法 | ✅ 正确做法 |
|------|-----------|-----------|
| 章节数量 | 6-8个章节 | 只有4个核心部分 |
| 根因分析 | 添加"根因分析"章节 | 不分析根因 |
| 修复建议 | 建议修改代码 | 不给修复建议 |
| 代码引用 | 贴代码片段 | 不引用代码 |
| 数据问题 | 分析为什么数据异常 | 只客观描述数据状态 |
| 文档数量 | 多个文档（BUG报告+分析+方案） | 只有1个BUG报告 |
| 复现步骤 | 简单描述 | 详细且可操作 |
| 错误日志 | 部分日志 | 完整日志和堆栈 |

---

## 快速参考

**编写BUG报告时，记住**:

✅ **要做的**:
- 描述问题现象
- 提供详细的复现步骤
- 粘贴完整的错误日志
- 查询并客观描述数据状态
- 说明预期和实际的差异

❌ **不要做的**:
- 分析代码
- 定位根因
- 给修复建议
- 引用代码片段
- 分析为什么数据异常
- 添加测试计划、影响评估等额外章节
- 生成多个文档

**记住**: 只生成一个BUG报告文档，只包含4个核心部分！
