# BUG-【先享月费】流水导出运营商为空及导出后页面运营商丢失

## 【BUG描述】
先享月费流水页面存在两个问题：
1. 导出Excel文件中"所属运营商"字段内容为空
2. 导出操作后，页面列表"所属运营商"列变空，运营商下拉筛选框也无选项

## 【前置条件】
- 环境：任货行后台管理系统
- 页面：先享月费流水页面
- 流水数据有关联用户，用户表有运营商数据

## 【复现步骤】
1. 登录任货行后台
2. 进入先享月费流水页面
3. 选择运营商筛选条件为"龙通卡(LTK)"
4. 页面列表正常显示"所属运营商"为"龙通卡"
5. 点击导出
6. 下载Excel查看"所属运营商"列
7. 观察页面列表和运营商下拉框

## 【期望结果】
1. 导出Excel的"所属运营商"列应显示"龙通卡"
2. 导出后页面列表"所属运营商"仍显示"龙通卡"
3. 导出后运营商下拉筛选框仍有选项可选

## 【实际结果】
1. 导出Excel的"所属运营商"列为空
2. 导出后页面列表"所属运营商"列变空
3. 导出后运营商下拉筛选框无选项

## 【附件】
请求示例：
```bash
curl -X POST 'http://xxx/hcbadmin/monthlyPaylaterWalletFlow/excel.do' \
  --data-urlencode 'operatorCode=LTK'
```

## 【原因定位】
`MonthlyPaylaterWalletFlowController.java` 的 `exportExcel` 方法存在两处遗漏：

**问题1：导出数据未转换运营商名称**
第106-109行只转换了bizTypeName和fundDirection，缺少operatorCodeName转换：
```java
for (MonthlyPaylaterWalletFlow flow : reportList) {
    flow.setBizTypeName(MonthlyFlowTypeEnum.getNameByCode(flow.getBizType()));
    flow.setFundDirection(WalletFundDirectionEnum.getNameByCode(flow.getFundDirection()));
    // 缺少: flow.setOperatorCodeName(OperatorCode.getName(flow.getOperatorCode()));
}
```

**问题2：返回页面缺少运营商下拉列表和列表数据转换**
第131-136行缺少operatorCodeList和varList的operatorCodeName转换：
```java
mv.addObject("varList", varList);
mv.addObject("pd", pd);
mv.setViewName("truck/monthly/monthly_paylater_wallet_flow_list");
mv.addObject("bizTypeList", MonthlyFlowTypeEnum.listOperatorType());
mv.addObject("QX", Jurisdiction.getHC());
// 缺少: mv.addObject("operatorCodeList", OperatorCode.list());
// 缺少: varList循环设置operatorCodeName
```

对比list方法第64行和第69行有这些代码。

## 【修复建议】
在exportExcel方法中补充：

1. 导出数据转换（第106-109行循环内添加）：
```java
flow.setOperatorCodeName(OperatorCode.getName(flow.getOperatorCode()));
```

2. 返回页面数据（第127-136行）：
```java
varList = monthlyPaylaterWalletFlowService.list(page);
for (MonthlyPaylaterWalletFlow flow : varList) {
    flow.setBizTypeName(MonthlyFlowTypeEnum.getNameByCode(flow.getBizType()));
    flow.setFundDirection(WalletFundDirectionEnum.getNameByCode(flow.getFundDirection()));
    flow.setOperatorCodeName(OperatorCode.getName(flow.getOperatorCode()));
}
// ...
mv.addObject("operatorCodeList", OperatorCode.list());
```

