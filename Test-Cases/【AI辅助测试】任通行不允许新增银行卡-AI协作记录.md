# 【AI辅助测试】任通行「不允许新增银行卡」功能 — AI协作全程记录

**需求ID**: ID1017497
**测试日期**: 2026-04-07 ~ 2026-04-08
**系统**: 任通行小程序 + 管理后台
**用途**: 会议分享 — 展示 AI 辅助测试工作的完整流程

---

## 一、需求背景

**我提给 AI 的原始需求描述：**

> 任通行新增字典配置「不允许新增银行卡配置」，包含两个配置项：
> - **渠道公司**：名下所有有效车辆（正常、黑名单、待激活）都属于该渠道公司，则不允许新增银行卡
> - **白名单**：豁免的用户车牌，应对特殊情况
>
> 当用户名下所有有效车辆均属于已配置渠道公司，且名下任意车辆均不属于白名单，则隐藏小程序两处按钮：
> - 扣款设置页面的「新增代扣签约」按钮
> - 账户信息页面的「新增签约银行卡」按钮

---

## 二、AI 分析代码逻辑

AI 自动拉取了 git 提交记录，找到了 5 次相关提交：

| 提交 Hash | 说明 |
|-----------|------|
| `71d57477` | 初始实现 |
| `cdc37dd6` | 联调修改 |
| `449bb2e9` | 联调修改1 |
| `fbdb6689` | 联调修改2 |
| `61c35c03` | **逻辑优化（最终版）** |

**AI 读取代码后，整理出最终版核心逻辑：**

```java
// IndexServiceImpl.isEnableBindBankCard(Long userId)
public Boolean isEnableBindBankCard(Long userId) {
    List<String> whiteList   = getDictValueListByType("enable_bind_bank_card_whitelist"); // 白名单（车牌号）
    List<String> channelList = getDictValueListByType("enable_bind_bank_card_channel");   // 禁止渠道（channel_id）
    List<String> statusList  = [NORMAL(1), BLACKLIST(2), TO_BE_ACTIVATED(3)];             // 有效状态

    List<EtcCardUser> list = 查询该用户所有车辆（不过滤状态）;
    boolean flag = true;

    for (EtcCardUser car : list) {
        if (whiteList.contains(car.getCarNum()))   return true;   // ① 白名单优先，直接放行
        if (!statusList.contains(car.getStatus())) continue;      // ② 非有效状态跳过
        if (!channelList.contains(car.getChannelId())) return true; // ③ 有一辆不在禁止渠道，放行
        flag = false;                                              // ④ 在禁止渠道，标记
    }
    return flag; // 所有有效车都在禁止渠道 → false（隐藏按钮）
}
```

**AI 总结的判断优先级：**

```
白名单命中（任意车牌）→ 显示按钮
    ↓ 否
非有效状态车辆 → 跳过不计入
    ↓
有效车辆中有一辆不在禁止渠道 → 显示按钮
    ↓ 否
所有有效车辆均在禁止渠道 → 隐藏按钮
```

**AI 查询数据库，发现测试环境已有配置：**

| 字典类型 | 当前配置值 | 说明 |
|----------|-----------|------|
| `enable_bind_bank_card_channel` | `0000` | 默认渠道-广东 |
| `enable_bind_bank_card_whitelist` | `蒙A44343`、`蒙A113355`、`黑A113355` | 白名单车牌 |

---

## 三、AI 编写测试用例（25条）

AI 根据代码逻辑和数据库现状，自动生成了完整测试用例，分为 6 大模块：

### 模块划分

| 模块 | 用例编号 | 测试重点 |
|------|---------|---------|
| 后台字典配置 | TC-001 ~ TC-005 | 渠道配置新增/删除、白名单配置 |
| 按钮隐藏场景 | TC-006 ~ TC-011 | 正常/黑名单/待激活状态 + 禁止渠道 |
| 按钮显示场景 | TC-012 ~ TC-016 | 无配置、混合渠道、无车辆、全注销 |
| 白名单豁免 | TC-017 ~ TC-020 | 白名单优先级、状态无关、精确匹配 |
| 双按钮位置验证 | TC-021 ~ TC-023 | 扣款设置 + 账户信息两处同步 |
| 回归测试 | TC-024 ~ TC-025 | 正常用户全流程、渠道切换后恢复 |

### 典型用例示例

**TC-007 单辆车-黑名单状态-属于禁止渠道-隐藏按钮**

- 前置条件：名下仅 1 辆车，`status=2`（黑名单），`channel_id=0000`，车牌不在白名单
- 测试步骤：登录小程序 → 扣款设置页 → 账户信息页
- 预期结果：两处按钮均隐藏

**TC-013 多辆车-部分属于禁止渠道-部分不属于-显示按钮**

- 前置条件：车辆A `channel_id=0000`（禁止），车辆B `channel_id=其他`（不禁止），均为有效状态
- 预期结果：按钮显示（存在有效车辆不在禁止渠道）

---

## 四、AI 辅助造数与问题排查

### 4.1 修正 SQL 查询

