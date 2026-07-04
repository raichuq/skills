# Torn API 编程指南

> **Skill 用途**: 指导 AI 正确、高效地调用 Torn API 进行编程。
> **创建日期**: 2026-07-04
> **参考数据**: Torn API 官方文档 + OpenAPI Spec v6.0.0 + torn-pda-master + torntools-extension 9.0.5
> **时效性**: 使用前请先检查 OpenAPI 规范是否有更新（见第 9 节）

---

## 1. 概述

### 1.1 基本信息

Torn API 提供只读接口，用于获取玩家、派系、公司等的游戏数据。

| 项目 | V1 | V2 |
|------|----|----|
| **Base URL** | `https://api.torn.com/` | `https://api.torn.com/v2/` |
| **OpenAPI Spec** | 无 | `https://www.torn.com/swagger/openapi.json` |
| **文档页** | `https://www.torn.com/api.html` | 同上（Swagger 链接） |

### 1.2 核心原则

```
原则 1: V2 优先 — Torn 正在向 V2 迁移，能用 V2 就不用 V1
原则 2: 能合并的请求尽量合并，减少请求次数
原则 3: 使用最低权限的 API Key (Public > Minimal > Limited > Full)
原则 4: 必须处理错误码，不能假定请求总是成功
原则 5: 遵守速率限制 (100 req/min/user)
```

### 1.3 速率限制

- **100 次/分钟/用户**（跨所有 API Key）
- 超限返回错误码 `5`（Too many requests）
- 使用无效 Key 可能导致 IP 临时封禁
- 建议实现请求队列控制频率

### 1.4 缓存

**服务缓存**（30 秒）：
- 相同的连续请求返回相同数据
- 可绕过：添加 `timestamp` 查询参数（使用当前 Unix 时间戳）
- `comment` 参数**不会**绕过缓存
- 命中缓存的请求不计入配额

**全局缓存**（不可绕过）：
- 以下 selection 全局缓存，所有用户获取相同数据：
  - `market` → `itemmarket`
  - `market` → `properties`
  - `market` → `rentals`
  - `company` → `companies`
  - `user` → `bazaar`
  - `torn` → `bounties`
  - `user` → `bounties`

---

## 2. API Key 与认证

### 2.1 访问级别

| 级别 | 颜色码 | 说明 |
|------|--------|------|
| **Public** | 白色 `#ffffff` | 基础信息、市场数据、全局数据 |
| **Minimal Access** | 绿色 `#b3e6b3` | 个人状态、冷却、旅行、教育 |
| **Limited Access** | 黄色 `#ffdd99` | 攻击、事件、消息、背包、股票 |
| **Full Access** | 红色 `#ff9999` | 日志（log） |

**注意**: V1 和 V2 中某些相同 selection 可能有不同的权限要求（见第 5 节交叉对照）。

### 2.2 认证方式

**V1** — 仅支持查询参数：
```
GET https://api.torn.com/user/?selections=profile&key=16CHARKEY
```

**V2** — 支持两种方式：
```
# 方式 1: Authorization 请求头（推荐）
Authorization: ApiKey 16CHARKEY

# 方式 2: 查询参数
GET https://api.torn.com/v2/user/basic?key=16CHARKEY
```

### 2.3 自定义密钥

可以在 Torn 网站上生成自定义密钥，精确选择所需的 selections。
- 自定义密钥默认只能访问 `default`、`timestamp`、`lookup` selection
- 需要手动勾选其他需要的 selections
- **自定义密钥应视为 Full Access 密钥**——谨慎分享

### 2.4 Key Info 端点（检查密钥权限）

```
# V1
GET https://api.torn.com/key/?selections=info&key=YOUR_KEY
# 或 V2（推荐）
GET https://api.torn.com/v2/key/info
Authorization: ApiKey YOUR_KEY
```

返回示例：
```json
{
  "key": "16CHARKEY...",
  "access_type": "Limited",
  "access_level": 3,
  "selections": ["profile", "bars", "cooldowns", ...]
}
```

`access_level` 值：1=Public, 2=Minimal, 3=Limited, 4=Full

### 2.5 通用参数

| 参数 | 适用 | 说明 |
|------|------|------|
| `comment` | V1+V2 | 备注信息，显示在 API 使用日志中。用于标注工具/服务来源 |
| `timestamp` | V1+V2 | Unix 时间戳，用于绕过服务缓存 |
| `striptags` | V2 | `true`/`false`，是否去除返回文本中的 HTML 标签 |

---

## 3. API V1 完整参考

### 3.1 请求格式

```
GET https://api.torn.com/{section}/{ID?}?selections={sel1,sel2,...}&key={KEY}&comment={COMMENT}&limit={N}&from={TS}&to={TS}
```

| 参数 | 必需 | 说明 |
|------|------|------|
| `section` | 是 | 类别: `user`, `faction`, `company`, `market`, `property`, `torn`, `key` |
| `ID` | 否 | 可选，不传时查询自己/默认数据 |
| `selections` | 是 | 逗号分隔的 selection 列表 |
| `key` | 是 | 16 字符 API Key |
| `comment` | 否 | 日志标注 |
| `limit` | 否 | 返回行数限制（用于 list 类数据） |
| `from` | 否 | 起始 Unix 时间戳 |
| `to` | 否 | 结束 Unix 时间戳 |

### 3.2 User 端点

所有 User selection 在 V1 中：
```
GET https://api.torn.com/user/{ID?}?selections=sel1,sel2&key=KEY
```

