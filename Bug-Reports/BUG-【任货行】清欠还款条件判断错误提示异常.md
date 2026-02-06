# BUG-【任货行】清欠还款条件判断错误提示异常

## 【BUG描述】

清欠还款接口（`com.hcb.getDebtRepaymentAmount`）在用户不满足还款条件时（冻结金额+待冻结金额覆盖全部欠款），返回错误提示"还款信息校验异常，还款金额低于0元，请联系客服处理"，与实际业务语义不符，应该提示"您暂不满足该还款条件，请登录小程序进行还款支付"。

**影响范围**：所有清欠还款用户
**频率**：100%复现

---

## 【前置条件】

### 测试环境
- 环境：测试环境
- 数据库：192.168.1.60:3306/hcb
- 接口地址：`com.hcb.getDebtRepaymentAmount`

### 测试账号
- 车牌号：蒙ZMM777
- 车牌颜色：1（黄色）
- 运营商：MTK（内蒙古）
- 钱包ID：7b4bfaa4a8de11f084f1141877512abb

### 测试数据（核心条件）
```sql
-- 钱包数据
BALANCE = -3000.00        -- 欠款3000元
FREEZE_AMOUNT = 500.00    -- 冻结金额500元
TOBE_FREEZE_AMT = 2500.00 -- 待冻结金额2500元

-- 需还款金额 = 3000 - 500 - 2500 = 0
```

### 其他条件
- ✅ 用户有欠款（余额<0）
- ✅ 用户有清欠标签（PURGE_Wallet）
- ✅ 冻结金额 < 欠款绝对值（500 < 3000）
- ❌ 冻结金额 + 待冻结金额 = 欠款（500 + 2500 = 3000）

---

## 【复现步骤】

### Step 1: 准备测试数据

```sql
-- 1. 查询用户钱包
SELECT TRUCKUSERWALLET_ID, BALANCE, FREEZE_AMOUNT, TOBE_FREEZE_AMT 
FROM hcb_hlj_truckuserwallet 
WHERE TRUCKUSERWALLET_ID = '7b4bfaa4a8de11f084f1141877512abb';

-- 2. 确保钱包数据满足条件
UPDATE hcb_hlj_truckuserwallet 
SET BALANCE = -3000.00, 
    FREEZE_AMOUNT = 500.00, 
    TOBE_FREEZE_AMT = 2500.00
WHERE TRUCKUSERWALLET_ID = '7b4bfaa4a8de11f084f1141877512abb';

-- 3. 确认用户有清欠标签
SELECT * FROM hcb_label_config_manage_detail 
WHERE ref_id = '7b4bfaa4a8de11f084f1141877512abb' 
  AND label_id = '2008733626665279488'  -- PURGE_Wallet
  AND is_delete = 0;
```

### Step 2: 调用清欠还款接口

```json
{
  "carNum": "蒙ZMM777",
  "vehicleColor": "1",
  "relativeurl": "com.hcb.getDebtRepaymentAmount",
  "caller": "chefuAPP",
  "timestamp": 1769046685706,
  "hashcode": "45e0b0748fad7408e5b45ae91f41e146"
}
```

### Step 3: 观察返回结果

查看接口返回的错误信息。

---

## 【期望结果】

### 期望返回
```json
{
  "ret": "11",
  "msg": "您暂不满足该还款条件，请登录小程序进行还款支付"
}
```

### 业务逻辑
当用户的（冻结金额 + 待冻结金额）≥ 欠款绝对值时，说明用户的欠款已经被完全覆盖（通过冻结和优惠减免），无需还款，应该提示用户不满足清欠还款条件。

### 4个条件判断
清欠还款需要同时满足：
1. ✅ 钱包余额 < 0（有欠款）
2. ✅ 冻结金额 < 余额绝对值
3. ✅ 用户有 PURGE_Wallet 标签
4. ❌ **实际还款金额 > 0**（本案例不满足）

计算公式：
```
实际还款金额 = |余额| - 冻结金额 - 待冻结金额
             = 3000 - 500 - 2500
             = 0
```

因为实际还款金额 = 0，不满足第4个条件，应该提示"您暂不满足该还款条件"。

---

## 【实际结果】

### 实际返回
```json
{
  "ret": "11",
  "msg": "还款信息校验异常，还款金额低于0元，请联系客服处理"
}
```

