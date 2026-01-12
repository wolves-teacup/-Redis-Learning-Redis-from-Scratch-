# 从零起步学习Redis || 第二章:Redis中数据类型的深层剖析讲解(上)
**发布时间**：2026-01-09 15:13:31  
**文章标签**：#redis #数据库 #java #后端 #缓存

## 前言
在前文中已经带大家初步了解了Redis，并在本地环境进行了Redis的配置，接下来我会讲解一下Redis中一些常用数据类型的底层实现，并与所学过的知识进行对比解析。

## Redis 高层概念
Redis 中每个键对应一个 `redisObject`（常简称 `robj`），它包含 `type` / `encoding` / `refcount` / `ptr` 等字段，`ptr` 指向真正的数据（字符串时指向 SDS 或整数编码）。换句话说：Redis 外面看到的“字符串”在内存里是一个 `robj` 指向底层的数据结构（SDS 或直接整数）。

```c
typedef struct redisObject {
    unsigned type:4;      // 类型：string, list, hash, set, zset
    unsigned encoding:4;  // 编码：int, embstr, raw ...
    int refcount;         // 引用计数（GC 用）
    void *ptr;            // 指向实际的数据（String 就指向 SDS 或整数）
} robj;
```

## 1. String（字符串）
Redis 的字符串可以存储二进制安全数据（字节数组），最大 512MB。

### 内存组织示意
```
dictEntry （哈希表节点）
├── key -> redisObject(type=string, ptr->SDS("mykey"))
└── val -> redisObject(type=string, encoding=embstr, ptr->SDS("hello"))
```

### 底层实现：SDS（Simple Dynamic String）
SDS 保存长度（len）、已分配空间（alloc）、字符串内容（buf），结构定义如下：
```c
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len;         // 当前字符串长度
    uint8_t alloc;       // buf 分配的总空间
    unsigned char flags; // 标记是哪种 sdshdr 类型
    char buf[];          // 实际数据（字节数组）
};
```

#### SDS 与 C 字符串的对比优势
- 解决 C 字符串不能二进制安全的问题（C 字符串以 `\0` 结尾）
- 获取长度复杂度为 O(1)（C 字符串需遍历）
- 动态扩容更高效，避免频繁 realloc
- 可预分配尾部空间以优化 append 操作

### String 的编码方式
Redis 会根据字符串的内容和长度选择不同的编码方式，以平衡内存占用和性能：
1. **int 编码**：内容是整数时，直接存储 long 值（无需 SDS 结构）
2. **embstr 编码**：长度 ≤ 44 字节的短字符串
   - 单块内存分配：`[ robj | sdshdr | buf ]`
   - 优势：少一次 malloc、内存连续、缓存友好，适合大量短键/短值场景
   - 特性：不可变（修改时会自动转为 raw 编码）
3. **raw 编码**：较长字符串或需频繁修改的字符串
   - 两次内存分配：`[ robj ]` 和 `[ sdshdr | buf ]`（robj 与 SDS 分开存储）
   - 优势：灵活，支持频繁修改；缺点：多一次分配与指针间接访问

### String 存储流程
1. 向 Redis 存入 String 时，key 和 value 都会被包装成 `robj` 对象
2. key 的 `robj` 会计算 hash 值，用作字典索引
3. value 的 `robj` 根据内容选择编码方式：
   - 整数 → int 编码（直接存 long 值）
   - 短字符串（≤44B）→ embstr 编码（robj + SDS 一次分配）
   - 长字符串 → raw 编码（robj 和 SDS 分开两次分配）
4. SDS（sdshdr + buf）负责管理字符串长度和存储实际数据

### 总结
Redis 的字符串本质上是二进制安全的动态字节数组，底层由 SDS 实现，且会根据字符串长度/内容在 raw / embstr / int 三种编码间切换，以节省内存和提升性能。

## 2. List（列表）
### 逻辑概念
Redis List 是有序的、可重复的字符串集合，支持在列表两端（头部/尾部）进行插入/删除操作，也可通过索引访问元素。

### 底层数据结构演变
Redis List 的底层实现并非固定不变，而是根据存储元素的数量和大小动态调整，主要经历三个阶段：
1. 早期：`ziplist`（压缩列表）和 `linkedlist`（双向链表）
2. 当前主流：`quicklist`（快速列表）

### 2.1 ziplist（压缩列表）
`ziplist` 是一种特殊编码的双向链表，基于连续内存空间实现，设计目标是最大程度节约内存。

#### 结构组成
| 字段       | 说明                          |
|------------|-------------------------------|
| zlbytes    | 整个 ziplist 占用的字节数     |
| zltail     | 最后一个元素的偏移量          |
| zllen      | 元素个数                      |
| entryX     | 具体元素（含自身编码格式）    |
| zlend      | 结束标记（1字节，值为 0xFF）  |

#### 特点
- 内存紧凑：所有元素连续存储，无指针开销
- 变长编码：元素前缀记录长度，支持不同长度的字符串和整数
- 双向遍历：每个 entry 记录上一个 entry 的长度，支持反向遍历

#### 优点
- 极度节省内存：适合存储大量小元素
- 缓存友好：连续内存访问，CPU 缓存局部性好

