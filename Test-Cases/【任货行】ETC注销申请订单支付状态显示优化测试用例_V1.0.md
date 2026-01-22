# 【任货行】ETC注销申请订单支付状态显示优化测试用例

## 1. 功能测试

### 1.1 支付状态=5(支付中) 显示测试

#### TC-001 支付中订单显示继续支付按钮
- **前置条件**：
  - 用户已登录任货行小程序
  - 存在PAYMENT_STATUS='5'(支付中)的注销申请订单
  - 数据准备：`UPDATE hcb_hlj_cancelbussine SET PAYMENT_STATUS='5', STATUS='0' WHERE CANCELBUSSINE_ID='test001';`
- **测试步骤**：
  1. 进入"我的订单"或"注销申请列表"页面
  2. 查看支付中状态订单
  3. 验证是否显示"继续支付"或"支付中"提示
- **预期结果**：支付中订单显示正确状态标识

#### TC-002 支付中订单可重新发起支付
- **前置条件**：
  - TC-001执行成功
  - 订单PAYMENT_STATUS='5'
- **测试步骤**：
  1. 点击订单操作按钮
  2. 验证是否可以重新发起支付
  3. 发起支付后验证状态
- **预期结果**：
  - 可以重新发起支付
  - 支付成功后PAYMENT_STATUS变为'1'

#### TC-003 支付中订单超时处理
- **前置条件**：
  - 订单PAYMENT_STATUS='5'
  - 创建时间超过30分钟
- **测试步骤**：
  1. 查看超时的支付中订单
  2. 验证订单状态显示
  3. 尝试重新支付
- **预期结果**：
  - 显示"支付超时"提示
  - 允许重新支付

### 1.2 支付状态=6(支付失败) 显示测试

#### TC-004 支付失败订单显示正确状态
- **前置条件**：
  - 存在PAYMENT_STATUS='6'(支付失败)的订单
  - 数据准备：`UPDATE hcb_hlj_cancelbussine SET PAYMENT_STATUS='6', STATUS='0' WHERE CANCELBUSSINE_ID='test002';`
- **测试步骤**：
  1. 进入订单列表
  2. 查看支付失败订单
  3. 验证状态显示和提示信息
- **预期结果**：
  - 显示"支付失败"状态
  - 提示失败原因（如有）

#### TC-005 支付失败订单可重新支付
- **前置条件**：
  - TC-004执行成功
  - 订单PAYMENT_STATUS='6'
- **测试步骤**：
  1. 点击订单操作按钮
  2. 点击"重新支付"按钮
  3. 完成支付流程
- **预期结果**：
  - 显示"重新支付"按钮
  - 可以重新发起支付

#### TC-006 支付失败原因显示
- **前置条件**：
  - 订单PAYMENT_STATUS='6'
  - RESULT_JSON字段包含失败原因
- **测试步骤**：
  1. 查看订单详情
  2. 验证失败原因显示
- **预期结果**：显示具体失败原因（余额不足、银行卡限额等）

### 1.3 其他支付状态显示测试

#### TC-007 未支付(0)订单显示
- **前置条件**：PAYMENT_STATUS='0'
- **测试步骤**：
  1. 查看未支付订单
  2. 验证显示和操作按钮
- **预期结果**：显示"未支付"，可发起支付

#### TC-008 已支付(1)订单显示
- **前置条件**：PAYMENT_STATUS='1'
- **测试步骤**：
  1. 查看已支付订单
  2. 验证显示和操作按钮
- **预期结果**：显示"已支付"，不可再支付

#### TC-009 已退款(2)订单显示
- **前置条件**：PAYMENT_STATUS='2'
- **测试步骤**：
  1. 查看已退款订单
  2. 验证显示
- **预期结果**：显示"已退款"

#### TC-010 退款失败(3)订单显示
- **前置条件**：PAYMENT_STATUS='3'
- **测试步骤**：
  1. 查看退款失败订单
  2. 验证显示和操作
- **预期结果**：显示"退款失败"，可重新申请退款

#### TC-011 退款中(4)订单显示
- **前置条件**：PAYMENT_STATUS='4'
- **测试步骤**：
  1. 查看退款中订单
  2. 验证显示
- **预期结果**：显示"退款中"

### 1.4 STATUS与PAYMENT_STATUS组合测试

#### TC-012 STATUS=0 + PAYMENT_STATUS=5 组合
- **前置条件**：
  - STATUS='0'(未提交)
  - PAYMENT_STATUS='5'(支付中)