| Selection | 权限 | V2 也有? | 说明 |
|-----------|------|----------|------|
| `profile` | Public | ✅ | 个人资料（状态、生命、等级等） |
| `basic` | Public | ✅ | 基本信息（无 profile 详细） |
| `bazaar` | Public | ✅ | 摊位物品列表 |
| `bounties` | Public | ✅ | 悬赏信息 |
| `cooldowns` | Minimal | ✅ | 冷却时间 |
| `bars` | Minimal | ✅ | 能量/生命/神经条 |
| `education` | Minimal | ✅ | 教育信息 |
| `events` | Limited | ✅ | 事件列表 |
| `attacks` | Limited | ✅ | 攻击记录 |
| `attacksfull` | Limited | ✅ | 简版攻击记录 |
| `messages` | Limited | ✅ | 消息 |
| `inventory` | Limited | ✅ | 背包物品 |
| `stocks` | Limited | ✅ | 股票持仓 |
| `money` | Limited | ✅ | 资金信息 |
| `networth` | Limited | ✅ | 净资产 |
| `newevents` | Limited | ✅ | 未读事件 |
| `newmessages` | Limited | ✅ | 未读消息 |
| `log` | Full | ✅ | 日志（仅 V1? 实际 V2 也有） |
| `personalstats` | Public | ✅ | 个人统计 |
| `travel` | Minimal | ✅ | 旅行信息 |
| `crimes` | Minimal | ✅ | 犯罪 |
| `honors` | Minimal | ✅ | 荣誉 |
| `merits` | Minimal | ✅ | 功绩 |
| `perks` | Minimal | ❌ | 特权 |
| `icons` | Public | ✅ | 图标状态 |
| `discord` | Public | ❌ | Discord 绑定信息 |
| `criminalrecord` | Public | ❌ | 犯罪记录 |
| `display` | Public | ❌ | 展示物品 |
| `jobpoints` | Minimal | ✅ | 工作点数 |
| `workstats` | Minimal | ✅ | 工作统计 |
| `skills` | Minimal | ✅ | 技能 |
| `weaponexp` | Minimal | ✅ | 武器经验 |
| `items` | Public | ❌ | 注意: V1 user items 是物品信息 |
| `gym` | Minimal | ❌ | 健身房信息 |
| `property` | Public | ✅ | 当前房产 |
| `properties` | Public | ✅ | 名下所有房产 |
| `revives` | Minimal | ✅ | 复活记录 |
| `revivesfull` | Minimal | ✅ | 简版复活记录 |
| `missions` | Minimal | ✅ | 任务 |
| `notifications` | Minimal | ✅ | 通知 |
| `refills` | Minimal | ✅ | 补充次数 |
| `travel` | Minimal | ✅ | 旅行 |
| `medals` | Minimal | ✅ | 奖章 |
| `equipment` | Minimal | ✅ | 装备/衣物 |
| `ammo` | Minimal | ✅ | 弹药 |

### 3.3 Faction 端点

```
GET https://api.torn.com/faction/{ID?}?selections=sel1,sel2&key=KEY
```

| Selection | 权限 | V2 也有? | 说明 |
|-----------|------|----------|------|
| `basic` | Public | ✅ | 派系基本信息 |
| `chain` | Public | ✅ | 当前 chain |
| `chains` | Public | ✅ | 历史 chains |
| `hof` | Public | ✅ | 荣誉堂 |
| `members` | Public | ✅ | 成员列表 |
| `territory` | Public | ✅ | 领土 |
| `rankedwars` | Public | ✅ | 排名战 |
| `attacks` | Limited | ✅ | 攻击记录 |
| `attacksfull` | Limited | ✅ | 简版攻击记录 |
| `contributors` | Limited | ✅ | 贡献者（需 `&stat=NAME`） |
| `crimes` | Minimal | ✅ | 有组织犯罪 |
| `applications` | Minimal | ✅ | 申请列表 |
| `news` | Minimal | ✅ | 新闻 |
| `positions` | Minimal | ✅ | 职位 |
| `upgrades` | Minimal | ✅ | 升级 |
| `stats` | Minimal | ✅ | 挑战统计 |
| `armor` | Minimal | ❌ | 装甲库 |
| `weapons` | Minimal | ❌ | 武器库 |
| `medical` | Minimal | ❌ | 医疗库 |
| `drugs` | Minimal | ❌ | 毒品库 |
| `boosters` | Minimal | ❌ | 增幅库 |
| `temporary` | Minimal | ❌ | 临时物品库 |
| `caches` | Minimal | ❌ | 缓存库 |
| `currency` | Limited | ❌ | 资金明细 |
| `donations` | Limited | ❌ | 捐款 |
| `reports` | Limited | ✅ | 报告 |

### 3.4 Company 端点

```
GET https://api.torn.com/company/{ID?}?selections=sel1,sel2&key=KEY
```

| Selection | 权限 | V2 也有? | 说明 |
|-----------|------|----------|------|
| `profile` | Public | ✅ | 公司资料 |
| `employees` | Public | ✅ | 雇员列表 |
| `companies` | Public | ✅ | 公司类型列表 |
| `applications` | Limited | ✅ | 申请 |
| `news` | Minimal | ✅ | 新闻 |
| `stock` | Limited | ✅ | 公司库存 |
| `detailed` | Limited | ❌ | 详细信息 |

### 3.5 Market 端点

```
GET https://api.torn.com/market/{ID?}?selections=sel1,sel2&key=KEY
```

| Selection | 权限 | V2 也有? | 说明 |
|-----------|------|----------|------|
| `bazaar` | Public | ✅ | 摊位目录 |
| `itemmarket` | Public | ✅ | 物品市场（全局缓存） |
| `pointsmarket` | Public | ❌ | 点数市场 |
| `properties` | Public | ✅ | 房产市场（全局缓存） |
| `rentals` | Public | ✅ | 租赁市场（全局缓存） |

### 3.6 Property 端点

```
GET https://api.torn.com/property/{ID}?selections=sel1,sel2&key=KEY
```

| Selection | 权限 | V2 也有? | 说明 |
|-----------|------|----------|------|
| `property` | Public | ✅ | 房产信息 |

### 3.7 Torn（全局信息）端点

```
GET https://api.torn.com/torn/{ID?}?selections=sel1,sel2&key=KEY
```

| Selection | 权限 | V2 也有? | 说明 |
|-----------|------|----------|------|
| `items` | Public | ✅ | 物品数据库 |
| `stocks` | Public | ✅ | 股票数据库 |
| `education` | Public | ✅ | 教育信息 |
| `medals` | Public | ✅ | 奖章 |
| `honors` | Public | ✅ | 荣誉 |
| `bank` | Public | ❌ | 银行利率 |
| `cityshops` | Public | ❌ | 城市商店 |
| `companies` | Public | ✅ | 公司类型 |
| `gyms` | Public | ❌ | 健身房 |
| `stats` | Public | ✅ | 统计数据 |
| `properties` | Public | ✅ | 房产类型 |
| `territory` | Public | ✅ | 领土 |
| `bounties` | Public | ✅ | 悬赏（全局缓存） |
| `organisedcrimes` | Public | ❌ | 有组织犯罪 |
| `pawnshop` | Public | ❌ | 当铺 |
| `raids` | Public | ❌ | 突袭 |
| `rankedwars` | Public | ✅ | 排名战 |
| `searchforcash` | Public | ❌ | 寻找现金 |
| `shoplifting` | Public | ❌ | 入店行窃 |
| `pokertables` | Public | ❌ | 牌桌 |
| `cards` | Public | ❌ | 卡牌 |
| `competition` | Public | ✅ | 竞赛 |
| `factiontree` | Public | ✅ | 派系技能树 |
| `factionhof` | Public | ✅ | 派系荣誉堂 |
| `logcategories` | Public | ✅ | 日志类别 |
| `logtypes` | Public | ✅ | 日志类型 |
| `itemstats` | Public | ❌ | 物品统计 |

