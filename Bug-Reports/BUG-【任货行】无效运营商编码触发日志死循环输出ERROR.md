# BUG-【任货行】无效运营商编码触发日志死循环输出ERROR

## 【问题描述】

**问题现象**：
当调用账单扩展表数据迁移接口时，传入无效的运营商编码（如不存在的运营商表），系统会持续输出ERROR日志，可能导致日志文件快速增长，最终撑爆磁盘导致服务器瘫痪。

**实际结果**：
- 抛出 `IllegalStateException: no table route info` 异常
- ERROR日志可能重复输出
- 没有任务停止机制
- 日志文件持续增长

**预期结果**：
- 参数校验在入口处完成
- 无效参数直接返回错误提示
- 只输出1次ERROR日志
- 不进入业务逻辑

## 【请求参数】

**接口地址**：
```
http://localhost:8087/hcbTimer-bill/test/testRemoveDeductRecordExtend?operatorCode=INVALID
```

**请求方法**：GET

**参数说明**：
- `operatorCode`: 运营商编码（传入不存在的值，如 INVALID、ABC、123 等）

**有效运营商编码**（正常情况）：
- CTK, HTK, JTK, LTK, MTK, TXB, XTK

**无效运营商编码**（触发BUG）：
- INVALID, ABC, TEST, 任意不存在的值

## 【复现步骤】

1. 连接测试服务器，打开日志实时监控
```bash
tail -f /home/tomcat/tomcat-hcbTimer-bill/logs/catalina.out
```

2. 调用接口，传入无效运营商编码
```bash
curl -s "http://localhost:8087/hcbTimer-bill/test/testRemoveDeductRecordExtend?operatorCode=INVALID"
```

3. 观察日志输出：
   - 查看是否有 `IllegalStateException: no table route info`
   - 观察ERROR是否重复输出
   - 检查日志文件大小是否快速增长

4. 检查日志文件大小变化
```bash
# 执行前
ls -lh /home/tomcat/tomcat-hcbTimer-bill/logs/catalina.out

# 等待30秒

# 执行后
ls -lh /home/tomcat/tomcat-hcbTimer-bill/logs/catalina.out
```

5. 统计ERROR重复次数
```bash
tail -n 200 /home/tomcat/tomcat-hcbTimer-bill/logs/catalina.out | grep -c "IllegalStateException"
```

## 【日志记录&截图标注】

**错误日志**：
```
Caused by: org.apache.ibatis.exceptions.PersistenceException: 
### Error querying database.  Cause: java.lang.IllegalStateException: no table route info
### The error may exist in file [/home/tomcat/tomcat-hcbTimer-bill/webapps/hcbTimer-bill/WEB-INF/classes/mybatis/truck/DeductRecordExtendMapper.xml]
### The error may involve defaultParameterMap
### The error occurred while setting parameters
### SQL: select id, create_time, update_time, bill_date, deduct_bill_no, deduct_bill_id, operator_code, car_num, truck_user_id, bill_amount,
                id_code, wallet_id, last_deduct_time, deduct_bank_card_times, total_bank_card_number, service_rate, fee_rate_rule_ids,
                fee_rate_rule_names, pay_way, pay_status, pay_success_time, bank_channel_code, pay_order_no, wallet_flow_id, bank_msg,
                pay_type, master_merchant_no, sub_merchant_no, pay_amount
         from hcb_deduct_record_extend
         where create_time < ?
         and operator_code = ?
         and id > ?
         and pay_status = '1'
         ORDER BY id asc
         LIMIT 10
### Cause: java.lang.IllegalStateException: no table route info
```

**危险等级**：
- ⚠️ 高危：如果ERROR重复输出，可能导致日志文件暴增
- ⚠️ 磁盘写满可能导致服务器瘫痪
- ⚠️ 影响其他服务的日志输出

**验证命令**：
```bash
# 检查日志是否有死循环（相同ERROR重复出现）
tail -n 100 /home/tomcat/tomcat-hcbTimer-bill/logs/catalina.out | sort | uniq -c | sort -rn | head -n 10

# 查看日志增长速度
watch -n 1 'ls -lh /home/tomcat/tomcat-hcbTimer-bill/logs/catalina.out'
```

## 【预期结果】

1. **参数校验**：
   - 在方法入口处校验 operatorCode 是否有效
   - 无效则直接返回错误提示，不进入业务逻辑

2. **异常处理**：
   - 只输出1次ERROR日志
   - 返回友好的错误信息：`{"success": false, "message": "无效的运营商编码: INVALID，支持的运营商: CTK,HTK,JTK,LTK,MTK,TXB,XTK"}`
   - 任务立即停止，不重试

3. **日志输出**：
   - 错误日志格式：`ERROR - 无效的运营商编码: INVALID`
   - 不输出完整堆栈信息（或只输出1次）
   - 日志文件大小增长 <1KB

4. **建议修复**：
```java
// 伪代码示例
private static final Set<String> VALID_OPERATORS = 
    new HashSet<>(Arrays.asList("CTK", "HTK", "JTK", "LTK", "MTK", "TXB", "XTK"));

public void testRemoveDeductRecordExtend(String operatorCode) {
    // 参数校验
    if (StringUtils.isBlank(operatorCode) || !VALID_OPERATORS.contains(operatorCode)) {
        log.error("无效的运营商编码: {}, 支持的运营商: {}", operatorCode, VALID_OPERATORS);
        return "无效的运营商编码";
    }
    
    // 业务逻辑...
}
```

