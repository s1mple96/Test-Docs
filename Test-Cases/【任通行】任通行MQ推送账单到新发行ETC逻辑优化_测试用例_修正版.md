# 任通行通过MQ推送账单到新发行ETC逻辑优化测试用例（修正版）

## 1. 需求背景分析（修正）

### 1.1 正确的业务流程理解
- **任通行系统(rtx)** → MQ推送 → **新发行ETC系统(fenmi_etc)**
- **数据流向**: `rtx.rtx_etc_deduc_bill` → MQ队列 → `fenmi_etc.etc_deduct_bill`
- **核心问题**: 任通行异步发送账单到MQ时异常，导致新发行ETC系统未收到账单，造成数据不一致

### 1.2 系统架构（修正）
```
任通行系统(rtx)                    新发行ETC系统(fenmi_etc)
├─ rtx.rtx_etc_deduc_bill    →    ├─ fenmi_etc.etc_deduct_bill
├─ rtx.sys_mq_error_log      ↗    ├─ 消费MQ消息
├─ 定时任务推送              MQ     ├─ 数据落库
└─ 推送状态维护                    └─ 回调确认
```

### 1.3 涉及系统和组件
- **发送方**: rtx-quartz (定时任务), rtx-app (业务应用)
- **接收方**: fenmi-etc-job (消息消费)
- **MQ系统**: RabbitMQ (192.168.1.58:5673)
- **数据库**: rtx库 → fenmi_etc库
- **监控工具**: Arthas (192.168.1.60服务器)

---

## 2. 核心表结构分析

### 2.1 任通行账单表 (rtx.rtx_etc_deduc_bill)
```sql
-- 源端：任通行系统账单表
主要字段：
- id: bigint(32) PRIMARY KEY
- bill_date: varchar(12) 账单日期
- operator_code: varchar(10) 运营商代码 (TXB=任通行内蒙)
- bill_amt: decimal(13,2) 账单金额
- bill_status: int(1) 账单状态 (0=待推送)
- bill_count: bigint(11) 账单数量
- pay_status: int(1) 支付状态
```

### 2.2 新发行ETC账单表 (fenmi_etc.etc_deduct_bill)
```sql
-- 目标端：新发行ETC系统账单表
主要字段：
- id: bigint(32) PRIMARY KEY
- bill_date: varchar(12) 账单日期
- operator_code: varchar(10) 运营商代码
- bill_amt: decimal(13,2) 账单金额
- bill_status: int(1) 账单状态 (0=待处理)
- bill_count: bigint(11) 账单数量
- pay_status: int(1) 支付状态
```

### 2.3 MQ错误日志表 (rtx.sys_mq_error_log) - 已优化
```sql
-- MQ错误日志表（推送方维护）
关键字段：
- id: bigint(20) PRIMARY KEY
- biz_id: varchar(64) 业务id（账单ID）
- correlation_id: varchar(64) 投递id
- msg: text 消息内容
- msg_type: int 发送类型 (0=字符串, 1=对象)
- routing_key: varchar(100) 路由键
- status: int 状态
- send_time: datetime 最近一次发送时间
- retry_max_count: int 最大重试次数
```

---

## 3. 修正版测试用例

### 3.1 账单推送基础功能测试

#### TC-001 任通行账单推送到新发行ETC正常流程测试（修正）
- **前置条件**: 任通行系统(rtx-quartz)正常运行，MQ服务可用，新发行ETC系统(fenmi-etc-job)正常
- **测试步骤**:
  1. 在 `rtx.rtx_etc_deduc_bill` 表中插入内蒙账单测试数据（operator_code='TXB'）
  2. 触发任通行定时任务 `FmYqtSettBillTask.pushNonAnHuiFmDeductBillTask()`
  3. 验证消息成功发送到MQ队列 `bill.deduct.record.queue`
  4. 验证新发行ETC系统成功消费消息
  5. 检查 `fenmi_etc.etc_deduct_bill` 表是否正确落库
  6. 验证新发行ETC系统回调任通行确认状态
