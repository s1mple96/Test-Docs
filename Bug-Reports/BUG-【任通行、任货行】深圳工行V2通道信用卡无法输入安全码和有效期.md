# BUG-【任通行、任货行】深圳工行V2通道信用卡无法输入安全码和有效期

## 【BUG描述】
深圳工行V2通道(`SZ_ICBC_QUICK_PAY_V2`)绑定信用卡时,前端未显示安全码(CVV2)和有效期输入框,导致信用卡无法完成签约

## 【前置条件】
- 测试环境: 任通行小程序
- 测试页面: 办理ETC页面 / 代扣签约页面
- 测试卡: 建设银行信用卡 `6259651812710403655`

## 【复现步骤】
1. 进入办理ETC页面
2. 输入建设银行信用卡卡号: `6259651812710403655`
3. 后端返回:
   ```json
   {
     "bankChannelCode": "SZ_ICBC_QUICK_PAY_V2",
     "cardType": "02"
   }
   ```
4. 观察页面表单
5. 点击"发送验证码"

## 【期望结果】
1. 页面应该显示信用卡专属字段:
   - 卡背面安全码(CVV2)输入框
   - 有效期(MM/YY)输入框
2. 用户填写完整信息后才能发送验证码
3. 信用卡签约流程正常

## 【实际结果】
1. 页面未显示安全码和有效期输入框
2. 直接显示储蓄卡的表单
3. 提交时缺少信用卡必填字段,导致签约失败或报错

## 【附件】
**前端代码** (`fm-bank-form.vue`):
```javascript
showCreditCard: function() {
  return "02" == this.bankCardType && 
         this.bankForm.bankChannelCode === s.ICBC_QUICK_PAY  // 只判断旧版通道
}
```

**需求文档** (`[需求]任通行、任货行深圳工行代扣签约接口迭代及协议展示.html` 第18行):
```
注:此版本工行快捷支付通道暂只支持工行储蓄卡(银行编码:ICBC)和非工行银联储蓄卡
```
> 注: 需求只说明支持储蓄卡,未明确禁止信用卡

**测试数据**:
- 建行信用卡: `6259651812710403655`
- 返回通道: `SZ_ICBC_QUICK_PAY_V2`
- 卡类型: `02`

## 【原因定位】
前端代码只判断了旧版工行通道(`ICBC_QUICK_PAY`),未包含V2通道(`SZ_ICBC_QUICK_PAY_V2`)

**代码位置**: `moduleApplyEtc/components/fm-bank-form/fm-bank-form.vue`

```javascript
computed: {
  showCreditCard: function() {
    // 只判断旧版通道,缺少V2通道判断
    return "02" == this.bankCardType && 
           this.bankForm.bankChannelCode === s.ICBC_QUICK_PAY
  }
}
```

**常量定义** (`704.js`):
```javascript
ICBC_QUICK_PAY: "PING_AN_QUICK_PAY"  // 旧版通道常量
// 缺少: SZ_ICBC_QUICK_PAY_V2 的常量定义
```

## 【修复建议】
前端修改 `fm-bank-form.vue`:

```javascript
computed: {
  showCreditCard: function() {
    // 支持旧版和V2通道的信用卡
    const supportCreditCardChannels = [
      s.ICBC_QUICK_PAY,           // 旧版工行通道
      'SZ_ICBC_QUICK_PAY_V2'      // 新版深圳工行V2通道
    ];
    
    return "02" == this.bankCardType && 
           supportCreditCardChannels.includes(this.bankForm.bankChannelCode);
  }
}
```

**或者更通用的方案**:
```javascript
computed: {
  showCreditCard: function() {
    // 只要是信用卡就显示安全码字段,让后端校验是否支持
    return "02" == this.bankCardType;
  }
}
```

