# 测试用例编写 - 项目配置参考

## 系统与数据库映射关系

| 系统名称 | 数据库名 | 说明 |
|---------|---------|------|
| 任通行 (RTX) | `rtx` | ETC通行服务 |
| 任货行 (HCB) | `hcb` | 货车ETC服务 |
| 新发行 | `fenmi_etc` | 新发行ETC服务 |
| 分米支付 | `fenmi_pay` | 支付服务 |
| 综合服务平台 | - | 综合管理系统 |
| 客货系统 | - | 客货系统 |

## 测试用例文件保存

**所有测试用例统一保存到**: `temp/测试用例/`

```
temp/测试用例/
├── 【任货行】xxx测试用例_V1.0.md
├── 【任通行】xxx测试用例_V1.0.md
├── 【新发行】xxx测试用例_V1.0.md
├── 【任通行、任货行】xxx测试用例_V1.0.md  (多系统共用)
└── ... (所有测试用例文件)
```

**⚠️ 重要**: 
- 所有测试用例直接保存在 `temp/测试用例/` 根目录，不创建子文件夹
- 通过文件名中的【系统名称】前缀区分不同系统
- 按文件修改时间排序，方便快速找到最新生成的测试用例

## 代码仓库路径

### 本地代码路径

代码一般存放在 `java` 目录下:

| 系统 | 本地代码路径 | 说明 |
|-----|------------|------|
| 任通行 (rtx) | `java/rtx/` | 单仓库,直接 pull |
| 任货行 (hcb) | `java/hcb/` | **多子目录,需进入每个子目录逐个 pull** |
| 分米支付 (fenmi) | `java/fenmi/` | 单仓库,直接 pull |

### 拉取最新代码流程

**⚠️ 重要**: 只拉取 `test` 分支（测试环境分支），不拉取其他分支

**任通行 (RTX) - 直接拉取 test 分支**:
```bash
cd java/rtx
git checkout test
git pull origin test
```

**分米支付 (Fenmi) - 直接拉取 test 分支**:
```bash
cd java/fenmi
git checkout test
git pull origin test
```

**任货行 (HCB) - 需进入子目录逐个拉取 test 分支** ⚠️:
```bash
cd java/hcb

# 进入每个子目录拉取 test 分支
cd hcb-admin && git checkout test && git pull origin test && cd ..
cd hcb-api && git checkout test && git pull origin test && cd ..
cd hcb-timer-bill && git checkout test && git pull origin test && cd ..
cd hcb-timer-business && git checkout test && git pull origin test && cd ..
# ... 其他子目录
```

或使用脚本批量拉取 test 分支:
```bash
cd java/hcb
for dir in */; do
  echo "Pulling test branch in $dir"
  cd "$dir" && git checkout test && git pull origin test && cd ..
done
```

### 代码分析前的准备

1. **拉取最新代码**: 根据系统选择对应的拉取方式
   - **⚠️ 只拉取 `test` 分支**（测试环境分支）
   - 不要拉取 master、dev 或其他分支
2. **找到相关模块**: 根据功能定位到具体的 Service、Controller
3. **查看业务逻辑**: 分析代码中的业务规则、数据流转
4. **提取测试点**: 根据代码逻辑提取测试场景

### ⚠️ 分支管理注意事项

**编写测试用例只使用 test 分支**:
- test 分支 = 测试环境分支 ✅
- master/main 分支 = 生产环境分支 ❌ 不拉取
- dev 分支 = 开发环境分支 ❌ 不拉取
- 其他分支 ❌ 不拉取

**原因**: 测试用例需要与测试环境的代码保持一致，测试环境部署的是 test 分支

## 代码部署路径（服务器）

### 任通行系统

