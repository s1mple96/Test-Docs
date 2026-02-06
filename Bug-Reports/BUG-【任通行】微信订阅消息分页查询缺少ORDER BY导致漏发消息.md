# BUG-【任通行】微信订阅消息分页查询缺少ORDER BY导致漏发消息

## 【BUG描述】
微信订阅消息-ETC欠费提醒定时任务在查询失败账单时，由于SQL缺少ORDER BY子句，导致分页查询结果顺序不确定，造成部分用户漏发消息。生产环境有几十万条账单数据时，该问题会频繁出现，影响用户体验。

## 【前置条件】
- 环境：生产环境
- 数据：数据库中存在大量（几十万条）status=2（代扣失败）的账单记录
- 定时任务：微信订阅消息-ETC欠费提醒定时任务（cron: 0 0 9 * * ?）
- 测试车辆：渝AL25A5（钱包ID: 1612262137303232513，账单ID: 2013997867492057089）

## 【复现步骤】
1. 准备测试数据：确保数据库中有大量（>2000条）status=2的账单记录
2. 在22号执行微信订阅消息定时任务，观察消息发送记录
3. 确认某台车（如渝AL25A5）在22号收到了欠费提醒消息
4. 保持该车账单状态不变（status=2，未补扣成功）
5. 在23号再次执行定时任务
6. 查询该车的消息发送记录

## 【期望结果】
- 每天定时任务执行时，所有status=2（代扣失败）的账单都应该被查询到
- 每台欠费车辆每天都应该收到一次欠费提醒消息
- 渝AL25A5在22号和23号都应该收到消息

## 【实际结果】
- 渝AL25A5在22号07:22:40收到了消息（消息记录ID: 2014115152121495554）
- 23号定时任务执行后，该车未收到消息
- 查询消息发送记录表（wx_msg_send_record_202601），23号没有该车的发送记录
- 但该车账单状态仍为status=2（代扣失败），符合发送条件

**对比案例**：
- 粤BBJ2820在同一天（22号）收到了2次消息（12:00和14:00各一次）
- 说明同一台车在不同时间可以多次发送，不存在去重机制

## 【附件】

### 1. 消息发送记录
```json
// 渝AL25A5 - 22号发送成功
{
  "id": "2014115152121495554",
  "sendDate": "2026-01-22",
  "sendTime": "2026-01-22 07:22:40",
  "carNumber": "渝AL25A5",
  "bizSourceId": "1612262137303232513",
  "sendStatus": 2,
  "resultMsg": "ok"
}

// 渝AL25A5 - 23号无记录

// 粤BBJ2820 - 22号发送2次
{
  "sendTime": "2026-01-22 12:10:02",
  "carNumber": "粤BBJ2820",
  "bizSourceId": "1651832488148455425"
},
{
  "sendTime": "2026-01-22 14:10:04",
  "carNumber": "粤BBJ2820",
  "bizSourceId": "1651832488148455425"
}
```

### 2. 账单数据
```json
// 渝AL25A5的账单（22号6点垫付扣款）
{
  "id": "2013997867492057089",
  "createTime": "2026-01-21 23:31:25",
  "updateTime": "2026-01-22 06:00:49",
  "updateBy": "钱包余额补扣-完成修改",
  "plateNum": "渝AL25A5",
  "status": "2",  // 代扣失败
  "payType": "7",  // 钱包垫付
  "operatorCode": "MTK",
  "totalFee": "148.74"
}
```

### 3. 钱包流水
```json
// 22号6点垫付扣款，余额变负
{
  "id": "2014095863261044737",
  "createTime": "2026-01-22 06:00:49",
  "operatorType": 33,
  "operatorTypeName": "通行扣款-垫付",
  "sourceId": "2013997867492057089",
  "beforeAmount": 0.0,
  "tradeAmount": 148.74,
  "surplusAmount": -148.74
}

// 无"钱包补扣入金"记录，说明未补扣成功
```

## 【原因定位】

### 1. 核心问题：SQL缺少ORDER BY子句

**问题代码位置**：
- 文件：`java/rtx/rtx-service/src/main/resources/mapper/EtcDeducrecordMapper.xml`
- 行号：856-882
- SQL ID：`selectDeductRecordIds`

