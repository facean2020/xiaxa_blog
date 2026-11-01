+++
date = '2026-11-01T15:00:00+08:00'
draft = false
slug = 'macos-accountsd-high-cpu'
title = '记一次 macOS accountsd 高负载排查：storefront 字段打架引发的连锁风暴'
tags = ['macOS', '调试', '排查', '系统问题']
+++

# 记一次 macOS accountsd 高负载排查：storefront 字段打架引发的连锁风暴

## 问题现象

某天下午发现 MacBook 风扇狂转，打开活动监视器一看，`accountsd` 这个平时几乎看不见的系统守护进程占了 127%～171% 的 CPU，同时 `itunescloudd`（92%）、Music.app（65%）、`commerce`（40%）也都不消停。系统负载一度到了 17，机器肉眼可见地卡顿。

`accountsd` 是 macOS 的账户守护进程（Accounts.framework），负责统一管理 iCloud、邮件、日历、iTunes Store 等各类互联网账户。它平时应该安安静静待在后台，持续高负载几乎可以断定是出了问题。

<!--more-->

## 排查过程

### 第一层：先看谁在烧 CPU

```bash
ps aux | grep -E "accountsd|itunes|commerce" | grep -v grep
```

结果除了 `accountsd`，还有 `itunescloudd`（iTunes Cloud 守护进程）和 Music.app 在同一时间点集体高负载——这不太可能是巧合，应该是一条链路上的连锁反应。

### 第二层：采样看调用栈

用 macOS 自带的 `sample` 对进程采样，看看它到底在忙什么：

```bash
sample <PID> 2 -file /tmp/accountsd_sample.txt
```

调用栈里出现了几个关键线索：

- `AMSAccountNotificationPlugin` 在反复执行 `ams_delta:`（账号属性差异对比）
- `SQLQueue for Accounts4.sqlite` —— 账号数据库在被高频读写
- `com.apple.AMSAccountNotificationPlugin.processingQueue` —— 账号变更通知插件的处理队列

`itunescloudd` 那边则是 `ICUserIdentityStore`、`ICURLBagProvider` 在不停查询账号身份和商店配置。看起来是：**某个组件在反复改账号 → `accountsd` 处理变更通知 → 另一个组件又改回去 → 无限循环**。

### 第三层：日志里的蛛丝马迹

用系统日志工具抓账号相关的记录：

```bash
log show --last 2m --predicate 'process == "accountsd"' --style compact
```

日志里出现了决定性的一行：

```text
AMSAccountNotificationPlugin: The account's storefront ID changed
    from H:8511cabb82 to H:8d4bd4922f.
```

统计了一下频率：**2 分钟内 storefront ID 翻转了 2168 次**，平均每秒约 18 次。每次翻转都会触发一整套账号变更通知——Passbook、IDS、联系人、TV 服务等七八个插件全部被叫醒处理一遍，再触发缓存拆除和 Core Data 大合并。CPU 就是这么烧掉的。

再顺着日志查「谁在保存账号」：

```text
Daemon save called for account ... client=(2098)   ← commerce (CommerceKit)
Daemon save called for account ... client=(2099)   ← accountsd 自己
```

两个写入方在 2 分钟内分别写了 1087 次和 1082 次，势均力敌地互相覆盖。

### 第四层：数据库里的真相

账号数据库在 `~/Library/Accounts/Accounts4.sqlite`（受 macOS 隐私保护，需要「完全磁盘访问权限」才能读）。打开之后发现了问题的核心。

`storefrontID` 属性是一个按媒体类型分键的字典，解码后长这样：

| 记录 | itunes storefront | appstore storefront |
| --- | --- | --- |
| `local` 残留记录 | **143465-2,42** | 143465-19,42 |
| 真实 Apple ID 记录 | **143465-19,42** | 143465-19,42 |

两个记录的 itunes storefront 值就是日志里打架的那对。再直接轮询数据库，亲眼看到值在磁盘上每 2 秒翻转一次：

