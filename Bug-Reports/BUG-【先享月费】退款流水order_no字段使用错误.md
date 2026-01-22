# BUG报告：【先享月费】退款流水order_no字段使用错误

---

## 【问题描述】

在先享月费账单退款功能中，生成的月费钱包流水记录中，`source_id`和`order_no`字段使用了相同的值（账单ID），导致订单号失去唯一性。

**对比扣款流水**：扣款时`order_no`是UUID（正确），但退款时`order_no`是账单ID（错误）。

---

## 【请求参数】

**接口**：`/monthlyPaylaterBill/refundBill`

**请求参数**：
```json
{
  "id": 9101  // 账单ID
}
```

---

## 【复现步骤】

1. 创建一笔先享月费账单并扣款成功（账单ID=9101，金额80元）

2. 在管理后台对该账单进行人工退款操作

3. 查询月费钱包流水表：
   ```sql
   SELECT 
       source_id as 来源ID,
       order_no as 订单编号,
       trade_amount as 金额,
       remark as 备注
   FROM hcb_monthly_paylater_wallet_flow 
   WHERE user_id = '145789578cb24dfea109280d6126991f'
   ORDER BY create_time DESC;
   ```

4. 观察退款流水的`source_id`和`order_no`字段

---

## 【日志记录&截图标注】

### **实际结果（有问题）**

**退款流水**：
```
来源ID    订单编号    金额      备注
9101      9101       80.00    test先享月费-人工退款  
❌ source_id和order_no相同，都是账单ID
```

**对比扣款流水**（正常）：
```
来源ID    订单编号                              金额      备注
9101      e6fc0e9a62124a7a9e908a5a14b42639    80.00    admin先享月费扣款
✅ source_id是账单ID，order_no是UUID
```

### **代码位置**

**文件**：`hcbadmin/src/main/java/com/hcb/bizService/monthly/MonthlyPaylaterBillBizService.java`  
**方法**：`monthlyWalletBillRefund`  
**行号**：第54-56行

**问题代码**：
```java
Tuple3<Boolean, String, Long> refundResult = monthlyWalletOperatorBizService.monthlyWalletTrade(
    refundBill.getWalletId(), 
    MonthlyFlowTypeEnum.MONTHLY_BILL_MANUAL_REFUND,
    refundBill.getActualAmount(), 
    String.valueOf(refundBill.getId()),    // source_id = 9101
    String.valueOf(refundBill.getId()),    // ❌ order_no = 9101 (错误！应该是UUID)
    refundBill.getUserId(), 
    Jurisdiction.getUsername()
);
```

**对比扣款代码**（正确）：
```java
// 文件：hcbtimer/.../MonthlyPaylaterWalletBillBizServiceImpl.java
// 方法：processBillPayment，行号：236-244
Tuple3<Boolean, String, Long> refundResp = monthlyWalletOperatorBizService.monthlyWalletTrade(
    bill.getWalletId(),
    MonthlyFlowTypeEnum.MONTHLY_BILL_PAY,
    actualAmount,
    bill.getId().toString(),    // source_id = 9101
    UuidUtil.get32UUID(),       // ✅ order_no = UUID (正确！)
    bill.getUserId(),
    Constants.ADMIN
);
```

---

## 【预期结果】

### **退款流水应该是**：
```
来源ID    订单编号                              金额      备注
9101      a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6    80.00    test先享月费-人工退款
✅ source_id是账单ID，order_no是唯一UUID
```

### **修复建议**：

将第56行的：
```java
String.valueOf(refundBill.getId())    // ❌ 错误
```

改为：
```java
UuidUtil.get32UUID()                  // ✅ 正确
```

---

**影响**：
- 无法通过`order_no`唯一标识退款交易
- 影响流水追踪和对账
- 不符合设计规范

**优先级**：P2（中等）
