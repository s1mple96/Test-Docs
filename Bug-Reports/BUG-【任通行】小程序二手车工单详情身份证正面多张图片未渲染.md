# BUG-【任通行】小程序二手车工单详情身份证正面多张图片未渲染

## 【BUG描述】
客车小程序查看二手车工单详情时，当身份证正面照片字段（`idCardPicture`）包含2张图片时，小程序只显示第一张或不显示，而H5页面能正常展示所有图片。

## 【前置条件】
- 测试环境：http://788360p9o5.yicp.fun
- 测试账号：任意已登录小程序的用户
- 测试数据：
  - 二手车工单ID：存在身份证正面照片包含多张图片的工单
  - 数据示例：
    ```json
    "idCardPicture": "https://huochebaoimg.oss-cn-shenzhen.aliyuncs.com/images/rtxfiles/2026/01/07/df6d2f851f1f4853958b084795070950.jpg,https://huochebaoimg.oss-cn-shenzhen.aliyuncs.com/images/rtxfiles/2026/01/07/d758778e69d045c0af31e3c949665e8e.jpg"
    ```

## 【复现步骤】
1. 登录任通行客车小程序
2. 进入【服务大厅】-【工单管理】-【工单列表】
3. 找到二手车订单（`business_type='3'`）
4. 点击【查看详情】
5. 查看身份证正面照片区域

## 【期望结果】
- 身份证正面照片区域应显示2张图片（逗号分隔）
- 两张图片都能正常加载和预览
- 与H5页面展示效果一致

## 【实际结果】
- 小程序只显示第一张图片，第二张图片未渲染
- 或小程序身份证正面照片区域无法正常显示
- H5页面能正常展示2张图片

## 【附件】
### 数据库字段内容
```sql
SELECT id, phone, car_num, id_card_picture 
FROM rtx_customer_used_car_order 
WHERE id_card_picture LIKE '%,%';

-- 示例数据：
-- idCardPicture: "https://huochebaoimg.oss-cn-shenzhen.aliyuncs.com/images/rtxfiles/2026/01/07/df6d2f851f1f4853958b084795070950.jpg,https://huochebaoimg.oss-cn-shenzhen.aliyuncs.com/images/rtxfiles/2026/01/07/d758778e69d045c0af31e3c949665e8e.jpg"
```

### 对比情况
- **H5页面**：正常展示2张图片
- **小程序**：只展示1张或不展示

## 【原因定位】
小程序前端渲染逻辑未正确处理 `idCardPicture` 字段中逗号分隔的多张图片URL：

**可能原因**：
1. 小程序前端未对 `idCardPicture` 字段做 `split(',')` 处理
2. 图片组件只绑定了字符串，而非数组
3. H5前端正确处理了逗号分隔的图片URL，而小程序未做相同处理

**其他图片字段对比**：
- `idCardBackUrl`（身份证反面）：单张图片，无逗号分隔，显示正常
- `drivingLicensePhotos`（行驶证照片）：逗号分隔多张图片，需验证是否有同样问题

## 【修复建议】
1. **小程序前端修改**：
   - 在渲染 `idCardPicture` 时，先判断是否包含逗号
   - 如果包含逗号，使用 `split(',')` 分割成数组
   - 循环渲染所有图片

2. **参考H5实现**：
   ```javascript
   // 伪代码示例
   const idCardPictures = idCardPicture ? idCardPicture.split(',') : [];
   // 循环渲染 idCardPictures 数组
   ```

3. **统一处理逻辑**：
   - 检查其他多图片字段（如 `drivingLicensePhotos`、`carContractPhotos`）是否有同样问题
   - 统一多图片字段的渲染逻辑

4. **建议排查范围**：
   - 小程序工单详情页组件：查看图片渲染逻辑
   - 对比H5工单详情页实现
   - 确认所有逗号分隔的图片字段都能正确渲染