### 3.8 Key 端点

```
GET https://api.torn.com/key/?selections=info&key=KEY    # 密钥信息
GET https://api.torn.com/key/?selections=log&key=KEY     # 密钥使用日志
```

---

## 4. API V2 完整参考

### 4.1 请求格式

```
# 无 ID（查询自己的数据）
GET https://api.torn.com/v2/{section}/{selection}?key=KEY&selections=a,b&legacy=x,y

# 有 ID（查询特定目标的数据）
GET https://api.torn.com/v2/{section}/{id}/{selection}?key=KEY&selections=...

# 无具体 selection（通用端点）
GET https://api.torn.com/v2/{section}?selections=a,b,c&key=KEY
```

**V2 特性总结**:
- 每个 selection 有专属端点（如 `/user/basic`, `/user/bars`）
- ID 作为路径参数（如 `/user/{id}/basic`）
- `selections` 查询参数可指定多个 V2 selection
- `legacy` 查询参数可包含 V1 兼容的 selection
- 认证支持 `Authorization: ApiKey XXX` 请求头（推荐）

### 4.2 User 端点 (V2)

| 端点 | 权限 | 说明 |
|------|------|------|
| `GET /user/basic` | **Public** | 自己的基本信息 |
| `GET /user/{id}/basic` | **Public** | 指定用户的基本信息 |
| `GET /user/profile` | **Public** | 自己的详细资料 |
| `GET /user/{id}/profile` | **Public** | 指定用户的详细资料 |
| `GET /user/personalstats` | **Public** | 自己的个人统计 |
| `GET /user/{id}/personalstats` | **Public** | 指定用户的个人统计 |
| `GET /user/faction` | **Public** | 自己的派系信息 |
| `GET /user/{id}/faction` | **Public** | 指定用户的派系信息 |
| `GET /user/job` | **Public** | 自己的工作信息 |
| `GET /user/{id}/job` | **Public** | 指定用户的工作信息 |
| `GET /user/icons` | **Public** | 自己的图标 |
| `GET /user/{id}/icons` | **Public** | 指定用户的图标 |
| `GET /user/hof` | **Public** | 自己的荣誉堂 |
| `GET /user/{id}/hof` | **Public** | 指定用户的荣誉堂 |
| `GET /user/competition` | **Public** | 自己的竞赛信息 |
| `GET /user/{id}/competition` | **Public** | 指定用户的竞赛信息 |
| `GET /user/discord` | **Public** | 自己的 Discord 信息 |
| `GET /user/{id}/discord` | **Public** | 指定用户的 Discord |
| `GET /user/forumposts` | **Public** | 自己的论坛帖子 |
| `GET /user/{id}/forumposts` | **Public** | 指定用户的论坛帖子 |
| `GET /user/forumthreads` | **Public** | 自己的论坛主题 |
| `GET /user/{id}/forumthreads` | **Public** | 指定用户的论坛主题 |
| `GET /user/properties` | **Public** | 自己的房产 |
| `GET /user/{id}/properties` | **Public** | 指定用户的房产 |
| `GET /user/property` | **Public** | 当前房产 |
| `GET /user/{id}/property` | **Public** | 指定用户的当前房产 |
| `GET /user/bounties` | **Public** | 自己的悬赏 |
| `GET /user/{id}/bounties` | **Public** | 指定用户的悬赏 |
| `GET /user/ammo` | **Minimal** | 弹药信息 |
| `GET /user/bars` | **Minimal** | 状态条 |
| `GET /user/calendar` | **Minimal** | 日历 |
| `GET /user/cooldowns` | **Minimal** | 冷却 |
| `GET /user/{crimeId}/crimes` | **Minimal** | 犯罪统计 |
| `GET /user/education` | **Minimal** | 教育 |
| `GET /user/enlistedcars` | **Minimal** | 登记赛车 |
| `GET /user/equipment` | **Minimal** | 装备 |
| `GET /user/jobpoints` | **Minimal** | 工作点数 |
| `GET /user/jobranks` | **Minimal** | 工作职位 |
| `GET /user/honors` | **Minimal** | 成就荣誉 |
| `GET /user/medals` | **Minimal** | 奖章 |
| `GET /user/merits` | **Minimal** | 功绩 |
| `GET /user/missions` | **Minimal** | 任务 |
| `GET /user/notifications` | **Minimal** | 通知 |
| `GET /user/organizedcrime` | **Minimal** | 当前 OC |
| `GET /user/organizedcrimes` | **Minimal** | 招募中的 OC |
| `GET /user/races` | **Minimal** | 比赛 |
| `GET /user/racingrecords` | **Minimal** | 比赛记录 |
| `GET /user/refills` | **Minimal** | 补充 |
| `GET /user/skills` | **Minimal** | 技能 |
| `GET /user/travel` | **Minimal** | 旅行 |
| `GET /user/virus` | **Minimal** | 病毒编码 |
| `GET /user/weaponexp` | **Minimal** | 武器经验 |
| `GET /user/workstats` | **Minimal** | 工作统计 |
| `GET /user/forumfeed` | **Minimal** | 论坛动态 |
| `GET /user/forumfriends` | **Minimal** | 好友动态 |
| `GET /user/forumsubscribedthreads` | **Minimal** | 订阅的主题 |
| `GET /user/itemmods` | **Public** | 可用物品模组 |
| `GET /user/attacks` | **Limited** | 攻击记录（支持 `from`/`to`/`filters`/`limit`） |
| `GET /user/attacksfull` | **Limited** | 简版攻击记录 |
| `GET /user/battlestats` | **Limited** | 战斗统计 |
| `GET /user/casino` | **Limited** | 赌场信息 |
| `GET /user/events` | **Limited** | 事件 |
| `GET /user/inventory` | **Limited** | 背包 |
| `GET /user/itemmarket` | **Limited** | 物品市场挂单 |
| `GET /user/list` | **Limited** | 好友/敌人/目标列表 |
| `GET /user/messages` | **Limited** | 消息 |
| `GET /user/money` | **Limited** | 资金 |
| `GET /user/newevents` | **Limited** | 未读事件 |
| `GET /user/newmessages` | **Limited** | 未读消息 |
| `GET /user/reports` | **Limited** | 报告 |
| `GET /user/revives` | **Limited** | 复活记录 |
| `GET /user/revivesFull` | **Limited** | 简版复活记录 |
| `GET /user/stocks` | **Limited** | 股票 |
| `GET /user/trades` | **Limited** | 交易 |
| `GET /user/{tradeId}/trade` | **Limited** | 交易详情 |
| `GET /user/log` | **Full** | 日志 |

