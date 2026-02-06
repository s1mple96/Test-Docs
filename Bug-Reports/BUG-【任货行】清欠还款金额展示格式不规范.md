# BUG-【任货行】清欠还款金额展示格式不规范

## 【BUG描述】

清欠还款相关金额展示时，部分金额字段未严格按照财务规范保留小数点后2位，导致金额显示不统一、不规范，影响用户体验和财务对账。

**影响范围**：所有清欠还款用户
**频率**：100%复现

---

## 【前置条件】

### 测试环境
- 环境：测试环境
- 数据库：192.168.1.60:3306/hcb
- 接口：`com.hcb.getDebtRepaymentAmount`

### 测试账号
- 车牌号：蒙ZMM777
- 车牌颜色：1（黄色）
- 运营商：MTK

### 测试数据
```sql
-- 钱包数据
BALANCE = -2500.50
FREEZE_AMOUNT = 100.00
TOBE_FREEZE_AMT = 100.00

-- 计算结果
需还款金额 = 2300.50
手续费率 = 6%
充值手续费 = 2300.50 × 0.06 = 138.03
```

---

## 【复现步骤】

### Step 1: 调用接口获取还款金额

```json
{
  "carNum": "蒙ZMM777",
  "vehicleColor": "1",
  "relativeurl": "com.hcb.getDebtRepaymentAmount"
}
```

### Step 2: 使用SQL验证金额计算

```sql
SET @rate = 6;

SELECT 
  tu.CAR_NUM as '车牌号',
  tw.BALANCE as '余额',
  CEILING((ABS(tw.BALANCE) - tw.FREEZE_AMOUNT - tw.TOBE_FREEZE_AMT) * @rate / 100 * 100) / 100 as 'S手续费',
  ABS(tw.BALANCE) + CEILING((ABS(tw.BALANCE) - tw.FREEZE_AMOUNT - tw.TOBE_FREEZE_AMT) * @rate / 100 * 100) / 100 as 'P总欠款',
  (ABS(tw.BALANCE) - tw.FREEZE_AMOUNT - tw.TOBE_FREEZE_AMT) + 
  CEILING((ABS(tw.BALANCE) - tw.FREEZE_AMOUNT - tw.TOBE_FREEZE_AMT) * @rate / 100 * 100) / 100 as '实际支付'
FROM hcb_truckuser tu
INNER JOIN hcb_hlj_truckuserwallet tw ON tu.TRUCKUSERWALLET_ID = tw.TRUCKUSERWALLET_ID
WHERE tu.CAR_NUM = '蒙ZMM777';
```

### Step 3: 查看各个环节的金额展示

- 接口返回JSON
- 数据库查询结果
- 前端页面展示
- 日志记录

### Step 4: 对比金额格式

观察各个环节金额的小数位数是否统一为2位。

---

## 【期望结果】

### 财务金额展示规范

根据财务规范，所有金额字段必须**严格保留小数点后2位**：

1. **整数金额**：`100` → `100.00`
2. **一位小数**：`100.5` → `100.50`
3. **两位小数**：`100.53` → `100.53`
4. **多位小数**：`100.5300` → `100.53`

### 预期展示格式

```json
{
  "ret": "1",
  "msg": "请求成功!",
  "data": {
    "preDiscountAmount": "2638.53",      // ✅ 2位小数
    "freezeAmount": "100.00",            // ✅ 2位小数
    "actualAmount": "2438.53",           // ✅ 2位小数
    "tobeFreezeAmt": "100.00",           // ✅ 2位小数
    "serviceFee": "138.03",              // ✅ 2位小数（如果返回）
    "name": "骆志敏",
    "carNum": "蒙ZMM777",
    "truckUserId": "6457f83940ad4501bc2f01474093c8c4"
  }
}
```

### SQL查询结果（预期）

| 车牌号 | 手续费 | 总欠款 | 实际支付 |
|--------|--------|--------|---------|
| 蒙ZMM777 | **138.03** | **2638.53** | **2438.53** |

**要求**：所有金额字段都是2位小数，不能出现 `138.0300` 或 `2638.5300` 这种格式。

---

## 【实际结果】

### 问题1：SQL查询结果格式不规范

```sql
-- 执行查询
SET @rate = 6;
SELECT ... CEILING(...) / 100 as 'S手续费' ...;
```

**实际输出**：
```
S手续费	    P总欠款	      实际支付
138.0300	2638.5300	  2438.5300
```

❌ **问题**：金额显示为4位小数，不符合财务规范

### 问题2：可能存在的其他格式问题

#### 数据库存储
```sql
SELECT BALANCE, FREEZE_AMOUNT, TOBE_FREEZE_AMT 
FROM hcb_hlj_truckuserwallet;

-- 可能的结果
BALANCE: -2500.5   ❌ (应该是 -2500.50)
FREEZE_AMOUNT: 100 ❌ (应该是 100.00)
```

#### 日志记录
```
充值手续费：138.03
总欠款：2638.5300  ❌
实际支付：2438.53
```

#### 前端展示
```
总欠款：2638.53 元  ✅
手续费：138.0 元    ❌ (可能只有1位小数)
实际支付：2438 元   ❌ (可能没有小数)
```

