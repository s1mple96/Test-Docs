# 任通行通过MQ推送账单到新发行ETC逻辑优化测试用例（最终修正版）

## 1. 需求背景分析（最终修正）

### 1.1 运营商代码明确
- **MTK** - 蒙通卡（内蒙古运营商）⭐ **需求中的内蒙账单**
- **TXB** - 通行宝（江苏运营商）
- **JTK** - 捷通卡（广西运营商）

### 1.2 正确的业务流程
- **问题**: 20240802发现内蒙账单（**MTK运营商**）与运营商数据不一致
- **根本原因**: 任通行系统推送MTK账单到新发行ETC系统时MQ异常，导致数据未成功落库
- **数据流向**: `rtx.rtx_etc_deducrecord_mtk` → MQ → `fenmi_etc.etc_deduct_record_mtk_2025`

### 1.3 系统架构（最终修正）
```
任通行系统(rtx)                           新发行ETC系统(fenmi_etc)
├─ rtx_etc_deducrecord_mtk         →      ├─ etc_deduct_record_mtk_2025
├─ rtx_etc_deducrecord_txb         →      ├─ etc_deduct_record_txb_2025  
├─ rtx_etc_deducrecord_jtk         →      ├─ etc_deduct_record_jtk_2025
├─ sys_mq_error_log (错误日志)      MQ     ├─ 消费MQ消息
└─ 定时任务推送补偿                       └─ 数据落库 + 回调确认
```

### 1.4 涉及系统和组件
- **发送方**: rtx-quartz (定时任务), rtx-app (业务应用)
- **接收方**: fenmi-etc-job (消息消费)
- **MQ系统**: RabbitMQ (192.168.1.58:5673)
- **数据库**: 按运营商+年份分表存储
- **监控工具**: Arthas (192.168.1.60服务器)

---

## 2. 核心表结构分析（最终修正）

### 2.1 任通行扣费记录表（推送方）

#### 2.1.1 MTK内蒙古扣费记录表 (rtx.rtx_etc_deducrecord_mtk)
```sql
-- 内蒙古运营商扣费记录（需求重点关注）
主要字段：
- id: bigint(32) PRIMARY KEY 
- bill_date: varchar(32) 账单日期
- trading_number: varchar(100) 交易流水号
- plate_num: varchar(50) 车牌号
- total_fee: varchar(10) 总费用
- operator_code: varchar(10) 运营商代码 (MTK)
- bill_status: char(2) 账单状态
- callback_flag: varchar(2) 回调标识
- callback_msg: varchar(255) 回调消息
```

#### 2.1.2 TXB江苏扣费记录表 (rtx.rtx_etc_deducrecord_txb)
```sql
-- 江苏运营商扣费记录（对比验证用）
字段结构与MTK表相同
```

### 2.2 新发行ETC扣费记录表（接收方）

#### 2.2.1 MTK内蒙古扣费记录表 (fenmi_etc.etc_deduct_record_mtk_2025)
```sql
-- 新发行ETC系统接收的内蒙古扣费记录（按年份分表）
主要字段：
- id: bigint(32) PRIMARY KEY
- bill_date: varchar(32) 账单日期  
- trading_number: varchar(100) 交易流水号
- car_num: varchar(50) 车牌号
- total_fee: decimal(10,2) 总费用
- operator_code: varchar(10) 运营商代码 (MTK)
- bill_status: int(2) 账单状态
- callback_flag: varchar(2) 回调标识
- bill_year: int(4) 账单年份
```

### 2.3 MQ错误日志表 (rtx.sys_mq_error_log)
```sql
-- MQ推送错误日志（推送方维护）
关键字段：
- id: bigint(20) PRIMARY KEY
- biz_id: varchar(32) 业务id（扣费记录ID）
- correlation_id: varchar(64) 投递id
- msg: text 消息内容
- routing_key: varchar(100) 路由键
- status: int 状态
- error_msg: varchar(255) 错误信息
```

---

## 3. 修正版测试用例

### 3.1 MTK内蒙古账单推送核心测试

#### TC-001 MTK内蒙古扣费记录推送正常流程测试（最终修正）
- **前置条件**: 任通行系统正常运行，MQ服务可用，新发行ETC系统正常
- **测试重点**: 验证内蒙古运营商(MTK)扣费记录的完整推送流程
- **测试步骤**:
  1. 在 `rtx.rtx_etc_deducrecord_mtk` 表插入测试扣费记录（operator_code='MTK'）
  2. 触发任通行定时任务推送MTK扣费记录
  3. 验证消息成功发送到MQ队列
  4. 验证新发行ETC系统成功消费MTK消息
  5. 检查 `fenmi_etc.etc_deduct_record_mtk_2025` 表数据落库情况
  6. 验证新发行ETC系统回调任通行确认处理结果
  7. 检查任通行端回调标识更新