- **预期结果**: 账单从rtx推送到fenmi_etc成功，数据一致，状态更新正确

**测试数据准备**:
```sql
-- 在任通行系统插入测试账单
INSERT INTO rtx.rtx_etc_deduc_bill (
    id, bill_date, operator_code, bill_amt, bill_count,
    bill_status, pay_status, create_time, update_time
) VALUES (
    20240821001001, '20240821', 'TXB', 500.00, 10,
    0, 0, NOW(), NOW()
);
```

**验证方法**:
```sql
-- 验证任通行端数据
SELECT id, bill_date, operator_code, bill_amt, bill_status 
FROM rtx.rtx_etc_deduc_bill 
WHERE id = 20240821001001;

-- 验证新发行ETC端数据
SELECT id, bill_date, operator_code, bill_amt, bill_status 
FROM fenmi_etc.etc_deduct_bill 
WHERE bill_date = '20240821' AND operator_code = 'TXB';
```

#### TC-002 MQ推送异常处理和错误日志记录测试
- **前置条件**: 模拟MQ连接异常或消息发送失败
- **测试步骤**:
  1. 在rtx.rtx_etc_deduc_bill插入测试账单
  2. 故意断开MQ连接或配置错误参数
  3. 触发账单推送任务
  4. 验证异常被正确捕获
  5. 检查 `rtx.sys_mq_error_log` 表是否记录错误信息
  6. 验证预警机制是否触发
- **预期结果**: 异常被正确处理，错误信息完整记录在rtx库中，预警及时触发

**验证方法**:
```sql
-- 检查MQ错误日志
SELECT id, biz_id, correlation_id, routing_key, status, error_msg, send_time 
FROM rtx.sys_mq_error_log 
WHERE create_time > NOW() - INTERVAL 1 HOUR 
ORDER BY create_time DESC;
```

#### TC-003 MQ消息重试机制测试
- **前置条件**: 配置MQ重试机制，模拟临时发送失败
- **测试步骤**:
  1. 插入rtx账单数据
  2. 模拟网络抖动导致MQ临时不可用
  3. 触发推送，观察重试机制
  4. 恢复MQ连接
  5. 验证最终推送成功
  6. 检查重试次数和状态更新
- **预期结果**: 重试机制按配置工作，最终成功推送，重试记录准确

### 3.2 新发行ETC系统消息处理测试

#### TC-004 新发行ETC系统消息消费测试
- **前置条件**: fenmi-etc-job消费者正常运行
- **测试步骤**:
  1. 从rtx系统发送测试账单消息到MQ
  2. 监控新发行ETC系统消费过程
  3. 验证消息处理逻辑的正确性
  4. 检查fenmi_etc.etc_deduct_bill数据落库
  5. 验证消费确认机制
- **预期结果**: 新发行ETC系统正确消费消息，数据准确落库

**监控方法**:
```bash
# 监控MQ队列状态
curl -u admin:admin "http://192.168.1.58:15672/api/queues/%2Ffenmi_test/bill.deduct.record.queue"

# 使用Arthas监控fenmi-etc-job进程（PID: 689361）
/usr/local/java/jdk1.8.0_171/bin/java -jar arthas-boot.jar 689361 -c "monitor *BillConsumer* * -n 5"
```

#### TC-005 新发行ETC消息处理失败恢复测试
- **前置条件**: 模拟新发行ETC系统处理异常
- **测试步骤**:
  1. 发送正常消息到队列
  2. 在fenmi-etc-job消费过程中制造异常
  3. 验证消息重新入队或进入死信队列
  4. 检查错误处理和告警机制
  5. 恢复后验证消息能正常处理
- **预期结果**: 消费失败得到正确处理，消息不丢失

#### TC-006 消息幂等性保护测试
- **前置条件**: 同一账单消息可能被重复发送
- **测试步骤**:
  1. 从rtx发送重复的账单消息
  2. 验证新发行ETC系统幂等性保护
  3. 检查重复消息不会重复入库
  4. 验证账单数据的唯一性约束
- **预期结果**: 系统具备幂等性保护，重复消息不导致数据异常

