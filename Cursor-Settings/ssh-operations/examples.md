# SSH操作示例

## 场景1：排查任通行登录失败问题

**用户反馈**: 手机号 `13800138000` 无法登录

### 完整排查流程

**Step 1: 查询用户数据**
```python
ssh_db_query(
    server="test",
    type="mysql",
    database="rtx",
    dbUser="root",
    dbPassword="fm123456",
    dbHost="192.168.1.60",
    query="SELECT id, mobile, status, create_time FROM tb_user_info WHERE mobile = '13800138000';"
)
```

**Step 2: 查看登录相关日志**
```bash
ssh_execute(
    server="test",
    command="grep '13800138000' /home/java-server/rtx-app/log.txt | tail -50"
)
```

**Step 3: 检查Redis验证码**
```bash
ssh_execute(
    server="test",
    command="redis-cli -a Fenmi_2020 GET rtx:login:code:13800138000"
)
```

**Step 4: 查看最新异常日志**
```bash
ssh_execute(
    server="test",
    command="grep -A 5 'Exception' /home/java-server/rtx-app/log.txt | tail -30"
)
```

---

## 场景2：检查任货行账单解析是否完成

**需求**: 确认 2024年1月运营商账单是否已解析入库

### 完整检查流程

**Step 1: 查询账单解析记录**
```python
ssh_db_query(
    server="test",
    type="mysql",
    database="hcb",
    dbUser="root",
    dbPassword="fm123456",
    dbHost="192.168.1.60",
    query="SELECT id, bill_month, operator_name, status, parse_time FROM tb_bill_parse_log WHERE bill_month = '202401' ORDER BY parse_time DESC;"
)
```

**Step 2: 查看解析定时任务日志**
```bash
ssh_tail(
    server="test",
    file="/home/tomcat/tomcat-hcbTimer-bill/logs/catalina.out",
    lines=200,
    follow=False
)
```

**Step 3: 检查账单文件是否存在**
```bash
ssh_execute(
    server="test",
    command="ls -lh /home/operator-bill/202401/"
)
```

**Step 4: 统计解析入库的账单明细数量**
```python
ssh_db_query(
    server="test",
    type="mysql",
    database="hcb",
    dbUser="root",
    dbPassword="fm123456",
    dbHost="192.168.1.60",
    query="SELECT COUNT(*) as total FROM tb_bill_detail WHERE bill_month = '202401';"
)
```

---

## 场景3：验证分米支付退款功能

**需求**: 测试订单号 `FM20240101123456` 的退款流程

### 完整验证流程

**Step 1: 查询订单信息**
```python
ssh_db_query(
    server="test",
    type="mysql",
    database="fenmi_pay",
    dbUser="root",
    dbPassword="fm123456",
    dbHost="192.168.1.60",
    query="SELECT order_no, status, amount, user_id FROM tb_pay_order WHERE order_no = 'FM20240101123456';"
)
```

**Step 2: 查看退款接口日志**
```bash
ssh_execute(
    server="test",
    command="grep 'FM20240101123456' /home/fenmi/fenmi-pay/log.txt | grep -i refund | tail -20"
)
```

**Step 3: 查询退款记录**
```python
ssh_db_query(
    server="test",
    type="mysql",
    database="fenmi_pay",
    dbUser="root",
    dbPassword="fm123456",
    dbHost="192.168.1.60",
    query="SELECT * FROM tb_refund_record WHERE order_no = 'FM20240101123456';"
)
```

**Step 4: 检查是否有异常日志**
```bash
ssh_execute(
    server="test",
    command="grep -C 3 'FM20240101123456' /home/fenmi/fenmi-pay/log.txt | grep -E '(ERROR|Exception)'"
)
```

---

## 场景4：监控服务器资源占用

**需求**: 定时任务执行时服务器资源飙升，需要定位原因

### 完整监控流程

**Step 1: 查看资源概览**
```bash
ssh_monitor(
    server="test",
    type="overview"
)
```

**Step 2: 查看CPU占用最高的进程**
```bash
ssh_process_manager(
    server="test",
    action="list",
    sortBy="cpu",
    limit=10
)
```

**Step 3: 查看内存占用情况**
```bash
ssh_monitor(
    server="test",
    type="memory"
)
```