- **预期结果**: MTK扣费记录从rtx成功推送到fenmi_etc，数据完整一致，状态正确更新

**测试数据准备**:
```sql
-- 在任通行系统插入MTK测试扣费记录
INSERT INTO rtx.rtx_etc_deducrecord_mtk (
    id, bill_date, trading_number, plate_num, total_fee, service_fee, fee,
    operator_code, bill_status, callback_flag, create_time, update_time
) VALUES (
    20240821001001, '20240821', 'MTK20240821001001', '蒙A12345', '50.00', '5.00', '45.00',
    'MTK', '0', '0', NOW(), NOW()
);
```

**验证方法**:
```sql
-- 验证任通行端MTK数据
SELECT id, bill_date, plate_num, total_fee, operator_code, bill_status, callback_flag 
FROM rtx.rtx_etc_deducrecord_mtk 
WHERE id = 20240821001001;

-- 验证新发行ETC端MTK数据
SELECT id, bill_date, car_num, total_fee, operator_code, bill_status, callback_flag 
FROM fenmi_etc.etc_deduct_record_mtk_2025 
WHERE bill_date = '20240821' AND car_num = '蒙A12345';

-- 数据一致性对比
SELECT 
    r.plate_num as rtx_plate,
    f.car_num as fenmi_plate,
    r.total_fee as rtx_fee,
    f.total_fee as fenmi_fee,
    r.bill_status as rtx_status,
    f.bill_status as fenmi_status
FROM rtx.rtx_etc_deducrecord_mtk r
LEFT JOIN fenmi_etc.etc_deduct_record_mtk_2025 f 
    ON r.trading_number = f.trading_number
WHERE r.id = 20240821001001;
```

#### TC-002 MTK账单推送异常处理和错误日志测试
- **前置条件**: 模拟MTK账单推送时MQ异常
- **测试步骤**:
  1. 在rtx.rtx_etc_deducrecord_mtk插入测试数据
  2. 故意断开MQ连接或配置错误参数
  3. 触发MTK账单推送任务
  4. 验证异常被正确捕获和处理
  5. 检查 `rtx.sys_mq_error_log` 表MTK相关错误记录
  6. 验证MTK账单预警机制触发
- **预期结果**: MTK推送异常被正确处理，错误信息完整记录

**错误日志验证**:
```sql
-- 检查MTK相关MQ错误日志
SELECT id, biz_id, correlation_id, routing_key, status, error_msg, send_time 
FROM rtx.sys_mq_error_log 
WHERE biz_id = '20240821001001'
   OR msg LIKE '%MTK%'
   OR msg LIKE '%蒙A12345%'
ORDER BY create_time DESC;
```

#### TC-003 MTK账单推送重试机制测试
- **前置条件**: 配置MTK账单推送重试机制
- **测试步骤**:
  1. 插入MTK扣费记录
  2. 模拟网络抖动导致MTK推送临时失败
  3. 观察重试机制对MTK数据的处理
  4. 恢复网络连接
  5. 验证MTK账单最终推送成功
  6. 检查重试次数和状态变化记录
- **预期结果**: MTK推送重试机制正常工作，最终成功

### 3.2 多运营商推送对比测试

#### TC-004 MTK与TXB运营商推送对比测试
- **前置条件**: 同时测试MTK和TXB两个运营商的推送
- **测试步骤**:
  1. 分别在rtx_etc_deducrecord_mtk和rtx_etc_deducrecord_txb插入测试数据
  2. 同时触发两个运营商的推送任务
  3. 对比两个运营商的推送处理逻辑
  4. 验证分别落库到对应的年份表
  5. 检查两个运营商推送的成功率和性能差异
- **预期结果**: 两个运营商推送机制一致，都能正常工作

**多运营商验证**:
```sql
-- 同时插入MTK和TXB测试数据
INSERT INTO rtx.rtx_etc_deducrecord_mtk (
    id, bill_date, trading_number, plate_num, total_fee, operator_code, bill_status, create_time
) VALUES (
    20240821002001, '20240821', 'MTK20240821002001', '蒙B12345', '60.00', 'MTK', '0', NOW()
);

INSERT INTO rtx.rtx_etc_deducrecord_txb (
    id, bill_date, trading_number, plate_num, total_fee, operator_code, bill_status, create_time  
) VALUES (
    20240821002002, '20240821', 'TXB20240821002002', '苏A12345', '70.00', 'TXB', '0', NOW()
);

-- 验证两个运营商数据推送结果
SELECT 'MTK' as operator, COUNT(*) as pushed_count 
FROM fenmi_etc.etc_deduct_record_mtk_2025 
WHERE bill_date = '20240821' AND car_num LIKE '蒙B%'
UNION ALL
SELECT 'TXB' as operator, COUNT(*) as pushed_count 
FROM fenmi_etc.etc_deduct_record_txb_2025 
WHERE bill_date = '20240821' AND car_num LIKE '苏A%';
```

