# BUG-【先享月费】人工退款未自动打客诉标签

## 【问题描述】

在"先享月费账单人工退款"功能中，客服执行退款操作后，系统**没有自动给用户打上"先享月费客诉用户"标签**（`MonthlyPayLaterComplaint_Wallet`），导致：
1. 客服需要手动打标签，容易遗漏
2. 下次扣款时无法自动识别客诉用户
3. 可能导致重复投诉和纠纷

**严重等级**：⚠️ **P2级 - 功能缺失**

## 【请求参数】

- **退款接口**：`POST /hcbadmin/monthlyPaylaterBill/refundBill.do`
- **请求参数**：`id=9301`（账单ID）
- **测试用户**：骆志敏（身份证：441622199602075195，钱包ID：7b4bfaa4a8de11f084f1141877512abb）
- **测试时间**：2025年11月18日 10:19

## 【复现步骤】

### 1. 准备测试数据

创建一笔已扣款成功的账单：
```sql
INSERT INTO hcb_monthly_paylater_wallet_bill 
(id, wallet_id, user_id, id_code, user_name, car_num, 
 bill_amount, actual_amount, bill_status, bill_date, operator_code, 
 pay_success_time, pay_flow_id, create_by, update_by)
VALUES 
(9301, '7b4bfaa4a8de11f084f1141877512abb', '145789578cb24dfea109280d6126991f', 
 '441622199602075195', '骆志敏', '黑A88888', 
 150.00, 150.00, 1, 202411, 'LTK', 
 NOW(), '1990364720896815106', 'system', 'system');
```

确认用户当前无客诉标签：
```sql
SELECT * FROM hcb_label_config_manage_detail 
WHERE ref_id = '7b4bfaa4a8de11f084f1141877512abb'
AND label_id = '1900000000000000001'  -- 客诉标签ID
AND is_delete = 0;
-- 结果：无记录
```

### 2. 执行退款操作

在管理后台执行退款：
1. 登录管理后台
2. 进入"先享月费账单管理"
3. 找到账单ID=9301
4. 点击"退款"按钮
5. 确认退款

或使用接口：
```bash
curl -X POST 'https://788360p9o5.yicp.fun/hcbadmin/monthlyPaylaterBill/refundBill.do' \
  -H 'Cookie: JSESSIONID=8856df23-cb80-43ed-9389-c552a011b1d5' \
  --data-urlencode 'id=9301'
```

### 3. 验证结果

**退款结果**：
```sql
SELECT id, bill_amount, refund_amount, bill_status, refund_time, refund_user
FROM hcb_monthly_paylater_wallet_bill 
WHERE id = 9301;
```
结果：
- 账单状态：已退款(3) ✅
- 退款金额：150.00元 ✅
- 退款时间：2025-11-18 10:19:27 ✅

**客诉标签检查**：
```sql
SELECT d.id, m.label_name, d.create_user, d.create_date
FROM hcb_label_config_manage_detail d
LEFT JOIN hcb_label_config_manage m ON d.label_id = m.id
WHERE d.ref_id = '7b4bfaa4a8de11f084f1141877512abb'
AND m.label_key = 'MonthlyPayLaterComplaint_Wallet'
AND d.is_delete = 0;
```
结果：
- **无记录** ❌（未自动打标签）

## 【日志记录&截图标注】

### 退款成功的数据

| 字段 | 值 |
|------|-----|
| 账单ID | 9301 |
| 账单金额 | 150.00元 |
| 退款金额 | 150.00元 |
| 账单状态 | 已退款(3) |
| 退款时间 | 2025-11-18 10:19:27 |
| 退款人 | test |

### 客诉标签状态

查询结果：**无记录**（未打标签）

### 业务逻辑矛盾

系统在**扣款时会检查客诉标签**：
- 代码位置：`MonthlyPaylaterWalletBillBizServiceImpl.java` 第189-194行
- 逻辑：如果用户有客诉标签，则**不允许扣款**

```java
// 判断用户是否为客诉用户
boolean isComplaintUser = walletLabelList.stream()
    .anyMatch(label -> LabelKeyEnum.MONTHLY_PAY_LATER_COMPLAINT_WALLET.getCode()
                      .equals(label.getString("labelKey")));
if (isComplaintUser) {
    return "用户为客诉用户，不允扣款";  // ⭐ 扣款时会检查
}
```

但**退款时不打标签**：
- 代码位置：`MonthlyPaylaterBillBizService.java` 第45-77行
- 问题：退款成功后，只更新账单状态，**没有打标签的逻辑**

## 【预期结果】

### 退款成功后应该自动打上客诉标签

**完整业务流程**：
```
用户投诉 → 客服退款 → 自动打客诉标签 → 下次扣款时检查标签 → 拒绝扣款 → 人工处理
```

**预期数据**：
```sql
-- 退款后，应该在标签详情表中生成记录
SELECT * FROM hcb_label_config_manage_detail 
WHERE ref_id = '7b4bfaa4a8de11f084f1141877512abb'
AND label_id = '1900000000000000001'
AND is_delete = 0;
```

**预期结果**：
| 字段 | 预期值 |
|------|--------|
| label_id | 1900000000000000001 |
| ref_id | 7b4bfaa4a8de11f084f1141877512abb |
| label_type | 1（钱包） |
| is_delete | 0（未删除） |
| identity_card | 441622199602075195 |
| create_user | test（退款操作人） |
| create_date | 2025-11-18 10:19:27（退款时间） |

**业务价值**：
1. ✅ 自动标记客诉用户，无需手动操作
2. ✅ 下次扣款时自动识别并拒绝，避免重复投诉
3. ✅ 便于客服管理和跟踪客诉用户

---

**BUG报告时间**：2025-11-18 10:30  
**报告人**：测试团队  
**严重等级**：⚠️ P2级 - 功能缺失