### 4.3 Faction 端点 (V2)

| 端点 | 权限 | 说明 |
|------|------|------|
| `GET /faction/basic` | **Public** | 自己派系基本信息 |
| `GET /faction/{id}/basic` | **Public** | 指定派系基本信息 |
| `GET /faction/chain` | **Public** | 当前 chain |
| `GET /faction/{id}/chain` | **Public** | 指定派系 chain |
| `GET /faction/chains` | **Public** | 历史 chains |
| `GET /faction/{id}/chains` | **Public** | 指定派系历史 chains |
| `GET /faction/chainreport` | **Public** | 最新 chain 报告 |
| `GET /faction/{chainId}/chainreport` | **Public** | 指定 chain 报告 |
| `GET /faction/hof` | **Public** | 荣誉堂 |
| `GET /faction/{id}/hof` | **Public** | 指定派系荣誉堂 |
| `GET /faction/members` | **Public** | 成员列表 |
| `GET /faction/{id}/members` | **Public** | 指定派系成员 |
| `GET /faction/rackets` | **Public** | 当前 rackets |
| `GET /faction/rankedwars` | **Public** | 排名战历史 |
| `GET /faction/{id}/rankedwars` | **Public** | 指定派系排名战历史 |
| `GET /faction/raids` | **Public** | 突袭历史 |
| `GET /faction/{id}/raids` | **Public** | 指定派系突袭历史 |
| `GET /faction/territory` | **Public** | 领土 |
| `GET /faction/{id}/territory` | **Public** | 指定派系领土 |
| `GET /faction/territorywars` | **Public** | 领土战历史 |
| `GET /faction/{id}/territorywars` | **Public** | 指定派系领土战 |
| `GET /faction/wars` | **Public** | 战争和协议 |
| `GET /faction/{id}/wars` | **Public** | 指定派系战争 |
| `GET /faction/warfare` | **Public** | 派系战争 |
| `GET /faction/search` | **Public** | 搜索派系 |
| `GET /faction/territoryownership` | **Public** | 领土所有权 |
| `GET /faction/{raidWarId}/raidreport` | **Public** | 突袭报告 |
| `GET /faction/{rankedWarId}/rankedwarreport` | **Public** | 排名战报告 |
| `GET /faction/{territoryWarId}/territorywarreport` | **Public** | 领土战报告 |
| `GET /faction/applications` | **Minimal** | 申请列表 |
| `GET /faction/crimes` | **Minimal** | 有组织犯罪 |
| `GET /faction/{crimeId}/crime` | **Minimal** | 指定犯罪 |
| `GET /faction/news` | **Minimal** | 新闻 |
| `GET /faction/positions` | **Minimal** | 职位配置 |
| `GET /faction/stats` | **Minimal** | 挑战统计 |
| `GET /faction/upgrades` | **Minimal** | 升级 |
| `GET /faction/attacks` | **Limited** | 攻击记录 |
| `GET /faction/attacksfull` | **Limited** | 简版攻击记录 |
| `GET /faction/balance` | **Limited** | 资金余额 |
| `GET /faction/contributors` | **Limited** | 挑战贡献者 |
| `GET /faction/reports` | **Limited** | 报告 |
| `GET /faction/revives` | **Limited** | 复活记录 |
| `GET /faction/revivesFull` | **Limited** | 简版复活记录 |

### 4.4 Company 端点 (V2)

| 端点 | 权限 | 说明 |
|------|------|------|
| `GET /company/profile` | **Public** | 自己的公司资料 |
| `GET /company/{id}/profile` | **Public** | 指定公司资料 |
| `GET /company/employees` | **Public** | 自己的雇员 |
| `GET /company/{id}/employees` | **Public** | 指定公司的雇员 |
| `GET /company/{typeId}/companies` | **Public** | 指定类型的公司列表 |
| `GET /company/search` | **Public** | 搜索公司 |
| `GET /company/snapshot` | **Public** | 公司每日快照 |
| `GET /company/news` | **Minimal** | 新闻 |
| `GET /company/applications` | **Limited** | 申请 |
| `GET /company/stock` | **Limited** | 公司库存 |

### 4.5 Market 端点 (V2)

| 端点 | 权限 | 说明 |
|------|------|------|
| `GET /market/bazaar` | **Public** | 摊位目录 |
| `GET /market/{id}/bazaar` | **Public** | 指定物品的摊位 |
| `GET /market/{id}/itemmarket` | **Public** | 物品市场（全局缓存） |
| `GET /market/{propertyTypeId}/properties` | **Public** | 房产市场（全局缓存） |
| `GET /market/{propertyTypeId}/rentals` | **Public** | 租赁市场（全局缓存） |
| `GET /market/auctionhouse` | **Public** | 拍卖行列表 |
| `GET /market/{id}/auctionhouse` | **Public** | 指定物品拍卖 |
| `GET /market/{id}/auctionhouselisting` | **Public** | 拍卖行 listing |

### 4.6 Property 端点 (V2)

| 端点 | 权限 | 说明 |
|------|------|------|
| `GET /property/{id}/property` | **Public** | 房产信息 |

### 4.7 Torn 端点 (V2)

| 端点 | 权限 | 说明 |
|------|------|------|
| `GET /torn/items` | **Public** | 物品数据库 |
| `GET /torn/{ids}/items` | **Public** | 指定物品 |
| `GET /torn/{id}/itemdetails` | **Public** | 物品详情 |
| `GET /torn/itemammo` | **Public** | 弹药信息 |
| `GET /torn/itemmods` | **Public** | 武器升级 |
| `GET /torn/stocks` | **Public** | 股票数据库 |
| `GET /torn/{stockId}/stocks` | **Public** | 指定股票+图表 |
| `GET /torn/education` | **Public** | 教育数据库 |
| `GET /torn/medals` | **Public** | 奖章 |
| `GET /torn/{ids}/medals` | **Public** | 指定奖章 |
| `GET /torn/honors` | **Public** | 荣誉 |
| `GET /torn/{ids}/honors` | **Public** | 指定荣誉 |
| `GET /torn/merits` | **Public** | 功绩 |
| `GET /torn/calendar` | **Public** | 日历 |
| `GET /torn/crimes` | **Public** | 犯罪信息 |
| `GET /torn/{crimeId}/subcrimes` | **Public** | 子犯罪信息 |
| `GET /torn/organizedcrimes` | **Public** | 有组织犯罪 |
| `GET /torn/bounties` | **Public** | 悬赏（全局缓存） |
| `GET /torn/properties` | **Public** | 房产类型 |
| `GET /torn/territory` | **Public** | 领土 |
| `GET /torn/hof` | **Public** | 荣誉堂 |
| `GET /torn/factionhof` | **Public** | 派系荣誉堂 |
| `GET /torn/factiontree` | **Public** | 派系技能树 |
| `GET /torn/competition` | **Public** | 竞赛 |
| `GET /torn/attacklog` | **Public** | 攻击日志 |
| `GET /torn/logcategories` | **Public** | 日志类别 |
| `GET /torn/logtypes` | **Public** | 日志类型 |
| `GET /torn/{logCategoryId}/logtypes` | **Public** | 指定类别的日志类型 |
| `GET /torn/elimination` | **Public** | 淘汰赛 |
| `GET /torn/{id}/eliminationteam` | **Public** | 淘汰赛队伍 |

