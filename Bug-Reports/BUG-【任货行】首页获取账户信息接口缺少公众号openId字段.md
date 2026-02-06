# BUG-【任货行】首页获取账户信息接口缺少公众号openId字段

## 1. BUG描述

首页调用获取账户信息接口时，返回的用户信息中缺少 `wxPublicOpenId` 字段，导致前端无法判断用户是否已关注公众号，无法控制【关注公众号】组件的展示。

**影响**：
- ✅ 其他用户信息字段（手机号、昵称、余额等）正常返回
- ❌ 缺少 `wxPublicOpenId` 字段，前端无法判断是否展示【关注公众号】组件
- ❌ 新功能【首页关注公众号】无法正常工作

---

## 2. 复现步骤

### 前置条件
- 测试环境：http://test.chefenxiang.com
- 测试账号：手机号 `15818376788`
- 用户ID：`2bad2f3fd59311f084f1141877512abb`
- 数据库：hcb (192.168.1.60:3306)

### 操作步骤

**Step 1**: 用户登录任货行小程序，进入首页

**Step 2**: 小程序调用获取账户信息接口

```json
POST http://test.chefenxiang.com/hcbapi/app/proxy.action
{
  "userInfoId": "2bad2f3fd59311f084f1141877512abb",
  "dataFlag": 1,
  "relativeurl": "com.hcb.carOwner.getAccountInfo",
  "caller": "chefuAPP",
  "timestamp": 1769737638534,
  "hashcode": "c2e6889a84ca0ba45bef440bee0247ff"
}
```

**Step 3**: 查询数据库确认用户的公众号openId状态

```bash
# 查询用户信息表
mysql -h 192.168.1.60 -u root -pfm123456 hcb -e "
SELECT USERINFO_ID, PHONE, UNIONID, PUBLIC_OPEN_ID 
FROM hcb_userinfo 
WHERE PHONE = '15818376788'"

# 查询公众号用户关联表
mysql -h 192.168.1.60 -u root -pfm123456 hcb -e "
SELECT * FROM hcb_wx_public_user 
WHERE union_id = 'obOtB56pZ-ekARb2PnPLYIr53mUk'"
```

**数据查询结果**：
```
# 用户信息表
USERINFO_ID: 2bad2f3fd59311f084f1141877512abb
PHONE: 15818376788
UNIONID: obOtB56pZ-ekARb2PnPLYIr53mUk
PUBLIC_OPEN_ID: NULL

# 公众号用户关联表
(Empty set)
```

**Step 4**: 观察接口返回结果

---

## 3. 预期结果

接口应该返回用户的公众号openId字段，用于前端判断是否展示【关注公众号】组件。

**预期返回**：
```json
{
  "ret": "1",
  "msg": "请求成功!",
  "params": {
    "userInfoId": "2bad2f3fd59311f084f1141877512abb",
    "openId": "oTpKf4vchzE1-cTZtKxbEB_gKJ_A",
    "phone": "15818376788",
    "nickName": "骆志敏",
    "balance": "0.00",
    "wxPublicOpenId": "",  ✅ 应该有这个字段（即使为空字符串）
    ...
  }
}
```

**预期**：
- ✅ 接口返回包含 `wxPublicOpenId` 字段
- ✅ 当字段为空时，前端展示【关注公众号】组件
- ✅ 当字段有值时，前端不展示【关注公众号】组件

---

## 4. 实际结果

接口返回的JSON中缺少 `wxPublicOpenId` 字段。

**实际返回**：
```json
{
  "ret": "1",
  "msg": "请求成功!",
  "params": {
    "userInfoId": "2bad2f3fd59311f084f1141877512abb",
    "openId": "oTpKf4vchzE1-cTZtKxbEB_gKJ_A",
    "phone": "15818376788",
    "nickName": "骆志敏",
    "userName": "骆志敏",
    "balance": "0.00",
    "status": "1",
    "extendQrUrl": "",
    "channelId": "0000",
    "recomCount": "",
    "rewardAmt": "",
    "applyNum": 0,
    "successApplyNum": 0,
    "alreadAmount": "718.0",
    "hasRealauthen": "1",
    "lastLoginTime": "2026-01-09 09:40:56",
    "email": "",
    "extensQRFlag": "1",
    "headUrl": "",
    "reSign": "",
    "agreementList": ""
    ❌ 缺少 "wxPublicOpenId" 字段
  }
}
```

**实际**：
- ❌ 接口返回中完全没有 `wxPublicOpenId` 字段
- ❌ 前端无法获取用户的公众号openId状态
- ❌ 前端无法判断是否需要展示【关注公众号】组件
- ❌ 新功能无法按照测试用例正常工作

---

**相关测试用例**: `temp/测试用例/【任货行】首页新增快捷关注服务号功能测试用例_V1.0.md`  
**接口配置**: `com.hcb.carOwner.getAccountInfo`
