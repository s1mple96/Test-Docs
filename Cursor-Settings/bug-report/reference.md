# BUG 报告编写 - 项目配置参考

## BUG 报告文件规范

### 文件保存位置

**必须保存到**: `temp/BUG报告/`

### 文件命名规范

**格式**: `BUG-【模块名】简短描述.md`

**示例**:
- `BUG-【任通行】快速缴费充值失败.md`
- `BUG-【任货行】黑名单导出缺少字段.md`
- `BUG-【新发行】退费打款校验异常.md`
- `BUG-【任货行定时任务】账单解析空指针.md`

## 系统与模块映射

| 系统 | 常见模块 | 数据库 |
|-----|---------|--------|
| 任通行 | 快速缴费、账单查询、补换卡、充值还款 | `rtx` |
| 任货行 | 黑名单、注销、钱包、账单、定时任务 | `hcb` |
| 新发行 | 退费、拒办、申办 | `fenmi_etc` |
| 综合服务平台 | 统计报表、预警配置 | - |

## 日志文件路径

### 任通行系统

| 服务 | 日志路径 |
|-----|---------|
| rtx-app | `/home/java-server/rtx-app/log.txt` |
| rtx-admin | `/home/java-server/rtx-admin/log.txt` |
| rtx-quartz | `/home/java-server/rtx-quartz/log.txt` |

### 任货行系统

| 服务 | 日志路径 |
|-----|---------|
| hcb-api | `/home/tomcat/tomcat-hcbapi/logs/catalina.out` |
| hcb-admin | `/home/tomcat/tomcat-hcbadmin/logs/catalina.out` |
| hcb-Timer-bill | `/home/tomcat/tomcat-hcbTimer-bill/logs/catalina.out` |
| hcb-Timer-business | `/home/tomcat/tomcat-hcbTimer-business/logs/catalina.out` |

### 其他系统

| 服务 | 日志路径 |
|-----|---------|
| fenmi-pay | `/home/fenmi/fenmi-pay/log.txt` |
| fenmi-etc | `/home/fenmi/fenmi-etc/log.txt` |

## 数据库连接信息

```bash
# MySQL 连接信息
主机: 192.168.1.60
端口: 3306
用户: root
密码: fm123456
```

**常用数据库**:
- `rtx` - 任通行
- `hcb` - 任货行
- `fenmi_pay` - 分米支付
- `fenmi_etc` - 新发行

## ⭐ 常用测试账号数据

**复现BUG时，优先使用以下测试数据查询相关信息：**

| 数据类型 | 测试数据 | 用途 |
|---------|---------|------|
| **身份证号** | `441622199602075195` | 查询用户信息、车辆信息、订单数据等 |
| **手机号** | `15818376788` | 查询用户账号、登录记录、充值记录等 |

**使用场景**:
```sql
-- 通过身份证号查询用户
SELECT * FROM tb_user_info WHERE id_card = '441622199602075195';

-- 通过手机号查询用户
SELECT * FROM tb_user_info WHERE mobile = '15818376788';

-- 查询任通行用户
SELECT * FROM rtx_truckuser WHERE mobile = '15818376788';

-- 查询任货行车辆
SELECT * FROM hcb_vehicle WHERE id_card = '441622199602075195';

-- 查询订单记录
SELECT * FROM tb_order WHERE mobile = '15818376788' ORDER BY create_time DESC LIMIT 10;
```

**⚠️ 复现BUG时的注意事项**:
- 先用这些测试数据验证BUG是否可复现
- 在BUG报告的"前置条件"中明确使用的测试数据
- 提供完整的数据查询SQL，方便开发验证

---

## 查询日志的命令模板

### 查看最新日志

```bash
# 任通行
ssh_execute(server="test", command="tail -n 100 /home/java-server/rtx-app/log.txt")

# 任货行
ssh_execute(server="test", command="tail -n 100 /home/tomcat/tomcat-hcbapi/logs/catalina.out")
```

### 搜索错误日志

```bash
# 搜索ERROR级别日志
ssh_execute(server="test", command="grep 'ERROR' /home/java-server/rtx-app/log.txt | tail -n 50")

# 搜索特定关键词
ssh_execute(server="test", command="grep '充值失败' /home/java-server/rtx-app/log.txt | tail -n 20")
```

### 查看实时日志

```bash
# 实时查看日志
ssh_tail(
  server="test",
  file="/home/java-server/rtx-app/log.txt",
  lines=50,
  follow=true
)
```

## 数据查询模板

### 查询订单数据

```bash
ssh_db_query(
  server="test",
  type="mysql",
  database="rtx",
  query="SELECT * FROM rtx_order WHERE order_id='xxx'",
  dbUser="root",
  dbPassword="fm123456",
  dbHost="192.168.1.60",
  dbPort=3306
)
```

### 查询用户数据

```bash
ssh_db_query(
  server="test",
  type="mysql",
  database="rtx",
  query="SELECT * FROM rtx_truckuser WHERE car_num='黑A113355'",
  dbUser="root",
  dbPassword="fm123456",
  dbHost="192.168.1.60",
  dbPort=3306
)
```

### 查询钱包数据

```bash
ssh_db_query(
  server="test",
  type="mysql",
  database="hcb",
  query="SELECT * FROM HCB_HLJ_TRUCKUSERWALLET WHERE ID_CODE='xxx'",
  dbUser="root",
  dbPassword="fm123456",
  dbHost="192.168.1.60",
  dbPort=3306
)
```

## BUG 报告前置条件模板

### 任通行BUG

```markdown
### 前置条件
- 测试环境: 192.168.1.60
- 系统: 任通行APP
- 数据库: rtx
- 测试账号: 车牌号 黑A113355
- 问题订单: order_id=xxx
```

### 任货行BUG

```markdown
### 前置条件
- 测试环境: 192.168.1.60
- 系统: 任货行APP / 任货行管理后台
- 数据库: hcb
- 测试账号: 车牌号 贵H86206
- 问题数据: 身份证号 xxx / 订单ID xxx
```

### 定时任务BUG

```markdown
### 前置条件
- 测试环境: 192.168.1.60
- 系统: 任货行定时任务 (hcb-Timer-bill)
- 数据库: hcb
- 定时任务: 注销流程静止期处理
- 问题数据: 注销订单ID xxx
```

## 常用测试环境信息

| 项目 | 地址 |
|-----|------|
| 测试服务器 | 192.168.1.60 |
| 任通行APP | http://192.168.1.60/rtx-app |
| 任货行API | http://192.168.1.60:8080 |
| 任货行管理后台 | http://192.168.1.60:8081 |
| MySQL | 192.168.1.60:3306 |
| Redis | 192.168.1.60:6379 |

## 快速参考

**编写BUG报告时**:
1. 确定模块 → 确定日志路径（查看上面的表格）
2. 复现BUG → 收集错误日志（使用ssh_execute或ssh_tail）
3. 查询数据 → 使用ssh_db_query（如果涉及数据问题）
4. 编写报告 → 保存到 `temp/BUG报告/BUG-【模块】描述.md`