| 服务名称 | 部署路径 | 日志路径 | 说明 |
|---------|---------|---------|------|
| rtx-app | `/home/java-server/rtx-app/` | `/home/java-server/rtx-app/log.txt` | 应用主服务 |
| rtx-admin | `/home/java-server/rtx-admin/` | `/home/java-server/rtx-admin/log.txt` | 管理后台 |
| rtx-quartz | `/home/java-server/rtx-quartz/` | `/home/java-server/rtx-quartz/log.txt` | 定时任务 |
| report-admin | `/home/java-server/report-admin/` | `/home/java-server/report-admin/log.txt` | 报表服务 |
| operator-etc | `/home/java-server/operator-etc/` | - | 运营商服务 |

### 任货行系统

| 服务名称 | 部署路径 | 日志路径 | 说明 |
|---------|---------|---------|------|
| hcb-api | `/home/tomcat/tomcat-hcbapi/` | `/home/tomcat/tomcat-hcbapi/logs/catalina.out` | API服务 |
| hcb-admin | `/home/tomcat/tomcat-hcbadmin/` | `/home/tomcat/tomcat-hcbadmin/logs/catalina.out` | 管理后台 |
| hcb-Timer-bill | `/home/tomcat/tomcat-hcbTimer-bill/` | `/home/tomcat/tomcat-hcbTimer-bill/logs/catalina.out` | 账单定时任务 |
| hcb-Timer-business | `/home/tomcat/tomcat-hcbTimer-business/` | `/home/tomcat/tomcat-hcbTimer-business/logs/catalina.out` | 业务定时任务 |

### 其他系统

| 服务名称 | 部署路径 | 日志路径 | 说明 |
|---------|---------|---------|------|
| fenmi-pay | `/home/fenmi/fenmi-pay/` | `/home/fenmi/fenmi-pay/log.txt` | 分米支付 |
| fenmi-etc | `/home/fenmi/fenmi-etc/` | `/home/fenmi/fenmi-etc/log.txt` | 新发行ETC |
| fenmi-provider | `/home/fenmi/fenmi-provider/` | `/home/fenmi/fenmi-provider/log.txt` | 分米服务 |
| refund-scheduler | `/opt/refund-scheduler/` | `/opt/refund-scheduler/refund.log` | 退费调度 |

## 任货行运营商账单文件路径

| 运营商代码 | 运营商名称 | 文件路径 | 文件格式 | 金额单位 | 文件编码 |
|-----------|----------|---------|---------|---------|---------|
| MTK | 内蒙古 | `/home/data/test/MTK/` | JSON | 分 | GBK |
| LTK | 黑龙江 | `/home/data/test/LTK/` | JSON | 分 | GBK |
| TXB | 江苏 | `/home/data/test/TXB/` | TXT(^分隔) | 分 | UTF-8 |
| HTK | 河北 | `/home/data/test/HTK/` | JSON | 分 | GBK |

**文件命名规则示例**:
- MTK: `HCB_MTK_yyyyMMddHHmm.json` (如: `HCB_MTK_202601191630.json`)
- LTK: `HCB_LTK_yyyyMMddHHmm.json`
- TXB: `FMHL_QLQSMX_yyyyMMdd` (如: `FMHL_QLQSMX_20260119`)

## 数据库连接信息

### MySQL配置

```bash
# 测试服务器MySQL连接信息
主机: 192.168.1.60
端口: 3306
用户: root
密码: fm123456
```

**常用数据库操作**:

```bash
# 查询表结构
mysql -h 192.168.1.60 -u root -pfm123456 库名 -e 'DESC 表名;'

# 查询数据
mysql -h 192.168.1.60 -u root -pfm123456 库名 -e 'SELECT * FROM 表名 LIMIT 10;'

# 插入测试数据
mysql -h 192.168.1.60 -u root -pfm123456 库名 -e 'INSERT INTO 表名 (...) VALUES (...);'
```

**使用SSH MCP工具查询**:

```javascript
user-ssh-manager-ssh_db_query(
  server="test",
  type="mysql",
  database="rtx",  // 或 hcb、fenmi_pay 等
  query="SELECT * FROM 表名 LIMIT 10",
  dbUser="root",
  dbPassword="fm123456",
  dbHost="192.168.1.60",
  dbPort=3306
)
```

