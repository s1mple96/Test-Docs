# BUG-【任通行】出纳退款明细列表和导出缺少通行费多扣退款申请类型

## 1. BUG描述

出纳退款明细菜单的列表查询和导出功能，在不筛选业务类型时，缺少"通行费多扣退款申请"(biz_type=9)的记录。

**影响**：
- ✅ 手动筛选"通行费多扣退款申请"后查询和导出正常，数据完整
- ❌ 不筛选业务类型时，列表查询缺少通行费多扣退款申请的记录
- ❌ 不筛选业务类型时，导出全部缺少通行费多扣退款申请的记录
- ❌ 导致用户无法在默认列表中看到通行费多扣退款申请，数据不完整

---

## 2. 复现步骤

### 前置条件
- 测试环境：http://192.168.1.60:8089/report-admin
- 测试账号：有出纳退款明细导出权限的账号
- 测试数据：数据库中存在已完成的通行费多扣退款申请记录(biz_type=9, status=2)

### 操作步骤

**Step 1**: 验证数据库中是否有通行费多扣退款记录

```bash
mysql -h localhost -u root -pfm123456 rtx -e "SELECT COUNT(*) AS total FROM rtx_bpm_process_management WHERE biz_type = 9 AND status = 2;"
```

**数据查询结果**：
```
total
4
```
说明数据库中有4条已完成的通行费多扣退款申请记录。

**Step 2**: 验证列表查询 - 手动筛选"通行费多扣退款申请"

```bash
curl -X GET 'http://788360p9o5.yicp.fun/rtx-admin/bpmProcess/cashierRefundInfoList?pageNum=1&pageSize=10&bizType=9' \
-H 'Authorization: Bearer xxx'
```

**Step 3**: 验证列表查询 - 不筛选业务类型

```bash
curl -X GET 'http://788360p9o5.yicp.fun/rtx-admin/bpmProcess/cashierRefundInfoList?pageNum=1&pageSize=10&bizType=' \
-H 'Authorization: Bearer xxx'
```

**Step 4**: 对比列表查询结果

1. 查看Step 2的返回数据是否包含biz_type=9的记录
2. 查看Step 3的返回数据是否包含biz_type=9的记录

**Step 5**: 验证导出 - 手动筛选"通行费多扣退款申请"

1. 登录任通行后台管理系统
2. 进入"流程审核管理 → 出纳退款明细"菜单
3. 在"业务类型"下拉框中选择"通行费多扣退款申请"
4. 点击"导出"按钮
5. 下载并打开导出的Excel文件

**Step 6**: 验证导出 - 不筛选业务类型

1. 进入"流程审核管理 → 出纳退款明细"菜单
2. 不选择任何筛选条件（或清空所有筛选）
3. 点击"导出"按钮
4. 下载并打开导出的Excel文件

**Step 7**: 对比导出结果

1. 统计手动筛选导出的记录数
2. 统计导出全部的记录数
3. 查看导出全部的Excel中是否包含"通行费多扣退款申请"类型的数据

---

## 3. 预期结果

**预期**：
- ✅ 列表查询不筛选业务类型时，应返回所有业务类型的记录（包括biz_type=9）
- ✅ 导出全部时，应包含所有业务类型的已完成记录
- ✅ Excel中应包含"通行费多扣退款申请"(biz_type=9)的4条记录
- ✅ 列表和导出的记录数应一致，包含所有业务类型

---

## 4. 实际结果

**实际**：
- ❌ 列表查询不筛选业务类型时，返回的数据中不包含biz_type=9的记录
- ❌ 导出全部时，Excel中不包含"通行费多扣退款申请"(biz_type=9)的记录
- ❌ 列表和导出的记录数都少于实际应有的记录数
- ✅ 手动筛选"通行费多扣退款申请"(bizType=9)后，列表查询和导出都正常

**数据对比**：
- 数据库中biz_type=9且status=2的记录数：4条
- 手动筛选bizType=9的查询结果：包含4条（正常）
- 不筛选业务类型的查询结果：不包含biz_type=9（异常）
- 手动筛选导出的记录数：4条（正常）
- 导出全部包含的biz_type=9记录数：0条（异常）

---

**涉及接口**：
1. 列表查询: `/rtx-admin/bpmProcess/cashierRefundInfoList`
2. 导出: `/report-admin/bpmProcess/cashierRefundInfoExport`

**复现请求**：
```bash
# 列表查询 - 手动筛选（正常）
GET /rtx-admin/bpmProcess/cashierRefundInfoList?pageNum=1&pageSize=10&bizType=9

# 列表查询 - 不筛选（异常）
GET /rtx-admin/bpmProcess/cashierRefundInfoList?pageNum=1&pageSize=10&bizType=

# 导出 - 手动筛选（正常）
GET /report-admin/bpmProcess/cashierRefundInfoExport?bizType=9

# 导出 - 不筛选（异常）
GET /report-admin/bpmProcess/cashierRefundInfoExport
```
