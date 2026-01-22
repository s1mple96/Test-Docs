# BUG-【任货行】归档任务每次执行都删除所有统计记录导致无法增量统计

## 【问题描述】
执行历史钱包流水归档统计任务时，任务会先删除统计表（`hcb_etc_card_balance_flow_statistics`）的所有记录，然后重新统计所有黑名单用户。这导致代码中"跳过本月已统计用户"的逻辑永远无法生效，每次都要全量统计，性能低下。

**实际结果：**
- 第1次执行归档任务，用户A的updateTime为 `2025-12-04 10:00:00`
- 第2次执行归档任务，用户A的updateTime变为 `2025-12-04 10:05:00`（未跳过，重新统计）

**预期结果：**
- 第1次执行归档任务，用户A的updateTime为 `2025-12-04 10:00:00`
- 第2次执行归档任务，用户A的updateTime仍为 `2025-12-04 10:00:00`（已跳过，statisticsDate为本月的用户不再处理）

## 【请求参数】
**归档任务接口：**
```
GET http://127.0.0.1:8087/hcbTimer-bill/test/etcCardBalanceFlowStatistics
```

**测试用户：**
任意已有统计记录且statisticsDate为当前月份（2025-12）的黑名单用户

**问题代码位置：**
`com.etc.api.bizService.etccardbalanceflow.EtcCardBalanceFlowBizServiceImpl.statistics()`

## 【复现步骤】
1. 执行归档统计任务：
```bash
curl -X GET "http://127.0.0.1:8087/hcbTimer-bill/test/etcCardBalanceFlowStatistics"
```

2. 查询某个用户的统计记录并记录updateTime：
```javascript
mongo 192.168.1.60:27017/jdbdb --eval "
var user = db.hcb_etc_card_balance_flow_statistics.findOne({statisticsDate: '2025-12'});
print('用户: ' + user.truckUserWalletId);
print('updateTime: ' + user.updateTime);
"
```

3. 等待5秒后再次执行归档任务：
```bash
curl -X GET "http://127.0.0.1:8087/hcbTimer-bill/test/etcCardBalanceFlowStatistics"
```

4. 再次查询同一用户的updateTime：
```javascript
mongo 192.168.1.60:27017/jdbdb --eval "
var user = db.hcb_etc_card_balance_flow_statistics.findOne({truckUserWalletId: '用户钱包ID'});
print('updateTime: ' + user.updateTime);
"
```

5. 观察结果：updateTime发生了变化，说明用户被重新统计，"跳过本月已统计用户"的逻辑未生效

## 【日志记录&截图标注】
**问题代码（第82行）：**
```java
// EtcCardBalanceFlowBizServiceImpl.statistics() 方法
public void statistics() {
    // ... 省略前面代码 ...
    
    // 第82行：删除统计表所有记录
    WriteResult remove = this.mongoTemplate.remove(new Query(), tableName);
    logger.info("清空统计表，删除记录数：{}", remove.getN());
    
    // ... 后续统计逻辑 ...
    
    // 第143-144行："跳过本月已统计用户"的逻辑（但因为第82行已删除所有记录，此逻辑永远不会生效）
    if (flowStatistics != null && currentMonth.equals(flowStatistics.getStatisticsDate())) {
        continue; // 本月已统计，跳过
    }
    
    // ... 继续统计 ...
}
```

**逻辑矛盾：**
- 第82行删除了所有统计记录
- 第143-144行判断"如果本月已统计则跳过"
- 但因为第82行已经删除了所有记录，所以`flowStatistics`永远为null，跳过逻辑永远不会执行

**测试验证结果：**
```bash
# 第1次执行归档
mongo 192.168.1.60:27017/jdbdb --eval "
db.hcb_etc_card_balance_flow_statistics.findOne({truckUserWalletId: 'a71df799b41b44e9a859d1c86ae192fa'}, {updateTime: 1})
"
# 结果：{ "updateTime" : "2025-12-04 10:30:15" }

# 第2次执行归档（5秒后）
mongo 192.168.1.60:27017/jdbdb --eval "
db.hcb_etc_card_balance_flow_statistics.findOne({truckUserWalletId: 'a71df799b41b44e9a859d1c86ae192fa'}, {updateTime: 1})
"
# 结果：{ "updateTime" : "2025-12-04 10:30:20" }
# updateTime变化了！说明重新统计了，未跳过
```

**性能影响：**
每次归档任务都要统计所有黑名单用户（假设1000个用户），而不是只统计新增或变化的用户，导致：
- 任务执行时间长
- 数据库压力大
- MongoDB查询频繁

## 【预期结果】
归档统计任务应该支持增量统计：
1. **不要每次都删除所有记录**，或者改为只删除非当月的历史统计数据：
```java
// 修改建议：只删除非当月的历史数据
String currentMonth = LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyy-MM"));
Query deleteQuery = new Query(Criteria.where("statisticsDate").ne(currentMonth));
WriteResult remove = this.mongoTemplate.remove(deleteQuery, tableName);
```

2. **正确执行"跳过本月已统计用户"的逻辑**：
- 如果用户的`statisticsDate`已经是当月，且`updateTime`在合理时间范围内（如24小时内），则跳过该用户
- 只统计新增的黑名单用户或本月尚未统计的用户

3. **性能提升**：
- 首次执行：统计所有黑名单用户（如1000个）
- 再次执行（同一天内）：跳过已统计用户（0个需要统计）
- 次日执行：只统计新增黑名单用户（如10个）

