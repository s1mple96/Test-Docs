# BUG-【任通行】通行费多扣退款申请流水截图上传文件类型限制失效

## 【BUG描述】
通行费多扣退款申请弹窗中，"流水截图"上传组件提示"请上传png/jpg/jpeg格式文件"，但点击后文件选择对话框显示"所有文件(*.*)"，未限制文件类型，用户可以选择任意格式文件上传。

## 【前置条件】
- 环境：测试环境
- 账号：有`bpmProcess:audit:tollRefund`权限的账号
- 车辆：客车类型车辆（如：粤A113355）
- 路径：任通行管理系统 → 客服系统 → 客户信息 → 业务流程申请单

## 【复现步骤】
1. 登录任通行管理系统
2. 进入"客服系统 → 客户信息 → 业务流程申请单"
3. 选择一个客车车辆
4. 点击"通行费多扣退款申请"按钮
5. 在弹窗中找到"流水截图"上传组件（提示：请上传png/jpg/jpeg格式文件）
6. 点击上传区域
7. 观察文件选择对话框的"文件类型"下拉框

## 【期望结果】
文件选择对话框的"文件类型"下拉框应**仅显示**图片格式：
- PNG 文件 (*.png)
- JPG 文件 (*.jpg)
- JPEG 文件 (*.jpeg)

不应显示"所有文件(*.*)"选项，限制用户只能选择图片文件。

## 【实际结果】
文件选择对话框的"文件类型"下拉框显示：
- **所有文件 (*.*)**

文件类型限制失效，用户可以选择任意格式文件（如.txt、.pdf、.doc等），与提示文字不符。

## 【附件】
截图显示：
- 弹窗标题："通行费多扣退款"
- 上传区域提示："请上传 png/jpg/jpeg 格式文件"
- 文件选择对话框中"文件类型"仅有"所有文件 (*.*)"选项

## 【原因定位】
前端上传组件未设置`accept`属性或设置为`*.*`，导致文件类型限制失效：

**当前配置（推测）**：
```vue
<el-upload>
  <!-- 未设置accept属性，或设置为 accept="*.*" -->
</el-upload>
```

**应该配置**：
```vue
<el-upload accept=".png,.jpg,.jpeg">
  <!-- 限制只能选择png/jpg/jpeg格式 -->
</el-upload>
```

## 【修复建议】

### 方案1：修改前端上传组件配置
```vue
<el-upload
  :accept="'.png,.jpg,.jpeg'"
  :before-upload="beforeUpload"
>
  <div class="upload-area">
    <i class="el-icon-plus"></i>
    <div class="el-upload__text">请上传 png/jpg/jpeg 格式文件</div>
  </div>
</el-upload>
```

### 方案2：在beforeUpload中校验
```javascript
beforeUpload(file) {
  const isImage = ['image/png', 'image/jpeg', 'image/jpg'].includes(file.type);
  const isLt10M = file.size / 1024 / 1024 < 10;
  
  if (!isImage) {
    this.$message.error('只能上传 PNG/JPG/JPEG 格式的图片！');
    return false;
  }
  if (!isLt10M) {
    this.$message.error('图片大小不能超过 10MB！');
    return false;
  }
  return true;
}
```

### 建议同时修改
1. 设置`accept`属性限制文件类型选择
2. 在`beforeUpload`中进行二次校验，防止用户手动修改文件扩展名绕过限制