**问题SQL**：
```xml
<select id="selectDeductRecordIds" resultType="com.rtx.project.dto.bill.ExecuteBillDeductParam">
    select
        t1.id as deductRecordId,
        t1.etc_card_user_id as cardUserId,
        t1.operator_code as operatorCode,
        t2.user_wallet_id as walletId,
        t1.create_time as createTime
    FROM rtx_etc_deducrecord t1 
    LEFT JOIN rtx_etc_card_user t2 ON t1.etc_card_user_id = t2.id
    where t1.operator_code = #{operatorCode}
      and t1.status = #{status}
      and t1.id > #{pageMaxId}
    limit 2000
    <!-- ❌ 缺少 ORDER BY t1.id ASC -->
</select>
```

### 2. 分页逻辑问题

**问题代码位置**：
- 文件：`java/rtx/rtx-service/src/main/java/com/rtx/project/bizservice/bill/deduct/impl/DeductBillSceneServiceImpl.java`
- 方法：`walletAdvanceWxMsgSend`
- 行号：411-424

**问题代码**：
```java
public void walletAdvanceWxMsgSend(String operatorCode) {
    Long maxId = 0L;  // 每次从0开始
    List<String> walletAdvanceList = PayTypeEnum.getAllCode();
    while (true) {
        // 查询 id > maxId 的账单，但SQL没有ORDER BY
        List<ExecuteBillDeductParam> records = deductRecordService.selectDeductRecordIds(
            EtcDeducrecordStatusEnum.WITHHOLDING_FAILED.getCode(), 
            operatorCode, 
            maxId, 
            walletAdvanceList
        );
        if (CollectionUtils.isEmpty(records)) {
            break;
        }
        // ❌ 取最后一条的ID作为maxId，但数据是乱序的
        maxId = records.get(records.size() - 1).getDeductRecordId();
        List<Long> cardUserIds = records.stream()
            .map(ExecuteBillDeductParam::getCardUserId)
            .collect(Collectors.toList());
        etcCardUserService.sendArrearsWechatMsg(cardUserIds);
    }
}
```

### 3. 问题原理

**没有ORDER BY时的行为**：
- MySQL返回顺序不确定（可能按主键索引，也可能按物理存储顺序）
- 每次执行返回顺序可能不同
- 生产环境有几十万条数据时，问题更明显

**举例说明**：

假设有5条账单：ID: 100, 200, 300, 400, 500

**22号执行**（假设返回顺序）：
```
第1次：id > 0, LIMIT 2000 → 返回 [100, 500]（乱序，实际会返回更多，这里简化）
       maxId = 500（取最后一条）
第2次：id > 500, LIMIT 2000 → 返回 []
结果：发送了 100, 500，漏掉了 200, 300, 400
```

**23号执行**（假设返回顺序）：
```
第1次：id > 0, LIMIT 2000 → 返回 [200, 300]（乱序）
       maxId = 300（取最后一条）
第2次：id > 300, LIMIT 2000 → 返回 [400, 500]
       maxId = 500
第3次：id > 500, LIMIT 2000 → 返回 []
结果：发送了 200, 300, 400, 500，漏掉了 100
```

**渝AL25A5（ID: 2013997867492057089）**：
- 22号被查询到并发送
- 23号因为返回顺序不同，被跳过了

### 4. 为什么粤BBJ2820发送了2次？

**原因**：
- 12:00执行时，欠费76.12，发送消息
- 12:00-14:00之间，又产生新账单，欠费增加到154.25
- 14:00执行时，再次查询到该车，再次发送
- **说明没有去重机制，每次查询到都会发送**

## 【修复建议】

### 方案1：SQL添加ORDER BY（推荐，最简单）

**修改文件**：`java/rtx/rtx-service/src/main/resources/mapper/EtcDeducrecordMapper.xml`

```xml
<select id="selectDeductRecordIds" resultType="com.rtx.project.dto.bill.ExecuteBillDeductParam">
    select
        t1.id as deductRecordId,
        t1.etc_card_user_id as cardUserId,
        t1.operator_code as operatorCode,
        t2.user_wallet_id as walletId,
        t1.create_time as createTime
    FROM rtx_etc_deducrecord t1 
    LEFT JOIN rtx_etc_card_user t2 ON t1.etc_card_user_id = t2.id
    where t1.operator_code = #{operatorCode}
      and t1.status = #{status}
      and t1.id > #{pageMaxId}
    ORDER BY t1.id ASC  <!-- ✅ 添加排序 -->
    limit 2000
</select>
```