### 3.3 新发行ETC系统MTK消息处理测试

#### TC-005 新发行ETC系统MTK消息消费测试
- **前置条件**: fenmi-etc-job能正确处理MTK消息
- **测试步骤**:
  1. 从rtx发送MTK扣费记录消息到MQ
  2. 监控新发行ETC系统消费MTK消息的过程
  3. 验证MTK消息解析和处理逻辑
  4. 检查数据正确落库到etc_deduct_record_mtk_2025表
  5. 验证MTK消费确认机制
- **预期结果**: 新发行ETC系统正确消费MTK消息，数据准确落库

**MTK消息监控**:
```bash
# 监控MTK相关的MQ队列
curl -u admin:admin "http://192.168.1.58:15672/api/queues/%2Ffenmi_test" | grep -i mtk

# 使用Arthas监控fenmi-etc-job处理MTK消息
/usr/local/java/jdk1.8.0_171/bin/java -jar arthas-boot.jar 689361 -c "watch *MTK* * '{params, returnObj}' -n 5"
```

#### TC-006 MTK消息幂等性保护测试
- **前置条件**: 同一MTK扣费记录可能被重复推送
- **测试步骤**:
  1. 发送重复的MTK扣费记录消息
  2. 验证新发行ETC系统对MTK消息的幂等性保护
  3. 检查重复的MTK消息不会重复落库
  4. 验证trading_number的唯一性约束
- **预期结果**: MTK消息具备幂等性保护，重复消息不导致数据异常

### 3.4 MTK回调机制测试

#### TC-007 新发行ETC回调任通行MTK处理确认测试
- **前置条件**: 新发行ETC处理完MTK扣费记录后回调任通行
- **测试步骤**:
  1. 新发行ETC成功处理MTK扣费记录并落库
  2. 触发回调任通行的MTK处理确认接口
  3. 验证MTK回调数据的完整性
  4. 检查任通行接收MTK回调并更新callback_flag
  5. 验证MTK扣费记录状态的变更
- **预期结果**: MTK回调机制正常，任通行能收到确认并更新状态

**MTK回调验证**:
```sql
-- 检查任通行端MTK回调状态更新
SELECT id, bill_date, plate_num, total_fee, bill_status, callback_flag, callback_msg, update_time 
FROM rtx.rtx_etc_deducrecord_mtk 
WHERE id = 20240821001001;

-- 对比处理前后的callback_flag变化
SELECT id, bill_date, plate_num, 
       CASE callback_flag 
           WHEN '0' THEN '未回调' 
           WHEN '1' THEN '回调成功' 
           WHEN '2' THEN '回调失败' 
           ELSE '未知状态' 
       END as callback_status,
       callback_msg
FROM rtx.rtx_etc_deducrecord_mtk 
WHERE bill_date = '20240821';
```

### 3.5 MTK定时补偿机制测试

#### TC-008 MTK超时扣费记录自动补偿测试
- **前置条件**: 配置MTK扣费记录处理超时时间和补偿机制
- **测试步骤**:
  1. 在rtx插入MTK扣费记录但阻止推送
  2. 等待超过预设的超时时间
  3. 验证补偿定时任务识别超时的MTK记录
  4. 检查超时MTK记录被重新推送
  5. 验证补偿后的MTK数据状态更新
- **预期结果**: 补偿机制自动发现并重推超时MTK扣费记录

**MTK补偿监控**:
```sql
-- 监控需要补偿的MTK扣费记录
SELECT id, bill_date, plate_num, total_fee, bill_status, callback_flag, create_time, update_time
FROM rtx.rtx_etc_deducrecord_mtk 
WHERE bill_status = '0' 
  AND callback_flag = '0'
  AND create_time < NOW() - INTERVAL 30 MINUTE
ORDER BY create_time DESC;

-- 检查MTK补偿处理结果
SELECT 
    COUNT(*) as total_mtk_records,
    SUM(CASE WHEN callback_flag = '1' THEN 1 ELSE 0 END) as success_count,
    SUM(CASE WHEN callback_flag = '0' THEN 1 ELSE 0 END) as pending_count,
    SUM(CASE WHEN callback_flag = '2' THEN 1 ELSE 0 END) as failed_count
FROM rtx.rtx_etc_deducrecord_mtk 
WHERE bill_date = '20240821';
```