- **测试步骤**：
  1. 查看订单列表
  2. 验证订单显示逻辑
- **预期结果**：
  - 订单状态显示"未提交"
  - 支付状态显示"支付中"
  - 可继续提交或重新支付

#### TC-013 STATUS=0 + PAYMENT_STATUS=6 组合
- **前置条件**：
  - STATUS='0'(未提交)
  - PAYMENT_STATUS='6'(支付失败)
- **测试步骤**：
  1. 查看订单列表
  2. 验证显示和操作
- **预期结果**：
  - 显示"支付失败"
  - 可继续提交流程

#### TC-014 STATUS=3 + PAYMENT_STATUS=5 组合
- **前置条件**：
  - STATUS='3'(已提交)
  - PAYMENT_STATUS='5'(支付中)
- **测试步骤**：
  1. 查看订单
  2. 等待支付回调
- **预期结果**：
  - 显示"已提交，支付中"
  - 支付完成后状态自动更新

#### TC-015 STATUS=3 + PAYMENT_STATUS=6 组合
- **前置条件**：
  - STATUS='3'(已提交)
  - PAYMENT_STATUS='6'(支付失败)
- **测试步骤**：
  1. 查看订单
  2. 尝试重新支付
- **预期结果**：
  - 显示"已提交，支付失败"
  - 可重新支付

### 1.5 支付状态变更测试

#### TC-016 支付中→支付成功流程
- **前置条件**：PAYMENT_STATUS='5'
- **测试步骤**：
  1. 发起支付
  2. 完成支付
  3. 验证状态更新
  4. 查询数据库：`SELECT PAYMENT_STATUS FROM hcb_hlj_cancelbussine WHERE CANCELBUSSINE_ID='test001';`
- **预期结果**：
  - PAYMENT_STATUS从'5'变为'1'
  - 显示"已支付"

#### TC-017 支付中→支付失败流程
- **前置条件**：PAYMENT_STATUS='5'
- **测试步骤**：
  1. 发起支付
  2. 支付失败（余额不足等）
  3. 验证状态更新
- **预期结果**：
  - PAYMENT_STATUS从'5'变为'6'
  - 显示"支付失败"

#### TC-018 支付失败→支付成功流程
- **前置条件**：PAYMENT_STATUS='6'
- **测试步骤**：
  1. 点击重新支付
  2. 完成支付
  3. 验证状态
- **预期结果**：
  - PAYMENT_STATUS从'6'变为'1'
  - 显示"已支付"

### 1.6 异常场景测试

#### TC-019 支付回调超时场景
- **前置条件**：
  - PAYMENT_STATUS='5'
  - 超过30分钟未收到回调
- **测试步骤**：
  1. 模拟支付超时
  2. 查看订单状态
  3. 尝试操作
- **预期结果**：
  - 显示"支付超时"提示
  - 允许重新支付或查询支付结果

#### TC-020 支付状态异常值处理
- **前置条件**：
  - PAYMENT_STATUS为NULL或非法值
- **测试步骤**：
  1. 查看订单
  2. 验证显示
- **预期结果**：
  - 显示默认状态或"状态异常"
  - 不影响其他功能

#### TC-021 并发支付处理
- **前置条件**：同一订单
- **测试步骤**：
  1. 同时在两个设备发起支付
  2. 验证支付状态
- **预期结果**：
  - 只有一个支付成功
  - 另一个提示"订单已支付"

### 1.7 支付状态文案测试

#### TC-022 支付状态文案准确性
- **前置条件**：各种支付状态
- **测试步骤**：
  1. 验证每个状态的显示文案
  2. 检查是否有错别字
- **预期结果**：
  - 0: "未支付"
  - 1: "已支付"
  - 2: "已退款"
  - 3: "退款失败"
  - 4: "退款中"
  - 5: "支付中"
  - 6: "支付失败"

#### TC-023 支付状态颜色区分
- **前置条件**：不同支付状态
- **测试步骤**：
  1. 查看不同状态订单
  2. 验证颜色标识
- **预期结果**：
  - 成功状态：绿色
  - 失败状态：红色
  - 进行中：橙色

## 2. 回归测试

#### TC-024 原有支付流程不受影响
- **前置条件**：已支付订单
- **测试步骤**：
  1. 查看原有已支付订单
  2. 验证显示正常
- **预期结果**：原有功能正常

#### TC-025 退款流程不受影响
- **前置条件**：需要退款的订单
- **测试步骤**：
  1. 发起退款
  2. 验证退款状态显示
- **预期结果**：退款流程正常

