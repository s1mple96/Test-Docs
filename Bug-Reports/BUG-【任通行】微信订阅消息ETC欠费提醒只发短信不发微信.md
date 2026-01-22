# BUG-【任通行】微信订阅消息ETC欠费提醒只发短信不发微信

## 【BUG描述】
执行定时任务"微信订阅消息-ETC欠费提醒定时任务"（`WeChatSubscriptionMessageTask.walletAdvanceWxMsgTask()`）时，用户只收到短信通知，未收到微信小程序订阅消息推送。该问题会导致本次需求的核心功能无法实现，影响范围为所有存在支付方式=7（钱包垫付）且账单状态=2（代扣失败）的用户。

## 【前置条件】
- **测试环境**: test环境
- **测试账号**: 
  - 用户ID: 1906909894150443009
  - 手机号: 15818376788
  - 车牌号: 黑A113355
  - 微信OpenId: o9h285Ka3kzx-FTZRUDE9WKt0nvI
- **测试数据**:
  - 账单ID: 2009515094681206786
  - 支付方式: 7（钱包垫付）
  - 账单状态: 2（代扣失败）
  - 钱包余额: -0.04元（欠费）
- **定时任务配置**: 微信订阅消息-ETC欠费提醒定时任务（手动执行）

## 【复现步骤】
1. 登录任通行运营后台 -> 定时任务管理
2. 找到定时任务"微信订阅消息-ETC欠费提醒定时任务"
3. 点击"执行一次"按钮，手动触发任务
4. 观察执行结果：
   - 查看短信是否收到
   - 查看微信小程序是否收到订阅消息
5. 查看日志：`/home/java-server/rtx-quartz/log.txt` 和 `/home/fenmi/fenmi-msg/log.txt`

## 【期望结果】
根据需求文档《【任通行】微信小程序消息的相关优化及问题处理》：
1. ✅ 用户手机收到短信通知：
   - 内容：`【任通行】尊敬的黑A1****5，您的ETC于2026-01-13 11:20:22扣款失败...`
2. ✅ 用户微信小程序收到订阅消息：
   - 模板：ETC欠费提醒
   - 内容：车牌号、欠费金额、还款日期
   - 消息文案：`账号已欠费，请尽快缴费，以免影响通行。`（新文案）
3. ✅ 两种通知同时发送，确保用户能及时收到提醒

## 【实际结果】
1. ✅ 用户手机收到了短信通知（时间：11:20:22）
2. ❌ 用户微信小程序**没有收到**订阅消息
3. ⚠️ 分米消息系统日志显示：
   - 收到了微信消息发送请求
   - 只执行了 `AliSmsHandler`（短信发送）
   - 没有执行 `WechatHandler`（微信消息发送）
4. ⚠️ 任通行系统返回结果：
   ```json
   {
     "code": 200,
     "data": {
       "sendStatus": "SUCCESS",
       "responseMsg": "发送成功"
     },
     "success": false  // 注意这里是 false
   }
   ```

## 【附件】

### 1. 任通行定时任务日志
```log
2026-01-13 11:20:20.452 ERROR - 支付失败账单-发送微信ETC欠费提醒-开始
2026-01-13 11:20:22.084 INFO - sendWxMsg parameter fixParam:WxMsgParam(
  openId=o9h285Ka3kzx-FTZRUDE9WKt0nvI, 
  carNum=黑A113355, 
  paraMap={
    date3=2026-01-14 11:20:22, 
    thing4=账号已欠费，请尽快缴费，以免影响通行。, 
    car_number1=黑A113355, 
    amount2=-0.04
  }
)
2026-01-13 11:20:22.250 ERROR - 发送分米微信消息,返回结果：
{
  "code":200,
  "data":{
    "sendId":"2010914792639414274",
    "sendStatus":"SUCCESS",
    "responseMsg":"发送成功"
  },
  "msg":"操作成功",
  "success":false
}
```