**优点**：
- 修改简单，只需加一行
- 确保分页顺序正确
- 不会漏数据

**缺点**：
- 如果数据量极大，ORDER BY可能影响性能（但id是主键，影响很小）

### 方案2：代码中取真正的最大ID

**修改文件**：`java/rtx/rtx-service/src/main/java/com/rtx/project/bizservice/bill/deduct/impl/DeductBillSceneServiceImpl.java`

```java
public void walletAdvanceWxMsgSend(String operatorCode) {
    Long maxId = 0L;
    List<String> walletAdvanceList = PayTypeEnum.getAllCode();
    while (true) {
        List<ExecuteBillDeductParam> records = deductRecordService.selectDeductRecordIds(
            EtcDeducrecordStatusEnum.WITHHOLDING_FAILED.getCode(), 
            operatorCode, 
            maxId, 
            walletAdvanceList
        );
        if (CollectionUtils.isEmpty(records)) {
            break;
        }
        // ✅ 取真正的最大ID，而不是最后一条
        maxId = records.stream()
            .map(ExecuteBillDeductParam::getDeductRecordId)
            .max(Long::compare)
            .orElse(maxId);
        
        List<Long> cardUserIds = records.stream()
            .map(ExecuteBillDeductParam::getCardUserId)
            .collect(Collectors.toList());
        etcCardUserService.sendArrearsWechatMsg(cardUserIds);
    }
}
```

**优点**：
- 不修改SQL
- 能正确处理乱序数据

**缺点**：
- 治标不治本，SQL还是乱序的
- 如果有其他地方调用这个SQL，还是会有问题

### 方案3：添加Redis去重机制（推荐，长期优化）

**修改文件**：`java/rtx/rtx-service/src/main/java/com/rtx/project/service/impl/EtcCardUserServiceImpl.java`

```java
@Override
public void sendArrearsWechatMsg(List<Long> carUserIds) {
    if (CollectionUtils.isEmpty(carUserIds)) {
        return;
    }
    List<Long> carUserIdList = carUserIds.stream().distinct().collect(Collectors.toList());
    for (Long carUserId : carUserIdList) {
        try {
            // ✅ 添加Redis去重：每天只发一次
            String key = "wx:arrears:msg:" + carUserId + ":" + LocalDate.now();
            if (redisCache.hasKey(key)) {
                log.info("今天已发送欠费提醒，跳过：carUserId={}", carUserId);
                continue;
            }
            
            EtcCardUser etcCardUser = this.selectEtccardUserById(carUserId);
            if (Objects.isNull(etcCardUser)) {
                continue;
            }
            // ... 原有发送逻辑 ...
            
            FenMiWxMessageUtils.sendWxServiceMsg(param, WxMsgSceneTypeEnum.SCENE_TYPE_3);
            smsService.sendMsgServiceMsg(etcCardUser.getPhone(), Constants.WX_MSG_NOTICE_TYPE_5, map);
            
            // ✅ 标记已发送，过期时间25小时
            redisCache.setCacheObject(key, "1", 25L, TimeUnit.HOURS);
            
        } catch (Exception e) {
            log.error("sendArrearsWechatMsg error, carUserId={}", carUserId, e);
        }
    }
}
```

**优点**：
- 彻底避免重复发送
- 即使分页有问题，也不会重复发送
- 提升用户体验

**缺点**：
- 需要依赖Redis
- 代码改动较大

### 推荐修复顺序

1. **立即修复**：方案1（SQL加ORDER BY）- 最简单，解决根本问题
2. **长期优化**：方案3（Redis去重）- 提升健壮性，避免重复发送
3. **监控告警**：记录每次执行处理的账单数量，异常时告警

### 测试验证

修复后需验证：
1. 查询结果按ID升序排列
2. 分页查询不会漏数据
3. 同一台车每天只发送一次消息（如果加了去重）
4. 生产环境大数据量下稳定运行
