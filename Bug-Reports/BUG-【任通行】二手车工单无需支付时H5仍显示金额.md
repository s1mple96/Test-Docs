# BUG-【任通行】二手车工单无需支付时H5仍显示金额

## 【BUG描述】
后台创建二手车工单并勾选"无需支付"后，用户在H5页面仍然看到需要支付0.02元的提示，造成用户误解以为需要付款，但实际点击提交后并不拉起支付。这种不一致的体验会让用户困惑。

## 【前置条件】
- 测试环境：http://788360p9o5.yicp.fun
- 测试账号：后台管理员账号
- 测试数据：
  - 车牌号：苏ZKN720
  - 手机号：15818376788
  - 工单ID：2008429097042300930
  - 二手车订单ID：2008429097063272450
  - 工单支付状态：`pay_status='0'`(无须支付)
  - 车辆产品策略：`bid_is_equipment_fee_required='0'`(申办时未缴费)
  - 设备费金额：`equipment_amount=0.02`

## 【复现步骤】
1. 后台【客户工单管理】-【新增】
2. 选择业务类型"二手车买卖"
3. 填写手机号、车牌号：苏ZKN720
4. **勾选【是否支付注销费】为"无需支付"**
5. 上传相关图片，保存工单
6. 公众号查询工单，获取H5链接
7. 打开H5页面：http://788360p9o5.yicp.fun/rhtxUsedCarH5/#/home/uploadFile?fromType=rtx&usedCarOrderId=2008429097063272450
8. 观察页面显示金额

**接口调用**：
```bash
curl 'http://788360p9o5.yicp.fun/rtx-app/usedCarOrderApp/getCustomerOrderPayAmt?source=2&vehicleColor=0&plateNum=%E8%8B%8FZKN720'
```

**接口返回**：
```json
{
  "code": 200,
  "msg": "操作成功",
  "data": {
    "amt": 0.02
  },
  "success": true
}
```

## 【期望结果】
- 后台勾选"无需支付"后，H5页面应显示"无需支付"或金额为0
- 查询金额接口应返回 `{"amt": 0}`
- 用户看到的信息与实际支付行为一致

## 【实际结果】
- H5页面显示"需支付0.02元"
- 查询金额接口返回 `{"amt": 0.02}`
- 但点击提交后不拉起支付（因为 `pay_status='0'`）
- **用户体验差**：显示需要付款，但实际不用付，造成困惑

## 【附件】

### 数据库状态
**rtx_customer_work_order 表**：
```sql
id: 2008429097042300930
phone: 15818376788
plate_num: 苏ZKN720
status: 0 (待提交)
pay_status: 0 (无须支付)  ← 后台已设置无需支付
payment_amount: 0.00
```

**rtx_customer_used_car_order 表**：
```sql
id: 2008429097063272450
customer_work_order_id: 2008429097042300930
source: 2 (任通行)
car_num: 苏ZKN720
status: 0 (未支付)
payment_amount: 0.00
```

**rtx_user_product_policy 表**：
```sql
etc_card_user_id: 1979081793189675009
car_num: 苏ZKN720
bid_is_equipment_fee_required: 0 (申办时未缴费，需补缴)
equipment_amount: 0.02
```

### 接口返回
```json
{
  "code": 200,
  "msg": "操作成功",
  "data": {
    "amt": 0.02
  },
  "success": true
}
```

## 【原因定位】

### 问题根源
金额查询接口 `getCustomerOrderPayAmt` 只根据车辆的产品策略 `bid_is_equipment_fee_required` 判断金额，**没有考虑工单的 `pay_status` 状态**。

**代码位置**：`CustomerUsedCarOrderAppController.java:322-360`

```java
@GetMapping(value = "/getCustomerOrderPayAmt")
public AjaxResult<?> getCustomerOrderPayAmt(@RequestParam String source, @RequestParam String plateNum
        , @RequestParam String vehicleColor) {
    JSONObject rej = new JSONObject();
    // ... 省略校验
    if (CustomerUsedCarOrderEnum.Source.RTX_OFFICIAL_ACCOUNTS_LINK.getCode().equals(source)) {
        // 查询车辆信息
        List<EtcCardUser> etcCardUserList = iEtcCardUserService.list(etcCardUserQW);
        RtxUtils.check(CollectionUtils.isEmpty(etcCardUserList), "该车不存在，无法添加申请！");
        
        // 查询产品策略
        UserProductPolicy userProductPolicy = iUserProductPolicyService
                .selectDataByEtcCardUserId(etcCardUserList.get(0).getId());
        String feeReq = userProductPolicy.getBidIsEquipmentFeeRequired();
        
        // ← BUG: 只看产品策略，不看工单的 pay_status
        rej.put("amt", (StringUtils.isBlank(feeReq) || "0".equals(feeReq))
                ? userProductPolicy.getEquipmentAmount() : BigDecimal.ZERO);
    }
    return AjaxResult.success(rej);
}
```

