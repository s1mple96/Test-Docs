# BUG-【任通行】通行费多扣退款申请图片上传后缺少预览删除功能

## 【BUG描述】
通行费多扣退款申请弹窗中，"退回账户信息"区域的图片上传组件，上传图片后缺少预览和删除功能，用户无法查看已上传图片或删除错误上传的图片。

## 【前置条件】
- 环境：测试环境
- 账号：有`bpmProcess:audit:tollRefund`权限的账号
- 车辆：客车类型车辆（如：粤A113355）
- 路径：任通行管理系统 → 客服系统 → 客户信息 → 业务流程申请单 → 通行费多扣退款申请

## 【复现步骤】
1. 登录任通行管理系统
2. 进入"客服系统 → 客户信息 → 业务流程申请单"
3. 选择一个客车车辆（如：粤A113355）
4. 点击"通行费多扣退款申请"按钮
5. 在"退回账户信息"区域，找到"图片上传"组件
6. 点击上传一张图片
7. 观察图片上传后的显示状态

## 【期望结果】
图片上传成功后，应该：
1. 显示缩略图预览
2. 鼠标悬停时显示操作按钮：
   - 预览按钮（👁 图标）：点击可放大查看图片
   - 删除按钮（🗑 图标）：点击可删除该图片
3. 可以重新上传或更换图片

## 【实际结果】
图片上传成功后：
- 显示了图片缩略图
- **缺少预览按钮**：无法点击放大查看图片
- **缺少删除按钮**：无法删除已上传的图片
- 如果上传错误，只能刷新页面重新操作

## 【附件】
截图显示：
- 弹窗标题："通行费多扣退款"
- "退回账户信息"区域有"图片上传"组件
- 图片上传后只显示缩略图，无预览/删除图标

## 【原因定位】
前端上传组件未配置`list-type`和相关事件处理：

**当前配置（推测）**：
```vue
<el-upload
  :on-success="handleSuccess"
  :show-file-list="false"
>
  <!-- 未配置预览和删除功能 -->
</el-upload>
```

**应该配置**：
```vue
<el-upload
  :file-list="fileList"
  list-type="picture-card"
  :on-preview="handlePreview"
  :on-remove="handleRemove"
  :on-success="handleSuccess"
>
  <i class="el-icon-plus"></i>
</el-upload>

<!-- 预览对话框 -->
<el-dialog :visible.sync="dialogVisible">
  <img width="100%" :src="dialogImageUrl" alt="">
</el-dialog>
```

## 【修复建议】

### 1. 修改上传组件配置
```vue
<template>
  <el-upload
    action="/api/upload"
    :file-list="imageList"
    list-type="picture-card"
    :limit="1"
    :on-preview="handlePreview"
    :on-remove="handleRemove"
    :on-success="handleSuccess"
  >
    <i class="el-icon-plus"></i>
  </el-upload>
  
  <!-- 图片预览对话框 -->
  <el-dialog :visible.sync="previewVisible" append-to-body>
    <img width="100%" :src="previewUrl" />
  </el-dialog>
</template>

<script>
export default {
  data() {
    return {
      imageList: [],
      previewVisible: false,
      previewUrl: ''
    }
  },
  methods: {
    // 预览图片
    handlePreview(file) {
      this.previewUrl = file.url;
      this.previewVisible = true;
    },
    // 删除图片
    handleRemove(file, fileList) {
      this.imageList = fileList;
      this.$message.success('删除成功');
    },
    // 上传成功
    handleSuccess(response, file, fileList) {
      this.imageList = fileList;
      this.$message.success('上传成功');
    }
  }
}
</script>
```

### 2. 或使用自定义样式
```vue
<template>
  <div class="image-upload">
    <el-upload
      :show-file-list="false"
      :on-success="handleSuccess"
    >
      <img v-if="imageUrl" :src="imageUrl" class="avatar">
      <i v-else class="el-icon-plus avatar-uploader-icon"></i>
    </el-upload>
    
    <!-- 自定义操作按钮 -->
    <div v-if="imageUrl" class="image-actions">
      <i class="el-icon-zoom-in" @click="handlePreview"></i>
      <i class="el-icon-delete" @click="handleRemove"></i>
    </div>
  </div>
</template>
```

### 建议
1. 使用`list-type="picture-card"`展示缩略图
2. 配置`on-preview`事件实现预览功能
3. 配置`on-remove`事件实现删除功能
4. 限制上传数量`:limit="1"`（如果只需一张图片）