**我反馈 TC-006 的 SQL 不正确，AI 立即给出修正版：**

原始错误写法（只过滤了渠道和状态，没限制「仅 1 辆车」）：
```sql
-- ❌ 错误：会把多辆车用户也查出来
SELECT * FROM rtx_etc_card_user WHERE channel_id='0000' AND status='1' LIMIT 5;
```

AI 修正后的正确写法：
```sql
-- ✅ 正确：先用子查询限制「名下仅 1 辆车」
SELECT t.driver_user_id, t.car_num, t.channel_id, t.status
FROM rtx_etc_card_user t
INNER JOIN (
  SELECT driver_user_id
  FROM rtx_etc_card_user
  WHERE driver_user_id IS NOT NULL
  GROUP BY driver_user_id
  HAVING COUNT(*) = 1
) u ON t.driver_user_id = u.driver_user_id
WHERE t.channel_id = '0000'
  AND t.status = '1'
LIMIT 20;
-- 查询结果：driver_user_id=1840644592836440065，车牌粤B12D1B
```

### 4.2 排查登录报错「用户不存在」

**我遇到登录报错，AI 定位根因：**

```json
{
  "code": 500,
  "msg": "登录用户：13877857502 用户不存在"
}
```

**AI 分析过程：**
1. 定位到报错代码位置：`SmsCodeUserDetailsServiceImpl.java:42`
2. 查询 `rtx_driver_user_info` 表 → 该手机号无记录
3. 查询 `rtx_etc_card_user` 表 → 该手机号有车辆，`driver_user_id=1840644592836440065`
4. **根因**：车辆表有数据，但车主登录表缺少对应记录（数据不一致）

**AI 执行修复 SQL：**
```sql
INSERT INTO rtx_driver_user_info
  (id, create_time, update_time, user_name, phone, status, nick_name, wx_pub_oid_source)
VALUES
  (1840644592836440065, NOW(), NOW(), '13877857502', '13877857502', '0', '测试用户13877857502', '0');
-- 执行结果：Rows affected: 1 ✅
```

### 4.3 TC-007 造数

**库内无满足条件的用户，AI 自动造数：**

- 找到测试账号：`driver_user_id=1447478959637221378`，手机 `18631601378`，仅 1 辆车，状态=黑名单(2)
- 将 `channel_id` 更新为 `0000`（原值 `f4745c1b59e94ea18f1fef2190efa2bd`）

```sql
UPDATE rtx_etc_card_user
SET channel_id = '0000', update_time = NOW()
WHERE id = 1447478959771439106
  AND driver_user_id = 1447478959637221378;
-- 执行结果：Rows affected: 1 ✅
```

**AI 同时提供了测后回滚 SQL：**
```sql
-- 测完恢复原渠道
UPDATE rtx_etc_card_user
SET channel_id = 'f4745c1b59e94ea18f1fef2190efa2bd', update_time = NOW()
WHERE id = 1447478959771439106
  AND driver_user_id = 1447478959637221378;
```

### 4.4 TC-015 找无车用户

**AI 查询并推荐符合条件的账号：**

```sql
SELECT d.id AS driver_user_id, d.phone, d.user_name
FROM rtx_driver_user_info d
LEFT JOIN rtx_etc_card_user t ON t.driver_user_id = d.id
WHERE t.id IS NULL AND d.status = '0'
LIMIT 20;
```

推荐账号：**`13418807900`**（`driver_user_id=1442713233802104833`，车辆数=0）

---

## 五、测试数据汇总

| 用例 | 手机号 | driver_user_id | 车牌 | 状态 | 渠道 | 备注 |
|------|--------|----------------|------|------|------|------|
| TC-006 | 13877857502 | 1840644592836440065 | 粤B12D1B | 正常(1) | 0000 | AI 补插车主记录 |
| TC-007 | 18631601378 | 1447478959637221378 | 陕F989898 | 黑名单(2) | 0000 | AI 修改渠道，测后回滚 |
| TC-015 | 13418807900 | 1442713233802104833 | 无 | — | — | 无车辆账号 |

---

## 六、AI 协作价值总结

| 环节 | 传统方式 | AI 辅助后 |
|------|---------|----------|
| 理解代码逻辑 | 手动翻代码，找开发确认 | AI 自动读 git diff，整理判断流程 |
| 编写测试用例 | 手写，容易遗漏边界场景 | AI 结合代码逻辑自动覆盖 25 个场景 |
| 造数 SQL | 手写，容易写错 | AI 实时查询验证，给出可直接执行的 SQL |
| 问题排查 | 看日志 + 问开发 | AI 定位代码行 + 查库 + 给出修复方案 |
| 数据回滚 | 容易忘记 | AI 主动提供回滚 SQL |

---

## 七、相关文档

- 测试用例：`temp/测试用例/【任通行】支持不允许新增银行卡测试用例_V1.0.md`
- 测试总结：`temp/测试总结/【任通行】支持不允许新增银行卡-测试总结_20260408.md`
- 应用日志：`/home/java-server/rtx-app/log.txt`
