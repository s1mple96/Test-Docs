# BUG-【任货行】MTK解黑定时任务未过滤诉讼还款注销用户

## 1. BUG描述

蒙通卡(MTK)运营商的解黑定时任务，将诉讼还款注销用户（`BLACK_RESOURE='E'`）错误地自动解黑。

**影响**：
- ✅ 正常欠费下黑用户可以正常解黑
- ❌ 诉讼还款注销用户被错误解黑（应该禁止自动解黑，需要人工审核）

---

## 2. 复现步骤

### 前置条件
- 测试环境：http://192.168.1.60:8087/hcbTimer-bill
- 测试车辆：蒙ZMM777（黄牌）
- 用户钱包ID：7b4bfaa4a8de11f084f1141877512abb
- 测试用例：TC-024 诉讼还款注销用户-解黑程序不解黑

### 操作步骤

**Step 1**: 准备测试数据

```bash
mysql -h192.168.1.60 -uroot -pfm123456 hcb -e "
UPDATE hcb_truckuser 
SET YTB_BLACK_STATUS = '3',  -- 待解黑
    STATUS = '1',             -- 正常
    BLACK_RESOURE = 'E'       -- 注销下黑
WHERE CAR_NUM = '蒙ZMM777' 
  AND VEHICLECOLOR = '1'"
```

**Step 2**: 查询执行前状态

```bash
mysql -h192.168.1.60 -uroot -pfm123456 hcb -e "
SELECT CAR_NUM, STATUS, BLACK_RESOURE, YTB_BLACK_STATUS 
FROM hcb_truckuser 
WHERE CAR_NUM = '蒙ZMM777' AND VEHICLECOLOR = '1'"
```

**执行前状态**：
```
CAR_NUM: 蒙ZMM777
STATUS: 1 (正常)
BLACK_RESOURE: E (注销下黑)
YTB_BLACK_STATUS: 3 (待解黑)
```

**Step 3**: 执行 MTK 解黑定时任务

```bash
curl http://192.168.1.60:8087/hcbTimer-bill/test/testNMBlackFileTask
```

**Step 4**: 查询执行后状态

```bash
mysql -h192.168.1.60 -uroot -pfm123456 hcb -e "
SELECT CAR_NUM, STATUS, BLACK_RESOURE, YTB_BLACK_STATUS 
FROM hcb_truckuser 
WHERE CAR_NUM = '蒙ZMM777' AND VEHICLECOLOR = '1'"
```

**执行后状态**：
```
CAR_NUM: 蒙ZMM777
STATUS: 1 (正常)
BLACK_RESOURE: E (注销下黑)
YTB_BLACK_STATUS: 2 (已解黑)
```

**Step 5**: 查看日志

```bash
tail -n 100 /home/tomcat/tomcat-hcbTimer-bill/logs/catalina.out | grep "蒙ZMM777"
```

---

## 3. 预期结果

用户不应被自动解黑，数据保持不变：

```
CAR_NUM: 蒙ZMM777
STATUS: 1 (正常)
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
CAR_NUM: 蒙ZMM777
STATUS: 1 (正常)
BLACK_RESOURE: E (注销下黑)
YTB_BLACK_STATUS: 2 (已解黑) ❌ 被改为已解黑
```

**日志内容**：
```log
[2026-01-27 11:27:00.632] [hcbTimer-bill] [ERROR] - ============= 进入 trunckerUserBlackTimerTask.MTKBlackTask(); ==========
[2026-01-27 11:27:00.632] [hcbTimer-bill] [ERROR] - 蒙通卡下黑任务开启....2026-01-27 11:27:00
[2026-01-27 11:27:03.640] [hcbTimer-bill] 成功    200
[2026-01-27 11:27:04.392] [hcb-api] [ERROR] - ERROR   - mongodb保存监听器, 收到消息: {"reqParam":"{\"CAR_NUM\":\"蒙ZMM777\",\"STATUS\":\"1\",\"BLACK_RESOURE\":\"E\",\"YTB_BLACK_STATUS\":\"2\",\"BLACK_TIME\":\"2026-01-27 11:27:04\",\"TRUCKUSER_ID\":\"6457f83940ad4501bc2f01474093c8c4\"}","tableName":"LTK_TRUCKUSER_HIS"}
[2026-01-27 11:27:04.642] [hcbTimer-bill] [ERROR] - 蒙通卡下黑任务结束....2026-01-27 11:27:04
```

**实际**：
- ❌ 用户 `YTB_BLACK_STATUS` 从 `'3'` 变成了 `'2'`（已解黑）
- ❌ 诉讼还款注销用户被错误地自动解黑
- ❌ 日志中没有跳过该用户的记录

---

**日志路径**: `/home/tomcat/tomcat-hcbTimer-bill/logs/catalina.out`