#### 缺点
- 增删改性能差：中间插入/删除元素需移动大量内存（realloc 操作）
- 不适合大元素/多元素：元素过大或过多时，realloc 开销极大
- 连锁更新风险：某 entry 长度变化可能导致后续 entry 的 `pre_entry_len` 字段连锁更新，极端情况下性能急剧下降

#### 使用场景
早期 Redis 版本中，或在 HASH、ZSET 等结构中，当元素数量少且单个元素较小时使用。

### 2.2 linkedlist（双向链表 / adlist）
`linkedlist` 是传统双向链表，Redis 内部具体实现为 `adlist`。

#### 结构定义
```c
typedef struct listNode {
    struct listNode *prev; // 指向上一个节点
    struct listNode *next; // 指向下一个节点
    void *value;           // 存储实际值
} listNode;

typedef struct list {
    listNode *head;        // 链表头
    listNode *tail;        // 链表尾
    unsigned long len;     // 链表长度
    // ... 其他函数指针
} list;
```

#### 特点
- 节点分离：每个节点独立分配内存，通过指针连接
- 双向访问：支持从头尾双向遍历
- 头尾操作高效：头部/尾部插入/删除复杂度为 O(1)

#### 优点
- 插入删除效率高：已知节点位置时，任意位置操作复杂度 O(1)
- 适合大元素：节点独立，对元素大小不敏感

#### 缺点
- 内存开销大：每个节点需额外存储 prev/next 两个指针
- 缓存不友好：节点内存不连续，易导致 CPU 缓存失效

#### 使用场景
早期 Redis 版本中，当 List 元素数量超过 ziplist 阈值或元素过大时，自动转为 linkedlist。

### 2.3 quicklist（快速列表）- 当前主流实现
`quicklist` 是 Redis 3.2 版本后 List 的主要底层实现，结合了 ziplist 的内存效率和 linkedlist 的操作效率。

#### 核心设计
本质是“ziplist 组成的双向链表”：每个 quicklist 节点（`quicklistNode`）存储一个完整的 ziplist，而非单个数据元素。

#### 结构定义
```c
// quicklist 结构体
typedef struct quicklist {
    quicklistNode *head; // 链表头
    quicklistNode *tail; // 链表尾
    unsigned long len;   // 所有 ziplist 包含的元素总数
    unsigned int count;  // quicklistNode 的数量
    int fill : 16;       // ziplist 填充因子（配置项：list-max-ziplist-size）
    unsigned int compress : 16; // 深度压缩（配置项：list-compress-depth）
    // ... 其他字段
} quicklist;

// quicklistNode 结构体（链表节点）
typedef struct quicklistNode {
    struct quicklistNode *prev; // 指向上一个 quicklistNode
    struct quicklistNode *next; // 指向下一个 quicklistNode
    unsigned char *zl;          // 指向存储的 ziplist
    unsigned int sz;            // ziplist 的字节大小
    unsigned int count : 16;    // ziplist 中的元素数量
    unsigned int encoding : 2;  // 编码方式
    unsigned int container : 2; // 是否为原始数据或嵌入的 ziplist
    unsigned int recompress : 1; // ziplist 是否被压缩过
    unsigned int attempted_compress : 1; // 尝试压缩的次数
    unsigned int extra : 10; // 额外空间
} quicklistNode;
```

#### 工作原理
1. quicklist 由多个 quicklistNode 通过指针连接形成双向链表
2. 每个 quicklistNode 内部包含一个 ziplist，实际数据存储在 ziplist 中
3. 通过配置参数控制 ziplist 大小和压缩策略，平衡性能与内存

#### 优点
1. 内存效率高
   - 局部连续性：ziplist 内部元素连续存储，继承内存紧凑优点
   - 减少指针开销：仅 quicklistNode 需 prev/next 指针，大幅降低整体开销
2. 操作效率高
   - 头尾操作 O(1)：直接操作头尾 quicklistNode 中的 ziplist，满则新建、空则删除
   - 中间操作可控：ziplist 大小可配置，中间插入/删除的移动开销有限，支持 ziplist 分裂
3. 高度可配置：通过两个核心参数调整行为
   - `list-max-ziplist-size`：控制每个 ziplist 的大小
     - 正数：表示最多包含的元素个数（如 5 → 每个 ziplist 最多 5 个元素）
     - 负数：表示最大字节阈值（如 -2 → 每个 ziplist 不超过 8KB，默认值）
   - `list-compress-depth`：控制压缩深度（LZF 算法）
     - 0：不压缩（默认）
     - 1：头尾各 1 个节点不压缩，其余压缩
     - 以此类推，数值越大，未压缩节点越多

#### 缺点
ziplist 的连锁更新问题仍可能存在于单个 quicklistNode 内部，但由于 ziplist 大小被限制，影响范围大幅缩小。

## 后续预告
本文主要讲解了 Redis 中 String 和 List 的底层实现，接下来会继续讲解 Set、ZSet 等数据类型的底层实现。

## 结语
学习过程中有任何问题都可以在评论区提出，我会及时为大家解答，感谢大家的支持！