### SQL查询验证
```sql
SET @rate = 6;

SELECT 
  tu.CAR_NUM as '车牌号',
  tw.BALANCE as '余额',
  tw.FREEZE_AMOUNT as '冻结',
  tw.TOBE_FREEZE_AMT as '待冻结',
  ABS(tw.BALANCE) - tw.FREEZE_AMOUNT - tw.TOBE_FREEZE_AMT as '需还款金额',
  CASE 
    WHEN tw.BALANCE < 0 
     AND tw.FREEZE_AMOUNT < ABS(tw.BALANCE)
     AND label.id IS NOT NULL
     AND (ABS(tw.BALANCE) - tw.FREEZE_AMOUNT - tw.TOBE_FREEZE_AMT) > 0
    THEN '✅ 满足' ELSE '❌ 不满足' 
  END as '最终判断'
FROM hcb_truckuser tu
INNER JOIN hcb_hlj_truckuserwallet tw ON tu.TRUCKUSERWALLET_ID = tw.TRUCKUSERWALLET_ID
LEFT JOIN hcb_label_config_manage_detail label 
  ON tw.TRUCKUSERWALLET_ID = label.ref_id 
  AND label.label_id = '2008733626665279488'
  AND label.is_delete = 0
WHERE tu.CAR_NUM = '蒙ZMM777';
```

结果：
| 车牌号 | 余额 | 冻结 | 待冻结 | 需还款金额 | 最终判断 |
|--------|------|------|--------|-----------|---------|
| 蒙ZMM777 | -3000 | 500 | 2500 | **0** | ❌ 不满足 |

### 问题说明
错误提示"还款信息校验异常，还款金额低于0元，请联系客服处理"：
- ❌ 语义不准确：实际是"还款金额=0元"，不是"低于0元"
- ❌ 用户体验差：提示"联系客服"让用户误以为是系统异常
- ❌ 与其他条件不一致：其他不满足条件时提示"您暂不满足该还款条件"

---

## 【附件】

### 1. 接口调用记录
```json
// 请求
{
  "carNum": "蒙ZMM777",
  "vehicleColor": "1"
}

// 响应
{
  "ret": "11",
  "msg": "还款信息校验异常，还款金额低于0元，请联系客服处理"
}
```

### 2. 数据库查询结果
```json
{
  "车牌号": "蒙ZMM777",
  "钱包余额": -3000.00,
  "冻结金额": 500.00,
  "待冻结金额": 2500.00,
  "需还款金额": 0.00,
  "条件1_余额小于0": "✅",
  "条件2_冻结小于欠款": "✅",
  "条件3_有清欠标签": "✅",
  "条件4_还款金额大于0": "❌",
  "最终判断": "❌ 不满足条件"
}
```

### 3. 相关文件
- 代码分析：`temp/清欠还款条件判断Bug分析.md`
- SQL查询：`temp/清欠还款金额计算-简洁版.sql`

---

## 【原因定位】

### 代码文件
`java/hcb/hcbapi/src/main/java/com/hcb/api/service/information/topuprecord/impl/TopupRecordService.java`

### Bug代码位置
第 1095-1109 行

### 问题分析