### 逻辑问题
1. 接口参数只有 `source`, `plateNum`, `vehicleColor`，**没有工单ID**
2. 无法查询到工单的 `pay_status` 状态
3. 只能根据车辆的产品策略判断金额
4. 导致后台设置的"无需支付"被忽略

### 业务冲突
- **后台设置**：`pay_status='0'`(无须支付) → 应该金额为0
- **产品策略**：`bid_is_equipment_fee_required='0'`(申办时未缴费) → 返回金额0.02
- **两者冲突**：后台设置被产品策略覆盖

## 【修复建议】

### 方案1：接口增加工单ID参数（推荐）
修改接口，增加工单ID参数，优先判断工单的 `pay_status`。

**修改后的接口**：
```java
@GetMapping(value = "/getCustomerOrderPayAmt")
public AjaxResult<?> getCustomerOrderPayAmt(
        @RequestParam String source, 
        @RequestParam String plateNum,
        @RequestParam String vehicleColor,
        @RequestParam(required = false) Long workOrderId) {  // 新增参数
    
    JSONObject rej = new JSONObject();
    
    // 如果传入工单ID，优先判断工单的 pay_status
    if (workOrderId != null) {
        CustomerWorkOrder workOrder = customerWorkOrderService.selectCustomerWorkOrderById(workOrderId);
        if (workOrder != null && CustomerWorkOrderEnum.PayStatus.NOT_NEED.getCode().equals(workOrder.getPayStatus())) {
            rej.put("amt", BigDecimal.ZERO);
            return AjaxResult.success(rej);
        }
    }
    
    // 原有逻辑：根据产品策略判断
    if (CustomerUsedCarOrderEnum.Source.RTX_OFFICIAL_ACCOUNTS_LINK.getCode().equals(source)) {
        List<EtcCardUser> etcCardUserList = iEtcCardUserService.list(etcCardUserQW);
        RtxUtils.check(CollectionUtils.isEmpty(etcCardUserList), "该车不存在，无法添加申请！");
        
        UserProductPolicy userProductPolicy = iUserProductPolicyService
                .selectDataByEtcCardUserId(etcCardUserList.get(0).getId());
        String feeReq = userProductPolicy.getBidIsEquipmentFeeRequired();
        rej.put("amt", (StringUtils.isBlank(feeReq) || "0".equals(feeReq))
                ? userProductPolicy.getEquipmentAmount() : BigDecimal.ZERO);
    }
    
    return AjaxResult.success(rej);
}
```

**前端调用**：
```javascript
// H5页面先查询工单详情，获取工单ID
const workOrderId = 2008429097042300930;

// 调用金额接口时传入工单ID
const response = await axios.get('/rtx-app/usedCarOrderApp/getCustomerOrderPayAmt', {
    params: {
        source: 2,
        plateNum: '苏ZKN720',
        vehicleColor: 0,
        workOrderId: workOrderId  // 传入工单ID
    }
});
```

### 方案2：前端先查询工单状态
前端先调用工单详情接口，判断 `pay_status`，如果为 `'0'`(无须支付)，直接显示金额为0，不调用金额查询接口。

**前端逻辑**：
```javascript
// 1. 查询工单详情
const workOrder = await getWorkOrderDetail(workOrderId);

// 2. 判断 pay_status
if (workOrder.pay_status === '0') {
    // 无须支付，直接显示金额为0
    this.paymentAmount = 0;
} else {
    // 需要支付，调用金额查询接口
    const response = await getCustomerOrderPayAmt({
        source: 2,
        plateNum: workOrder.plate_num,
        vehicleColor: workOrder.vehicle_color
    });
    this.paymentAmount = response.data.amt;
}
```

### 方案3：后台创建时同步产品策略
后台创建工单时，如果勾选"无需支付"，同时将车辆的 `bid_is_equipment_fee_required` 设置为 `'1'`(已缴费)，确保数据一致性。

**不推荐理由**：
- 修改产品策略会影响其他业务
- 产品策略是车辆维度，不应该被工单修改
- 治标不治本

## 【推荐方案】
**方案1（接口增加工单ID参数）** 是最彻底的解决方案：
1. 接口改动小，只增加一个可选参数
2. 向后兼容，不影响其他调用方
3. 业务逻辑清晰：工单设置优先于产品策略
4. 数据一致性好：后台设置与H5显示一致

## 【影响范围】
- 所有后台创建的"无需支付"二手车工单
- 影响用户体验，造成用户困惑
- 可能导致用户投诉或客服咨询增加