**Step 4: 检查是否有僵尸进程**
```bash
ssh_execute(
    server="test",
    command="ps aux | grep -E '(defunct|Z)' | grep -v grep"
)
```

**Step 5: 查看磁盘使用率**
```bash
ssh_monitor(
    server="test",
    type="disk"
)
```

---

## 场景5：准备测试数据

**需求**: 为测试用户 `13900139000` 准备卡券数据

### 完整准备流程

**Step 1: 确认表结构**
```python
ssh_db_query(
    server="test",
    type="mysql",
    database="rtx",
    dbUser="root",
    dbPassword="fm123456",
    dbHost="192.168.1.60",
    query="DESC tb_coupon_user;"
)
```

**Step 2: 检查是否已有卡券**
```python
ssh_db_query(
    server="test",
    type="mysql",
    database="rtx",
    dbUser="root",
    dbPassword="fm123456",
    dbHost="192.168.1.60",
    query="SELECT * FROM tb_coupon_user WHERE mobile = '13900139000';"
)
```

**Step 3: 插入测试卡券**
```python
ssh_db_query(
    server="test",
    type="mysql",
    database="rtx",
    dbUser="root",
    dbPassword="fm123456",
    dbHost="192.168.1.60",
    query="INSERT INTO tb_coupon_user (mobile, coupon_id, status, expire_time) VALUES ('13900139000', 1001, 1, '2024-12-31 23:59:59');"
)
```

**Step 4: 验证插入成功**
```python
ssh_db_query(
    server="test",
    type="mysql",
    database="rtx",
    dbUser="root",
    dbPassword="fm123456",
    dbHost="192.168.1.60",
    query="SELECT * FROM tb_coupon_user WHERE mobile = '13900139000' ORDER BY id DESC LIMIT 1;"
)
```

---

## 场景6：实时监控账单解析日志

**需求**: 账单解析定时任务正在执行，需要实时查看日志

### 完整监控流程

**Step 1: 启动日志实时监控**
```bash
ssh_tail(
    server="test",
    file="/home/tomcat/tomcat-hcbTimer-bill/logs/catalina.out",
    follow=True,
    lines=50
)
```

**Step 2: 过滤特定关键字**
```bash
ssh_tail(
    server="test",
    file="/home/tomcat/tomcat-hcbTimer-bill/logs/catalina.out",
    follow=True,
    grep="解析成功"
)
```

**Step 3: 查看是否有错误日志**
```bash
ssh_tail(
    server="test",
    file="/home/tomcat/tomcat-hcbTimer-bill/logs/catalina.out",
    follow=True,
    grep="ERROR"
)
```

---

## 场景7：下载日志到本地分析

**需求**: 日志太大，需要下载到本地用工具分析

### 完整下载流程

**Step 1: 检查日志文件大小**
```bash
ssh_execute(
    server="test",
    command="ls -lh /home/java-server/rtx-app/log.txt"
)
```

**Step 2: 如果文件太大，先压缩**
```bash
ssh_execute(
    server="test",
    command="gzip -c /home/java-server/rtx-app/log.txt > /tmp/rtx-app-log.txt.gz"
)
```

**Step 3: 下载到本地**
```bash
ssh_download(
    server="test",
    remotePath="/tmp/rtx-app-log.txt.gz",
    localPath="e:/Fenmi-Test/temp/rtx-app-log.txt.gz"
)
```

---

## 常用命令速查

### 查找文件
```bash
# 查找最近修改的日志文件
ssh_execute(
    server="test",
    command="find /home -name 'log.txt' -mtime -1"
)
```

### 查看端口占用
```bash
ssh_execute(
    server="test",
    command="netstat -tlnp | grep 8080"
)
```

### 检查服务是否运行
```bash
ssh_execute(
    server="test",
    command="ps aux | grep java | grep rtx-app"
)
```

### 查看最近的系统日志
```bash
ssh_execute(
    server="test",
    command="tail -100 /var/log/messages"
)
```

### 统计日志中某个关键字出现次数
```bash
ssh_execute(
    server="test",
    command="grep -c 'NullPointerException' /home/java-server/rtx-app/log.txt"
)
```