### Redis配置

```bash
# Redis连接信息
主机: 192.168.1.60
端口: 6379
密码: chefu666
```

**常用Redis操作**:

```bash
# 查询key
/home/redis/bin/redis-cli -h 192.168.1.60 -p 6379 -a chefu666 -n 0 GET 'key'

# 设置key
/home/redis/bin/redis-cli -h 192.168.1.60 -p 6379 -a chefu666 -n 0 SET 'key' 'value'

# 删除key
/home/redis/bin/redis-cli -h 192.168.1.60 -p 6379 -a chefu666 -n 0 DEL 'key'
```

### MongoDB配置

```bash
# MongoDB连接信息（任货行部分数据）
主机: 192.168.1.60
端口: 27017
数据库: jdbdb
```

## SSH服务器信息

```bash
# 测试服务器SSH连接
主机: 192.168.1.60
用户: root
密码: fenmi123
MCP服务器名: test
```

## 编写测试用例时的查询流程

### 1. 确定系统和数据库

根据功能模块确定对应的系统和数据库:

```markdown
- 任通行功能 → rtx 库, 代码路径: java/rtx/
- 任货行功能 → hcb 库, 代码路径: java/hcb/
- 支付相关 → fenmi_pay 库, 代码路径: java/fenmi/
```

### 2. 拉取最新代码

**⚠️ 重要**: 编写测试用例前先拉取最新代码（只拉取 `test` 分支）

```bash
# 任通行 - 直接拉取 test 分支
cd java/rtx
git checkout test
git pull origin test

# 任货行 - 进入子目录逐个拉取 test 分支
cd java/hcb
for dir in */; do
  echo "Pulling test branch in $dir"
  cd "$dir" && git checkout test && git pull origin test && cd ..
done

# 分米支付 - 直接拉取 test 分支
cd java/fenmi
git checkout test
git pull origin test
```

### 3. 分析代码逻辑

```bash
# 1. 找到相关的 Controller
# 例: 查找账单相关的 Controller
grep -r "BillController" java/hcb/

# 2. 查看 Service 业务逻辑
# 例: 阅读账单解析的 Service
cat java/hcb/hcb-timer-bill/src/.../BillAnalyzeService.java

# 3. 查看 Mapper SQL
# 例: 查看数据库操作的 SQL
cat java/hcb/hcb-timer-bill/src/.../BillMapper.xml
```

### 4. 查询表结构

```bash
# 使用SSH MCP工具查询
ssh_db_query(
  server="test",
  type="mysql",
  database="rtx",
  query="DESC 表名",
  dbUser="root",
  dbPassword="fm123456",
  dbHost="192.168.1.60",
  dbPort=3306
)
```

### 5. 查询业务数据

```bash
# 查询现有数据了解业务逻辑
ssh_db_query(
  server="test",
  type="mysql",
  database="rtx",
  query="SELECT * FROM 表名 WHERE 条件 LIMIT 10",
  dbUser="root",
  dbPassword="fm123456",
  dbHost="192.168.1.60",
  dbPort=3306
)
```

### 6. 查看服务器日志

```bash
# 任通行日志
ssh_execute(server="test", command="tail -n 100 /home/java-server/rtx-app/log.txt")

# 任货行日志
ssh_execute(server="test", command="tail -n 100 /home/tomcat/tomcat-hcbapi/logs/catalina.out")
```

## 常用测试数据准备SQL模板

### ⭐ 常用测试账号数据

**编写测试用例时，优先使用以下测试数据查询相关信息：**

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

-- 查询任通行用户（按手机号）
SELECT * FROM rtx_truckuser WHERE mobile = '15818376788';

-- 查询任货行车辆（按身份证）
SELECT * FROM hcb_vehicle WHERE id_card = '441622199602075195';