### 2. 分米消息系统日志
```log
2026-01-13 11:20:22.176 [http-nio-9209-exec-8] INFO - 微信消息发送请求参数：
WechatSubscribeMsgRequest{
  templateCode='wx_template_02', 
  sendType=1, 
  userOpenId='o9h285Ka3kzx-FTZRUDE9WKt0nvI',
  bizSceneDesc='黑A113355-账号欠费'
}

2026-01-13 11:20:22.347 [http-nio-9209-exec-1] INFO - AliSmsHandler 执行短信发送
  - templateParam: {"date":"2026-01-13 11:20:22","carNum":"黑A1****5"}
  - template_id: 1714459497609797633

2026-01-13 11:20:23.314 [http-nio-9209-exec-1] INFO - AliSmsHandler success!
2026-01-13 11:20:23.343 [http-nio-9209-exec-1] INFO - sms-ali短信 已成功发送
```

**❌ 关键问题**: 日志中没有 `WechatHandler` 或 `WechatMiniAppHandler` 的执行记录！

### 3. 数据库查询结果

**账单数据**:
```sql
SELECT id, plate_num, pay_type, status FROM rtx_etc_deducrecord_ltk 
WHERE plate_num = '黑A113355' AND pay_type = 7 ORDER BY create_time DESC;

结果:
id: 2009515094681206786
plate_num: 黑A113355
pay_type: 7 (钱包垫付)
status: 2 (代扣失败) ✅ 符合条件
```

**微信模板配置** (更正):
```sql
SELECT * FROM fenmi_admin.wx_msg_template_manage 
WHERE template_code = 'wx_template_02';

结果:
id: 1824713102594080769
template_code: wx_template_02
app_config_code: RTX_XCX
wx_template_id: 3p9XqHmBwE5_qB4DXp5tXsiL3MRMZjr-SIj9PFxcTBM
msg_title: ETC欠费提醒
keyword: car_number1,amount2,date3,thing4
template_status: 1 (启用) ✅ 模板存在且启用
```

**短信模板配置**:
```sql
SELECT * FROM fenmi_admin.msg_template WHERE id = 1714459497609797633;

结果:
id: 1714459497609797633
template_code: SMS_461065152
template_name: 钱包1.0，银行卡代扣或微信代扣失败时触发
channel_code: 10 (短信渠道) ✅ 存在
status: 1 (启用)
```

## 【原因定位】

### 根因分析（三个维度）

#### 1. **代码层面 - sendType参数配置错误**

**问题代码位置**: `java/rtx/rtx-common/src/main/java/com/rtx/common/enums/wx/WxMsgSceneTypeEnum.java`

```java
// 第17行
SCENE_TYPE_3(3, "账号欠费", 1, "moduleService/pay/pay", 
             WeixinMsgTemplate.ETC_OVERDUE_PAYMENT_REMINDER, true),
//                          ↑ sendType=1
```

**代码注释说明**:
```java
/**
 * 发送类型（0:实时,1:延时）延时是指通过定时任务每隔10分钟发送一次
 */
private Integer sendType;
```

**实际行为**: 
- 代码注释说 `sendType=1` 表示"延时发送"
- 但分米消息系统实际将 `sendType=1` 解析为**"仅发送短信"**
- 导致微信消息处理器未被触发

**消息发送流程**:
```java
FenMiWxMessageUtils.sendWxServiceMsg(param, WxMsgSceneTypeEnum.SCENE_TYPE_3)
  ↓
req.setSendType(sceneTypeEnum.getSendType());  // 传入 sendType=1
  ↓
调用分米消息系统 API
  ↓
分米系统根据 sendType=1 判断为"仅发短信"
  ↓
只启动 AliSmsHandler，不启动 WechatHandler
```

#### 2. **配置层面 - sendType导致只发短信** (已更正)

**微信模板配置状态**:
- ✅ 微信模板 `wx_template_02` 存在且启用（`wx_msg_template_manage` 表）
- ✅ 短信模板 `SMS_461065152` 存在且启用（`msg_template` 表）
- ⚠️ **核心问题**: `sendType=1` 导致分米消息系统只处理短信，不处理微信

**分米消息系统处理逻辑**:
```
收到请求: sendType=1, templateCode='wx_template_02'
  ↓
判断 sendType=1 → 只发送短信
  ↓
查找短信模板 SMS_461065152
  ↓
执行 AliSmsHandler ✅
  ↓
跳过 WechatHandler ❌
```