#### 当前代码逻辑
```java
public DebtRepaymentAmountVO queryDebtRepaymentAmount(String walletId, String operatorCode) {
    TruckUserWallet wallet = truckuserwalletService.getTruckUserWalletById(walletId);
    
    // 检查1：钱包是否存在
    if (wallet == null) {
        throw new CustomException("用户钱包信息不存在，请联系客服!", Convert.toInt(Constants.FAILURE_CODE));
    }
    
    // 检查2：是否有欠款
    if (wallet.getBalance().compareTo(BigDecimal.ZERO) >= 0) {
        throw new CustomException("您当前暂无欠款", NO_DEBT_ERROR_CODE);
    }
    
    // 检查3：冻结金额是否覆盖全部欠款 ⚠️ BUG在这里！
    if (wallet.getFreezeAmount().compareTo(wallet.getBalance().abs()) >= 0) {
        throw new CustomException("您暂不满足该还款条件，请登录小程序进行还款支付", NO_REPAYMENT_CONDITION_ERROR_CODE);
    }
    
    // 检查4：是否有清欠标签
    boolean isPurgeWalletUser = labelConfigManageDetailService.isExistConfigManageDetailByLabelKey(LabelKeyEnum.PURGE_WALLET, walletId);
    if (!isPurgeWalletUser) {
        throw new CustomException("您暂不满足该还款条件，请登录小程序进行还款支付", NO_REPAYMENT_CONDITION_ERROR_CODE);
    }
    
    // 计算需还款金额
    BigDecimal balanceAbs = wallet.getBalance().abs();
    BigDecimal freezeAmount = Objects.isNull(wallet.getFreezeAmount()) ? BigDecimal.ZERO : wallet.getFreezeAmount();
    BigDecimal tobeFreezeAmt = Objects.isNull(wallet.getTobeFreezeAmt()) ? BigDecimal.ZERO : wallet.getTobeFreezeAmt();
    BigDecimal totalFee = balanceAbs.subtract(freezeAmount).subtract(tobeFreezeAmt);
    
    // 检查5：需还款金额是否<=0 ⚠️ 实际命中这里
    if (totalFee.compareTo(BigDecimal.ZERO) <= 0) {
        throw new CustomException("还款信息校验异常，还款金额低于0元，请联系客服处理", NO_REPAYMENT_CONDITION_ERROR_CODE);
    }
    
    // ... 后续计算
}
```

#### Bug根因

**第1095行的检查逻辑不完整**：

```java
// ❌ 只检查了冻结金额
if (wallet.getFreezeAmount().compareTo(wallet.getBalance().abs()) >= 0) {
    throw new CustomException("您暂不满足该还款条件...");
}
```

**问题**：
1. 只判断了 `冻结金额 >= 欠款`
2. **没有包含待冻结金额（tobeFreezeAmt）**
3. 导致检查不完整，未能提前拦截

#### 执行流程（蒙ZMM777案例）

```
1. 检查1：钱包存在 ✅
   └─ wallet != null → 通过

2. 检查2：有欠款 ✅
   └─ -3000 < 0 → 通过

3. 检查3：冻结金额是否覆盖欠款 ❌ BUG！
   └─ 只检查：freezeAmount(500) >= balanceAbs(3000)
   └─ 500 >= 3000 → false → 通过（不应该通过！）
   
   ⚠️ 正确逻辑应该是：
   └─ (freezeAmount + tobeFreezeAmt) >= balanceAbs
   └─ (500 + 2500) >= 3000 → true → 应该在这里拦截！

4. 检查4：有清欠标签 ✅
   └─ isPurgeWalletUser = true → 通过

5. 计算需还款金额
   └─ totalFee = 3000 - 500 - 2500 = 0

6. 检查5：需还款金额>0 ❌
   └─ 0 <= 0 → true → 抛出异常 ❌
   └─ "还款信息校验异常，还款金额低于0元，请联系客服处理"
```

#### 为什么提示不正确？

因为第1095行的检查不完整，本应在检查3拦截的情况，漏过去了，最后在检查5时才被拦截，但检查5使用的是针对"系统异常"的错误提示，而不是"业务条件不满足"的提示。

---

## 【修复建议】

### 方案1：调整检查顺序（推荐）

**优点**：
- 代码改动小
- 逻辑更清晰简洁
- 只需一个统一的判断

**修改内容**：

1. **删除第1095-1097行**（不完整的检查）：
```java
// ❌ 删除这3行
if (wallet.getFreezeAmount().compareTo(wallet.getBalance().abs()) >= 0) {
    throw new CustomException("您暂不满足该还款条件，请登录小程序进行还款支付", NO_REPAYMENT_CONDITION_ERROR_CODE);
}
```

2. **修改第1107-1109行**的异常提示：
```java
BigDecimal totalFee = balanceAbs.subtract(freezeAmount).subtract(tobeFreezeAmt);

// ✅ 修改异常信息
if (totalFee.compareTo(BigDecimal.ZERO) <= 0) {
    // 从："还款信息校验异常，还款金额低于0元，请联系客服处理"
    // 改为：
    throw new CustomException("您暂不满足该还款条件，请登录小程序进行还款支付", NO_REPAYMENT_CONDITION_ERROR_CODE);
}
```

**修改后的完整代码**：

