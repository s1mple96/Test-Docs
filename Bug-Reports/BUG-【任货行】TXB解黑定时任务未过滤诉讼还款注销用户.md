# BUG-【任货行】TXB解黑定时任务未过滤诉讼还款注销用户

## 1. BUG描述

通行宝(TXB)运营商的解黑定时任务，将诉讼还款注销用户（`BLACK_RESOURE='E'`）错误地自动解黑。

**影响**：
- ✅ 正常欠费下黑用户可以正常解黑
- ❌ 诉讼还款注销用户被错误解黑（应该禁止自动解黑，需要人工审核）

---

## 2. 复现步骤

### 前置条件
- 测试环境：http://192.168.1.60:8087/hcbTimer-bill
- 测试车辆：苏CKU316（黄牌）
- 测试用例：TC-024 诉讼还款注销用户-解黑程序不解黑

### 操作步骤

**Step 1**: 准备测试数据

```bash
mysql -h192.168.1.60 -uroot -pfm123456 hcb -e "
UPDATE hcb_truckuser 
SET YTB_BLACK_STATUS = '3',  -- 待解黑
    STATUS = '2',             -- 黑名单
    BLACK_RESOURE = 'E'       -- 注销下黑
WHERE CAR_NUM = '苏CKU316' 
  AND VEHICLECOLOR = '1'"
```

**Step 2**: 查询执行前状态

```bash
mysql -h192.168.1.60 -uroot -pfm123456 hcb -e "
SELECT CAR_NUM, STATUS, BLACK_RESOURE, YTB_BLACK_STATUS 
FROM hcb_truckuser 
WHERE CAR_NUM = '苏CKU316' AND VEHICLECOLOR = '1'"
```

**执行前状态**：
```
CAR_NUM: 苏CKU316
STATUS: 2 (黑名单)
BLACK_RESOURE: E (注销下黑)
YTB_BLACK_STATUS: 3 (待解黑)
```

**Step 3**: 执行 TXB 解黑定时任务

```bash
curl http://192.168.1.60:8087/hcbTimer-bill/test/testTXBBlackTask
```

**Step 4**: 查询执行后状态

```bash
mysql -h192.168.1.60 -uroot -pfm123456 hcb -e "
SELECT CAR_NUM, STATUS, BLACK_RESOURE, YTB_BLACK_STATUS 
FROM hcb_truckuser 
WHERE CAR_NUM = '苏CKU316' AND VEHICLECOLOR = '1'"
```

**执行后状态**：
```
CAR_NUM: 苏CKU316
STATUS: 1 (正常)
BLACK_RESOURE: (空)
YTB_BLACK_STATUS: 2 (已解黑)
```

**Step 5**: 查看日志

```bash
tail -n 100 /home/tomcat/tomcat-hcbTimer-bill/logs/catalina.out | grep "苏CKU316"
```

---

## 3. 预期结果

用户不应被自动解黑，数据保持不变：

```
CAR_NUM: 苏CKU316
STATUS: 2 (黑名单)
BLACK_RESOURE: E (注销下黑)
YTB_BLACK_STATUS: 3 (待解黑) ✅ 保持待解黑
```

**预期**：
- ✅ 诉讼还款注销用户不会被定时任务处理
- ✅ `YTB_BLACK_STATUS` 保持 `'3'`（待解黑）
- ✅ 日志中应有跳过该用户的记录

---

## 4. 实际结果

用户被错误解黑：

```
CAR_NUM: 苏CKU316
STATUS: 1 (正常) ❌ 被改为正常
BLACK_RESOURE: (空) ❌ 被清空
YTB_BLACK_STATUS: 2 (已解黑) ❌ 被改为已解黑
```

**实际**：
- ❌ 用户 `YTB_BLACK_STATUS` 从 `'3'` 变成了 `'2'`（已解黑）
- ❌ 用户 `STATUS` 从 `'2'` 变成了 `'1'`（正常）
- ❌ 用户 `BLACK_RESOURE` 从 `'E'` 变成了空（被清空）
- ❌ 诉讼还款注销用户被错误地自动解黑

---

**日志路径**: `/home/tomcat/tomcat-hcbTimer-bill/logs/catalina.out`
