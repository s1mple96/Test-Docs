# BUG-【任通行】二手车工单设备费金额计算逻辑错误

## 【BUG描述】
二手车工单获取设备费金额接口逻辑写反，导致需要收取设备费时返回0元，不需要收取时反而返回设备费金额，影响所有二手车工单的设备费收取。

## 【前置条件】
- 测试环境：http://788360p9o5.yicp.fun
- 测试车辆：黑A113355（vehicle_color=4）
- 数据库：rtx库
- 产品策略配置：
  - `rtx_user_product_policy.bid_is_equipment_fee_required` = '1'（需要设备费）
  - `rtx_user_product_policy.equipment_amount` = 0.02

## 【复现步骤】
1. 修改产品策略为需要设备费
```sql
UPDATE rtx_user_product_policy 
SET bid_is_equipment_fee_required='1' 
WHERE etc_card_user_id=1927255865551912962;
```

2. 调用获取金额接口
```bash
curl 'http://788360p9o5.yicp.fun/rtx-app/usedCarOrderApp/getCustomerOrderPayAmt?source=2&vehicleColor=4&plateNum=%E9%BB%91A113355'
```

3. 观察返回结果

4. 修改产品策略为不需要设备费
```sql
UPDATE rtx_user_product_policy 
SET bid_is_equipment_fee_required='0' 
WHERE etc_card_user_id=1927255865551912962;
```

5. 再次调用接口观察结果

## 【期望结果】
- 当 `bid_is_equipment_fee_required='1'`（需要设备费）时，接口返回：
```json
{"code":200,"msg":"操作成功","data":{"amt":0.02},"success":true}
```

- 当 `bid_is_equipment_fee_required='0'`（不需要设备费）时，接口返回：
```json
{"code":200,"msg":"操作成功","data":{"amt":0},"success":true}
```

## 【实际结果】
- 当 `bid_is_equipment_fee_required='1'`（需要设备费）时，接口返回：
```json
{"code":200,"msg":"操作成功","data":{"amt":0},"success":true}
```
❌ 返回0元，无法收取设备费

- 当 `bid_is_equipment_fee_required='0'`（不需要设备费）时，接口返回：
```json
{"code":200,"msg":"操作成功","data":{"amt":0.02},"success":true}
```
❌ 返回0.02元，不应该收费

**逻辑完全相反**

## 【附件】

### 数据库配置
```sql
mysql> SELECT id, etc_card_user_id, bid_is_equipment_fee_required, equipment_amount 
       FROM rtx_user_product_policy 
       WHERE etc_card_user_id=1927255865551912962;
+---------------------+---------------------+-------------------------------+------------------+
| id                  | etc_card_user_id    | bid_is_equipment_fee_required | equipment_amount |
+---------------------+---------------------+-------------------------------+------------------+
| 1927256236844285954 | 1927255865551912962 | 0                             | 0.02             |
+---------------------+---------------------+-------------------------------+------------------+
```

### 字段说明
```sql
bid_is_equipment_fee_required varchar(2) comment '申办是否需要缴纳设备费 0-否,1-是'
```

## 【原因定位】

### 问题代码位置
**文件**: `java/rtx/rtx-app-server/src/main/java/com/rtx/app/controller/CustomerUsedCarOrderAppController.java`

**行号**: 343-345行

### 错误代码
```java
String feeReq = userProductPolicy.getBidIsEquipmentFeeRequired();
rej.put("amt", (StringUtils.isBlank(feeReq) || "0".equals(feeReq))
        ? userProductPolicy.getEquipmentAmount() : BigDecimal.ZERO);
```

### 问题分析
当前逻辑：
- `feeReq='0'`（不需要设备费）→ 返回 `equipment_amount`（金额） ❌
- `feeReq='1'`（需要设备费）→ 返回 `BigDecimal.ZERO`（0元） ❌

**逻辑完全写反了！**

## 【修复建议】

### 修改方案
将343-345行代码修改为：

```java
String feeReq = userProductPolicy.getBidIsEquipmentFeeRequired();
rej.put("amt", "1".equals(feeReq) 
        ? userProductPolicy.getEquipmentAmount() : BigDecimal.ZERO);
```

### 修改后逻辑
- `feeReq='1'`（需要设备费）→ 返回 `equipment_amount`（金额） ✅
- `feeReq='0'`（不需要设备费）→ 返回 `BigDecimal.ZERO`（0元） ✅

### 影响范围
- 接口：`/rtx-app/usedCarOrderApp/getCustomerOrderPayAmt`
- 影响：所有任通行二手车工单的设备费计算
- 优先级：**高**（影响收费逻辑）

### 测试验证
修改后需验证：
1. `bid_is_equipment_fee_required='1'` 时返回正确金额
2. `bid_is_equipment_fee_required='0'` 时返回0
3. 回归测试二手车工单完整流程

