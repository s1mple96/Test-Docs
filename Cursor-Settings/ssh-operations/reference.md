# SSH操作详细配置参考

## 服务器配置信息

### 测试服务器（test）
```
Host: 192.168.1.60
User: root
Port: 22
用途: 测试环境专用服务器
```

**⚠️ 禁止操作生产服务器！**

---

## 完整系统路径清单

### 任通行（RTX）

**应用服务**:
- 路径: `/home/java-server/rtx-app/`
- 日志: `/home/java-server/rtx-app/log.txt`
- 配置: `/home/java-server/rtx-app/config/application.yml`
- 端口: 8081

**后台管理**:
- 路径: `/home/java-server/rtx-admin/`
- 日志: `/home/java-server/rtx-admin/log.txt`
- 端口: 8082

**定时任务**:
- 路径: `/home/java-server/rtx-job/`
- 日志: `/home/java-server/rtx-job/log.txt`
- 端口: 8083

**本地代码路径**: `e:/Fenmi-Test/java/rtx/`

---

### 任货行（HCB）

**应用服务（API）**:
- 路径: `/home/tomcat/tomcat-hcbapi/`
- 日志: `/home/tomcat/tomcat-hcbapi/logs/catalina.out`
- 端口: 8080

**后台管理**:
- 路径: `/home/tomcat/tomcat-hcbadmin/`
- 日志: `/home/tomcat/tomcat-hcbadmin/logs/catalina.out`
- 端口: 8888

**账单定时任务**:
- 路径: `/home/tomcat/tomcat-hcbTimer-bill/`
- 日志: `/home/tomcat/tomcat-hcbTimer-bill/logs/catalina.out`

**补换定时任务**:
- 路径: `/home/tomcat/tomcat-hcbTimer-swap/`
- 日志: `/home/tomcat/tomcat-hcbTimer-swap/logs/catalina.out`

**运营商账单文件存放路径**:
- 联通: `/home/operator-bill/unicom/YYYYMM/`
- 移动: `/home/operator-bill/cmcc/YYYYMM/`
- 电信: `/home/operator-bill/telecom/YYYYMM/`

**本地代码路径**: `e:/Fenmi-Test/java/hcb/` (包含多个子项目)
- `hcb-admin/` - 后台管理
- `hcb-api/` - API服务
- `hcb-timer/` - 定时任务

---

### 分米支付（Fenmi）

**支付服务**:
- 路径: `/home/fenmi/fenmi-pay/`
- 日志: `/home/fenmi/fenmi-pay/log.txt`
- 端口: 9001

**ETC服务**:
- 路径: `/home/fenmi/fenmi-etc/`
- 日志: `/home/fenmi/fenmi-etc/log.txt`
- 端口: 9002

**定时任务**:
- 路径: `/home/fenmi/fenmi-job/`
- 日志: `/home/fenmi/fenmi-job/log.txt`
- 端口: 9003

**钱包服务**:
- 路径: `/home/fenmi/fenmi-wallet/`
- 日志: `/home/fenmi/fenmi-wallet/log.txt`
- 端口: 9004

**本地代码路径**: `e:/Fenmi-Test/java/fenmi/`

---

## 数据库配置

### MySQL

**连接信息**:
```
Host: 192.168.1.60
Port: 3306
User: root
Password: fm123456
```

**数据库清单**:

| 数据库名 | 系统 | 说明 |
|---------|------|------|
| `rtx` | 任通行 | 用户、订单、卡券等 |
| `hcb` | 任货行 | 车辆、司机、账单等 |
| `fenmi_pay` | 分米支付 | 支付订单、退款等 |
| `fenmi_etc` | 分米ETC | ETC通行记录 |
| `fenmi_wallet` | 分米钱包 | 钱包余额、流水 |

**常用表清单**:

**任通行（rtx）**:
- `tb_user_info` - 用户信息
- `tb_order` - 订单表
- `tb_coupon_user` - 用户卡券
- `tb_recharge_record` - 充值记录