## 3. 数据库测试

#### TC-026 支付状态枚举值完整性
- **前置条件**：数据库连接
- **测试步骤**：
  1. 查询所有支付状态：`SELECT DISTINCT PAYMENT_STATUS FROM hcb_hlj_cancelbussine;`
  2. 验证每个值都有对应显示
- **预期结果**：所有枚举值都能正确显示

#### TC-027 支付状态更新日志
- **前置条件**：支付状态发生变更
- **测试步骤**：
  1. 支付前后查询UPDATE_TIME
  2. 验证更新时间
- **预期结果**：
  - UPDATE_TIME正确更新
  - 状态变更有记录

## 测试数据准备SQL

```sql
-- 1. 创建支付中订单（用于TC-001到TC-003）
UPDATE hcb_hlj_cancelbussine 
SET PAYMENT_STATUS='5', STATUS='0', PLATE_NUM='川BBB2A2' 
WHERE CANCELBUSSINE_ID='0039494b4ac14c4491de1fa11ad2bda9';

-- 2. 创建支付失败订单（用于TC-004到TC-006）
UPDATE hcb_hlj_cancelbussine 
SET PAYMENT_STATUS='6', STATUS='0', PLATE_NUM='苏ZBBB99' 
WHERE CANCELBUSSINE_ID='00d0675f7b294c99ab65a714c41a1796';

-- 3. 创建各种支付状态订单（用于TC-007到TC-011）
-- 未支付
UPDATE hcb_hlj_cancelbussine SET PAYMENT_STATUS='0', STATUS='0' WHERE CANCELBUSSINE_ID='test01';
-- 已支付
UPDATE hcb_hlj_cancelbussine SET PAYMENT_STATUS='1', STATUS='3' WHERE CANCELBUSSINE_ID='test02';
-- 已退款
UPDATE hcb_hlj_cancelbussine SET PAYMENT_STATUS='2', STATUS='7' WHERE CANCELBUSSINE_ID='test03';
-- 退款失败
UPDATE hcb_hlj_cancelbussine SET PAYMENT_STATUS='3', STATUS='3' WHERE CANCELBUSSINE_ID='test04';
-- 退款中
UPDATE hcb_hlj_cancelbussine SET PAYMENT_STATUS='4', STATUS='3' WHERE CANCELBUSSINE_ID='test05';

-- 4. STATUS与PAYMENT_STATUS组合数据（用于TC-012到TC-015）
UPDATE hcb_hlj_cancelbussine SET STATUS='0', PAYMENT_STATUS='5' WHERE CANCELBUSSINE_ID='combo01';
UPDATE hcb_hlj_cancelbussine SET STATUS='0', PAYMENT_STATUS='6' WHERE CANCELBUSSINE_ID='combo02';
UPDATE hcb_hlj_cancelbussine SET STATUS='3', PAYMENT_STATUS='5' WHERE CANCELBUSSINE_ID='combo03';
UPDATE hcb_hlj_cancelbussine SET STATUS='3', PAYMENT_STATUS='6' WHERE CANCELBUSSINE_ID='combo04';

-- 5. 查询各支付状态数量
SELECT PAYMENT_STATUS, COUNT(*) as count 
FROM hcb_hlj_cancelbussine 
GROUP BY PAYMENT_STATUS;

-- 6. 查询支付中和支付失败的订单
SELECT CANCELBUSSINE_ID, STATUS, PAYMENT_STATUS, PLATE_NUM, CREATE_TIME, UPDATE_TIME 
FROM hcb_hlj_cancelbussine 
WHERE PAYMENT_STATUS IN ('5', '6') 
ORDER BY CREATE_TIME DESC 
LIMIT 10;

-- 7. 验证支付状态变更
SELECT CANCELBUSSINE_ID, PAYMENT_STATUS, UPDATE_TIME 
FROM hcb_hlj_cancelbussine 
WHERE CANCELBUSSINE_ID='test001';
```

## 测试优先级

| 优先级 | 用例编号 | 说明 |
|--------|---------|------|
| **P0** | TC-001, TC-004, TC-007, TC-008, TC-016, TC-017 | 核心支付状态显示和变更 |
| **P1** | TC-002, TC-005, TC-012, TC-013, TC-018, TC-022 | 重要操作和组合场景 |
| **P2** | TC-003, TC-006, TC-014, TC-015, TC-019, TC-026 | 边界值和异常场景 |
| **P3** | TC-009-TC-011, TC-020, TC-021, TC-023-TC-025, TC-027 | 回归和完整性测试 |
