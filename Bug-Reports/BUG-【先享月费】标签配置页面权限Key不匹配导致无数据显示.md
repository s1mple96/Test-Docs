# BUG-【先享月费】标签配置页面权限Key不匹配导致无数据显示

## 【问题描述】
标签配置管理页面（`/hcbadmin/labelconfigmanage/list.do`）在重新构建部署后，页面显示"共14条"记录，但表格中无数据显示。用户拥有完整权限，数据库中确实存在14条标签配置记录。

## 【复现步骤】
1. 使用`test`用户登录后台系统（角色：测试专用，拥有所有权限）
2. 访问菜单：信息管理 → 标签配置管理
3. 观察页面显示

## 【实际结果】
- 页面底部显示"共14条"
- 表格中无数据显示（`<tbody>`为空）
- 浏览器F12查看HTML，确认数据未渲染

## 【预期结果】
- 表格中应显示14条标签配置数据
- 可以正常查看、编辑、删除标签

## 【根本原因】
权限Key命名不一致导致JSP权限检查失败：

### 1. 登录时存储权限（LoginController.java:513）
```java
public Map<String, String> getUQX(String USERNAME) {
    ...
    map.put("adds", pd.getString("ADD_QX"));
    map.put("dels", pd.getString("DEL_QX"));
    map.put("edits", pd.getString("EDIT_QX"));
    map.put("chas", pd.getString("CHA_QX"));    // ❌ 注意：key是"chas"（复数）
    map.put("toExcels", pd.getString("TOEXCEL_QX"));
    ...
}
```

### 2. JSP页面检查权限（labelconfigmanage_list.jsp:95）
```jsp
<c:if test="${QX.cha == 1 }">  <!-- ❌ 检查的是"cha"（单数） -->
    <c:forEach items="${varList}" var="var" varStatus="vs">
        <!-- 渲染数据 -->
    </c:forEach>
</c:if>
```

### 3. 为什么之前没问题？
- 其他页面的Controller会调用`Jurisdiction.buttonJurisdiction`方法
- 该方法触发`Jurisdiction.readMenu`，将"chas"转换为"cha"并存入Session
- **但是**：`LabelConfigManageController.list`方法的权限校验被注释掉了（第83行）：
  ```java
  //if(!Jurisdiction.buttonJurisdiction(MENU_URL, "cha")){return null;}  // 被注释！
  ```
- 导致没有触发权限转换，Session中只有"chas"，没有"cha"
- JSP检查`${QX.cha == 1}`失败（因为`QX.cha`为null）
- 数据不渲染

## 【影响范围】
- 标签配置管理页面无法正常使用
- 可能影响其他被注释掉权限校验的页面

## 【修复建议】

### 方案1：统一权限Key命名（推荐）
**文件**：`hcbadmin/src/main/java/com/hcb/controller/system/login/LoginController.java`

**修改**：`getUQX`方法（第510-514行）
```java
map.put("adds", pd.getString("ADD_QX"));
map.put("dels", pd.getString("DEL_QX"));
map.put("edits", pd.getString("EDIT_QX"));
map.put("chas", pd.getString("CHA_QX"));
map.put("toExcels", pd.getString("TOEXCEL_QX"));

// 添加兼容性：同时设置单数形式的key
map.put("add", pd.getString("ADD_QX"));
map.put("del", pd.getString("DEL_QX"));
map.put("edit", pd.getString("EDIT_QX"));
map.put("cha", pd.getString("CHA_QX"));      // ✅ 添加这行
map.put("toExcel", pd.getString("TOEXCEL_QX"));
```

**优点**：
- 一次性解决所有类似问题
- 向后兼容，不影响现有功能
- 无需修改JSP文件

### 方案2：恢复权限校验
**文件**：`hcbadmin/src/main/java/com/hcb/controller/information/labelconfigmanage/LabelConfigManageController.java`

**修改**：第83行
```java
// 原代码（被注释）
//if(!Jurisdiction.buttonJurisdiction(MENU_URL, "cha")){return null;}

// 修改为（取消注释）
if(!Jurisdiction.buttonJurisdiction(MENU_URL, "cha")){return null;}  // ✅ 取消注释
```

**优点**：
- 符合系统设计规范
- 触发完整的权限校验和转换流程

**缺点**：
- 只修复当前页面，可能还有其他页面存在类似问题

### 方案3：修改JSP（临时方案，不推荐）
**文件**：`hcbadmin/src/main/webapp/WEB-INF/jsp/information/labelconfigmanage/labelconfigmanage_list.jsp`

**修改**：第57、95、161行
```jsp
<!-- 原代码 -->
<c:if test="${QX.cha == 1 }">

<!-- 修改为 -->
<c:if test="${not empty QX.chas}">
```

**缺点**：
- 治标不治本
- 需要逐个修改所有受影响的JSP文件

## 【验证SQL】
```sql
-- 确认数据库中有标签配置数据
SELECT COUNT(*) as total FROM hcb_label_config_manage;
-- 结果：14

-- 确认test用户的查询权限
SELECT 
    u.USERNAME,
    r.ROLE_NAME,
    r.CHA_QX
FROM sys_user u
LEFT JOIN sys_role r ON u.ROLE_ID = r.ROLE_ID
WHERE u.USERNAME = 'test';
-- 结果：CHA_QX字段有值（长字符串）
```

## 【日志记录&截图标注】
无相关错误日志，因为这是前端渲染逻辑问题，后端未报错。

**浏览器F12 Elements面板**：
```html
<tbody>
<!-- 空的，没有<tr>元素 -->
</tbody>
```

**页面底部显示**：
```
共14条
```

## 【优先级】
**P1 - 高优先级**
- 影响核心功能使用
- 用户无法管理标签配置
- 修复简单，建议尽快处理

## 【测试环境】
- 服务器：test-hcb-admin-server
- 用户：test
- 角色：测试专用
- 浏览器：Chrome 141.0.0.0