### 3.6 MTK数据一致性验证测试

#### TC-009 MTK跨系统数据一致性测试
- **前置条件**: MTK扣费记录在rtx和fenmi_etc系统间完整流转
- **测试步骤**:
  1. 在rtx.rtx_etc_deducrecord_mtk创建完整的MTK测试数据
  2. 执行完整的MTK推送流程
  3. 详细对比两个系统的MTK数据
  4. 验证关键字段的一致性（车牌、金额、时间等）
  5. 检查MTK数据的完整性和准确性
- **预期结果**: 两系统MTK数据高度一致，无数据丢失或错误

**MTK数据一致性验证**:
```sql
-- MTK数据完整性对比分析
SELECT 
    'rtx端MTK记录' as data_source,
    COUNT(*) as record_count,
    SUM(CAST(total_fee AS DECIMAL(10,2))) as total_amount,
    COUNT(DISTINCT plate_num) as unique_plates
FROM rtx.rtx_etc_deducrecord_mtk 
WHERE bill_date = '20240821'
UNION ALL
SELECT 
    'fenmi_etc端MTK记录' as data_source,
    COUNT(*) as record_count,
    SUM(total_fee) as total_amount,
    COUNT(DISTINCT car_num) as unique_plates
FROM fenmi_etc.etc_deduct_record_mtk_2025 
WHERE bill_date = '20240821';

-- MTK详细数据对比
SELECT 
    r.id as rtx_id,
    f.id as fenmi_id,
    r.plate_num as rtx_plate,
    f.car_num as fenmi_plate,
    r.total_fee as rtx_fee,
    f.total_fee as fenmi_fee,
    CASE 
        WHEN r.total_fee = CAST(f.total_fee AS CHAR) THEN '一致'
        ELSE '不一致'
    END as fee_match,
    r.bill_status as rtx_status,
    f.bill_status as fenmi_status
FROM rtx.rtx_etc_deducrecord_mtk r
LEFT JOIN fenmi_etc.etc_deduct_record_mtk_2025 f 
    ON r.trading_number = f.trading_number
WHERE r.bill_date = '20240821'
ORDER BY r.create_time DESC;
```

---

## 4. 使用工具进行MTK专项分析测试

### 4.1 Arthas工具监控MTK推送（修正）

#### TC-010 基于Arthas监控MTK推送方法
- **前置条件**: 连接到rtx-quartz进程(PID: 699103)
- **测试步骤**:
  1. 连接Arthas: `/usr/local/java/jdk1.8.0_171/bin/java -jar arthas-boot.jar 699103`
  2. 查找MTK相关类: `sc *MTK* *FmYqtSettBillTask*`
  3. 监控MTK推送方法: `monitor *FmYqtSettBillTask* *MTK* -n 5`
  4. 观察MTK推送参数: `watch *FmYqtSettBillTask* * '{params, returnObj}' -n 3`
  5. 跟踪MTK推送调用链: `trace *FmYqtSettBillTask* *`
- **预期结果**: 清晰观察MTK推送的完整执行过程

### 4.2 数据库工具MTK专项分析

#### TC-011 MTK数据库分析测试
- **前置条件**: MTK数据在rtx和fenmi_etc库中的分布分析
- **测试步骤**:
  1. 连接MySQL服务器(192.168.1.60:3306)
  2. 分析rtx.rtx_etc_deducrecord_mtk表的数据分布
  3. 分析fenmi_etc.etc_deduct_record_mtk_2025表的数据分布
  4. 对比MTK数据的推送比例和成功率
  5. 分析MTK数据的时间分布和处理延迟
- **预期结果**: 全面了解MTK数据的处理情况和性能指标

