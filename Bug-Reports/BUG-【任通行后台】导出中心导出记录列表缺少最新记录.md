# BUG-【任通行后台】导出中心导出记录列表缺少最新记录

## 1. BUG描述

任通行管理后台“数据导出中心”提交导出任务成功后，导出任务记录在数据库中已生成且导出成功，但通过导出中心记录列表接口查询不到最新记录，页面导出记录不显示最新导出任务。

**影响**：
- ✅ 导出任务可正常提交并生成 Excel 文件（任务成功、可下载）
- ❌ 导出中心“导出任务记录”列表无法展示最新导出记录，导致用户无法在导出中心页面查看/操作最新任务

---

## 2. 复现步骤

### 前置条件
- 测试环境: 192.168.1.60
- 系统: 任通行管理后台（rtx-admin）
- 数据库: rtx
- 账号: admin（使用 Bearer Token 登录）
- 测试数据:
  - 身份证号: 441622199602075195
  - 车牌号: 黑A113355
  - walletId: 1906909894016225282

### 操作步骤

**Step 1**：提交导出任务（导出中心）

```bash
curl -X POST 'http://788360p9o5.yicp.fun/rtx-admin/center/export/createTask' \
  -H 'Authorization: Bearer <token>' \
  -H 'Content-Type: application/json;charset=UTF-8' \
  -d '{"carType":"1","idCard":"441622199602075195","exportTypes":["1"],"carNums":["黑A113355"],"walletId":"1906909894016225282"}'
```

**接口返回**（示例）：
```
code=200
data=["2043588707646410754"]
```

**补充复现（同一功能再次提交导出）接口返回**（示例）：
```
code=200
data=["2043603772080549889"]
```

**Step 2**：查询导出中心记录列表（应能看到最新任务）

```bash
curl -X GET 'http://788360p9o5.yicp.fun/rtx-admin/center/export/exportList?pageNum=1&pageSize=10' \
  -H 'Authorization: Bearer <token>'
```

**Step 3**：查询数据库确认任务已生成

```sql
SELECT id, create_by, create_time, type, status, export_name, export_source
FROM rtx.rtx_etc_export_record
WHERE id = 2043588707646410754
LIMIT 1;
```

**数据查询结果**（客观描述）：
```
id=2043588707646410754
create_by=admin
create_time=2026-04-13 15:14:51
type=1001
status=2
export_name=客车用户通行账单-20260413151450.xlsx
export_source=0
```

**补充：同一功能再次提交导出后数据库记录（客观描述）**：
```sql
SELECT id, create_by, create_time, type, status, export_name, export_source
FROM rtx.rtx_etc_export_record
WHERE id = 2043603772080549889
LIMIT 1;
```

```
id=2043603772080549889
create_by=admin
create_time=2026-04-13 16:14:42
type=1001
status=2
export_name=客车用户通行账单-20260413161442.xlsx
export_source=1
```

---

## 3. 预期结果

- 提交导出任务成功后，通过导出中心记录列表接口 `GET /rtx-admin/center/export/exportList` 能查询到本次最新导出任务记录
- 页面导出任务记录列表应展示该任务，并可进行后续操作（下载/重发邮件等，按权限与状态显示）

---

## 4. 实际结果

- `POST /center/export/createTask` 返回成功并返回任务ID（如 `2043588707646410754`）
- 数据库 `rtx.rtx_etc_export_record` 中该任务记录存在且 status=2（成功），但 `export_source=0`
- 调用 `GET /center/export/exportList` 查询不到该最新任务记录，页面“导出任务记录”列表不显示最新导出

**补充日志（客观描述）**：
```
ProgressBigExcelWriterListener : 文件导出发送mq创建任务 ... exportSource":1 ... id:2043603772080549889 type:1001
FileExportListener : 进入文件导出消息,msg：... "exportSource":1,"id":2043603772080549889,"status":"1","type":"1001"
FileExportListener : 保存文件导出记录,exportRecord:ExportRecord(id=2043603772080549889, type=1001, status=1, ... exportSource=1 ...)
FileExportListener : 进入文件导出消息,msg：{"id":2043603772080549889,"progress":"...","status":"2"}  （消息体未携带 exportSource/type/exportName/exportUrl）
```

**相关数据（最近导出记录示例）**：
```sql
SELECT id, create_by, create_time, type, status, export_name, export_source
FROM rtx.rtx_etc_export_record
WHERE create_time >= '2026-04-13 15:00:00'
ORDER BY id DESC
LIMIT 20;
```

```
2043588707646410754 ... type=1001 status=2 export_source=0
2043587626631663617 ... type=1001 status=2 export_source=0
2043587524290646017 ... type=1001 status=2 export_source=0
2043587501180030978 ... type=1001 status=2 export_source=0
2043603772080549889 ... type=1001 status=2 export_source=1
```

**日志路径**: `/home/java-server/rtx-admin-server/log.txt`

