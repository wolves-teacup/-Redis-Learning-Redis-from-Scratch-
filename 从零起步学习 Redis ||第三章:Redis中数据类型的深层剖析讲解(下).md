# 从零起步学习Redis || 第三章:Redis中数据类型的深层剖析讲解(下)
前言：
昨天讲解了Redis 中String和List的数据类型的相关知识，今天我们来讲解一下剩余的几种常用数据类型：Hash，Set，Zset

## 一、Hash（哈希）
### 底层实现
在底层，Redis 对 Hash 的实现有两种方式，取决于元素数量和 field/value 的长度：
1. ziplist（压缩列表，Redis 3.2 以后是 listpack）
当 Hash 中的键值对数量较少且 field 和 value 都比较短时，Redis 会使用压缩列表来存储。
压缩列表是一块连续的内存区域，内部紧凑存放 `field` 和 `value`。
- 优点：内存占用少，存储紧凑，适合小数据量。
- 缺点：随着元素增多，插入、查找的性能会下降，因为需要线性扫描。
- 默认条件：
  - `hash-max-ziplist-entries`（默认 512）—— 最大 entry 数量
  - `hash-max-ziplist-value`（默认 64 字节）—— 单个 value 最大长度
超过阈值会转成 hashtable。

2. hashtable（哈希表）
当 Hash 中的元素很多或 field/value 较大时，会自动转为哈希表存储。
本质上是一个字典（dict），底层采用哈希表 + 链地址法实现冲突解决。
- 优点：查找、插入、删除的时间复杂度接近 O(1)。
- 缺点：内存占用相对更多。

### 表格对比讲解
| 特性 | ziplist / listpack（压缩列表） | hashtable（哈希表） |
| ---- | ---- | ---- |
| 存储方式 | 一块连续内存，按顺序紧凑存储 field 和 value | 数组 + 哈希函数映射，冲突用链表（或 rehash 时双表） |
| 内存占用 | 紧凑，节省内存 | 额外指针和哈希桶，内存开销大 |
| 查找效率 | 需要遍历，时间复杂度 O(N) | 哈希查找，平均 O(1) |
| 适用场景 | 元素个数少（≤512，默认）且 field/value 较小（≤64 字节） | 元素多或 field/value 较大 |
| 优点 | 占用少、缓存友好 | 查找快、插入删除性能高 |
| 缺点 | 查找效率低，数据一多性能下降 | 内存浪费多，rehash 时开销较大 |
| 应用举例 | 小对象存储，如用户配置 | 大对象存储，如用户信息、购物车数据 |

### Hash 在项目中的应用场景
Redis 的 Hash 特别适合存储对象，比如用户信息、配置、统计数据等。以下给几个常见应用场景：
1. 缓存对象数据（替代 JSON 存储）
例如存储用户信息：
```bash
HSET user:1001 name "Tom" age "18" gender "male"
HGET user:1001 name  # -> "Tom"
HGETALL user:1001    # -> 获取整个对象
```
优点：
- 可以只更新某个字段，不需要整个 JSON 覆盖。
- 内存占用比 JSON/string 更小。

2. 存储计数器或统计信息
适合保存多维度的计数：
```bash
HINCRBY article:1001:stats views 1
HINCRBY article:1001:stats likes 1
HGETALL article:1001:stats  # -> {views: 101, likes: 20}
```
适用场景：文章浏览量、点赞数、商品销量。

3. 存储配置或元数据
例如系统配置：
```bash
HSET system:config site_name "MySite" theme "dark" max_users "1000"
```
修改配置只需要改某个字段，不影响其他部分。

4. Session/Token 存储
存储用户会话信息：
```bash
HSET session:token123 user_id 1001 expire_at 1690000000
```
便于快速读取和更新。

5. 电商购物车
用户购物车：
```bash
HSET cart:1001 product:2001 2   # 商品2001数量为2
HSET cart:1001 product:2002 1
HGETALL cart:1001               # -> {product:2001: 2, product:2002: 1}
```
高效支持单个商品增删改。

## 二、Set
Redis 的 Set 是一个无序、去重的集合，支持集合运算（交集、并集、差集）。

### 底层实现
1. intset（整数集合）
当 Set 里的元素：
- 全部是整数
- 且数量较少（默认 ≤ 512）
Redis 会用 intset 来存储。
intset 本质是一个紧凑的数组，内部元素有序存放，不允许重复。
- 优点：节省内存（紧凑存储，类似 ziplist）；查找时可以用二分搜索 O(log N)
- 缺点：只支持整数；超过数量阈值或混合非整数时会自动转为 hashtable

2. hashtable（字典）
当集合中：
- 元素是字符串（非整数）
- 或者数量超过 `set-max-intset-entries`（默认 512）
Redis 会用哈希表存储，每个元素作为 key，value 为空。
- 优点：查找、插入、删除的平均复杂度 O(1)；支持任意字符串
- 缺点：相比 intset 占用更多内存

### Set 在项目中的应用场景
Redis Set 因为具备去重、无序、集合运算的特性，在实际开发中非常有用。
1. 用户标签、兴趣、喜好
```bash
SADD user:1001:tags "sports" "music" "travel"
SADD user:1002:tags "music" "tech"
SINTER user:1001:tags user:1002:tags   # -> 共同兴趣
```
应用：推荐系统、兴趣匹配。

2. 去重功能
比如存储唯一的访问 IP：
```bash
SADD ip:today "192.168.1.1"
SADD ip:today "192.168.1.2"
SCARD ip:today   # -> 今日独立 IP 数
```
应用：统计 UV（Unique Visitor）