**任货行（hcb）**:
- `tb_vehicle` - 车辆信息
- `tb_driver` - 司机信息
- `tb_bill_parse_log` - 账单解析日志
- `tb_bill_detail` - 账单明细
- `tb_swap_apply` - 补换申请

**分米支付（fenmi_pay）**:
- `tb_pay_order` - 支付订单
- `tb_refund_record` - 退款记录
- `tb_bank_card` - 银行卡信息

---

### ⭐ 常用测试账号数据

**测试时优先使用以下测试数据查询相关信息：**

| 数据类型 | 测试数据 | 用途 |
|---------|---------|------|
| **身份证号** | `441622199602075195` | 查询用户信息、车辆信息、订单数据等 |
| **手机号** | `15818376788` | 查询用户账号、登录记录、充值记录等 |

**快速查询命令**:
```python
# 通过手机号查询任通行用户
ssh_db_query(
    server="test",
    type="mysql",
    database="rtx",
    dbUser="root",
    dbPassword="fm123456",
    dbHost="192.168.1.60",
    query="SELECT * FROM rtx_truckuser WHERE mobile = '15818376788';"
)

# 通过身份证号查询任货行车辆
ssh_db_query(
    server="test",
    type="mysql",
    database="hcb",
    dbUser="root",
    dbPassword="fm123456",
    dbHost="192.168.1.60",
    query="SELECT * FROM hcb_vehicle WHERE id_card = '441622199602075195';"
)

# 查询用户订单记录
ssh_db_query(
    server="test",
    type="mysql",
    database="rtx",
    dbUser="root",
    dbPassword="fm123456",
    dbHost="192.168.1.60",
    query="SELECT * FROM tb_order WHERE mobile = '15818376788' ORDER BY create_time DESC LIMIT 10;"
)
```

---

### Redis

**连接信息**:
```
Host: 192.168.1.60
Port: 6379
Password: chefu666
```

**常用Key前缀**:
- `rtx:login:code:` - 任通行登录验证码
- `rtx:user:token:` - 任通行用户Token
- `fenmi:pay:lock:` - 分米支付分布式锁
- `hcb:bill:cache:` - 任货行账单缓存

**Redis 命令模板**:
```bash
# GET 命令
ssh_execute(
    server="test",
    command="redis-cli -a chefu666 GET key_name"
)

# SET 命令
ssh_execute(
    server="test",
    command="redis-cli -a chefu666 SET key_name 'value'"
)

# DEL 命令
ssh_execute(
    server="test",
    command="redis-cli -a chefu666 DEL key_name"
)

# 查看所有key（谨慎使用）
ssh_execute(
    server="test",
    command="redis-cli -a chefu666 KEYS 'rtx:*'"
)

# 查看key的TTL
ssh_execute(
    server="test",
    command="redis-cli -a chefu666 TTL key_name"
)
```

---

### MongoDB

**连接信息**:
```
Host: 192.168.1.60
Port: 27017
Database: jdbdb
(MongoDB无需密码)
```

**集合清单**:
- `bill_detail` - 账单明细（任货行部分数据）
- `parse_log` - 解析日志

**MongoDB 查询模板**:
```python
# 查询
ssh_db_query(
    server="test",
    type="mongodb",
    database="jdbdb",
    dbUser="root",
    dbPassword="fm123456",
    dbHost="192.168.1.60",
    query="db.bill_detail.find({mobile: '13800138000'}).limit(10)"
)

# 统计
ssh_db_query(
    server="test",
    type="mongodb",
    database="jdbdb",
    dbUser="root",
    dbPassword="fm123456",
    dbHost="192.168.1.60",
    query="db.bill_detail.count({bill_month: '202401'})"
)

# 聚合
ssh_db_query(
    server="test",
    type="mongodb",
    database="jdbdb",
    dbUser="root",
    dbPassword="fm123456",
    dbHost="192.168.1.60",
    query="db.bill_detail.aggregate([{$match: {mobile: '13800138000'}}, {$group: {_id: '$bill_month', total: {$sum: 1}}}])"
)
```

---

## SSH MCP 工具完整参考

### ssh_execute - 执行命令

