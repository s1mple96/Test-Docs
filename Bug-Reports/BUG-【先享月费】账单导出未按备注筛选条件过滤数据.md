# BUG-【先享月费】账单导出未按备注筛选条件过滤数据

## 【BUG描述】
先享月费账单页面按备注筛选后点击导出，导出的Excel数据未按备注条件过滤，导出了全部数据。

## 【前置条件】
- 环境：任货行后台管理系统
- 账单表存在多条不同备注的数据（共8条）
- 其中备注为"钱包不存在"的只有1条

## 【复现步骤】
1. 登录任货行后台
2. 进入先享月费账单页面
3. 选择备注筛选条件为"钱包不存在"
4. 点击导出
5. 下载Excel查看数据条数

## 【期望结果】
导出的Excel应只包含备注为"钱包不存在"的1条记录

## 【实际结果】
导出的Excel包含全部8条记录，未按备注条件过滤

## 【附件】
请求示例：
```bash
curl -X POST 'http://xxx/hcbadmin/monthlyPaylaterBill/excel.do' \
  --data-urlencode 'remark=钱包不存在'
```

## 【原因定位】
`MonthlyPaylaterWalletBillMapper.xml` 中 `listAll` 查询（导出使用）缺少 remark 筛选条件。

datalistPage（列表查询）有的筛选条件：
- createDateStart/End、idCode、carNum、billStatus、operatorCode、paySuccessTime、refundTime

listAll（导出查询）同样只有以上条件，**缺少 remark 筛选**。

问题代码位置：
`java/hcb/hcbadmin/src/main/resources/mybatis1/monthly/MonthlyPaylaterWalletBillMapper.xml` 第75-112行

## 【修复建议】
在 `listAll` 的 where 条件中添加 remark 筛选：

```xml
<if test="remark != null and remark == 'EMPTY' ">
    and (remark IS NULL OR remark = '')
</if>
<if test="remark != null and remark != '' and remark != 'EMPTY' ">
    and remark = #{remark}
</if>
```

