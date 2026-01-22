# BUG报告

## BUG标题
【任通行、任货行】JDK11兼容性问题 - sun.misc.BASE64Encoder类不存在

## BUG等级
🔴 **严重** - 导致平安银行签约功能完全不可用

## 发现时间
2026-01-08

## 影响范围
- **任通行(rtx-app)**: 平安银行快捷支付签约
- **任货行(hcbapi)**: 平安银行快捷支付签约 (潜在风险)

## BUG描述

### 现象
调用平安银行签约接口时,系统抛出异常:
```
java.lang.NoClassDefFoundError: sun/misc/BASE64Encoder
```

### 触发条件
1. 用户选择**平安银行卡**进行签约
2. 点击"发送验证码"或"签约"
3. 系统调用 `OpenApiAESKeyCreator.createAESKey()` 方法

### 错误堆栈
```
java.lang.NoClassDefFoundError: sun/misc/BASE64Encoder
    at com.rtx.common.utils.bank.pingan.encrypt.OpenApiAESKeyCreator.createAESKey(OpenApiAESKeyCreator.java:57)
    at com.rtx.app.service.impl.UserQuickBizServiceImpl.applySignSendSms(UserQuickBizServiceImpl.java:71)
    at com.rtx.app.controller.EquipmentApplyController.submitIdentityWithBankSign(EquipmentApplyController.java:2859)
```

## 根本原因

### 技术背景
- `sun.misc.BASE64Encoder` 是JDK 8及以下版本的**内部API**
- JDK 9+ 已**移除**该类
- 官方推荐使用 `java.util.Base64`

### 环境问题
1. **代码编译环境**: JDK 8 (代码可以编译通过)
2. **实际运行环境**: JDK 11 (运行时找不到该类)

**验证命令**:
```bash
# 查看实际运行的JDK版本
ps aux | grep rtx-app | grep -v grep
# 输出: -jar ... rtx-app-server.jar
# 实际使用的是系统默认JDK 11
```

## 问题代码

**文件**: `e:\Fenmi-Test\java\rtx\rtx-common\src\main\java\com\rtx\common\utils\bank\pingan\encrypt\OpenApiAESKeyCreator.java`

**行号**: 第57行

```java:57:57:e:\Fenmi-Test\java\rtx\rtx-common\src\main\java\com\rtx\common\utils\bank\pingan\encrypt\OpenApiAESKeyCreator.java
return (new sun.misc.BASE64Encoder()).encode(src);
```

## 解决方案

### 推荐方案
使用JDK 8+ 标准API `java.util.Base64` 替换:

```java
// 修改前 (JDK 8内部API)
return (new sun.misc.BASE64Encoder()).encode(src);

// 修改后 (JDK 8+标准API)
return java.util.Base64.getEncoder().encodeToString(src);
```

**优点**:
- ✅ JDK 8/11/17+ 全版本兼容
- ✅ 官方推荐API
- ✅ 无需引入第三方依赖

### 影响评估
- **修改范围**: 1个文件,1行代码
- **影响功能**: 平安银行签约AES密钥生成
- **测试重点**: 
  - 平安银行发送短信
  - 平安银行签约验证
  - 平安银行快捷支付

## 复现步骤
1. 部署 `rtx-app` 服务到 JDK 11 环境
2. 小程序选择平安银行卡
3. 点击"发送验证码"
4. 触发错误: `NoClassDefFoundError: sun/misc/BASE64Encoder`

## 测试建议
1. ✅ 本地JDK 8环境测试
2. ✅ 测试环境JDK 11测试
3. ✅ 验证签约全流程
4. ✅ 回归测试其他银行签约

## 相关信息
- **JDK迁移指南**: https://docs.oracle.com/javase/9/migrate/toc.htm#JSMIG-GUID-7744EF96-5899-4FB2-B34E-86D49B2E89B6
- **Base64 API文档**: https://docs.oracle.com/javase/8/docs/api/java/util/Base64.html

## 备注
- 该问题在RTX系统已确认
- HCB系统使用相同代码,但实际运行JDK 8暂未触发
- 建议同步修复HCB,避免未来JDK升级时出现同样问题