**参数**:
```python
ssh_execute(
    server="test",          # 必需，服务器名称
    command="ls -lh",       # 必需，要执行的命令
    cwd="/home",            # 可选，工作目录
    timeout=30000           # 可选，超时时间（毫秒），默认30秒
)
```

**常见用途**:
- 查看文件列表
- 搜索日志关键字
- 执行 Redis 命令
- 检查进程状态
- 查看系统信息

---

### ssh_db_query - 数据库查询

**MySQL 参数**:
```python
ssh_db_query(
    server="test",
    type="mysql",
    database="rtx",
    dbUser="root",
    dbPassword="fm123456",
    dbHost="192.168.1.60",
    dbPort=3306,            # 可选，默认3306
    query="SELECT * FROM tb_user_info LIMIT 10;"
)
```

**MongoDB 参数**:
```python
ssh_db_query(
    server="test",
    type="mongodb",
    database="jdbdb",
    dbUser="root",
    dbPassword="fm123456",
    dbHost="192.168.1.60",
    dbPort=27017,           # 可选，默认27017
    query="db.bill_detail.find({}).limit(10)"
)
```

**⚠️ 查询规范**:
1. 查询前必须先 `DESC` 或 `SHOW TABLES` 确认表结构
2. DML 操作前必须先 `SELECT COUNT(*)` 确认影响范围
3. 添加 `LIMIT` 限制返回行数（建议 ≤ 1000）
4. 复杂查询增加 `timeout` 参数

---

### ssh_tail - 查看日志

**参数**:
```python
ssh_tail(
    server="test",
    file="/home/java-server/rtx-app/log.txt",  # 必需，日志文件路径
    lines=100,              # 可选，显示行数，默认10
    follow=False,           # 可选，是否实时跟踪，默认true
    grep="ERROR"            # 可选，过滤关键字
)
```

**实时监控模式** (`follow=True`):
- 新增的日志内容会自动显示
- 适用于监控定时任务执行、接口调用等
- 可配合 `grep` 参数过滤特定关键字

**静态查看模式** (`follow=False`):
- 只显示指定行数的最新日志
- 适用于快速查看最近的日志内容

---

### ssh_monitor - 监控资源

**参数**:
```python
ssh_monitor(
    server="test",
    type="overview",        # 必需，监控类型
    duration=60,            # 可选，持续时间（秒）
    interval=5              # 可选，更新间隔（秒）
)
```

**监控类型**:
- `overview` - 资源概览（CPU、内存、磁盘）
- `cpu` - CPU 使用率详情
- `memory` - 内存使用详情
- `disk` - 磁盘使用详情
- `network` - 网络流量
- `process` - 进程列表

---

### ssh_process_manager - 进程管理

**列出进程**:
```python
ssh_process_manager(
    server="test",
    action="list",
    sortBy="cpu",           # 可选，排序方式：cpu 或 memory
    limit=20,               # 可选，返回数量，默认20
    filter="java"           # 可选，过滤进程名
)
```

**查看进程信息**:
```python
ssh_process_manager(
    server="test",
    action="info",
    pid=12345               # 必需，进程ID
)
```

**杀死进程**:
```python
ssh_process_manager(
    server="test",
    action="kill",
    pid=12345,              # 必需，进程ID
    signal="TERM"           # 可选，信号：TERM/KILL/HUP/INT/QUIT
)
```

---

### ssh_upload - 上传文件

**参数**:
```python
ssh_upload(
    server="test",
    localPath="e:/Fenmi-Test/temp/test-data.json",   # 必需，本地文件路径
    remotePath="/home/test/test-data.json"            # 必需，远程文件路径
)
```

**常见用途**:
- 上传测试数据文件
- 上传配置文件
- 上传脚本文件

---

### ssh_download - 下载文件

**参数**:
```python
ssh_download(
    server="test",
    remotePath="/home/java-server/rtx-app/log.txt",   # 必需，远程文件路径
    localPath="e:/Fenmi-Test/temp/rtx-app-log.txt"    # 必需，本地保存路径
)
```

**常见用途**:
- 下载日志文件到本地分析
- 下载配置文件查看
- 下载数据文件

---

### ssh_sync - 文件同步