### 4.8 Racing 端点 (V2 only 🆕)

| 端点 | 权限 | 说明 |
|------|------|------|
| `GET /racing/cars` | **Public** | 车辆及赛车属性 |
| `GET /racing/carupgrades` | **Public** | 车辆升级 |
| `GET /racing/races` | **Public** | 比赛列表 |
| `GET /racing/{raceId}/race` | **Public** | 比赛详情 |
| `GET /racing/{trackId}/records` | **Public** | 赛道记录 |
| `GET /racing/tracks` | **Public** | 赛道信息 |

### 4.9 Forum 端点 (V2 only 🆕)

| 端点 | 权限 | 说明 |
|------|------|------|
| `GET /forum/categories` | **Public** | 论坛分类 |
| `GET /forum/threads` | **Public** | 所有主题 |
| `GET /forum/{categoryIds}/threads` | **Public** | 指定分类的主题 |
| `GET /forum/{threadId}/thread` | **Public** | 主题详情 |
| `GET /forum/{threadId}/posts` | **Public** | 主题帖子 |

### 4.10 Key 端点 (V2)

| 端点 | 权限 | 说明 |
|------|------|------|
| `GET /key/info` | **Public** | 密钥信息 |
| `GET /key/log` | **Public** | 密钥使用日志 |

---

## 5. V1 ↔ V2 交叉对照

### 5.1 V1-only 的选择（V2 中没有，需要继续使用 V1）

以下是只在 V1 中可用、V2 中没有对应端点的 selection：

| 类别 | V1 Selection | 说明 |
|------|-------------|------|
| user | `discord` | Discord 信息（V2 中已有 `GET /user/discord`—确认） |
| user | `criminalrecord` | 犯罪记录 |
| user | `display` | 展示物品 |
| user | `gym` | 健身房 |
| user | `perks` | 特权 |
| user | `items` | 物品信息（V2 用 `/torn/items`） |
| faction | `armor` | 装甲库 |
| faction | `weapons` | 武器库 |
| faction | `medical` | 医疗库 |
| faction | `drugs` | 毒品库 |
| faction | `boosters` | 增幅库 |
| faction | `temporary` | 临时物品库 |
| faction | `caches` | 缓存库 |
| faction | `currency` | 资金明细 |
| faction | `donations` | 捐款 |
| company | `detailed` | 公司详细信息 |
| market | `pointsmarket` | 点数市场 |
| torn | `bank` | 银行利率 |
| torn | `cityshops` | 城市商店 |
| torn | `gyms` | 健身房列表 |
| torn | `organisedcrimes` | 有组织犯罪（V2 有 `/torn/organizedcrimes`） |
| torn | `pawnshop` | 当铺 |
| torn | `raids` | 突袭 |
| torn | `searchforcash` | 寻找现金 |
| torn | `shoplifting` | 入店行窃 |
| torn | `pokertables` | 牌桌 |
| torn | `cards` | 卡牌 |
| torn | `itemstats` | 物品统计 |

### 5.2 V2-only 的选择（V1 中没有）

| 类别 | V2 端点 | 说明 |
|------|---------|------|
| 所有 Racing | `GET /racing/*` | 赛车系统全部 V2-only |
| 所有 Forum | `GET /forum/*` | 论坛系统全部 V2-only |
| user | `GET /user/list` | 好友/敌人/目标列表（V1 为 `friends` / `enemies`） |
| user | `GET /user/casino` | 赌场信息 |
| user | `GET /user/competition` | 竞赛 |
| user | `GET /user/{id}/forumposts` | 公共论坛帖子查询 |
| user | `GET /user/{id}/forumthreads` | 公共论坛主题查询 |
| user | `GET /user/forumfeed` | 论坛动态 |
| user | `GET /user/forumfriends` | 好友论坛动态 |
| user | `GET /user/forumsubscribedthreads` | 订阅的主题 |
| user | `GET /user/itemmarket` | 物品市场挂单 |
| user | `GET /user/{id}/hof` | 公共荣誉堂查询 |
| user | `GET /user/{id}/properties` | 公共房产查询 |
| user | `GET /user/{crimeId}/crimes` | 犯罪统计 |
| user | `GET /user/organizedcrime` | 当前 OC |
| user | `GET /user/organizedcrimes` | 招募中的 OC |
| user | `GET /user/races` | 比赛 |
| user | `GET /user/racingrecords` | 比赛记录 |
| user | `GET /user/enlistedcars` | 登记车辆 |
| user | `GET /user/virus` | 病毒编码 |
| user | `GET /user/jobranks` | 工作职位 |
| user | `GET /user/itemmods` | 装备模组 |
| user | `GET /user/{tradeId}/trade` | 交易详情 |
| faction | `GET /faction/search` | 搜索派系 |
| faction | `GET /faction/{id}/members` | 公共成员查询 |
| faction | `GET /faction/{id}/wars` | 公共战争查询 |
| faction | `GET /faction/{id}/hof` | 公共荣誉堂 |
| faction | `GET /faction/{id}/chain` | 公共 chain 查询 |
| faction | `GET /faction/{id}/chains` | 公共 chains 查询 |
| faction | `GET /faction/{id}/rankedwars` | 公共排名战查询 |
| faction | `GET /faction/{crimeId}/crime` | 指定犯罪详情 |
| faction | `GET /faction/rackets` | Rackets 列表 |
| faction | `GET /faction/warfare` | 派系战争 |
| faction | `GET /faction/territorywars` | 领土战 |
| faction | `GET /faction/territoryownership` | 领土所有权 |
| faction | `GET /faction/{rankedWarId}/rankedwarreport` | 排名战报告 |
| faction | `GET /faction/{raidWarId}/raidreport` | 突袭报告 |
| faction | `GET /faction/{territoryWarId}/territorywarreport` | 领土战报告 |
| company | `GET /company/search` | 搜索公司 |
| company | `GET /company/snapshot` | 公司快照 |
| market | `GET /market/auctionhouse` | 拍卖行 |
| market | `GET /market/{id}/bazaar` | 指定物品摊位 |
| torn | `GET /torn/{ids}/items` | 指定物品查询 |
| torn | `GET /torn/{id}/itemdetails` | 物品详情 |
| torn | `GET /torn/{stockId}/stocks` | 指定股票+图表 |
| torn | `GET /torn/{ids}/medals` | 指定奖章 |
| torn | `GET /torn/{ids}/honors` | 指定荣誉 |
| torn | `GET /torn/calendar` | 日历信息 |
| torn | `GET /torn/crimes` | 犯罪信息 |
| torn | `GET /torn/{crimeId}/subcrimes` | 子犯罪 |
| torn | `GET /torn/elimination` | 淘汰赛 |
| torn | `GET /torn/{id}/eliminationteam` | 淘汰赛队伍 |
| torn | `GET /torn/{logCategoryId}/logtypes` | 日志类型筛选 |
| torn | `GET /torn/factiontree` | 派系技能树 |
| torn | `GET /torn/attacklog` | 攻击日志 |