-- 查询订单记录
SELECT * FROM tb_order WHERE mobile = '15818376788' ORDER BY create_time DESC LIMIT 10;

-- 查询充值记录
SELECT * FROM tb_recharge_record WHERE mobile = '15818376788' ORDER BY create_time DESC LIMIT 10;
```

**⚠️ 重要**: 
- 编写测试用例时，先用这些数据查询测试环境是否已有相关记录
- 如果没有数据，可以基于这些信息准备测试数据
- 保证测试用例的可执行性和可复现性

---

### 任通行用户准备

```sql
-- 查询用户
SELECT * FROM rtx_truckuser WHERE car_num='黑A113355' LIMIT 1;

-- 准备用户数据
INSERT INTO rtx_truckuser (id, car_num, ...) VALUES (...);
```

### 任货行账单准备

```sql
-- 查询账单
SELECT * FROM hcb_deduct_record_extend_mtk WHERE car_num='贵H86206' LIMIT 1;

-- 准备账单数据
INSERT INTO hcb_deduct_record_extend_mtk (
  id, create_time, update_time, bill_date, 
  deduct_bill_id, operator_code, car_num, 
  bill_amount, pay_status
) VALUES (
  8990840457371287800, NOW(), NOW(), '2026-01-19',
  'HCB20260119001', 'MTK', '贵H86206',
  1.00, 0
);
```

## 测试用例前置条件编写规范

在编写测试用例的前置条件时，必须明确:

### 必须包含的信息

```markdown
- **前置条件**:
  - **系统**: 任通行APP / 任货行APP / 管理后台
  - **数据库**: rtx / hcb / fenmi_pay
  - **场景**: 新用户申办 / 已有用户操作 / 定时任务执行
  - **配置**: 渠道编码、运营商代码等
  - **测试数据**: 具体的车牌号、用户ID等
  - **环境**: 测试环境 (192.168.1.60)
```

### 示例

```markdown
#### TC-001 任通行充值-支付宝支付
- **前置条件**:
  - **系统**: 任通行APP
  - **数据库**: rtx
  - **场景**: 已登录用户充值
  - **用户数据**: 车牌号 黑A113355, user_id=1001
  - **账户余额**: 10元
  - **充值金额**: 100元
  - **支付方式**: 支付宝
  - **环境**: 测试环境 192.168.1.60
```

## 注意事项

1. **⚠️ 分支管理**: 编写测试用例只拉取 `test` 分支（测试环境分支），不拉取其他分支
2. **⚠️ 任货行拉取**: 任货行需要进入每个子目录逐个拉取，不能只在父目录拉取
3. **数据库名称**: 不同系统使用不同数据库,编写测试用例前务必确认
4. **日志路径**: 任通行和任货行的日志路径格式不同
5. **代码路径**: 任通行在 `/home/java-server/`, 任货行在 `/home/tomcat/`
6. **账单文件**: 任货行运营商账单文件路径和格式各不相同
7. **数据编码**: 江苏TXB使用UTF-8,其他运营商使用GBK
8. **分表规则**: 部分表按月分表(如 `_202601`), 注意表名后缀

## 快速参考

**拉取代码**:
- 任通行: `cd java/rtx && git checkout test && git pull origin test`
- 任货行: `cd java/hcb && for dir in */; do cd "$dir" && git checkout test && git pull origin test && cd ..; done`
- 分米支付: `cd java/fenmi && git checkout test && git pull origin test`

**测试任通行功能**: 数据库 `rtx`, 日志 `/home/java-server/rtx-app/log.txt`  
**测试任货行功能**: 数据库 `hcb`, 日志 `/home/tomcat/tomcat-hcbapi/logs/catalina.out`  
**查询表结构**: `mysql -h 192.168.1.60 -u root -pfm123456 库名 -e 'DESC 表名;'`  
**查看日志**: `ssh_execute(server="test", command="tail -n 100 日志路径")`
