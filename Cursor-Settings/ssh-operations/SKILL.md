---
name: SSH服务器测试操作
description: 在测试服务器上执行命令、查询数据库、查看日志、监控资源等测试相关操作
---

# SSH服务器测试操作

## 何时使用本技能

- 需要查看测试服务器的日志文件
- 需要查询测试环境数据库（MySQL/Redis/MongoDB）
- 需要监控服务器资源（CPU、内存、磁盘）
- 需要检查进程状态
- 需要查看文件内容或上传/下载文件

## ⚠️ 核心约束

**1. 服务器使用规范**
- ✅ 只使用 `test` 服务器（测试环境）
- ❌ 永远不要操作生产服务器
- ❌ 禁止修改服务器上的源代码
- ✅ 可以修改测试数据、查看日志

**2. 数据库操作规范**
- ✅ 查询前必须先确认表结构（`SHOW TABLES;` 或 `DESC 表名;`）
- ✅ DML 操作前必须先 `SELECT` 确认影响范围
- ✅ 单次修改 < 1万条，大批量分批处理
- ❌ 禁止 `DROP TABLE`、`TRUNCATE`、`DELETE` 无 WHERE 条件

**3. 日志查看规范**
- ✅ 先查看最新的日志（`tail -n 100`）
- ✅ 查找关键字（ERROR、Exception、成功/失败的业务日志）
- ✅ 检查是否有死循环（日志重复）
- ✅ 告知用户日志文件路径

## 标准操作流程

### 1️⃣ 执行命令

```bash
# 基本命令执行
ssh_execute(
    server="test",
    command="ls -lh /home/java-server/rtx-app/"
)

# 需要长时间运行的命令，增加超时时间
ssh_execute(
    server="test",
    command="find /home -name '*.log'",
    timeout=60000  # 60秒
)
```

### 2️⃣ 查询数据库

**MySQL 查询**:
```python
# 查询表结构
ssh_db_query(
    server="test",
    type="mysql",
    database="rtx",
    dbUser="root",
    dbPassword="fm123456",
    dbHost="192.168.1.60",
    query="DESC tb_user_info;"
)

# 查询数据
ssh_db_query(
    server="test",
    type="mysql",
    database="rtx",
    dbUser="root",
    dbPassword="fm123456",
    dbHost="192.168.1.60",
    query="SELECT * FROM tb_user_info WHERE mobile = '13800138000' LIMIT 10;"
)
```

**Redis 查询**:
```bash
ssh_execute(
    server="test",
    command="redis-cli -a Fenmi_2020 GET fenmi:user:token:123456"
)
```

**MongoDB 查询**:
```python
ssh_db_query(
    server="test",
    type="mongodb",
    database="jdbdb",
    dbUser="root",
    dbPassword="fm123456",
    dbHost="192.168.1.60",
    query="db.bill_detail.find({mobile: '13800138000'}).limit(10)"
)
```

### 3️⃣ 查看日志

```bash
# 查看最新100行日志
ssh_tail(
    server="test",
    file="/home/java-server/rtx-app/log.txt",
    lines=100,
    follow=False
)

# 实时监控日志（新增内容会自动显示）
ssh_tail(
    server="test",
    file="/home/java-server/rtx-app/log.txt",
    follow=True,
    grep="ERROR"  # 只显示包含 ERROR 的行
)

# 搜索日志中的关键字
ssh_execute(
    server="test",
    command="grep -n 'NullPointerException' /home/java-server/rtx-app/log.txt | tail -20"
)
```

### 4️⃣ 监控资源

```bash
# 查看服务器资源概览
ssh_monitor(
    server="test",
    type="overview"
)

# 查看CPU使用率
ssh_monitor(
    server="test",
    type="cpu"
)

# 查看进程列表（按CPU排序）
ssh_process_manager(
    server="test",
    action="list",
    sortBy="cpu",
    limit=20
)
```

### 5️⃣ 文件操作

```bash
# 上传文件到服务器
ssh_upload(
    server="test",
    localPath="e:/Fenmi-Test/temp/test-data.json",
    remotePath="/home/test/test-data.json"
)

# 下载文件到本地
ssh_download(
    server="test",
    remotePath="/home/java-server/rtx-app/log.txt",
    localPath="e:/Fenmi-Test/temp/rtx-app-log.txt"
)
```

## 系统路径快速参考

**任通行（RTX）**:
- 应用: `/home/java-server/rtx-app/`
- 后台: `/home/java-server/rtx-admin/`
- 定时任务: `/home/java-server/rtx-job/`
- 日志: `log.txt`
- 数据库: `rtx`

**任货行（HCB）**:
- 应用: `/home/tomcat/tomcat-hcbapi/`
- 后台: `/home/tomcat/tomcat-hcbadmin/`
- 账单定时: `/home/tomcat/tomcat-hcbTimer-bill/`
- 日志: `logs/catalina.out`
- 数据库: `hcb`

**分米支付（Fenmi）**:
- 支付: `/home/fenmi/fenmi-pay/`
- ETC: `/home/fenmi/fenmi-etc/`
- 定时任务: `/home/fenmi/fenmi-job/`
- 日志: `log.txt`
- 数据库: `fenmi_pay`, `fenmi_etc`, `fenmi_wallet`

## 数据库连接配置

**测试环境统一配置**:
- **Host**: `192.168.1.60`
- **User**: `root`
- **Password**: `fm123456`
- **Redis密码**: `chefu666`

**数据库清单**:
- `rtx` - 任通行
- `hcb` - 任货行
- `fenmi_pay` - 分米支付
- `fenmi_etc` - 分米ETC
- `fenmi_wallet` - 分米钱包
- `jdbdb` - MongoDB（任货行部分数据）

## 标准输出格式

执行完成后，提供简洁反馈：

```
✅ 查询完成
结果: [简短描述]
日志路径: [如适用]
```

## 常见错误处理

**错误1**: `database 'xxx' not found`
- 检查系统对应的数据库名是否正确（参考上方数据库清单）

**错误2**: `ssh_db_query` 超时
- 增加 `timeout` 参数（单位：毫秒）
- 添加 `LIMIT` 限制返回行数
- 检查 WHERE 条件是否有索引

**错误3**: 日志文件找不到
- 确认服务名称（rtx-app / rtx-admin / hcb-api 等）
- 确认日志文件名（log.txt / catalina.out）

---

**详细配置和示例**，请参考 `reference.md` 和 `examples.md`。