3. 关注/粉丝关系（社交应用）
```bash
SADD user:1001:follow 1002 1003 1004   # 用户1001关注的人
SADD user:1002:fans 1001               # 用户1002的粉丝
```
可以快速求交集：共同好友；可以计算差集：推荐可能认识的人。

4. 黑名单/白名单
```bash
SADD blacklist 1001 1002
SISMEMBER blacklist 1003   # -> 0 (不在黑名单)
```
应用：权限控制、封禁用户。

5. 抽奖/随机推荐
```bash
SADD lottery 101 102 103 104
SRANDMEMBER lottery 1   # 随机抽一个
SPOP lottery            # 抽出并移除
```
应用：抽奖系统、随机内容推荐。

## 三、Zset
### 底层实现
Zset 底层主要有两种结构，和 Hash/Set 类似，也是双结构实现：
1. ziplist（压缩列表，Redis 3.2 后改为 listpack）
当元素个数较少，且 member/score 较小时使用。
元素以 score + member 紧凑存储，按 score 顺序排列。
- 优点：节省内存。
- 缺点：查询复杂度 O(N)，不适合大数据量。
- 触发条件（默认配置）：
  - `zset-max-ziplist-entries` ≤ 128
  - `zset-max-ziplist-value` ≤ 64 字节
超过阈值时会自动转为 skiplist。

2. skiplist（跳表）+ dict
当数据量较大或元素较复杂时，Redis 使用双结构：
- 哈希表（dict）：存储 member → score 映射，支持 O(1) 查找。
- 跳表（skiplist）：按照 score 排序存储，支持范围查询和有序遍历。

### Zset 在项目中的应用场景
Zset 的最大特点就是有序性，在项目中常用于需要排序的数据。
1. 排行榜（经典场景）
```bash
ZADD rank 100 user1
ZADD rank 200 user2
ZADD rank 150 user3
ZREVRANGE rank 0 2 WITHSCORES  # 前三名
```
应用：游戏排行榜、积分榜、热门内容排名。

2. 延时队列 / 定时任务
利用 score 存储任务执行时间戳：
```bash
ZADD delay_queue 1690000000 "task1"
ZADD delay_queue 1690000050 "task2"
ZRANGEBYSCORE delay_queue 0 1690000000   # 取到期任务
```
应用：定时发送消息、延迟执行任务。

3. 消息队列（按优先级）
利用 score 存储任务优先级：
```bash
ZADD queue 1 "task_low"
ZADD queue 10 "task_high"
ZPOPMIN queue   # 取优先级最高的任务
```
应用：异步任务调度。

4. 时间序列数据（日志/点击流）
利用 score 存储时间戳，member 存储事件 ID：
```bash
ZADD click_log 1690000001 "click1"
ZADD click_log 1690000002 "click2"
ZRANGEBYSCORE click_log 1690000000 1690000100
```
应用：网站访问日志、用户行为数据。

5. 推荐系统
存储物品权重：
```bash
ZADD recommend 0.95 "item1"
ZADD recommend 0.87 "item2"
ZREVRANGE recommend 0 9
```
应用：推荐 top10 商品/内容。

6. 去重 + 排序
例如“最近登录的用户”：
```bash
ZADD recent_login 1690001000 "user1"
ZADD recent_login 1690002000 "user2"
ZREVRANGE recent_login 0 9
```
应用：最近活跃用户列表。

### 补充跳表（skiplist）内容
例子：有序集合存储一些数字
假设我们要往跳表里存 `10, 20, 30, 40`，节点高度是随机生成的。最终跳表大概长这样（简化表示）：
```bash
Level 2:   Head → 10 → 30
Level 1:   Head → 10 → 20 → 30 → 40
Level 0:   Head → 10 → 20 → 30 → 40
```

1. 查找（Search）
目标：查找 30
从最高层（Level 2）开始：Head → 10 → 30，找到目标。
如果查找 25：
- Level 2: Head → 10，下一步是 30 > 25，不能走 → 下到 Level 1
- Level 1: 从 10 → 20，下一步是 30 > 25，不能走 → 下到 Level 0
- Level 0: 从 20 → 30，发现 30 > 25，停止 → 说明 25 不存在。
思路：先快跳，再精细走。

2. 插入（Insert）
目标：插入 25
- 先“查找”25，记录每层最后一个小于 25 的节点（我们叫它 `update[]` 数组）：
  - Level 2: 10
  - Level 1: 20
  - Level 0: 20
- 随机生成新节点高度，比如高度=2（Level 0 和 Level 1）。
若随机高度为 `k`，则新节点会出现在第 0 层到第 k-1 层（共 `k` 个层级）
- 更新指针：
在 Level 0 和 Level 1，把 `20.forward` 改指向 `25`，`25.forward` 指向原来的下一个节点。
结果：25 被插入到合适位置。

3. 删除（Delete）
目标：删除 20
- 查找 20，同时记录每层前驱节点（`update[]`）：
  - Level 2: 前驱=10（20 不在这一层）
  - Level 1: 前驱=10
  - Level 0: 前驱=10
- 更新指针：
把前驱的 forward 改为跳过 20，直接指向 30。
如果某一层删完后该层空了，就降低跳表的最大层数。
结果：20 被移除，跳表结构依然有序。

### 总结规律
- 查找：从上层往下逐层跳，找到或确定不存在。
- 插入：查找时顺便记录前驱 → 随机高度 → 修改前驱指针。
- 删除：查找时顺便记录前驱 → 修改前驱指针跳过目标。

## 总结
至此，Redis 中常见的几种数据类型的底层实现就已完成讲解，后续会补充 bitmap 的底层实现。
最后，感谢大家的支持，学习过程中有任何问题都可以在评论区提出，我会及时为大家解答，谢谢大家！
