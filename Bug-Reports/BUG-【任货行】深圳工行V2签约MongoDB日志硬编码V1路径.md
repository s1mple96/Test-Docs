# BUG-【任货行】深圳工行V2签约MongoDB日志硬编码V1路径

## 基本信息
- **系统**: 任货行(hcb)
- **模块**: 银行签约
- **严重级别**: 中
- **发现日期**: 2026-01-09
- **影响范围**: 深圳工行V2通道签约日志记录

---

## BUG描述

任货行系统在调用深圳工行V2签约接口(`sign/V2`)时,MongoDB日志中硬编码保存了错误的API路径`api/cardbusiness/ncpay/agreement/sign/V1`,导致日志记录的`serviceUrl`与实际调用的接口不一致。

---

## 复现步骤

1. 任货行小程序进入银行卡签约页面
2. 使用深圳工行V2通道(`SZ_ICBC_QUICK_PAY_V2`)发送签约短信
3. 输入验证码完成签约
4. 查看MongoDB日志 `jdbdb.icbc_request_log`

---

## 实际结果

**MongoDB日志**:
```json
{
  "serviceUrl": "api/cardbusiness/ncpay/agreement/sign/V1",  // ❌ 错误:硬编码V1
  "request": { ... },
  "response": { ... }
}
```

**实际调用的API**:
```
https://gw.open.icbc.com.cn/api/cardbusiness/ncpay/agreement/sign/V2
```

---

## 预期结果

MongoDB日志中的 `serviceUrl` 应该正确记录为 `api/cardbusiness/ncpay/agreement/sign/V2`

---

## 根因分析

**文件**: `java/hcb/hcbapi/src/main/java/com/hcb/api/util/bank/icbc/SzIcbcBankApiUtils.java`  
**行号**: 773

**问题代码**:
```java
public static CardbusinessNcpayAgreementSignResponseV1VO noCardSignV2(
    CardbusinessNcpayAgreementSignRequestV1VO noCardSign) throws Exception {
    // ... 省略 ...
    
    // line 735: 实际调用 V2 接口
    request.setServiceUrl(SzIcbcConstants.BASE_URI + "api/cardbusiness/ncpay/agreement/sign/V2");
    
    // ... 省略 ...
    
    // line 773: 保存日志时硬编码 V1 ❌
    saveLog(JSONObject.toJSONString(request), 
            JSONObject.toJSONString(bizContent), 
            JSONObject.toJSONString(response), 
            "api/cardbusiness/ncpay/agreement/sign/V1");  // ❌ 错误
    return response;
}
```

**对比V1方法** (正确):
```java
public static CardbusinessNcpayAgreementSignResponseV1VO noCardSign(
    CardbusinessNcpayAgreementSignRequestV1VO noCardSign) throws Exception {
    // ... 省略 ...
    
    // line 131: 调用 V1 接口
    request.setServiceUrl(SzIcbcConstants.BASE_URI + "api/cardbusiness/ncpay/agreement/sign/V1");
    
    // ... 省略 ...
    
    // line 169: 保存日志使用 V1 ✅
    saveLog(JSONObject.toJSONString(request), 
            JSONObject.toJSONString(bizContent), 
            JSONObject.toJSONString(response), 
            "api/cardbusiness/ncpay/agreement/sign/V1");  // ✅ 正确
    return response;
}
```

---

## 修复建议

**修改文件**: `java/hcb/hcbapi/src/main/java/com/hcb/api/util/bank/icbc/SzIcbcBankApiUtils.java`  
**修改行号**: 773

**修改前**:
```java
saveLog(JSONObject.toJSONString(request), 
        JSONObject.toJSONString(bizContent), 
        JSONObject.toJSONString(response), 
        "api/cardbusiness/ncpay/agreement/sign/V1");  // ❌
```

**修改后**:
```java
saveLog(JSONObject.toJSONString(request), 
        JSONObject.toJSONString(bizContent), 
        JSONObject.toJSONString(response), 
        "api/cardbusiness/ncpay/agreement/sign/V2");  // ✅
```

---

## 影响范围

1. **日志错误**: MongoDB日志记录的API路径与实际调用不符,影响问题排查
2. **数据统计**: 基于日志的API调用统计会混淆V1和V2的调用量
3. **问题定位**: 出现问题时,无法通过日志准确区分是V1还是V2接口的错误

---

## 相关日志

**MongoDB查询**:
```javascript
db.icbc_request_log.find({
  "serviceUrl": "api/cardbusiness/ncpay/agreement/sign/V1"
}).sort({_id: -1}).limit(5)
```

**示例日志**:
```json
{
  "_id": ObjectId("..."),
  "serviceUrl": "api/cardbusiness/ncpay/agreement/sign/V1",  // ❌ 错误标记
  "request": "...",
  "response": "...",
  "createTime": ISODate("2026-01-09T...")
}
```

---

## 备注

- 该问题仅影响日志记录,不影响实际业务功能
- 建议同时检查其他V2接口的日志保存逻辑,确保没有类似问题
- 修复后需清理历史错误日志,或在统计时排除错误标记的数据