```java
public DebtRepaymentAmountVO queryDebtRepaymentAmount(String walletId, String operatorCode) {
    TruckUserWallet wallet = truckuserwalletService.getTruckUserWalletById(walletId);
    
    if (wallet == null) {
        throw new CustomException("用户钱包信息不存在，请联系客服!", Convert.toInt(Constants.FAILURE_CODE));
    }
    
    if (wallet.getBalance().compareTo(BigDecimal.ZERO) >= 0) {
        throw new CustomException("您当前暂无欠款", NO_DEBT_ERROR_CODE);
    }
    
    // ❌ 删除：不完整的检查
    
    boolean isPurgeWalletUser = labelConfigManageDetailService.isExistConfigManageDetailByLabelKey(LabelKeyEnum.PURGE_WALLET, walletId);
    if (!isPurgeWalletUser) {
        throw new CustomException("您暂不满足该还款条件，请登录小程序进行还款支付", NO_REPAYMENT_CONDITION_ERROR_CODE);
    }
    
    BigDecimal balanceAbs = wallet.getBalance().abs();
    BigDecimal freezeAmount = Objects.isNull(wallet.getFreezeAmount()) ? BigDecimal.ZERO : wallet.getFreezeAmount();
    BigDecimal tobeFreezeAmt = Objects.isNull(wallet.getTobeFreezeAmt()) ? BigDecimal.ZERO : wallet.getTobeFreezeAmt();
    
    BigDecimal totalFee = balanceAbs.subtract(freezeAmount).subtract(tobeFreezeAmt);
    
    // ✅ 修改：统一判断实际还款金额
    if (totalFee.compareTo(BigDecimal.ZERO) <= 0) {
        throw new CustomException("您暂不满足该还款条件，请登录小程序进行还款支付", NO_REPAYMENT_CONDITION_ERROR_CODE);
    }
    
    BigDecimal chargeRate = bankQuickPayV2Server.getCommissionByBianMa("WECHAT_PAY_RATE", operatorCode).divide(BigDecimal.valueOf(100));
    BigDecimal serviceFee = totalFee.multiply(chargeRate).setScale(2, RoundingMode.UP);
    BigDecimal preDiscountAmount = balanceAbs.add(serviceFee);
    BigDecimal actualAmount = totalFee.add(serviceFee);
    
    return new DebtRepaymentAmountVO(actualAmount, preDiscountAmount, freezeAmount, tobeFreezeAmt, serviceFee, wallet);
}
```

---

### 方案2：修复检查逻辑

**优点**：
- 提前拦截，性能更好
- 可以区分不同的不满足情况

**修改内容**：

修复第1095行，包含待冻结金额：

```java
// ✅ 修复：检查冻结金额 + 待冻结金额
BigDecimal freezeAmount = Objects.isNull(wallet.getFreezeAmount()) ? BigDecimal.ZERO : wallet.getFreezeAmount();
BigDecimal tobeFreezeAmt = Objects.isNull(wallet.getTobeFreezeAmt()) ? BigDecimal.ZERO : wallet.getTobeFreezeAmt();
BigDecimal totalFreeze = freezeAmount.add(tobeFreezeAmt);

if (totalFreeze.compareTo(wallet.getBalance().abs()) >= 0) {
    throw new CustomException("您暂不满足该还款条件，请登录小程序进行还款支付", NO_REPAYMENT_CONDITION_ERROR_CODE);
}
```

---

### 推荐方案

**推荐方案1**，理由：
1. ✅ 代码改动小（删除3行，修改1行）
2. ✅ 逻辑更清晰（统一用 totalFee 判断）
3. ✅ 减少重复代码
4. ✅ 错误提示统一

---

## 【测试验证】

修改后需要测试以下场景：

### 测试用例1：冻结金额 = 欠款
```
余额：-1000，冻结：1000，待冻结：0
预期：提示"您暂不满足该还款条件"
```

### 测试用例2：冻结 + 待冻结 = 欠款
```
余额：-1000，冻结：500，待冻结：500
预期：提示"您暂不满足该还款条件"
```

### 测试用例3：冻结 + 待冻结 > 欠款
```
余额：-1000，冻结：600，待冻结：500
预期：提示"您暂不满足该还款条件"
```

### 测试用例4：正常还款（冻结 + 待冻结 < 欠款）
```
余额：-1000，冻结：200，待冻结：100
预期：正常返回金额，可以还款
```

### 测试用例5：本次Bug案例
```
余额：-3000，冻结：500，待冻结：2500
预期：提示"您暂不满足该还款条件" ✅
```
