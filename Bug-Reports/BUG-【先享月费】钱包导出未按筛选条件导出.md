# BUG-【先享月费】钱包导出未按筛选条件导出

## 【问题描述】
先享月费钱包列表页面，设置筛选条件后点击"导出"按钮，导出的Excel文件未按照筛选条件导出数据，而是直接全量导出所有数据。

## 【请求参数】
```
接口：/hcbadmin/monthlyPaylaterWallet/excel.do
方法：POST

筛选条件：
- userName=测试用户14

实际情况：
导出了所有用户数据，未根据 userName=测试用户14 进行过滤
```

完整curl：
```bash
curl -X POST 'https://788360p9o5.yicp.fun/hcbadmin/monthlyPaylaterWallet/excel.do' \
  --data-urlencode 'walletId' \
  --data-urlencode 'idCode' \
  --data-urlencode 'userName=测试用户14' \
  --data-urlencode 'createDateStart' \
  --data-urlencode 'createDateEnd'
```

## 【复现步骤】
1. 登录后台系统（账号：test / 密码：a12345678）
2. 进入"钱包账户管理" → "先享月费钱包"
3. 在搜索条件中输入"用户名称=测试用户14"
4. 点击"搜索"按钮，列表只显示该用户的数据
5. 点击"导出"按钮
6. 下载并打开导出的Excel文件
7. 发现导出了所有用户的数据，不是只导出"测试用户14"的数据

## 【日志记录&截图标注】
1. 筛选条件：userName=测试用户14
2. 列表显示：只有1条"测试用户14"的数据
3. 导出结果：包含所有用户数据（30条）
4. 预期导出：只导出1条"测试用户14"的数据

## 【预期结果】
导出功能应该根据用户设置的筛选条件进行导出，只导出符合条件的数据，而不是全量导出所有数据。

