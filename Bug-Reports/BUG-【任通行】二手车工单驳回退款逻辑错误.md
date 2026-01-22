# BUG-【任通行】二手车工单驳回退款逻辑错误

## 【BUG描述】
后台驳回二手车工单时，系统提示"退款失败,交易单号：【】，找不到原交易订单信息"，导致无法正常驳回工单。该问题影响所有标记为"无须支付"但实际有金额的二手车工单。

## 【前置条件】
- 测试环境：http://788360p9o5.yicp.fun
- 测试账号：后台管理员账号
- 测试数据：
  - 工单ID：2008367669245616130
  - 车牌号：苏ZKN720
  - 手机号：15818376788
  - 工单状态：待审核（status='2'）
  - 支付状态：无须支付（pay_status='0'）
  - 支付金额：0.02元（payment_amount=0.02）
  - 二手车订单状态：支付成功（status='1'）
  - 支付流水号：out_trade_no 为空

## 【复现步骤】
1. 后台创建二手车工单，选择"无需支付"
2. 客户H5提交工单，实际产生了0.02元支付金额
3. 工单进入待审核状态
4. 后台点击"驳回"，填写驳回原因"123"
5. 提交驳回请求

**请求示例**：
```bash
curl -X POST 'http://788360p9o5.yicp.fun/rtx-admin/customerWorkOrder/rejected' \
  -H 'Content-Type: application/json;charset=UTF-8' \
  -d '{"id":"2008367669245616130","rejectReason":"123"}'
```

## 【期望结果】
- 驳回成功
- 如果有支付金额，系统自动退款
- 工单状态变更为"已驳回"（status='4'）
- 二手车订单状态变更为"已退款"（status='3'）

## 【实际结果】
```json
{
  "code": 500,
  "msg": "退款失败,交易单号：【】，找不到原交易订单信息",
  "data": null,
  "success": false
}
```

驳回失败，工单状态未变更。

## 【附件】

### 数据库状态
**rtx_customer_work_order 表**：
```sql
id: 2008367669245616130
phone: 15818376788
plate_num: 苏ZKN720
status: 2 (待审核)
pay_status: 0 (无须支付)
payment_amount: 0.02
pay_order_no: (空)
```

**rtx_customer_used_car_order 表**：
```sql
id: 2008367669337890817
customer_work_order_id: 2008367669245616130
status: 1 (支付成功)
payment_amount: 0.02
out_trade_no: (空)
```

### 错误日志
```
退款失败,交易单号：【】，找不到原交易订单信息
```

## 【原因定位】

### 问题1：数据状态矛盾
- `rtx_customer_work_order.pay_status='0'`（无须支付）
- 但 `payment_amount=0.02`（有金额）
- 这是数据状态不一致导致的

### 问题2：退款逻辑判断错误
代码位置：`CustomerWorkOrderServiceImpl.java:448-471`

```java
if (customerWorkOrder.getPaymentAmount().compareTo(BigDecimal.ZERO) > 0) {
    // 查询支付成功的订单
    CustomerUsedCarOrder find = new CustomerUsedCarOrder();
    find.setCustomerWorkOrderId(customerWorkOrder.getId());
    find.setStatus(CustomerUsedCarOrderEnum.Status.PAYMENT_SUCCESS.getCode());
    List<CustomerUsedCarOrder> customerUsedCarOrderList = iCustomerUsedCarOrderService.selectCustomerUsedCarOrderList(find);
    
    if (CollectionUtils.isEmpty(customerUsedCarOrderList)) {
        resultMap.put("status", "false");
        resultMap.put("msg", "未找到支付成功的支付流水号,驳回失败!");
        return resultMap;
    }
    
    CustomerUsedCarOrder customerUsedCarOrder = customerUsedCarOrderList.get(0);
    OrderRefundDTO refundDto = new OrderRefundDTO();
    refundDto.setTradeOrderNo(customerUsedCarOrder.getOutTradeNo()); // ← BUG: out_trade_no 为空
    // ... 调用退款接口
}
```

**问题分析**：
1. 代码仅判断 `payment_amount > 0` 就进入退款逻辑
2. 没有判断 `pay_status` 是否真的需要退款
3. 没有判断 `out_trade_no` 是否为空
4. 当 `out_trade_no` 为空时，调用退款接口必然失败

### 问题3：支付流水号缺失
- `rtx_customer_used_car_order.out_trade_no` 为空
- 但 `status='1'`（支付成功）
- 这表明支付流程可能有问题，或者是测试数据异常

## 【修复建议】

### 方案1：修改驳回逻辑（推荐）
```java
// 判断是否需要退款
if (customerWorkOrder.getPaymentAmount().compareTo(BigDecimal.ZERO) > 0 
    && !CustomerWorkOrderEnum.PayStatus.NOT_NEED.getCode().equals(customerWorkOrder.getPayStatus())) {
    
    CustomerUsedCarOrder find = new CustomerUsedCarOrder();
    find.setCustomerWorkOrderId(customerWorkOrder.getId());
    find.setStatus(CustomerUsedCarOrderEnum.Status.PAYMENT_SUCCESS.getCode());
    List<CustomerUsedCarOrder> customerUsedCarOrderList = iCustomerUsedCarOrderService.selectCustomerUsedCarOrderList(find);
    
    if (CollectionUtils.isEmpty(customerUsedCarOrderList)) {
        resultMap.put("status", "false");
        resultMap.put("msg", "未找到支付成功的支付流水号,驳回失败!");
        return resultMap;
    }
    
    CustomerUsedCarOrder customerUsedCarOrder = customerUsedCarOrderList.get(0);
    
    // 检查支付流水号是否存在
    if (StringUtils.isBlank(customerUsedCarOrder.getOutTradeNo())) {
        resultMap.put("status", "false");
        resultMap.put("msg", "支付流水号为空，无法退款！请联系技术人员处理。");
        return resultMap;
    }
    
    OrderRefundDTO refundDto = new OrderRefundDTO();
    refundDto.setTradeOrderNo(customerUsedCarOrder.getOutTradeNo());
    // ... 退款逻辑
}
```

### 方案2：修复数据一致性
对于 `pay_status='0'` 但 `payment_amount > 0` 的数据，需要：
1. 排查为什么会出现这种状态
2. 修复创建工单时的逻辑，确保 `pay_status` 和 `payment_amount` 一致
3. 对历史数据进行修正

### 方案3：增强数据校验
在工单创建和更新时，增加校验：
```java
// pay_status='0' 时，payment_amount 必须为 0
if (CustomerWorkOrderEnum.PayStatus.NOT_NEED.getCode().equals(payStatus)) {
    RtxUtils.check(paymentAmount.compareTo(BigDecimal.ZERO) > 0, 
        "无须支付的工单，支付金额必须为0！");
}

// payment_amount > 0 时，pay_status 不能为 '0'
if (paymentAmount.compareTo(BigDecimal.ZERO) > 0) {
    RtxUtils.check(CustomerWorkOrderEnum.PayStatus.NOT_NEED.getCode().equals(payStatus), 
        "有支付金额的工单，支付状态不能为无须支付！");
}
```

## 【影响范围】
- 所有标记为"无须支付"但实际产生金额的二手车工单
- 所有 `out_trade_no` 为空但状态为"支付成功"的订单
- 影响后台驳回功能，可能导致工单无法正常驳回