```text
14:56:36 itunes = 143465-2,42
14:56:42 itunes = 143465-19,42
14:56:44 itunes = 143465-2,42
```

## 根因

问题的完整机制是一个自我强化的反馈环：

```mermaid
flowchart LR
    A["commerce<br/>(CommerceKit)"] -->|"从商店 bag 读到<br/>itunes=143465-2,42"| B["写入账号记录"]
    B --> C["accountsd 通知风暴<br/>Passbook / IDS / 联系人…"]
    C --> D["按另一变体重建 bag 缓存"]
    D --> E["其它组件读到<br/>itunes=143465-19,42"]
    E --> B
```

具体来说：

1. **商店配置缓存（bag）是按 storefront 变体分文件存的**。在 `~/Library/Application Support/com.apple.iTunesCloud/URLBags/` 下，同一个账号同时存在两套配置文件：

   ```text
   ...-143465-19,42-zh_CN-macOS.plist
   ...-143465-2,42-zh_CN-macOS.plist
   ```

2. `commerce`（CommerceKit，负责 App Store / iTunes Store 商店事务）从这两套 bag 里读到不同的 storefront 值，交替写回账号记录。
3. 每次写入都触发 `accountsd` 的 AMS 插件做差异对比（`ams_delta:`）并广播账号变更通知，所有依赖账号的系统组件（广告、家长批准、内容分类、资产下载……）都被叫醒清缓存、重新联网取配置。
4. 新一轮的 bag 重新下载又产生了两个变体的缓存文件，回到第 1 步。

日志里能看到每次翻转时全系统的 AMS 客户端集体清缓存的壮观场面：

```text
askpermissiond       AMSBagNetworkDataSource: The storefront changed. Clearing the bag...
adprivacyd           AMSBagNetworkDataSource: The storefront changed. Clearing the bag...
CategoriesService    AMSBagNetworkDataSource: The storefront changed. Clearing the bag...
amsondevicestoraged  AMSBagNetworkDataSource: The storefront changed. Clearing the bag...
jetpackassetd        Storefront changed. oldStorefront = 143465-19,42 | newStorefront = 143465-2,42
```

这也解释了为什么网络流量异常偏高——十几个守护进程在以秒级频率反复拉取商店配置。

## 诱发条件

复盘下来，三个条件叠加才会触发：

1. **账号同时激活了 App Store 和 iTunes 两个媒体类型**（`activeMediaTypes = [appstore, itunes]`）。两者的 storefront 变体本来就不同（App Store 用 `143465-19,42`，iTunes Store 用 `143465-2,42`），这本身是正常的服务端配置。
2. **系统里残留了旧的 `local` 占位账号记录**（`isLocalAccount = true`，由 `amsaccountsd` 创建，用户名为 `local`）。它的 itunes storefront 值是旧变体，和正式账号记录不一致，成了持续的污染源。
3. **bag 缓存中同时存在两个变体的配置文件**，让 `commerce` 有了「两套真相」可以交替采信。

正常使用中，如果账号只激活一个媒体类型、或 bag 缓存没有分裂成两套，这个环是闭合不上的。我这台机器大概是历史上某次系统迁移 / iTunes 数据转换留下的旧记录，加上 Music.app 一段时间的使用，把三个条件凑齐了。

## 解决方案

按「先止血、再断根」的顺序处理。所有操作前先备份了数据库：

```bash
sqlite3 ~/Library/Accounts/Accounts4.sqlite \
  ".backup ~/Library/Accounts/Accounts4.backup-$(date +%Y%m%d-%H%M%S).sqlite"
```

### 1. 清理残留的 local 占位账号

删掉两条 `isLocalAccount` 的 ghost 记录（含属性、凭证关联行）：