**MTK专项分析SQL**:
```sql
-- MTK数据分布分析
SELECT 
    bill_date,
    COUNT(*) as mtk_count,
    COUNT(DISTINCT plate_num) as unique_vehicles,
    SUM(CAST(total_fee AS DECIMAL(10,2))) as total_amount,
    AVG(CAST(total_fee AS DECIMAL(10,2))) as avg_amount,
    COUNT(CASE WHEN callback_flag = '1' THEN 1 END) as success_callback,
    COUNT(CASE WHEN callback_flag = '0' THEN 1 END) as pending_callback,
    COUNT(CASE WHEN callback_flag = '2' THEN 1 END) as failed_callback
FROM rtx.rtx_etc_deducrecord_mtk 
WHERE bill_date >= '20240820'
GROUP BY bill_date
ORDER BY bill_date DESC;

-- MTK推送性能分析
SELECT 
    DATE(create_time) as push_date,
    HOUR(create_time) as push_hour,
    COUNT(*) as push_count,
    AVG(TIMESTAMPDIFF(MINUTE, create_time, update_time)) as avg_process_minutes
FROM rtx.rtx_etc_deducrecord_mtk 
WHERE create_time >= '2024-08-20'
  AND callback_flag = '1'
GROUP BY DATE(create_time), HOUR(create_time)
ORDER BY push_date DESC, push_hour DESC;
```

---

## 5. 测试执行计划（MTK专项）

### 5.1 MTK测试环境准备

```sql
-- 清理MTK测试数据
DELETE FROM rtx.rtx_etc_deducrecord_mtk WHERE id >= 20240821001000 AND id <= 20240821999999;
DELETE FROM fenmi_etc.etc_deduct_record_mtk_2025 WHERE bill_date = '20240821' AND car_num LIKE '%测试%';

-- 准备MTK测试数据集
INSERT INTO rtx.rtx_etc_deducrecord_mtk (
    id, bill_date, trading_number, plate_num, total_fee, service_fee, fee,
    operator_code, bill_status, callback_flag, create_time, update_time, create_by
) VALUES 
(20240821001001, '20240821', 'MTK20240821001001', '蒙A测试01', '50.00', '5.00', '45.00', 'MTK', '0', '0', NOW(), NOW(), 'TEST'),
(20240821001002, '20240821', 'MTK20240821001002', '蒙B测试02', '60.00', '6.00', '54.00', 'MTK', '0', '0', NOW(), NOW(), 'TEST'),
(20240821001003, '20240821', 'MTK20240821001003', '蒙C测试03', '70.00', '7.00', '63.00', 'MTK', '0', '0', NOW(), NOW(), 'TEST');

-- 验证MTK测试数据
SELECT id, bill_date, plate_num, total_fee, operator_code, bill_status, callback_flag 
FROM rtx.rtx_etc_deducrecord_mtk 
WHERE id BETWEEN 20240821001001 AND 20240821001003;
```

### 5.2 MTK测试执行顺序

#### 阶段1: MTK基础推送测试 (TC-001 ~ TC-003)
1. MTK正常推送流程测试
2. MTK推送异常处理测试
3. MTK推送重试机制测试

#### 阶段2: MTK对比和消费测试 (TC-004 ~ TC-006)  
1. MTK与其他运营商对比测试
2. MTK消息消费测试
3. MTK幂等性测试

#### 阶段3: MTK回调和补偿测试 (TC-007 ~ TC-008)
1. MTK回调确认测试
2. MTK补偿机制测试

#### 阶段4: MTK数据一致性和工具分析 (TC-009 ~ TC-011)
1. MTK数据一致性验证
2. MTK专项工具分析

### 5.3 MTK关键监控指标

#### 5.3.1 MTK推送指标
- MTK扣费记录推送成功率
- MTK消息发送延迟
- MTK错误日志数量
- MTK回调确认率

#### 5.3.2 MTK数据质量指标
- MTK数据一致性比例
- MTK金额匹配准确率
- MTK车牌信息完整性
- MTK处理时效性

---

## 6. 测试用例总结（MTK专项修正版）

本最终修正版测试用例文档共包含 **11个专项测试案例**，专门针对**内蒙古运营商(MTK)**的账单推送优化进行深度测试：

- **MTK账单推送核心测试**：3个测试案例 (TC-001 ~ TC-003)
- **MTK多运营商对比测试**：3个测试案例 (TC-004 ~ TC-006)
- **MTK回调和补偿测试**：2个测试案例 (TC-007 ~ TC-008)  
- **MTK数据一致性测试**：1个测试案例 (TC-009)
- **MTK专项工具分析**：2个测试案例 (TC-010 ~ TC-011)

### 最终修正要点 ⭐

✅ **正确识别内蒙运营商**: MTK=蒙通卡，不是TXB
✅ **正确的表结构**: 按运营商+年份分表存储
✅ **正确的数据流**: rtx_etc_deducrecord_mtk → etc_deduct_record_mtk_2025
✅ **正确的业务理解**: 扣费记录(deducrecord)是核心业务数据
✅ **针对性测试**: 专门测试MTK内蒙古账单的推送优化

这个修正版测试用例完全基于实际的表结构和业务逻辑，能够准确验证内蒙古账单推送优化的效果。
