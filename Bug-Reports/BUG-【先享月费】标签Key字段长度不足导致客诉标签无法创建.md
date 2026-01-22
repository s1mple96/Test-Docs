# BUG-【先享月费】标签Key字段长度不足导致客诉标签无法创建

## 【问题描述】
数据库表`hcb_label_config_manage`的`label_key`字段长度为25字符，但代码枚举类`LabelKeyEnum`中定义的客诉标签key `MonthlyPayLaterComplaint_Wallet`为32字符，超出字段长度限制，导致无法创建客诉标签。

## 【影响范围】
- 先享月费客诉标签无法正常创建
- 测试环境无法测试客诉用户扣款拦截功能
- 生产环境如果需要使用客诉标签功能将无法正常工作

## 【复现步骤】
1. 查看代码中标签枚举定义：
   - 文件：`com.etc.api.enums.LabelKeyEnum.java`
   - 第29行：`MONTHLY_PAY_LATER_COMPLAINT_WALLET("MonthlyPayLaterComplaint_Wallet")`
   - 字符数：32字符

2. 查看数据库表结构：
   ```sql
   DESC hcb_label_config_manage;
   ```
   - 字段：`label_key`
   - 类型：`varchar(25)`
   - 最大长度：25字符

3. 尝试插入客诉标签：
   ```sql
   INSERT INTO hcb_label_config_manage 
   (id, label_name, label_key, label_object, label_status, label_explain, create_user, create_date)
   VALUES ('test_id', '先享月费客诉用户', 'MonthlyPayLaterComplaint_Wallet', 1, 1, '客诉标签', 'admin', NOW());
   ```

4. 报错信息：
   ```
   Error executing query: 1406 (22001): Data too long for column 'label_key' at row 1
   ```

## 【预期结果】
客诉标签能正常创建到数据库，扣款时能正确拦截客诉用户

## 【实际结果】
因字段长度限制，客诉标签无法创建

## 【数据对比】
| 项目 | 长度 | 值 |
|------|------|-----|
| **代码枚举值** | **32字符** | `MonthlyPayLaterComplaint_Wallet` |
| **数据库字段长度** | **25字符** | `varchar(25)` |
| **差距** | **超出7字符** | ❌ 不匹配 |