**参数**:
```python
ssh_sync(
    server="test",
    source="local:/e/Fenmi-Test/temp/data/",     # 必需，源路径
    destination="remote:/home/test/data/",       # 必需，目标路径
    delete=False,           # 可选，是否删除目标中不存在的文件
    compress=True,          # 可选，是否压缩传输
    dryRun=False,           # 可选，是否只模拟不实际操作
    exclude=["*.log", "*.tmp"]  # 可选，排除的文件模式
)
```

**路径前缀**:
- `local:` - 本地路径
- `remote:` - 远程路径

**常见用途**:
- 同步测试数据文件夹
- 备份远程日志文件夹

---

## 测试环境访问地址

### 前端页面
- **任通行H5**: `http://test.rtx.fenmi.com`
- **任通行后台**: `http://test.rtx-admin.fenmi.com`
- **任货行H5**: `http://test.hcb.fenmi.com`
- **任货行后台**: `http://test.hcb-admin.fenmi.com`
- **分米支付H5**: `http://test.pay.fenmi.com`

### API接口
- **任通行API**: `http://192.168.1.60:8081`
- **任货行API**: `http://192.168.1.60:8080`
- **分米支付API**: `http://192.168.1.60:9001`

---

## 常见场景速查表

| 场景 | 推荐工具 | 关键参数 |
|------|---------|---------|
| 查看日志 | `ssh_tail` | `file`, `lines`, `grep` |
| 搜索日志关键字 | `ssh_execute` | `grep -n 'keyword' file` |
| 查询数据库 | `ssh_db_query` | `database`, `query` |
| 检查进程 | `ssh_process_manager` | `action="list"`, `sortBy="cpu"` |
| 查看资源 | `ssh_monitor` | `type="overview"` |
| 下载日志 | `ssh_download` | `remotePath`, `localPath` |
| 实时监控 | `ssh_tail` | `follow=True` |
| Redis操作 | `ssh_execute` | `redis-cli -a password` |

---

## 安全注意事项

1. **禁止操作生产服务器**: 只使用 `test` 服务器
2. **禁止危险命令**: 不要使用 `rm -rf`, `DROP TABLE`, `TRUNCATE` 等
3. **数据库操作规范**: 
   - 大批量操作前先测试小批量
   - 删除/更新前先 `SELECT` 确认
   - 添加 `LIMIT` 保护
4. **日志查看规范**: 
   - 不要一次性 `cat` 巨大的日志文件
   - 使用 `tail`, `grep` 等命令精准定位
5. **密码安全**: 
   - 密码已加密存储在配置中
   - 不要在日志或文档中明文记录密码

---

## 故障排查指南

### 问题1: 数据库连接超时
**可能原因**:
- 数据库服务未启动
- 网络不通
- 密码错误

**排查步骤**:
```bash
# 1. 检查数据库服务是否运行
ssh_execute(server="test", command="ps aux | grep mysql")

# 2. 测试数据库连接
ssh_execute(server="test", command="mysql -h 192.168.1.60 -u root -pfm123456 -e 'SELECT 1;'")
```

---

### 问题2: 日志文件找不到
**可能原因**:
- 路径错误
- 服务未启动
- 日志文件名错误

**排查步骤**:
```bash
# 1. 查找日志文件
ssh_execute(server="test", command="find /home -name 'log.txt' -o -name 'catalina.out'")

# 2. 检查服务是否运行
ssh_execute(server="test", command="ps aux | grep java")
```

---

### 问题3: Redis 连接失败
**可能原因**:
- Redis 服务未启动
- 密码错误

**排查步骤**:
```bash
# 1. 检查 Redis 服务
ssh_execute(server="test", command="ps aux | grep redis")

# 2. 测试 Redis 连接
ssh_execute(server="test", command="redis-cli -a chefu666 PING")
```

---

### 问题4: 命令执行超时
**可能原因**:
- 命令执行时间过长
- 服务器资源不足

**解决方案**:
```python
# 增加超时时间
ssh_execute(
    server="test",
    command="your_command",
    timeout=60000  # 60秒
)

# 或者拆分成多个小命令
```