---

## 【附件】

### 1. SQL查询实际结果截图

```bash
# 执行命令
/home/redis/bin/redis-cli -a chefu666 ...

# 输出示例
车牌号	    余额	    冻结	    待冻结	  费率	Q需还款	S手续费	  P总欠款	    实际支付
蒙ZMM777	-2500.50	100.00	100.00	6%	2300.50	138.0300	2638.5300	2438.5300
```

### 2. 接口返回数据

```json
// 当前返回（可能正确）
{
  "preDiscountAmount": 2638.53,    // Number类型，2位小数 ✅
  "actualAmount": 2438.53          // Number类型，2位小数 ✅
}

// 但如果转为字符串可能丢失末尾0
{
  "freezeAmount": "100",    // ❌ 应该是 "100.00"
  "tobeFreezeAmt": "100.0"  // ❌ 应该是 "100.00"
}
```

### 3. 涉及的金额字段

#### 接口返回字段
- `preDiscountAmount` - 总欠款（优惠前金额）
- `freezeAmount` - 过往留存金额（冻结金额）
- `actualAmount` - 实际支付金额
- `tobeFreezeAmt` - 优惠减免金额（待冻结金额）
- `serviceFee` - 充值手续费（未返回，但应该规范）

#### 数据库字段
- `BALANCE` - 钱包余额
- `FREEZE_AMOUNT` - 冻结金额
- `TOBE_FREEZE_AMT` - 待冻结金额
- `WITHHOLD_FREEZE_AMOUNT` - 代扣冻结金额
- `TEMP_AMOUNT` - 临时钱包余额

---

## 【原因定位】

### 原因1：SQL CEILING计算导致精度问题

**代码位置**：SQL查询语句

```sql
-- ❌ 问题代码
CEILING((ABS(tw.BALANCE) - tw.FREEZE_AMOUNT - tw.TOBE_FREEZE_AMT) * @rate / 100 * 100) / 100

-- 计算过程
2300.50 × 0.06 × 100 = 13803
13803 / 100 = 138.03
```

**问题**：
- `CEILING` 返回整数，但除以100后 MySQL 可能显示为 `138.0300`
- 没有使用 `FORMAT()` 或 `ROUND()` 函数格式化

### 原因2：Java BigDecimal 格式化缺失

**代码位置**：`TopupRecordService.java:1112-1116`

```java
// 当前代码
BigDecimal serviceFee = totalFee.multiply(chargeRate).setScale(2, RoundingMode.UP);
BigDecimal preDiscountAmount = balanceAbs.add(serviceFee);
BigDecimal actualAmount = totalFee.add(serviceFee);

return new DebtRepaymentAmountVO(actualAmount, preDiscountAmount, freezeAmount, tobeFreezeAmt, serviceFee, wallet);
```

**问题**：
- `serviceFee` 使用了 `.setScale(2, RoundingMode.UP)` ✅
- 但 `preDiscountAmount` 和 `actualAmount` 没有设置精度 ❌
- `freezeAmount` 和 `tobeFreezeAmt` 直接从数据库读取，可能没有格式化 ❌

### 原因3：数据库字段定义

```sql
-- 查看表结构
DESC hcb_hlj_truckuserwallet;

-- 字段定义
BALANCE decimal(11,2)           ✅ 2位小数
FREEZE_AMOUNT decimal(11,2)     ✅ 2位小数
TOBE_FREEZE_AMT decimal(11,2)   ✅ 2位小数
```

**结论**：数据库字段定义正确，问题在于计算和展示环节。

### 原因4：前端展示格式化

```javascript
// 可能的前端代码
<div>{data.actualAmount}</div>  // ❌ 直接展示，可能丢失末尾0

// 应该格式化
<div>{Number(data.actualAmount).toFixed(2)}</div>  // ✅
```

---

## 【修复建议】

### 修复1：Java 后端统一格式化（推荐）

**文件**：`TopupRecordService.java:1112-1117`

```java
// ✅ 修改后
BigDecimal serviceFee = totalFee.multiply(chargeRate).setScale(2, RoundingMode.UP);

// 统一设置精度为2位小数
BigDecimal preDiscountAmount = balanceAbs.add(serviceFee).setScale(2, RoundingMode.HALF_UP);
BigDecimal actualAmount = totalFee.add(serviceFee).setScale(2, RoundingMode.HALF_UP);

// 从数据库读取的金额也确保2位小数
BigDecimal freezeAmountFormatted = freezeAmount.setScale(2, RoundingMode.HALF_UP);
BigDecimal tobeFreezeAmtFormatted = tobeFreezeAmt.setScale(2, RoundingMode.HALF_UP);

return new DebtRepaymentAmountVO(
    actualAmount, 
    preDiscountAmount, 
    freezeAmountFormatted, 
    tobeFreezeAmtFormatted, 
    serviceFee, 
    wallet
);
```

**优点**：
- 从源头统一格式
- 所有下游系统都能获得规范格式
- 最彻底的解决方案

---

### 修复2：SQL查询格式化