### 3.3 回调机制测试

#### TC-007 新发行ETC回调任通行确认测试
- **前置条件**: 新发行ETC系统处理完消息后回调任通行
- **测试步骤**:
  1. 新发行ETC系统成功处理账单消息并落库
  2. 触发回调任通行的确认接口
  3. 验证回调数据的完整性和格式
  4. 检查任通行系统接收回调并更新状态
  5. 验证账单推送状态的变更
- **预期结果**: 回调机制正常，任通行能收到确认并更新推送状态

**回调验证**:
```sql
-- 检查任通行端推送状态更新
SELECT id, bill_date, operator_code, bill_status, update_time 
FROM rtx.rtx_etc_deduc_bill 
WHERE id = 20240821001001;
```

#### TC-008 回调失败重试机制测试
- **前置条件**: 回调任通行时可能出现网络异常
- **测试步骤**:
  1. 新发行ETC处理完账单
  2. 模拟任通行回调接口不可用
  3. 验证回调重试机制
  4. 恢复接口可用性
  5. 验证最终成功回调
- **预期结果**: 回调重试机制正常，最终能成功确认或记录失败

### 3.4 定时补偿机制测试

#### TC-009 超时账单自动补偿测试
- **前置条件**: 配置账单推送超时时间和补偿机制
- **测试步骤**:
  1. 在rtx插入账单但阻止MQ发送
  2. 等待超过预设超时时间
  3. 验证补偿定时任务自动触发
  4. 检查超时账单被重新推送
  5. 验证补偿记录和状态更新
- **预期结果**: 补偿机制自动发现并重推超时账单

**补偿监控**:
```sql
-- 监控需要补偿的账单
SELECT id, bill_date, operator_code, bill_amt, bill_status, create_time, update_time
FROM rtx.rtx_etc_deduc_bill 
WHERE bill_status = 0 
  AND create_time < NOW() - INTERVAL 30 MINUTE
  AND operator_code = 'TXB'
ORDER BY create_time DESC;
```

#### TC-010 补偿重试次数限制测试
- **前置条件**: 设置补偿重试的最大次数
- **测试步骤**:
  1. 创建一直推送失败的账单场景
  2. 让补偿机制多次重试
  3. 验证达到最大重试次数后停止
  4. 检查最终状态标记和错误记录
- **预期结果**: 达到最大重试次数后停止，状态正确标记为失败

### 3.5 数据一致性验证测试

#### TC-011 跨系统数据一致性测试
- **前置条件**: 账单在rtx和fenmi_etc系统间正常流转
- **测试步骤**:
  1. 在rtx.rtx_etc_deduc_bill创建账单
  2. 完整执行推送流程
  3. 对比两个系统的账单数据
  4. 验证关键字段的一致性（金额、数量、日期等）
  5. 检查数据完整性和准确性
- **预期结果**: 两系统数据高度一致，无数据丢失或错误

**数据对比验证**:
```sql
-- 任通行端数据
SELECT id, bill_date, operator_code, bill_amt, bill_count, bill_status 
FROM rtx.rtx_etc_deduc_bill 
WHERE operator_code = 'TXB' AND bill_date = '20240821';

-- 新发行ETC端数据
SELECT id, bill_date, operator_code, bill_amt, bill_count, bill_status 
FROM fenmi_etc.etc_deduct_bill 
WHERE operator_code = 'TXB' AND bill_date = '20240821';

-- 数据一致性对比
SELECT 
  r.bill_date,
  r.operator_code,
  r.bill_amt as rtx_amt,
  f.bill_amt as fenmi_amt,
  (r.bill_amt - f.bill_amt) as amt_diff,
  r.bill_count as rtx_count,
  f.bill_count as fenmi_count
FROM rtx.rtx_etc_deduc_bill r
LEFT JOIN fenmi_etc.etc_deduct_bill f 
  ON r.bill_date = f.bill_date AND r.operator_code = f.operator_code
WHERE r.operator_code = 'TXB' AND r.bill_date = '20240821';
```

