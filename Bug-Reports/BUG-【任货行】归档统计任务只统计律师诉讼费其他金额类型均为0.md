# BUG-【任货行】归档统计任务使用旧blackTime导致新流水未被统计

## 【问题描述】
执行历史钱包流水归档统计任务时，虽然删除了所有统计记录，但使用的是旧的blackTime，导致新增的历史流水因CREATE_TIME不满足条件而未被统计，所有金额字段（lateFee、lawyerFee、monthFee、lateFeeRefund）均为0。

**实际结果：**
- blackTime: 2024-01-05 17:26:51（旧值，未更新）
- lateFee: 0
- lawyerFee: 0
- monthFee: 0
- lateFeeRefund: 0

**预期结果：**
- blackTime: 2024-10-21 10:00:00（最新余额>0的流水时间）
- lateFee: -350.80
- lawyerFee: -130.00
- monthFee: -55.00
- lateFeeRefund: 55.50

## 【请求参数】
**测试用户：**
- 用户ID（USER_ID）: `bcf66d3e6279458e898ac4f990d08a3b`
- 钱包ID（TRUCKUSERWALLET_ID）: `a71df799b41b44e9a859d1c86ae192fa`

**归档任务接口：**
```
GET http://127.0.0.1:8087/hcbTimer-bill/test/etcCardBalanceFlowStatistics
```

**测试数据（MongoDB - hcb_etccardbalanceflow_backup_2024）：**
- 下黑时间标记：2024-10-01 10:00:00（余额=1000.00，OPERATOR_TYPE=1）
- 滞纳金流水（OPERATOR_TYPE=11）：2条，总计-350.80元
- 律师诉讼费流水（OPERATOR_TYPE=71）：2条，总计-130.00元
- 月管理费流水（OPERATOR_TYPE=100）：2条，总计-55.00元
- 滞纳金退费流水（OPERATOR_TYPE=108）：2条，总计+55.50元

所有流水的CREATE_TIME均 >= 下黑时间（2024-10-01 10:00:00）

**注意：** 该用户之前的统计记录中blackTime为2024-01-05 17:26:51

## 【复现步骤】
1. 选择一个已有统计记录的黑名单用户（钱包ID: `a71df799b41b44e9a859d1c86ae192fa`），原统计记录的blackTime为`2024-01-05 17:26:51`

2. 在MongoDB jdbdb数据库的`hcb_etccardbalanceflow_backup_2024`表为该用户插入新的测试数据：
   - 1条下黑时间标记流水（2024-10-01，余额=1000，OPERATOR_TYPE=1）
   - 4种类型的费用流水各2条（时间从2024-10-05到2024-10-21）

3. 执行归档统计任务：
```bash
curl -X GET "http://127.0.0.1:8087/hcbTimer-bill/test/etcCardBalanceFlowStatistics"
```

4. 查询统计结果：
```javascript
mongo 192.168.1.60:27017/jdbdb --eval "
db.hcb_etc_card_balance_flow_statistics.findOne({
  truckUserWalletId: 'a71df799b41b44e9a859d1c86ae192fa'
})"
```

5. 观察结果：
   - blackTime仍然是旧值`2024-01-05 17:26:51`（未更新）
   - 所有金额字段均为0（新流水因时间不符合条件而未被统计）

## 【日志记录&截图标注】
**问题分析：**

归档任务代码逻辑：
1. 第82行：删除统计表所有记录 `mongoTemplate.remove(new Query(), tableName)`
2. 重新统计所有黑名单用户
3. 但使用的blackTime来自旧的数据源，未重新计算

**实际执行情况：**
```bash
# 执行前的统计记录
blackTime: 2024-01-05 17:26:51 (旧值)

# 执行归档任务后
blackTime: 2024-01-05 17:26:51 (仍然是旧值！)
lateFee: 0
lawyerFee: 0
monthFee: 0
lateFeeRefund: 0
```

**手工验证新测试数据（证明数据正确）：**
```bash
mongo 192.168.1.60:27017/jdbdb --eval "
db.hcb_etccardbalanceflow_backup_2024.aggregate([
  {\$match: {
    USER_ID: 'bcf66d3e6279458e898ac4f990d08a3b',
    OPERATOR_TYPE: {\$in: ['11','71','100','108']},
    CREATE_TIME: {\$gte: '2024-10-01 10:00:00'}
  }},
  {\$group: {
    _id: '\$OPERATOR_TYPE',
    total: {\$sum: {\$toDouble: '\$AMOUNT'}}
  }}
])"
```

结果：
```
{ "_id" : "100", "total" : -55 }      // monthFee
{ "_id" : "71", "total" : -130 }      // lawyerFee
{ "_id" : "108", "total" : 55.5 }     // lateFeeRefund
{ "_id" : "11", "total" : -350.8 }    // lateFee
```

但由于blackTime使用了旧值（2024-01-05），查询条件`CREATE_TIME >= '2024-01-05'`会包含大量历史数据，而新测试数据（2024-10月）的统计被历史数据覆盖或因其他逻辑问题未生效。

**疑似问题代码位置：**
`com.etc.api.bizService.etccardbalanceflow.EtcCardBalanceFlowBizServiceImpl.statistics()`

- 第82行：删除所有统计记录
- 第125-152行：`statisticsSingleUser()`方法中获取blackTime的逻辑，可能未重新从MongoDB流水表计算最新的blackTime

## 【预期结果】
归档统计任务每次执行时应该：
1. **重新计算blackTime**：从MongoDB流水表中查找余额>0的最新流水时间作为新的blackTime
2. **使用新的blackTime统计金额**：统计CREATE_TIME >= 新blackTime的所有流水
3. **正确统计所有4种类型的金额**：
   - lateFee（OPERATOR_TYPE=11）：-350.80
   - lawyerFee（OPERATOR_TYPE=71）：-130.00
   - monthFee（OPERATOR_TYPE=100）：-55.00
   - lateFeeRefund（OPERATOR_TYPE=108）：55.50
4. **更新blackTime字段**：统计表中的blackTime应该更新为2024-10-21 10:00:00

统计表`hcb_etc_card_balance_flow_statistics`的blackTime和4个金额字段应该基于最新的MongoDB流水数据计算。

