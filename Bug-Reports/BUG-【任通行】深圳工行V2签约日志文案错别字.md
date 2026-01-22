# BUG-【任通行】深圳工行V2签约日志文案错别字

## 基本信息
- **系统**: 任通行(rtx)
- **模块**: 银行签约
- **严重级别**: 低
- **发现日期**: 2026-01-09
- **影响范围**: 深圳工行V2通道签约日志文案

---

## BUG描述

任通行系统在调用深圳工行V2签约接口时,日志中的扩展信息(`extendInfo`)字段存在错别字:"甚至工行支付签约V2",正确应为"深圳工行支付签约V2"。

---

## 复现步骤

1. 任通行小程序进入银行卡签约页面
2. 使用深圳工行V2通道(`SZ_ICBC_QUICK_PAY_V2`)发送签约短信
3. 输入验证码完成签约
4. 查看MongoDB日志 `rtx.icbc_request_log` 或应用日志

---

## 实际结果

**日志内容**:
```json
{
  "cardNo": "6222034000046037473",
  "extendInfo": "甚至工行支付签约V2",  // ❌ 错别字
  "holderName": "骆志敏",
  "idNo": "441622199602075195",
  "idType": "0",
  "merSerFlag": "1",
  "merSerId": "400047840027",
  "merSerPrtclno": "4000478400270201",
  "mobileNo": "15818376788",
  "trxDate": "2025-12-25",
  "trxSerno": "2025122515594402",
  "trxTime": "15:59:44",
  "verifyCode": "238452",
  "verifyCodeNo": "2025122500002153"
}
```

---

## 预期结果

```json
{
  "extendInfo": "深圳工行支付签约V2"  // ✅ 正确
}
```

---

## 根因分析

**文件**: `java/rtx/rtx-common/src/main/java/com/rtx/common/utils/bank/icbc/shenzhen/NewSzIcbcBankApiUtils.java`  
**行号**: 497

**问题代码**:
```java
public static CardbusinessNcpayAgreementSignResponseV1VO noCardSignV2(
    CardbusinessNcpayAgreementSignRequestV1VO noCardSign) throws Exception {
    
    // ... 省略 ...
    
    CardbusinessNcpayAgreementSignRequestV1VO.CardbusinessNcpayAgreementSignRequestV1Biz bizContent = 
        new CardbusinessNcpayAgreementSignRequestV1VO.CardbusinessNcpayAgreementSignRequestV1Biz();
    
    // ... 省略 ...
    
    // line 497: 错别字 "甚至" ❌
    bizContent.setExtendInfo("甚至工行支付签约V2");
    
    // ... 省略 ...
}
```

**其他相关方法**:
```java
// sendSmsV2() 方法 - line 422 (正确)
bizContent.setExtendInfo("深圳工行支付短信V2");  // ✅

// noCardSignV1() 方法 - line 555 (正确)  
bizContent.setExtendInfo("深圳工行支付签约V1");  // ✅
```

---

## 修复建议

**修改文件**: `java/rtx/rtx-common/src/main/java/com/rtx/common/utils/bank/icbc/shenzhen/NewSzIcbcBankApiUtils.java`  
**修改行号**: 497

**修改前**:
```java
bizContent.setExtendInfo("甚至工行支付签约V2");  // ❌ 错别字
```

**修改后**:
```java
bizContent.setExtendInfo("深圳工行支付签约V2");  // ✅ 正确
```

---

## 影响范围

1. **日志可读性**: 影响日志的可读性和专业性
2. **问题排查**: 通过关键字搜索日志时,错别字可能导致搜索遗漏
3. **代码规范**: 影响代码质量和团队形象

---

## 相关日志

**MongoDB查询**:
```javascript
// 查询包含错别字的日志
db.icbc_request_log.find({
  "request": /甚至工行支付签约V2/
}).count()

// 修复后应查询不到
db.icbc_request_log.find({
  "request": /深圳工行支付签约V2/
}).count()
```

**示例日志**:
```json
{
  "_id": ObjectId("..."),
  "bizContentClass": "com.icbc.api.request.CardbusinessNcpayAgreementSignRequestV1$CardbusinessNcpayAgreementSignRequestV1Biz",
  "method": "POST",
  "serviceUrl": "https://gw.open.icbc.com.cn/api/cardbusiness/ncpay/agreement/sign/V2",
  "bizContent": {
    "extendInfo": "甚至工行支付签约V2"  // ❌
  },
  "createTime": ISODate("2026-01-09T...")
}
```

---

## 备注

- 错别字: "甚至" → "深圳" (拼音输入法误选)
- 该问题仅影响日志文案,不影响实际业务功能
- 建议全局搜索是否还有其他类似的错别字
- 修复后历史日志仍会包含错误文案,无需清理

