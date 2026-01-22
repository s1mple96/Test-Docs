# BUG-【任货行】黑名单导出缺少起诉状态字段

## 【BUG描述】
黑名单导出功能中，导出的Excel文件缺少"起诉状态"字段列，导致导出数据不完整，影响业务使用

## 【前置条件】
- 测试环境：http://788360p9o5.yicp.fun
- 测试账号：洛智明
- 测试数据：系统中存在黑名单用户，且部分用户有起诉状态数据

## 【复现步骤】
1. 登录系统，进入黑名单管理页面
2. 点击"导出黑名单"按钮
3. 等待导出完成，在下载管理页面下载导出的Excel文件
4. 打开Excel文件，检查表头字段

## 【期望结果】
导出的Excel文件应包含以下字段（包括但不限于）：
- 车牌号
- 用户姓名
- 手机号
- 身份证号
- 起诉状态（必须包含）
- 拉黑时间
- 欠路费金额
- 等其他字段

与黑名单列表页面显示的字段保持一致

## 【实际结果】
导出的Excel文件缺少"起诉状态"字段列，只有以下字段：
- 车牌号
- 用户姓名
- 所属渠道
- 渠道手机号
- 手机号
- 身份证号
- ETC卡号
- 用户状态
- 拉黑天数
- 拉黑时间
- 激活时间
- 拉黑状态
- 拉黑原因
- 黑名单次数
- 欠路费金额
- 已冻结金额
- 违约待冻结金额
- 钱包余额
- 起诉状本金
- 利息欠款

（注：缺少"起诉状态"列）

## 【附件】
无

## 【原因定位】

**1. 黑名单列表页面有查询起诉状态的逻辑：**
```java
// TruckUserController.java:566
var.put("SUE_STATUS", findby.getString("SUE_STATUS"));
```

**2. 导出VO类中缺少该字段定义：**
```java
// BlackUserInfoVO.java
// 该类中没有定义sueStatus相关字段和@ExcelProperty注解
```

**3. 导出转换方法中未赋值该字段：**
```java
// TruckUserController.java:632
.map(var -> truckUserBizService.toBlackUserInfoVO(var))

// TruckUserBizService.java:123-218
// toBlackUserInfoVO方法中没有设置sueStatus字段
```

**根本原因：**
- `BlackUserInfoVO` 类缺少 `sueStatus` 字段定义和 `@ExcelProperty` 注解
- `TruckUserBizService.toBlackUserInfoVO()` 方法未将 `SUE_STATUS` 字段赋值到VO对象
- 导出时未查询起诉状态数据

## 【修复建议】

**1. 在 `BlackUserInfoVO` 类中添加起诉状态字段：**
```java
/**
 * 起诉状态
 */
@ExcelProperty(value = "起诉状态")
private String sueStatus;
```

**2. 在导出时查询起诉状态数据（参考列表页面逻辑）：**
```java
// TruckUserController.java exportBlackList方法中
// 在varList循环中添加起诉状态查询
PageData findby = new PageData();
findby.put("TRUCKUSER_ID", var.getString("TRUCKUSER_ID"));
findby.put("FLAG", Constants.YES_FLAG);
List<PageData> debtcollectionServiceBy = debtcollectionService.findBy(findby);
if (!CollectionUtils.isEmpty(debtcollectionServiceBy)) {
    findby = debtcollectionServiceBy.get(0);
    var.put("SUE_STATUS", findby.getString("SUE_STATUS"));
}
```

**3. 在 `TruckUserBizService.toBlackUserInfoVO()` 方法中添加字段赋值逻辑：**
```java
// 在toBlackUserInfoVO方法末尾添加
blackUserInfo.setSueStatus(StringUtils.isBlank(truckUser.getString("SUE_STATUS")) ? "" : SueStatusEnum.getName(truckUser.getString("SUE_STATUS")));
```

**4. 验证修复：**
- 导出Excel后检查是否包含"起诉状态"列
- 验证起诉状态数据是否准确显示

