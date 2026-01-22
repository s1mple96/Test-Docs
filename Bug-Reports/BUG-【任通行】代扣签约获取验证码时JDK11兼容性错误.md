# BUG-【任通行】代扣签约获取验证码时JDK11兼容性错误

## 【BUG描述】
任通行小程序在代扣签约页面,当银行卡被系统识别为**平安快捷支付渠道**(PING_AN_QUICK_PAY)后,点击"获取验证码"按钮时,系统抛出JDK兼容性异常,导致无法发送短信验证码,签约流程中断。该问题影响所有使用平安支付渠道的用户签约功能。

## 【前置条件】
1. **测试环境**: 任通行测试环境
2. **测试账号**: 骆志敏 (身份证: 441622199602075195)
3. **银行卡信息**: 
   - 银行卡号: 6226630407949360 (光大银行)
   - 持卡人: 骆志敏
   - 预留手机号: 15818376788
   - 支付渠道: 平安快捷支付 (PING_AN_QUICK_PAY)
4. **系统环境**: 
   - rtx-app服务运行在JDK 11环境
   - 代码编译使用JDK 8

## 【复现步骤】
1. 打开任通行小程序
2. 进入"申办ETC"或"新增银行卡"页面
3. 选择"代扣签约"方式
4. 填写银行卡信息:
   - 银行卡号: 6226630407949360 (光大银行)
   - 身份证号: 441622199602075195
   - 持卡人: 骆志敏
   - 手机号: 15818376788
   - 系统识别为**平安快捷支付渠道** (PING_AN_QUICK_PAY)
5. 点击"获取验证码"按钮
6. 系统弹出错误提示

## 【期望结果】
1. 系统成功调用平安银行接口发送短信验证码
2. 提示"验证码已发送至手机 139****3461"
3. 验证码输入框可用
4. 用户可以继续完成签约流程

## 【实际结果】
1. 系统抛出异常: `java.lang.NoClassDefFoundError: sun/misc/BASE64Encoder`
2. 小程序提示"服务器异常,请联系系统管理员"或"签约失败"
3. 验证码发送失败
4. 无法继续签约流程

## 【附件】

### 错误日志
```
java.lang.NoClassDefFoundError: sun/misc/BASE64Encoder
    at com.rtx.common.utils.bank.pingan.encrypt.OpenApiAESKeyCreator.createAESKey(OpenApiAESKeyCreator.java:57)
    at com.rtx.project.service.impl.UserQuickBizServiceImpl.applySignSendSms(UserQuickBizServiceImpl.java:71)
    at com.rtx.app.controller.EquipmentApplyController.submitIdentityWithBankSign(EquipmentApplyController.java:2859)
    ...
Caused by: java.lang.ClassNotFoundException: sun.misc.BASE64Encoder
```

### 请求参数
```json
{
  "bankCardType": "01",
  "bankCardNo": "6226630407949360",
  "bankName": "光大银行",
  "bankCode": "CEB",
  "bankChannelCode": "PING_AN_QUICK_PAY",
  "phoneNumber": "15818376788",
  "holderName": "骆志敏",
  "withholdIdCode": "441622199602075195",
  "cardExpiredDate": "",
  "bankCardInfoId": "1440968918956118016",
  "cardCvv2": ""
}
```

### 日志路径
`/home/java-server/rtx-app/log.txt`

## 【原因定位】

### 1. 代码层面
**问题代码**:
- **文件**: `e:\Fenmi-Test\java\rtx\rtx-common\src\main\java\com\rtx\common\utils\bank\pingan\encrypt\OpenApiAESKeyCreator.java`
- **行号**: 第57行
- **代码**:
```java
return (new sun.misc.BASE64Encoder()).encode(src);
```

### 2. 技术原因
- `sun.misc.BASE64Encoder` 是JDK 8及以下版本的**内部API**
- JDK 9开始已**标记为废弃**,JDK 11完全**移除**该类
- 代码编译时使用JDK 8可以通过,但运行时使用JDK 11找不到该类

### 3. 环境验证
通过SSH查看实际运行环境:
```bash
ps aux | grep rtx-app | grep -v grep
# 确认运行命令中没有指定JDK路径,使用系统默认JDK 11
```

### 4. 调用链
```
用户点击"获取验证码"
  ↓
EquipmentApplyController.submitIdentityWithBankSign() (line 2859)
  ↓
UserQuickBizServiceImpl.applySignSendSms() (line 71)
  ↓
OpenApiAESKeyCreator.createAESKey() (line 57)
  ↓
调用 sun.misc.BASE64Encoder (JDK 11中不存在)
  ↓
抛出 NoClassDefFoundError
```

## 【修复建议】

### 方案1: 使用JDK标准API (推荐)
**修改文件**: `OpenApiAESKeyCreator.java` 第57行

```java
// 修改前 (JDK 8内部API - 不兼容JDK 11+)
return (new sun.misc.BASE64Encoder()).encode(src);

// 修改后 (JDK 8+标准API - 全版本兼容)
return java.util.Base64.getEncoder().encodeToString(src);
```

**优点**:
- ✅ JDK 8/11/17+ 全版本兼容
- ✅ 官方推荐标准API
- ✅ 无需引入第三方依赖
- ✅ 代码改动最小

### 方案2: 修改运行环境 (临时方案)
修改rtx-app启动脚本,指定使用JDK 8:
```bash
# 修改启动命令,使用JDK 8路径
/home/soft/jdk1.8.0_171/bin/java -jar rtx-app-server.jar
```

**缺点**:
- ❌ 治标不治本
- ❌ 未来JDK升级仍会遇到问题
- ❌ 不符合技术演进方向

### 推荐方案: 方案1
使用标准API替换,一劳永逸解决兼容性问题。

### 回归测试重点
1. ✅ 平安银行发送短信验证码
2. ✅ 平安银行签约验证
3. ✅ 平安银行快捷支付
4. ✅ 其他银行签约功能(确保不受影响)

### 相关系统
- **任货行(hcbapi)**: 使用相同代码,目前运行JDK 8暂未触发,建议同步修复避免未来升级时出现同样问题

### 参考资料
- JDK迁移指南: https://docs.oracle.com/javase/9/migrate/
- Base64 API文档: https://docs.oracle.com/javase/8/docs/api/java/util/Base64.html