### 5.3 两者都有但权限不同的选择 ⚠️

一些 selection 在 V1 和 V2 中有不同的访问级别要求：

| Selection | V1 权限 | V2 权限 | 注意 |
|-----------|---------|---------|------|
| `user` → `crimes` | Minimal | Minimal | ✅ 一致 |
| `user` → `revives` | Minimal | **Limited** | ⚠️ V2 要求更高 |
| `user` → `revivesfull` | Minimal | **Limited** | ⚠️ V2 要求更高 |
| `faction` → `crimes` | Minimal | Minimal | ✅ 一致 |
| `faction` → `revives` | Minimal | **Limited** | ⚠️ V2 要求更高 |
| `faction` → `revivesfull` | Minimal | **Limited** | ⚠️ V2 要求更高 |
| `faction` → `reports` | Limited | Limited | ✅ 一致 |
| `company` → `employees` | Public | Public | ✅ 一致 |
| `company` → `applications` | Limited | Limited | ✅ 一致 |

### 5.4 两者都有但返回格式不同 ⚠️

| Selection | V1 格式 | V2 格式 | 应对 |
|-----------|---------|---------|------|
| `user` → `attacks` | 嵌套对象 keyed by ID | 数组 | 遍历方式不同 |
| `user` → `events` | 嵌套对象 keyed by ID | 数组 | 遍历方式不同 |
| `user` → `messages` | 嵌套对象 keyed by ID | 数组 | 遍历方式不同 |
| `user` → `inventory` | 嵌套对象 keyed by ID | 数组 | 遍历方式不同 |
| `user` → `attacksfull` | 嵌套对象 keyed by ID | 数组 | 遍历方式不同 |
| `faction` → `attacks` | 嵌套对象 keyed by ID | 数组 | 遍历方式不同 |
| `faction` → `members` | 嵌套对象 keyed by ID | 数组 | 遍历方式不同 |

**重要**: V1 的列表数据通常以 `{ "ID": {...}, "ID2": {...} }` 格式返回，而 V2 通常返回 `[{...}, {...}]` 数组格式。编写代码时需要根据使用的 API 版本选择正确的解析方式。

### 5.5 迁移建议

```
优先迁移到 V2 的选择:
  ✅ market/* (V2 结构清晰)
  ✅ torn/* (V2 有更多端点)
  ✅ user/basic, user/profile (V2 有公共查询)
  ✅ faction/basic, faction/members (V2 有公共查询)

继续使用 V1 的选择:
  ⏳ user/discord (V1 可合并请求)
  ⏳ faction/currency, donations (V1-only)
  ⏳ torn/bank, cityshops, pawnshop (V1-only)
```

---

## 6. 请求合并策略

### 6.1 V1 合并（逗号分隔 selections）

V1 的合并非常直接——通过 `selections` 参数传入逗号分隔的列表：

```
# ❌ 不推荐：分开请求
GET /user/?selections=profile&key=KEY
GET /user/?selections=bars&key=KEY
GET /user/?selections=cooldowns&key=KEY

# ✅ 推荐：合并为一个请求
GET /user/?selections=profile,bars,cooldowns&key=KEY
```

**torn-pda-master 的实际合并示例**:
```dart
// 合并 profile, bars, networth, cooldowns, notifications, travel, icons, money, education, messages
url += 'user/?selections=profile,bars,networth,cooldowns,notifications,travel,icons,money,education,messages';

// 合并 money, travel
url += 'user/?selections=money,travel';

// 合并 profile, battlestats, bars
url += 'user/?selections=profile,battlestats,bars';
```

**合并原则**:
- 同类数据尽量合并（User 与 User 合并，不要跨类别）
- 一次请求最多获取所需数据，不要多取
- 利用 `limit` 参数控制返回数量

### 6.2 V2 合并（selections 参数 + legacy 参数）

V2 支持两种合并方式：

**方式 1：`selections` 参数合并多个 V2 选择**

```
GET /v2/user?selections=profile,bars,cooldowns,money,events&key=KEY
```

这个方式使用通用端点 `/{section}`，通过 `selections` 参数指定多个 selection。适用于属于同一个 category 的多个选择。

**方式 2：`legacy` 参数合并 V1 兼容选择**

```
GET /v2/user?selections=attacks,events&legacy=profile,bars,cooldowns&key=KEY
```

`legacy` 参数用于那些还没有独立 V2 端点、或者你想用 V1 方式获取的 selection。

**torntools-extension 的实际合并模式** (JavaScript):
```javascript
// 使用 selections 和 legacySelections
const data = await fetchData("tornv2", {
    section: "user",
    selections: ["attacks", "events"],           // V2-native selections
    legacySelections: ["profile", "bars", "cooldowns"],  // V1-compat selections
    params: { cat: "all", timestamp: Date.now() / 1000 }
});
// 最终 URL 变为:
// https://api.torn.com/v2/user?selections=attacks,events&legacy=profile,bars,cooldowns&key=KEY
```

**合并原则（V2）**:
- 如果所有需要的数据都有 V2 专属端点，使用 `selections` 参数合并
- 如果需要的数据部分是 V1-only，使用 `legacy` 参数
- 如果只需要单个选择，直接使用专用端点 `GET /v2/{section}/{selection}`