**关键日志证据**:
- 11:20:22.176 收到微信消息请求（http-nio-9209-exec-8 线程）
- 11:20:22.323 开始处理短信（http-nio-9209-exec-1 线程）
- 11:20:23.314 短信发送成功
- ❌ 没有任何微信Handler的执行记录

#### 3. **数据库层面 - 消息发送记录表无微信记录**

```sql
-- 查询该时间段的消息发送记录
SELECT COUNT(*) FROM fenmi_admin.msg_send_record 
WHERE create_time BETWEEN '2026-01-13 11:20:00' AND '2026-01-13 11:25:00';

结果: 0 条记录
```

说明分米消息系统根本没有执行微信消息的发送逻辑，连失败记录都没有产生。

### 核心问题总结

1. **唯一原因**: `sendType=1` 被分米消息系统解析为"仅发短信"
2. **配置正常**: 
   - ✅ 微信模板 `wx_template_02` 在 `wx_msg_template_manage` 表中存在且启用
   - ✅ 短信模板 `SMS_461065152` 在 `msg_template` 表中存在且启用
3. **表象**: 任通行系统返回 `success=false`，但未做异常处理，导致误以为发送成功

## 【修复建议】

### 方案1：修改 sendType 参数（推荐）

**修改位置**: `java/rtx/rtx-common/src/main/java/com/rtx/common/enums/wx/WxMsgSceneTypeEnum.java`

```java
// 修改前
SCENE_TYPE_3(3, "账号欠费", 1, "moduleService/pay/pay", 
             WeixinMsgTemplate.ETC_OVERDUE_PAYMENT_REMINDER, true),

// 修改后（需要确认分米系统的 sendType 枚举值）
SCENE_TYPE_3(3, "账号欠费", 3, "moduleService/pay/pay", 
             WeixinMsgTemplate.ETC_OVERDUE_PAYMENT_REMINDER, true),
//                          ↑ 改为 3（假设3表示短信+微信）
```

**修改步骤**:
1. 先确认分米消息系统的 sendType 枚举值定义：
   - 0 = ?
   - 1 = 仅短信（已确认）
   - 2 = 仅微信？
   - 3 = 短信+微信？
2. 根据确认的枚举值，修改 `SCENE_TYPE_3` 的 sendType 参数
3. 重新编译部署
4. 测试验证

### 方案2：确认 sendType 枚举值定义（必要步骤）

**操作步骤**:
1. 查看分米消息系统源码，确认 sendType 的枚举值定义
2. 或者咨询分米消息系统开发人员
3. 明确以下枚举值的含义：
   ```
   sendType = 0 → ?
   sendType = 1 → 仅短信（已确认）
   sendType = 2 → 仅微信？
   sendType = 3 → 短信+微信？
   ```
4. 根据确认结果修改任通行代码

### 方案3：完善异常处理逻辑

**修改位置**: `java/rtx/rtx-common/src/main/java/com/rtx/common/utils/bank/fenmi/FenMiWxMessageUtils.java`

```java
// 修改 sendWxServiceMsg 方法
public static ApiResult sendWxServiceMsg(...) {
    ApiResult result = sendWxMessage(req);
    
    // 新增：判断 success 字段
    if (result != null && !result.isSuccess()) {
        log.error("发送微信消息失败，返回success=false，详细信息：{}", result);
        // 可以考虑告警或重试
    }
    
    return result;
}
```

### 推荐修复顺序

1. **第一步**: 确认 sendType 枚举值定义（方案2）
2. **第二步**: 修改 sendType 参数（方案1）
3. **第三步**: 完善异常处理逻辑（方案3）

### 补充说明

**微信模板配置已验证**:
```sql
-- 已确认模板存在
SELECT * FROM fenmi_admin.wx_msg_template_manage 
WHERE template_code = 'wx_template_02';

结果:
✅ template_code: wx_template_02
✅ msg_title: ETC欠费提醒
✅ template_status: 1 (启用)
✅ wx_template_id: 3p9XqHmBwE5_qB4DXp5tXsiL3MRMZjr-SIj9PFxcTBM
```

因此，**不需要补充微信模板配置**，只需修改 `sendType` 参数即可。

---

**提交日期**: 2026-01-13  
**提交人**: 测试工程师  
**严重程度**: 高（影响核心功能）  
**优先级**: P0（需求核心功能无法实现）