#### TC-012 异常恢复后数据一致性测试
- **前置条件**: 系统在异常中断后能恢复数据一致性
- **测试步骤**:
  1. 在推送过程中模拟系统异常
  2. 重启相关服务
  3. 检查数据状态和一致性
  4. 验证恢复机制是否工作
  5. 确认无数据丢失或重复
- **预期结果**: 系统能自动恢复或识别数据不一致问题

---

## 4. 使用工具进行深度分析测试

### 4.1 Arthas工具监控测试（修正）

#### TC-013 基于Arthas监控任通行推送方法
- **前置条件**: 连接到rtx-quartz进程(PID: 699103)
- **测试步骤**:
  1. 连接Arthas: `/usr/local/java/jdk1.8.0_171/bin/java -jar arthas-boot.jar 699103`
  2. 监控推送方法: `monitor com.rtx.quartz.task.fenmi.FmYqtSettBillTask pushNonAnHuiFmDeductBillTask -n 5`
  3. 观察方法参数和返回值: `watch com.rtx.quartz.task.fenmi.FmYqtSettBillTask pushNonAnHuiFmDeductBillTask '{params, returnObj}' -n 3`
  4. 跟踪方法调用链: `trace com.rtx.quartz.task.fenmi.FmYqtSettBillTask pushNonAnHuiFmDeductBillTask`
  5. 监控异常: `watch * * '{params, throwExp}' -e`
- **预期结果**: 清晰观察推送方法的执行过程和异常情况

#### TC-014 基于Arthas监控新发行ETC消费方法
- **前置条件**: 连接到fenmi-etc-job进程(PID: 689361)
- **测试步骤**:
  1. 连接Arthas: `/usr/local/java/jdk1.8.0_171/bin/java -jar arthas-boot.jar 689361`
  2. 查找消费者类: `sc *BillConsumer* *MessageListener*`
  3. 监控消费方法调用
  4. 观察消息处理过程
  5. 跟踪数据库操作链路
- **预期结果**: 清晰监控消息消费和处理过程

### 4.2 数据库工具分析测试（修正）

#### TC-015 跨数据库数据一致性验证
- **前置条件**: 账单数据在rtx和fenmi_etc库中需要保持一致
- **测试步骤**:
  1. 连接MySQL服务器(192.168.1.60:3306)
  2. 同时查询rtx.rtx_etc_deduc_bill和fenmi_etc.etc_deduct_bill
  3. 对比相同账单的数据完整性
  4. 验证字段值的匹配情况
  5. 检查数据同步的及时性
- **预期结果**: 跨库数据保持一致，无数据丢失或错误

#### TC-016 MQ错误日志分析测试
- **前置条件**: rtx.sys_mq_error_log表记录了推送错误信息
- **测试步骤**:
  1. 查询MQ错误日志数据
  2. 分析错误类型和频率
  3. 验证错误信息的完整性
  4. 检查重试机制的记录
  5. 分析错误恢复情况
- **预期结果**: 错误日志完整准确，能支持问题排查

---

## 5. 测试执行计划（修正）

### 5.1 测试环境准备

#### 5.1.1 数据库环境
```sql
-- 连接MySQL服务器
mysql -h192.168.1.60 -uroot -pfm123456

-- 验证任通行数据库
USE rtx;
SHOW TABLES LIKE '%bill%';
DESCRIBE rtx_etc_deduc_bill;
DESCRIBE sys_mq_error_log;

-- 验证新发行ETC数据库
USE fenmi_etc;
DESCRIBE etc_deduct_bill;

-- 清理测试数据（如果需要）
DELETE FROM rtx.rtx_etc_deduc_bill WHERE id >= 20240821001000 AND id <= 20240821001999;
DELETE FROM fenmi_etc.etc_deduct_bill WHERE bill_date = '20240821' AND operator_code = 'TXB';
```

#### 5.1.2 MQ环境验证
```bash
# 检查RabbitMQ状态
curl -u admin:admin http://192.168.1.58:15672/api/overview

# 检查关键队列
curl -u admin:admin "http://192.168.1.58:15672/api/queues/%2Ffenmi_test/bill.deduct.record.queue"

# 检查队列消息数量
curl -u admin:admin "http://192.168.1.58:15672/api/queues/%2Ffenmi_test" | grep bill
```