**文件**：`temp/清欠还款金额计算-简洁版.sql`

```sql
-- ✅ 使用 FORMAT 或 CAST
SELECT 
  FORMAT(CEILING((ABS(tw.BALANCE) - tw.FREEZE_AMOUNT - tw.TOBE_FREEZE_AMT) * @rate / 100 * 100) / 100, 2) as 'S手续费',
  FORMAT(ABS(tw.BALANCE) + CEILING(...) / 100, 2) as 'P总欠款',
  ...

-- 或使用 CAST
SELECT 
  CAST(CEILING(...) / 100 AS DECIMAL(11,2)) as 'S手续费',
  ...
```

**优点**：
- 查询结果直接规范
- 便于验证和调试

---

### 修复3：VO对象格式化

**新增工具类**：`MoneyFormatUtil.java`

```java
public class MoneyFormatUtil {
    /**
     * 格式化金额为2位小数
     * @param amount 金额
     * @return 格式化后的金额
     */
    public static BigDecimal format(BigDecimal amount) {
        if (amount == null) {
            return BigDecimal.ZERO.setScale(2, RoundingMode.HALF_UP);
        }
        return amount.setScale(2, RoundingMode.HALF_UP);
    }
    
    /**
     * 格式化金额为字符串（保留2位小数）
     * @param amount 金额
     * @return 格式化后的字符串
     */
    public static String formatString(BigDecimal amount) {
        return format(amount).toPlainString();
    }
}
```

**在VO中使用**：

```java
public class DebtRepaymentAmountVO {
    private BigDecimal actualAmount;
    
    // Getter 中格式化
    public BigDecimal getActualAmount() {
        return MoneyFormatUtil.format(actualAmount);
    }
    
    // 或提供格式化的字符串方法
    public String getActualAmountStr() {
        return MoneyFormatUtil.formatString(actualAmount);
    }
}
```

---

### 修复4：前端统一格式化

**新增格式化函数**：

```javascript
// utils/money.js
export function formatMoney(amount) {
  if (!amount && amount !== 0) return '0.00';
  return Number(amount).toFixed(2);
}

// 使用
import { formatMoney } from '@/utils/money';

<div>总欠款：{formatMoney(data.preDiscountAmount)} 元</div>
<div>实际支付：{formatMoney(data.actualAmount)} 元</div>
```

---

### 修复优先级

| 优先级 | 修复位置 | 影响范围 | 推荐度 |
|--------|---------|---------|--------|
| ⭐⭐⭐⭐⭐ | Java后端 | 全局 | ✅ 最推荐 |
| ⭐⭐⭐⭐ | VO对象 | 接口返回 | ✅ 推荐 |
| ⭐⭐⭐ | SQL查询 | 查询结果 | 可选 |
| ⭐⭐ | 前端展示 | 页面显示 | 补充 |

**建议**：
1. **必须做**：Java 后端统一格式化（修复1）
2. **建议做**：VO 对象格式化（修复3）
3. **可选做**：SQL 查询格式化（修复2）
4. **兜底做**：前端统一格式化（修复4）

---

## 【测试验证】

### 测试用例1：整数金额

```
余额：-1000
冻结：1000
待冻结：0

预期显示：
- 余额：-1000.00
- 冻结：1000.00
- 待冻结：0.00
```

### 测试用例2：一位小数

```
余额：-1000.5
冻结：100.5
待冻结：50.5

预期显示：
- 余额：-1000.50
- 冻结：100.50
- 待冻结：50.50
```

### 测试用例3：两位小数

```
余额：-2500.50
冻结：100.00
待冻结：100.00

计算结果：
- 需还款：2300.50
- 手续费：138.03
- 总欠款：2638.53
- 实际支付：2438.53

预期：所有金额都是2位小数
```

### 测试用例4：涉及向上取整

```
余额：-2500.55
冻结：100.00
待冻结：100.00

计算：
需还款 = 2300.55
手续费 = 2300.55 × 0.06 = 138.033 → 向上取整 = 138.04
总欠款 = 2500.55 + 138.04 = 2638.59
实际支付 = 2300.55 + 138.04 = 2438.59

预期：所有金额都是2位小数
```

---

## 【影响范围】

### 涉及的接口
- `com.hcb.getDebtRepaymentAmount` - 清欠还款金额查询
- `com.hcb.debtRepaymentWechatPay` - 清欠微信还款
- 其他涉及金额计算的接口

### 涉及的表
- `hcb_hlj_truckuserwallet` - 货车钱包
- `hcb_etccardbalanceflow` - 余额流水
- `hcb_topuprecord` - 充值记录

### 涉及的页面
- 清欠还款页面
- 账单详情页面
- 充值记录页面
- 钱包余额页面

---

## 【备注】

### 财务金额规范说明

1. **所有金额必须精确到分**：小数点后2位
2. **整数也要显示小数**：100 显示为 100.00
3. **末尾零不能省略**：100.5 显示为 100.50
4. **千分位可选**：1000.00 或 1,000.00（根据需求）

### 相关规范
- 《人民币现金管理暂行条例》
- 《会计法》金额规范
- 《电子支付业务规范》
