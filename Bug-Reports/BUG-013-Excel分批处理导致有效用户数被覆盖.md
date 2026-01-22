# BUG-013：Excel分批处理导致有效用户数被覆盖

## 【问题描述】
在大批量数据导入时（特别是超过3000条记录），Excel文件校验接口`validateUserFile.do`返回的有效用户数量始终为0或最后一批的数量，而不是所有批次的累计总数，导致大批量数据导入全部失败。

## 【请求参数】
**涉及接口：**
- 文件校验接口：`/hcbadmin/channelResignTask/validateUserFile.do`
- 数据保存接口：`/hcbadmin/channelResignTask/save.do`

**关键参数：**
- `file`: Excel文件，包含大量手机号数据
- `agreementId`: 协议ID
- `versionId`: 协议版本ID
- `userType`: "import"（导入用户模式）

**测试数据规模：**
- 小文件测试：11条数据 → ✅ 返回正确的有效用户数
- 中等文件测试：5000条数据（包含11条有效） → ❌ 返回0
- 大文件测试：50000条数据（包含100条有效） → ❌ 返回0

## 【复现步骤】
1. **准备测试数据**：
   - 创建包含5000条以上手机号的Excel文件
   - 确保其中包含部分有效用户手机号（状态为"1"的正常用户）
   
2. **执行文件校验**：
   ```bash
   curl -X POST 'http://788360p9o5.yicp.fun/hcbadmin/channelResignTask/validateUserFile.do' \
   -H 'Cookie: JSESSIONID=7a3e71e9-f5c7-4b54-be48-8482c735af15' \
   -F 'file=@"大批量测试数据.xlsx"'
   ```

3. **观察返回结果**：
   ```json
   {
     "data": 0,           // ❌ 实际应该 > 0
     "success": true,
     "message": "校验成功"
   }
   ```

4. **对比小批量测试**：
   - 使用相同的有效手机号单独测试
   - 小批量时能正确识别有效用户数

## 【日志记录&截图标注】

**代码执行流程分析：**
```java
// 1. Excel读取使用EasyExcel分批处理机制
ExcelUtil.readExcel(inputStream, ImportChannelPhoneDTO.class, lambda)
    .sheet().headRowNumber(1).doRead();

// 2. EasyExeclConsumerListeners分批处理逻辑
if (tmpContent.size() >= 3000) {  // 🔍 关键：每3000行触发一次
    consumer.accept(lineNo, excelDto);
    excelDto.setTmpContent(new ArrayList<>());  // 清空当前批次
}

// 3. lambda$validateUserFile$1 处理每批数据
private void lambda$validateUserFile$1(AtomicInteger counter, ...) {
    List<String> phoneList = excelDto.getTmpContent().stream()
        .map(data -> data.getPhone())
        .distinct()
        .collect(toList());
    
    int validUserCount = truckChannelService.countTruckChannelByPhones(phoneList);
    
    // ❌ 问题核心：直接覆盖而不是累加
    counter.set(validUserCount);  // 每次都覆盖之前的结果！
}
```

**问题执行时序：**
```
50000条数据处理过程：
批次1 (1-3000行):    包含20个有效用户 → counter.set(20)
批次2 (3001-6000行): 包含15个有效用户 → counter.set(15)  ❌ 覆盖了20
批次3 (6001-9000行): 包含0个有效用户  → counter.set(0)   ❌ 覆盖了15
...
批次17(48001-50000): 包含0个有效用户  → counter.set(0)   ❌ 最终结果

最终返回：data = 0 （实际应该是所有批次的累计：20+15+...）
```

**数据库验证：**
```sql
-- 验证测试数据中确实存在有效用户
SELECT COUNT(*) as valid_users 
FROM hcb_truckchannel 
WHERE PHONE IN ('测试文件中的手机号列表')
  AND STATUS IN ('0', '1', '2');  -- 非注销状态
-- 结果：确实有有效用户，但接口返回0
```

**边界测试验证：**
- 2999条数据：✅ 正确返回有效用户数（单批处理）
- 3000条数据：❌ 可能返回0（触发分批处理边界）
- 3001条数据：❌ 确定返回0（分成2批，后批覆盖前批）

## 【预期结果】
1. **累加逻辑**：每批处理的有效用户数应该累加，而不是覆盖
2. **数据一致性**：无论数据量大小，有效用户统计都应该准确
3. **分批透明**：分批处理机制对业务逻辑应该是透明的
4. **性能稳定**：大批量数据处理应该能够正常完成

**具体期望：**
- 50000条数据中的100个有效用户应该被正确识别
- `validateUserFile.do` 应该返回 `{"data": 100, "success": true}`
- 后续的数据保存流程应该能够正常执行
- 系统应该支持任意规模的数据导入

**正确的代码逻辑应该是：**
```java
// ✅ 修复方案：使用累加而不是覆盖
counter.addAndGet(validUserCount);  // 累加每批的结果

// 或者使用同步累加确保线程安全
synchronized(counter) {
    counter.set(counter.get() + validUserCount);
}
```

**建议的分批处理优化：**
- 考虑调整分批大小（当前3000可能过小）
- 实现更智能的内存管理
- 添加分批处理的监控日志
- 提供分批进度反馈给用户

---

**技术分析：**
- **根本原因**：AtomicInteger.set()覆盖逻辑错误，应该使用addAndGet()累加
- **涉及组件**：EasyExcel分批读取、ExcelUtil、ChannelResignTaskServiceImpl
- **影响范围**：所有超过3000条记录的Excel文件导入功能

**影响评估：**
- 🔴 **严重级别**：导致大批量数据导入完全失败
- 📊 **性能影响**：用户无法进行规模化数据操作
- 💼 **业务影响**：影响渠道重签任务的批量处理能力
- 🐛 **修复难度**：低（单行代码修改）但影响重大