#### 5.1.3 应用进程验证
```bash
# SSH连接服务器
ssh root@192.168.1.60

# 检查关键Java进程
ps -ef | grep -E "(rtx-quartz|fenmi-etc-job)" | grep -v grep

# 确认进程ID
# rtx-quartz: 699103 (推送方)
# fenmi-etc-job: 689361 (接收方)
```

### 5.2 测试数据准备（修正）

```sql
-- 在任通行系统准备测试数据
USE rtx;
INSERT INTO rtx_etc_deduc_bill (
    id, bill_date, operator_code, bill_amt, bill_count,
    bill_status, pay_status, create_time, update_time, create_by
) VALUES 
(20240821001001, '20240821', 'TXB', 500.00, 10, 0, 0, NOW(), NOW(), 'TEST'),
(20240821001002, '20240821', 'TXB', 300.00, 6, 0, 0, NOW(), NOW(), 'TEST'),
(20240821001003, '20240822', 'TXB', 800.00, 15, 0, 0, NOW(), NOW(), 'TEST');

-- 验证测试数据
SELECT id, bill_date, operator_code, bill_amt, bill_status, create_time 
FROM rtx_etc_deduc_bill 
WHERE id BETWEEN 20240821001001 AND 20240821001003;
```

### 5.3 测试执行顺序（修正）

#### 阶段1: 基础推送功能测试 (TC-001 ~ TC-006)
1. 任通行账单推送功能测试
2. 新发行ETC消息消费测试
3. 幂等性和异常处理测试

#### 阶段2: 回调和补偿机制测试 (TC-007 ~ TC-010)
1. 回调确认机制测试
2. 定时补偿任务测试

#### 阶段3: 数据一致性验证 (TC-011 ~ TC-012)
1. 跨系统数据一致性测试
2. 异常恢复测试

#### 阶段4: 工具深度分析测试 (TC-013 ~ TC-016)
1. Arthas监控测试
2. 数据库分析测试

### 5.4 关键监控指标（修正）

#### 5.4.1 推送方监控指标（rtx系统）
- 账单推送成功率
- MQ发送延迟
- 错误消息数量（sys_mq_error_log表）
- 补偿任务执行频率
- 推送状态更新及时性

#### 5.4.2 接收方监控指标（fenmi_etc系统）
- 消息消费成功率
- 消息处理延迟
- 数据落库准确性
- 回调确认成功率
- 队列积压情况

#### 5.4.3 数据一致性指标
- 跨系统账单数据匹配率
- 金额差异统计
- 数据同步延迟
- 异常数据处理情况

---

## 6. 测试用例总结（修正版）

本修正版测试用例文档共包含 **16个核心测试案例**，准确覆盖了任通行(rtx)通过MQ推送账单到新发行ETC(fenmi_etc)逻辑优化的所有关键功能点：

- **账单推送基础功能测试**：6个测试案例 (TC-001 ~ TC-006)
- **回调和补偿机制测试**：4个测试案例 (TC-007 ~ TC-010)  
- **数据一致性验证测试**：2个测试案例 (TC-011 ~ TC-012)
- **工具深度分析测试**：4个测试案例 (TC-013 ~ TC-016)

### 核心修正要点

✅ **正确的数据流向**: rtx.rtx_etc_deduc_bill → MQ → fenmi_etc.etc_deduct_bill
✅ **正确的系统角色**: 任通行(rtx)=推送方，新发行ETC(fenmi_etc)=接收方
✅ **正确的表结构**: 基于实际数据库表结构设计测试
✅ **正确的监控方法**: 针对实际进程PID和类名进行监控
✅ **正确的业务逻辑**: 内蒙账单(TXB)从任通行推送到新发行ETC系统

测试用例充分考虑了真实的业务场景和技术实现，确保能够有效验证系统优化后解决内蒙账单数据不一致问题的能力。