### 6.3 合并最佳实践（来自参考项目）

1. **同类数据一次获取**：torn-pda-master 将 `profile,bars,cooldowns,money,travel,icons` 等经常一起使用的数据合并为一个请求
2. **分离高频和低频数据**：高频更新的数据（bars, cooldowns）和低频数据（education, stocks）分开请求
3. **设置合适的 limit**：对事件、消息等有默认 100 的限制，按需调整
4. **使用 timestamp 绕过缓存**：需要最新数据时，添加 `timestamp` 参数
5. **添加 comment**：标注请求来源，方便调试

---

## 7. 错误码处理

### 7.1 完整错误码表

Torn API 返回的错误格式：
```json
{ "error": { "code": 0, "error": "Error message" } }
```

| 代码 | 名称 | 含义 | 处理策略 |
|------|------|------|---------|
| 0 | Unknown error | 未处理的服务端错误 | 记录日志，可重试 1-2 次 |
| 1 | Key is empty | 请求中未传入 API Key | 检查代码：确保 Key 已配置并正确传递 |
| 2 | Incorrect Key | Key 格式错误（不是 16 字符） | 提示用户重新生成 API Key |
| 3 | Wrong type | 请求了错误的基本类型 | 检查 section 名称是否正确 |
| 4 | Wrong fields | 请求了不正确的 selection 字段 | 检查 selection 拼写是否正确 |
| 5 | Too many requests | 超过 100 次/分钟限制 | **等待 60 秒后重试**，建议实现请求队列控制频率 |
| 6 | Incorrect ID | ID 值错误 | 检查传入的用户/派系/物品 ID |
| 7 | Incorrect ID-entity relation | 请求了其他用户的私有数据 | 用户只能查看自己的私有数据，公共数据需使用公共端点 |
| 8 | IP block | IP 因滥用被临时封禁 | **停止请求，等待一段时间**（建议 60s+），检查是否有无效 Key 请求 |
| 9 | API disabled | API 系统暂时关闭 | 提示用户 Torn API 可能正在维护，稍后再试 |
| 10 | Key owner is in federal jail | Key 主人在联邦监狱 | 提示用户出狱后才能使用 |
| 11 | Key change error | 每 60 秒只能修改一次 Key | 操作太频繁，等待后重试 |
| 12 | Key read error | 从数据库读取 Key 出错 | 服务端问题，可重试 |
| 13 | Key temporarily disabled | 所有者 >7 天未上线 | 提示用户上线激活 Key |
| 14 | Daily read limit reached | 今日云服务拉取记录过多 | 等待第二天重置，或优化请求频率 |
| 15 | Temporary error | 测试用错误 | 忽略或记录，找到稳定复现条件可报告 |
| 16 | Access level too low | Key 权限不足 | **提示用户升级 API Key 权限**，或改用更低权限要求的 selection |
| 17 | Backend error | 服务端错误 | 等待后重试，如果持续则报告 |
| 18 | API key paused | Key 被所有者暂停 | 提示用户在 Torn 设置中恢复 Key |
| 19 | Must migrate to crimes 2.0 | 需要迁移到 crimes 2.0 | 提示用户在游戏中升级 |
| 20 | Race not finished | 比赛尚未结束 | 等待比赛结束 |
| 21 | Incorrect category | `cat` 值错误 | 检查传入的 category 参数 |
| 22 | Only available in API v1 | 该选择仅 V1 可用 | **切换到 V1 API** |
| 23 | Only available in API v2 | 该选择仅 V2 可用 | **切换到 V2 API** |
| 24 | Closed temporarily | 暂时关闭 | 该功能暂时不可用，稍后再试 |
| 25 | Invalid stat requested | 请求了无效的统计信息 | 检查 `stat` 参数值 |
| 26 | Only category or stats can be requested | 只能请求 category 或 stats | 调整请求参数，不要同时传 |
| 27 | Must migrate to organized crimes 2.0 | 需要迁移到 OC 2.0 | 提示用户升级 |
| 28 | Incorrect log ID | 日志 ID 错误 | 检查 log ID |
| 29 | Category not available for interaction logs | 该类别不适用于交互日志 | 调整日志类别 |
| 30 | File does not exist | 文件不存在 | 检查请求的文件 ID |

### 7.2 参考项目的错误处理模式

**torn-pda-master 的错误处理** (Dart):
```dart
// 1. JSON 解析错误
try { jsonResponse = json.decode(response.body); }
catch (e) { return ApiError(errorId: 101, pdaErrorDetails: "..."); }

// 2. API 返回的 Torn 错误
if (jsonResponse['error'] != null) {
    final code = jsonResponse['error']['code'];
    final reason = jsonResponse['error']['error'];
    return ApiError(errorId: code, tornErrorDetails: reason);
}

// 3. HTTP 状态码错误
if (response.statusCode != 200) { ... }

// 4. 超时错误
on TimeoutException catch (_) { return ApiError(errorId: 100); }

// 5. 其他网络错误
catch (e) { return ApiError(pdaErrorDetails: "API CALL ERROR [$e]"); }
```

**torntools-extension 的错误处理** (JavaScript):
```javascript
// 1. DOMException（请求取消/超时）
if (result instanceof DOMException) { code = CUSTOM_API_ERROR.CANCELLED; }

// 2. TypeError（网络错误）
if (result.constructor.name === "TypeError") {
    if (error === "Failed to fetch") {
        if (!hasOrigins(url)) code = CUSTOM_API_ERROR.NO_PERMISSION;
        else code = CUSTOM_API_ERROR.NO_NETWORK;
    }
}

// 3. Torn API 错误
if (isTornAPICall(location)) {
    error = result.error.error;    // Torn 错误消息
    online = (result.error.code !== 9 && !(result instanceof HTTPException));
    // code 9 = API disabled，此时不在线
}
```

### 7.3 需要实现的错误处理层次

建议在代码中实现以下层次的错误处理：

```
第一层: HTTP/网络错误 → 检查网络连接，超时重试
第二层: JSON 解析错误 → 验证响应格式，记录原始响应
第三层: Torn 错误码 → 按上表策略处理每个错误码
第四层: 业务逻辑错误 → 数据格式不正确、字段缺失等
```

**关键错误码处理原则**:
- **错误码 5**（请求过多）：必须实现等待后重试，建议请求队列
- **错误码 8**（IP 封禁）：必须停止请求，较长时间后恢复
- **错误码 16**（权限不足）：向用户明确提示所需的权限等级
- **错误码 22/23**（V1/V2 互斥）：自动切换 API 版本
- **错误码 10/13/18**（Key 状态问题）：给用户清晰的引导信息

