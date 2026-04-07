# BUG-【新发行】浙商银行卡解绑校验未生效

## 1. BUG描述

用户名下只有一张浙商银行卡，且车辆外部合作银行已配置为"呼和浩特浙商"，解绑该浙商卡时，未触发新需求的"至少保留一张浙商银行卡"校验，而是提示老逻辑的"名下存在有效车辆，无法操作该银行卡！"。

**影响**：
- ❌ 浙商银行卡解绑时，新增的"至少保留一张浙商卡"校验未生效
- ❌ 错误提示文案，用户无法理解为什么不能解绑
- ✅ 解绑操作本身被拦截（虽然提示不对，但至少没让用户解绑成功）

---

## 2. 复现步骤

### 前置条件
- 测试环境: 192.168.1.60
- 系统: 新发行APP
- 数据库: fenmi_etc
- 测试账号: 15818376788 / 441622199602075195
- 测试车辆: 蒙A113355（etc_user_id: 1929825749801947137）
- 测试银行卡: 6222034000046037473（id: 2026189792397012994）

### 操作步骤

**Step 1**: 查询车辆信息

```sql
SELECT id, car_num, id_code, user_wallet_id, status, operator_code, product_id 
FROM etc_user 
WHERE car_num = '蒙A113355' AND id_code = '441622199602075195' AND del_flag = 0;
```

**查询结果**:
```
id: 1929825749801947137
car_num: 蒙A113355
status: 4（正常）
operator_code: MTK
product_id: 1743204710478303234
```

**Step 2**: 查询用户政策配置

```sql
SELECT p.id, p.etc_user_id, p.order_id, pd.rule_key, pd.rule_value 
FROM etc_user_policy p 
LEFT JOIN etc_user_policy_detail pd ON p.id = pd.policy_id 
WHERE p.etc_user_id = 1929825749801947137 AND pd.rule_key = 'partnerBankValue';
```

**查询结果**:
```
etc_user_id: 1929825749801947137
rule_key: partnerBankValue
rule_value: 1（呼和浩特浙商）
```

**Step 3**: 查询名下银行卡

```sql
SELECT id, bank_code, bank_card_name, bank_card_no, card_status, del_flag, wallet_id_card 
FROM etc_user_bank_card 
WHERE wallet_id_card = '441622199602075195' AND del_flag = 0;
```

**查询结果**:
```
id: 2026189792397012994
bank_code: CZB_SZ
bank_card_name: 深圳浙商
bank_card_no: 6222034000046037473
card_status: 1（有效）
del_flag: 0（未解绑）
```

**Step 4**: 登录新发行APP，尝试解绑银行卡

1. 登录账号: 15818376788
2. 进入"我的银行卡"页面
3. 选择银行卡: 6222034000046037473（深圳浙商）
4. 点击"解绑"按钮
5. 观察提示信息

---

## 3. 预期结果

根据需求，用户名下只有一张浙商银行卡，且车辆外部合作银行配置为"呼和浩特浙商"，解绑时应触发新需求的校验。

**预期提示**:
```
您名下需启用至少一张浙商银行卡不允解绑！
```

**预期行为**:
- ✅ 系统识别该卡为浙商银行卡（bank_code: CZB_SZ）
- ✅ 触发浙商银行卡数量校验逻辑
- ✅ 检测到名下只有1张有效浙商卡
- ✅ 拒绝解绑，提示"您名下需启用至少一张浙商银行卡不允解绑！"

---

## 4. 实际结果

**实际提示**:
```json
{
  "msg": "名下存在有效车辆，无法操作该银行卡！",
  "code": 500
}
```

**实际行为**:
- ❌ 系统未识别该卡为浙商银行卡
- ❌ 未触发新需求的浙商银行卡数量校验
- ❌ 走了老逻辑的"最后一张银行卡 + 有效车辆"校验
- ❌ 提示文案错误，用户无法理解为什么不能解绑

**数据状态**:
- 车辆已配置外部合作银行: partnerBankValue = 1（呼和浩特浙商）
- 名下只有1张银行卡: bank_code = CZB_SZ（深圳浙商）
- 银行卡状态: card_status = 1, del_flag = 0（有效且未解绑）
- 车辆状态: status = 4（正常）

**日志路径**: `/home/fenmi/fenmi-etc/log.txt`