```sql
DELETE FROM ZACCOUNTPROPERTY WHERE ZOWNER IN (1,13);
DELETE FROM ZCREDENTIALITEM WHERE ZACCOUNTIDENTIFIER IN ('<ghost-uuid-1>','<ghost-uuid-2>');
DELETE FROM ZACCOUNT WHERE Z_PK IN (1,13);
```

### 2. 清空分裂的 bag 缓存

让系统重新从服务器取一份干净配置：

```bash
rm -f ~/Library/Application\ Support/com.apple.iTunesCloud/URLBags/*/*
rm -f ~/Library/HTTPStorages/com.apple.{accountsd,amsaccountsd,amsondevicestoraged,itunescloudd}/*
```

单独做这一步，翻转停了约两分钟又复发——因为根源（两个媒体类型各自的 storefront 变体）还在。

### 3. 断根：停用 iTunes 媒体类型

既然这台机器上根本不用 iTunes/Music，把账号的 `activeMediaTypes` 改成只保留 `appstore`：

```python
new_val = plistlib.dumps({
    '$version': 100000, '$archiver': 'NSKeyedArchiver', '$top': {'root': plistlib.UID(1)},
    '$objects': ['$null',
        {'NS.objects': [plistlib.UID(2)], '$class': plistlib.UID(3)},
        'com.apple.AppleMediaServices.accountmediatype.appstore',
        {'$classname': 'NSMutableArray', '$classes': ['NSMutableArray', 'NSArray', 'NSObject']}],
}, fmt=plistlib.FMT_BINARY)
conn.execute("UPDATE ZACCOUNTPROPERTY SET ZVALUE=? WHERE ZOWNER=12 AND ZKEY='activeMediaTypes'", (new_val,))
```

这一步的作用是**去掉 `143465-2,42` 这个变体的同步源**：只有 App Store 一个媒体类型活跃时，`commerce` 只会采信 `143465-19,42` 一套 bag，另一个变体不再产生，环就闭不上了。

> 注意：改数据库前需要给终端授予「完全磁盘访问权限」（系统设置 → 隐私与安全性 → 完全磁盘访问权限），因为 `~/Library/Accounts/` 受 macOS 的 TCC 隐私保护。另外 Accounts 框架的 API 在新版本 macOS 上对第三方脚本已经封锁（JXA 拿不到账号列表），只能走数据库这条路线。

## 验证

修复后观察了 3 分钟以上：

| 指标 | 修复前 | 修复后 |
| --- | --- | --- |
| `accountsd` CPU | 127%～171% | 退出 CPU 榜单 |
| storefront 翻转（3 分钟） | 27653 次 | 2 次 |
| 账号保存（3 分钟） | 2143 次 | 2 次 |
| 系统负载（1 分钟均值） | 17 | 2.5 |

账号记录、TestFlight 账号、Apple Music 订阅信息均完好无损。

## 给 Apple 的反馈

这个问题的性质更像是系统侧的 bug 而不是用户配置错误：

- 两个媒体类型（appstore / itunes）的 storefront 变体不同是**服务端的正常配置**，但 AMS 的账号变更处理会把它当作「字段不一致」来回去纠正；
- bag 缓存按 storefront 变体分裂成多套文件后，没有任何收敛机制；
- 每次翻转的代价极高：全套账号通知插件 + 缓存拆除 + Core Data 合并 + 全系统 AMS 客户端重新拉 bag，还会连带网络风暴。

已经通过 Feedback Assistant 提交了详细报告，希望后续版本能在源头收敛这个环。

## 总结

- `accountsd` 无端高负载，九成是某个账号字段被反复改写导致的通知风暴，`sample` + `log show` 是定位这类问题的利器；
- 账号数据库里的「双记录 / 双变体」是这类打架的典型病灶，`~/Library/Accounts/Accounts4.sqlite` 用 SQLite + plist 解码就能看清真相；
- 修缓存只能止血，真正断根要找到循环的「双源头」并去掉一个；
- 动系统数据库之前，备份、备份、备份。
