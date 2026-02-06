# BUG-【任货行】冻结记录表freeze_type字段值错误

## 1. BUG描述

冻结记录查询页面显示所有记录的"冻结类型"都是"冻结"，即使操作原因是"清欠解冻"或"清欠待冻结抹平"（解冻操作），也显示为"冻结"。

**影响**：
- ❌ 无法区分冻结和解冻操作，页面显示混乱
- ❌ 冻结金额合计统计可能不准确
- ✅ 数据库记录的 `reason` 字段正确（如"清欠解冻"）

---

## 2. 复现步骤

### 前置条件
- 测试环境：http://788360p9o5.yicp.fun/hcbadmin
- 测试车牌：蒙ZMM777
- 测试操作：清欠还款支付（会触发解冻操作）

### 操作步骤

**Step 1**: 访问冻结记录查询页面

```bash
curl -X POST 'http://788360p9o5.yicp.fun/hcbadmin/freeze/list.do' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -H 'Cookie: JSESSIONID=42bad585-d202-471e-bfa6-096914bc258f' \
  --data-urlencode 'car_number=蒙ZMM777'
```

**Step 2**: 查询数据库 `hcb_freeze_record` 表

```bash
mysql -h192.168.1.60 -uroot -pfm123456 hcb -e "
SELECT 
  freeze_time, 
  car_number, 
  freeze_amount, 
  freeze_type, 
  reason 
FROM hcb_freeze_record 
WHERE car_number = '蒙ZMM777' 
ORDER BY freeze_time DESC 
LIMIT 10"
```

**数据查询结果**：
```
freeze_time         | car_number | freeze_amount | freeze_type | reason
--------------------|------------|---------------|-------------|-------------------------
2026-01-27 16:35:18 | 蒙ZMM777   | 888.00        | frozen      | 蒙ZMM777-清欠解冻
2026-01-27 16:35:18 | 蒙ZMM777   | 110.99        | frozen      | 蒙ZMM777-清欠待冻结抹平
2026-01-27 16:23:37 | 蒙ZMM777   | 0.00          | frozen      | 蒙ZMM777-清欠解冻
2026-01-27 16:23:37 | 蒙ZMM777   | 0.00          | frozen      | 蒙ZMM777-清欠待冻结抹平
2026-01-24 16:53:43 | 蒙ZMM777   | 0.00          | frozen      | 蒙ZMM777-清欠解冻
2026-01-24 16:53:43 | 蒙ZMM777   | 0.00          | frozen      | 蒙ZMM777-清欠待冻结抹平
```

**Step 3**: 观察页面显示和数据库字段

- 页面"冻结类型"列全部显示："冻结"
- 数据库 `freeze_type` 字段全部为：`frozen`

---

## 3. 预期结果

**预期**：
- ✅ 操作原因是"清欠解冻"的记录，`freeze_type` 应该是 `thaw`（解冻）
- ✅ 操作原因是"清欠待冻结抹平"的记录，`freeze_type` 应该是 `thaw`（解冻）
- ✅ 页面"冻结类型"列应该显示"解冻"
- ✅ 冻结金额合计应该区分冻结和解冻

**预期数据**：
```
freeze_time         | freeze_amount | freeze_type | reason
--------------------|---------------|-------------|-------------------------
2026-01-27 16:35:18 | 888.00        | thaw        | 蒙ZMM777-清欠解冻
2026-01-27 16:35:18 | 110.99        | thaw        | 蒙ZMM777-清欠待冻结抹平
```

---

## 4. 实际结果

**实际**：
- ❌ 所有记录的 `freeze_type` 字段都是 `frozen`（冻结）
- ❌ 即使操作原因是"清欠解冻"，`freeze_type` 也记录为 `frozen`
- ❌ 页面无法区分冻结和解冻操作
- ❌ 冻结金额合计统计将解冻金额也计入冻结总额

**实际数据**：
```
freeze_time         | freeze_amount | freeze_type | reason
--------------------|---------------|-------------|-------------------------
2026-01-27 16:35:18 | 888.00        | frozen ❌   | 蒙ZMM777-清欠解冻
2026-01-27 16:35:18 | 110.99        | frozen ❌   | 蒙ZMM777-清欠待冻结抹平
2026-01-27 16:23:37 | 0.00          | frozen ❌   | 蒙ZMM777-清欠解冻
2026-01-27 16:23:37 | 0.00          | frozen ❌   | 蒙ZMM777-清欠待冻结抹平
```

---

**数据库**：`hcb.hcb_freeze_record`  
**查询页面**：http://788360p9o5.yicp.fun/hcbadmin/freeze/list.do