---

## 8. 参考项目模式总结

### 8.1 torn-pda-master (Flutter)

**项目特点**: Android/iOS 应用，同时使用 V1 和 V2

**API 调用架构**:
```
ApiCallsV1.getProfile()
  → ApiCallerController.enqueueApiCall(apiSelection: ApiSelection_v1.sel)
    → _launchApiCall_v1()  # 构造 URL、发送 HTTP 请求、解析响应
      → JSON decode → 错误检查 → 返回数据或 ApiError

ApiCallsV2.getMarketItem()
  → ApiCallerController.enqueueApiCall(apiSelection_v2: ..., apiCall: ...)
    → _launchApiCall_v2()  # 创建 TornV2 客户端、调用方法、解析响应
      → 使用 Chopper 客户端 + Authorization header
```

**关键设计**:
- 请求队列系统：`_callQueue` 队列 + `_checkQueue` 定时器（每秒检查）
- 队列条件：当近 60 秒内请求数接近 `maxCallsAllowed`（95）时入队
- 统一的 `ApiError` 类：包含 `errorId`、`errorReason`、`tornErrorDetails`、`pdaErrorDetails`
- 错误历史记录：保留最近 30 条错误，按 V1/V2 分类
- 使用 `comment` 参数标注 `"PDA-App"`
- V2 使用 `source-app: torn-pda` 请求头

### 8.2 torntools-extension (Chrome Extension)

**项目特点**: Chrome 扩展，主要使用 V2 API

**API 调用架构**:
```
fetchData("tornv2", { section, selections, legacySelections, params })
  → 构造 URL: FETCH_PLATFORMS.tornv2 + path + ?selections=...&legacy=...&key=...
  → fetch() + AbortController (10s 超时)
  → JSON 解析 → handleError() 或 resolve()
```

**关键设计**:
- 统一的 `fetchData()` 函数支持多平台（tornv2, yata, tornstats 等）
- `selections` + `legacySelections` 的 V2 特有合并模式
- 错误分类为 `NO_NETWORK`, `NO_PERMISSION`, `CANCELLED`
- 10 秒硬超时（`AbortController`）
- 离线检测：`isTornAPICall()` + `api.torn.online` 状态跟踪
- Badge 显示错误状态
- 38 种平台支持（tornv2, yata, ffscouter 等）

### 8.3 共同最佳实践

1. **请求频率控制**：两个项目都实现了请求频率管理
2. **错误分类**：至少分 3 类（网络错误、Torn 错误、解析错误）
3. **超时控制**：15 秒（torn-pda）/ 10 秒（torntools）
4. **缓存策略**：利用 `timestamp` 参数按需绕过缓存
5. **Key 管理**：支持多个 Key、备用 Key
6. **日志记录**：使用 `comment` 参数标注请求来源
7. **最少数据原则**：只请求需要的数据

---

## 9. 时效性检查

### 9.1 本 Skill 的时效性

- **创建/更新日期**: 2026-07-04
- **OpenAPI 版本**: v6.0.0
- **参考数据**: Torn API 官方文档页面 + OpenAPI 规范 + 两个活跃的开源项目

### 9.2 使用前检查清单

由于 Torn API 经常更新，在调用 Torn API 编写代码前请检查：

```
□ 检查 OpenAPI 规范是否有更新:
   GET https://www.torn.com/swagger/openapi.json
   比较 info.version 是否是 v6.0.0 或更新

□ 检查 API 文档页面是否有更新:
   https://www.torn.com/api.html
   特别注意新增的 selections、废弃的 endpoints、权限变更

□ 检查 V1/V2 迁移进度:
   是否有新的 selection 从 V1-only 变为 V2 可用
   是否有旧的 V1 selection 被废弃

□ 检查 Patch Notes:
   https://www.torn.com/api.html# (Patch Notes 部分)
```

### 9.3 更新时的重点关注

当发现 API 更新时，重点检查以下内容：
1. **新 endpoints**：是否有新的 V2 endpoints 可用
2. **权限变化**：某些 selection 的访问级别是否改变
3. **废弃通知**：是否有 V1 selection 标记为废弃
4. **错误码变更**：是否有新的错误码或现有错误码含义变更
5. **返回格式变化**：V1 到 V2 的迁移进度

---

## 10. 快速参考速查

### V1 快速调用

```python
import requests

API_KEY = "your_16_char_key"

# 合并请求：一次调用获取个人资料+状态条+冷却
response = requests.get(
    "https://api.torn.com/user/",
    params={
        "selections": "profile,bars,cooldowns",
        "key": API_KEY,
        "comment": "my-tool"
    }
)
data = response.json()
if "error" in data:
    code = data["error"]["code"]
    # 处理错误...
else:
    profile = data.get("profile", {})
    bars = data.get("bars", {})
```

### V2 快速调用

```python
import requests

API_KEY = "your_16_char_key"
headers = {"Authorization": f"ApiKey {API_KEY}"}

# 使用专用端点
response = requests.get(
    "https://api.torn.com/v2/user/basic",
    headers=headers,
    params={"striptags": "true"}
)

# 合并多个 selections
response = requests.get(
    "https://api.torn.com/v2/user",
    headers=headers,
    params={
        "selections": "attacks,events",
        "legacy": "profile,bars,cooldowns"
    }
)
```

### V2 查询其他用户

```python
# 查询其他用户的公开信息
response = requests.get(
    f"https://api.torn.com/v2/user/{user_id}/basic",
    headers=headers
)
```

### 检查 Key 权限

```python
response = requests.get(
    "https://api.torn.com/v2/key/info",
    headers=headers
)
key_info = response.json()
# key_info.access_type: "Public" | "Minimal" | "Limited" | "Full"
```

### 错误处理模板

```python
def handle_torn_error(data):
    if "error" not in data:
        return None  # 没有错误

    code = data["error"]["code"]
    message = data["error"]["error"]

    if code == 0:    # 未知错误，重试
        return retry_request()
    elif code == 5:  # 请求过多，等待 60s
        time.sleep(60)
        return retry_request()
    elif code == 8:  # IP 封禁，等待 120s
        time.sleep(120)
        return retry_request()
    elif code == 16: # 权限不足
        raise PermissionError(f"需要更高级别的 API Key (当前: {get_key_level()})")
    elif code == 22: # 仅 V1
        return use_v1_instead()
    elif code == 23: # 仅 V2
        return use_v2_instead()
    elif code == 17: # 后端错误，重试
        time.sleep(5)
        return retry_request()
    else:
        log.error(f"Torn API 错误 {code}: {message}")
        return None
```