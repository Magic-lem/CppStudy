# Redis学习

## 一、Redis基础概念

### 1.1 Redis是什么，有什么特点？

Redis是一个基于**<font color='red'>内存</font>**的数据库，因此**读写速度非常快**，常用作缓存、消息队列、分布式锁和键值存储数据库。支持多种数据结构：<font color='cornflowerblue'>字符串</font>、<font color='cornflowerblue'>哈希表</font>、<font color='cornflowerblue'>列表</font>、<font color='cornflowerblue'>集合</font>、<font color='cornflowerblue'>有序集合</font>、<font color='cornflowerblue'>位图</font>等。内置了<font color='cornflowerblue'>事务、持久化、Lua脚本、多种集群方案（主从复制模式、哨兵模式、切片集群模式）、发布/订阅模式、内存淘汰机制、过期删除机制</font>等。

具有以下特点：

- **基于内存**：数据存储在内存中，具有快速的读写速度；
- **持久性**：可以通过快照和日志文件的持久化来保证数据的持久性，防止数据丢失；
- **多数据结构**：支持丰富的数据结构；
- **原子性操作**：支持原子性操作，能够保证一个操作是原子的，要么执行成功，要么不执行；
- **分布式**：提供了分布式特性，可以将数据分布在多个节点上。

### 1.2 为什么要使用Redis？

<font color='cornflowerblue'> **Redis 具备「高性能」和「高并发」两种特性**</font>。

- Redis读写非常快速（高性能），对于需要频繁读写的数据，与一般MySQL数据库相比（从硬盘中读取），Redis性能优势非常明显。将Redis作为缓存，存储一些MySQL数据库中频繁被使用的数据，可以减轻对MySQL等持久性数据库的压力。

- Redis可以很好地处理高并发请求（高并发），尤其是需要快速响应的场景。这也是它的高性能而决定的。

<img src="G:\code\study\CppStudy\docs\figures\redis作为缓存.png" alt="img" style="zoom:50%;" />

因此，Redis更适合处理**高速、高并发的数据访问**，以及需要**复杂数据结构和功能**的场景。在实际应用中，很多系统同时使用MySQL和Redis，其中Redis作为缓存大幅提高响应速度，减轻数据库的压力。



## 二、Redis 数据类型

### 2.1 Redis有哪些数据类型？

Redis提供了**五种**常见的数据类型：**<font color='cornflowerblue'>String（字符串）、Hash（哈希表）、List（列表）、Set（集合）、Zset（有序集合）</font>**

后面又增加了**四种**数据类型：**<font color='cornflowerblue'>BitMap（位图）、HyperLogLog（基数统计）、GEO（地理信息）、Stream（流）</font>**

<img src="G:\code\study\CppStudy\docs\figures\redis数据结构提纲.png" style="zoom:33%;" />

### 2.2 String 类型

#### 2.2.1 基本介绍

`String`是最基本的key-value结构，value可以不仅仅是字符串，也可以是数字，value可以容纳的数据长度为`512M`。例如：<img src="G:\code\study\CppStudy\docs\figures\string.png" alt="img" style="zoom:50;" />

#### 2.2.2 底层实现

`String`类型的底层实现是`int`和`SDS`（简单动态字符串数据类型）。

`SDS`和C语言字符串不同，它比C语言的字符串适用范围更广、效率更高、也更安全。具有以下特点：

- <font color='cornflowerblue'>`SDS`不仅可以保存文本数据，还可以保存二进制数据。</font>`SDS`所有的API都会以**处理二进制的方式**来处理SDS存放在`buf[]`数组里的数据，所以当然可以直接保存图片、音频、视频、压缩文件等二进制数据。
- <font color='cornflowerblue'>`SDS`获取字符串长度的时间复杂度是O(1)。</font>`SDS`维护了一个`len`属性来记录自身长度，而不是通过尾部加'\0'的方式。（这点和`std::string`一致）
- <font color='cornflowerblue'>`SDS`安全的，拼接字符串不会造成缓冲区溢出。</font>SDS在拼接字符串会检查SDS空间是否满足，不满足则会扩容，不会导致缓冲区溢出。



> 知道了底层数据类型，来看看String类型是如何利用`int`和`SDS`对数据进行构建对象的。

`String`类型对象的内部编码（encoding）有三种方式：<font color='red'>int、raw、embstr</font>。

**情况1：如果`String`对象保存的是一个整数值，且这个整数值可以用`long`类型表示。**

​		字符串对象会将该整数值保存在`ptr`属性中，并将字符串对象的编码类型（encoding）设置为`int`。

​		<img src="G:\code\study\CppStudy\docs\figures\int.png" alt="img" style="zoom:50%;" />

**情况2：如果`String`对象保存的是一个字符串，且这个字符串的长度小于等于32字节。**

​		`String`对象会选择使用`SDS`来保存这个字符串，并将编码类型设置为`embstr`。

​		<img src="G:\code\study\CppStudy\docs\figures\embstr.png" alt="img" style="zoom:50%;" />

**情况3：如果`String`对象保存的是一个字符串，且这个字符串的长度大于32字节。（注意和上面的区别，SDS和redisObject不连续）**

​			`String`对象会选择使用`SDS`来保存这个字符串，且用ptr指向这个`SDS`，并将编码类型设置为`embstr`。

​		<img src="G:\code\study\CppStudy\docs\figures\raw.png" alt="img" style="zoom:50%;" />

> embstr编码和raw编码的边界在redis不同版本是不一样的：
>
> - redis 2.+ 是 32 字节
> - redis 3.0-4.0 是 39 字节
> - redis 5.0 是44字节

`embstr`和`raw`编码方式都会使用`SDS`类型，它们之间的区别就在于字符串的长度。

1️⃣  由于`embstr`中字符串长度较短，只需要通过一次内存分配函数来分配一个**<font color='cornflowerblue'>连续的内存空间</font>**来保存`redisObject`和`SDS`。

2️⃣  而raw编码会调用两次内存分配函数，分配两块空间分别保存`redisObject`和`SDS`。



> **为什么要单独设置一个`embstr`编码方式？**
>
> - `embstr`编码将创建字符串对象所需的内存分配次数从 `raw` 编码的两次降低为一次；
> - 释放 `embstr`编码的字符串对象同样只需要调用一次内存释放函数；
> - 因为`embstr`编码的字符串对象的所有数据都保存在一块连续的内存里面可以更好的利用 CPU 缓存提升性能。

> **`embstr`编码方式缺点？**
>
> 如果字符串的长度增加需要重新分配内存时，整个redisObject和sds都需要重新分配空间，所<font color='cornflowerblue'>以**embstr编码的字符串对象实际上是只读的**</font>，redis没有为embstr编码的字符串对象编写任何相应的修改程序。当我们对embstr编码的字符串对象执行任何修改命令（例如append）时，程序会先将对象的编码从embstr转换成raw，然后再执行修改命令。

#### 2.2.3 常用指令

普通字符串的基本操作：

```shell
# 设置key-value
> SET name magic
OK

# 根据key获得value
> GET name
"magic"

# 判断某个key是否存在
> EXISTS name
(integer) 1

# 返回key所存储的字符串值的长度
> STRLEN name
(integer) 5

# 删除某个key对应的值
> DEL name
(integer) 1
```

字符串批量操作 :

```shell
# 批量设置key-value
> MSET key1 value1 key2 value2
OK

# 批量获取多个key对应的value
> MGET key1 key2
1) "value1"
2) "value2"
```

当value为整数时，可以对其使用以下操作：

```shell
# 设置 key-value 类型的值
> SET number 0
OK
# 将key对应的value + 1
> INCR number
(integer) 1
# 将key对应的value + 10
> INCRBY number 10
(integer) 11
# 将key对应的value - 1
> DECR number
(integer) 10
# 将key对应的value - 10
> DECRBY number 10
(integer) 0
```

设置过期时间相关操作（默认为永不过期）：

```shell
# 设置 key 在 60 秒后过期（为已经存在的key）
> EXPIRE name 60
(integer) 1

# 查看数据还有多久过期
> TTL name
(integer) 58

# 创建key-value时就指定过期时间
> SET key value EX 60
OK
> SETEX key  60 value
OK
```

不存在就插入：

```shell
# 不存在就插入（not exists），常用来实现分布式锁
> SET key value NX  # 或者 SETNX key value
(integer) 1
```

#### 2.2.4 应用场景

- **缓存对象**

  - 直接缓存这个对象的JSON，例如：`SET user:1 '{"name":"xiaolin", "age":18}'`。
  - 利用MSET分离式存储，例如：`MSET user:1:name xiaolin user:1:age 18`。

- **常规计数**

  Redis处理命令是单线程，所以命令的执行是原子的。`String`数据类型适合计数场景，比如计算访问次数、点赞数、转发数、库存量灯。例如：

  ```shell
  # 初始化文章的阅读量
  > SET article:readcount:1001 0
  OK
  > INCR article:readcount:1001	# 阅读量+1
  ```

- **分布式锁**

  SET 命令有个 NX 参数可以实现**「key不存在才插入」**，可以用它来实现分布式锁：

  - 如果 key 不存在，则显示插入成功，可以用来表示加锁成功；
  - 如果 key 存在，则会显示插入失败，可以用来表示加锁失败。

  一般而言，还需要对分布式锁加上过期时间，如下：

  ```shell
  SET lock_key unique_value NX PX 10000
  
  ## lock_key 就是 key 键；
  ## unique_value 是客户端生成的唯一的标识；
  ## NX 代表只在 lock_key 不存在时，才对 lock_key 进行设置操作；（没有被锁时才能加锁）
  ## PX 10000 表示设置 lock_key 的过期时间为 10s，这是为了避免客户端发生异常而无法释放锁。
  ```

  解锁的过程就是将 lock_key 键删除，但不能乱删，要保证执行操作的客户端就是加锁的客户端，即先判断先判断锁的 unique_value 是否为加锁客户端。因此，解锁是两个步骤，**无法保证原子性。**因此Redis需要引入<font color='cornflowerblue'>**`Lua`脚本**</font>来实现原子操作，解锁的`Lua`脚本如下：

  ```lua
  // 释放锁时，先比较 unique_value 是否相等，避免锁的误释放
  if redis.call("get",KEYS[1]) == ARGV[1] then
      return redis.call("del",KEYS[1])
  else
      return 0
  end
  ```

  用户通常需要**编写适当的 Lua 脚本并通过 Redis 客户端发送到 Redis 服务器执行**。这是因为分布式锁的解除需要原子操作，而 Lua 脚本可以保证在 Redis 中的操作是原子的。发送到Redis服务器方法如下：

  ```shell
  ## 后面的1是指有一个键
  > EVAL "if redis.call('get',KEYS[1]) == ARGV[1] then return redis.call('del',KEYS[1]) else return 0 end" 1 lock_key unique_value
  ```

  不过，为了简化这一过程，一些 Redis 客户端库已经内置了处理分布式锁的功能。这些库通常会封装 Lua 脚本的编写和执行过程，使用户可以更方便地使用分布式锁。

- **共享Session信息**

  通常我们在开发后台管理系统时，会使用 Session 来保存用户的会话(登录)状态，这些 Session 信息会被保存在服务器端，但这只适用于单系统应用，如果是<font color='red'>分布式系统此模式将不再适用</font>：例如用户一的 Session 信息被存储在服务器一，但第二次访问时用户一被分配到服务器二，这个时候服务器并没有用户一的 Session 信息，就会出现需要重复登录的问题。

  ![img](G:\code\study\CppStudy\docs\figures\Session1.png)

  因此，我们需要借助 Redis 对这些 Session 信息进行统一的存储和管理，这样无论请求发送到那台服务器，服务器都会去同一个 Redis 获取相关的 Session 信息，这样就解决了分布式系统下 Session 存储的问题。

  ![img](G:\code\study\CppStudy\docs\figures\Session2.png)



### 2.3 List 类型

#### 2.3.1 基本介绍

`List`列表是简单的字符串列表，**<font color='cornflowerblue'>按照插入的顺序排序</font>**，可以从**头部**或**尾部**向List列表添加元素。

列表的最大长度为 `2^32 - 1`，也即每个列表支持超过 `40 亿`个元素。

#### 2.3.2 底层实现

`List`类型的底层结构是由<font color='cornflowerblue'>**双向链表（linkedList）**</font>或<font color='cornflowerblue'>**压缩列表（zipList）**</font>实现的：

- 如果列表的元素小于512个（默认值，可由 `list-max-ziplist-entries` 配置），且列表中的值都小于64字节（默认值，可由 `list-max-ziplist-value` 配置），Redis会使用**压缩列表**作为List的底层数据结构。

- 不满足上述条件，则会使用**双向链表**作为List的底层结构。

  ![img](G:\code\study\CppStudy\docs\figures\链表压缩列表.png)

但是<font color='cornflowerblue'>**在 Redis 3.2 版本之后，List 数据类型底层数据结构就只由 quicklist 实现了，替代了双向链表和压缩列表**</font>。

> **什么是压缩列表（zipList）？**

压缩列表**<font color='cornflowerblue'>是一种特殊的双向链表</font>**，被设计成一种内存紧凑型的数据结构，**占用一块连续的内存空间**，不仅可以利用 CPU 缓存，而且**会针对不同长度的数据，进行相应编码**，这种方法可以有效地节省内存开销。zipList整体的结构布局如下图：

![在这里插入图片描述](G:\code\study\CppStudy\docs\figures\ziplist.png)

- `zlbytes`: 32 位无符号整型，记录 ziplist 整个结构体的占用空间大小。当需要修改 ziplist 时候不需要遍历即可知道其本身的大小。 这个 SDS 中记录字符串的长度有相似之处。
- `zltail`: 32 位无符号整型, 记录整个 ziplist 中最后一个 entry 的偏移量。所以在尾部进行 POP 操作时候也不需要先遍历一次。
- `zllen`: 16 位无符号整型, 记录 `entry` 的数量， 只能表示 2^16。但是 Redis 作了特殊的处理：当实体数超过 2^16 ,该值被固定为 2^16 - 1。 所以这种时候要知道所有实体的数量就必须要遍历整个结构了。
- `entry`: 真正存数据的结构。包含了前一个`entry`的长度`prelen`（用于从后向前遍历），`encoding`编码形式，和具体编码后的数据。
- `zlend`: 8 位无符号整型, 固定为 255 (0xFF)。为 ziplist 的结束标识。

![在这里插入图片描述](G:\code\study\CppStudy\docs\figures\ziplist2.png)

😀**优点：**

- **节省内存**：存储在一个内存块中，剩下了两个指针的内存，减少内存碎片；
- **提高缓存命中率**：连续空间，缓存可以高效访问；
- **快速序列化/反序列化**：数据结构紧凑，序列化和反序列化速度较快。

☹**缺点：**

- **插入删除操作效率低**：连续的内存块，插入或删除操作需要移动大量元素；
- **扩展性差**：只适用于小规模数据集，规模越大，每次插入或删除性能越差；
- **不适合存储复杂结构**：适合存储简单的数据结构。

> ==**什么是快速列表（quickList）？**==

快速列表实际上是双向链表和压缩列表的混合体，它将整体的数据分成多个段，**每个段用一个压缩列表来表示**，**不同段之间通过双向指针串起来**，最终形成一个双向链表的形式。

![img](G:\code\study\CppStudy\docs\figures\QuickList.png)

#### 2.3.3 常用指令

常用的命令即增删改查，如下图所示：![img](G:\code\study\CppStudy\docs\figures\list.png)

```shell
# LPUSH 将一个或多个值value插入到key列表的表头(最左边)，最后的值在最前面，如果不存在key列表，则创建
LPUSH key value1 value2 (...)

# RPUSH 插入到表尾（最右边）
RPUSH key value1 value2 (...)

# LPOP 移除并返回key列表表头元素
LPOP key

# RPOP 移除并返回key列表表尾元素
RPOP key

# LRANGE 获取列表key中指定区间内的元素
LRANGE key start stop

# BLPOP 从key列表表头弹出一个元素，没有就阻塞timeout秒，如果timeout=0则一直阻塞
BLPOP key [key ...] timeout
# BRPOP 从key列表表尾弹出一个元素，没有就阻塞timeout秒，如果timeout=0则一直阻塞
BRPOP key [key ...] timeout
```

#### 2.3.4 应用场景

- **消息队列**

  消息队列在存取消息时，必须要满足三个需求：**消息顺序保序、处理重复的消息和保证消息可靠性**。

  Redis中`List`类型和`Stream`类型可以满足。

  **1️⃣ 如何满足消息顺序保存？**

  List 本身就是按先进先出的顺序对数据进行存取的，可以通过`LPUSH + BRPOP`（或`RPUSH + BLPOP`）组合来实现消息的存取。

  ![img](G:\code\study\CppStudy\docs\figures\list消息队列.png)

  <font color='cornflowerblue'>使用`BRPOP`或者`BLPOP`的原因是能够在当List中不存在消息时阻塞，直到有新的数据写入队列，再开始读取新数据。</font>而不是一直循环读取，节省CPU开销。

  **2️⃣ 如何处理重复的消息？**

  通过为每个消息添加一个全局ID，消费者（从消息队列中取消息的角色）程序可以对比收到的消息ID和记录的消息ID，判断是否为重复的消息。

  **3️⃣ 如何保证消息可靠性？**

  当消费者程序从 List 中读取一条消息后，List 就不会再留存这条消息了。所以，如果消费者程序在处理消息的过程出现了故障或宕机，就会导致消息没有处理完成，那么，消费者程序再次启动后，就没法再次从 List 中读取消息了。

  为了留存消息，List 类型提供了 `BRPOPLPUSH` 命令，这个命令的**作用是让消费者程序从一个 List 中读取消息，同时，Redis 会把这个消息再插入到另一个 List（可以叫作备份 List）留存**。

  > List 作为消息队列有什么缺陷？

  **List 不支持多个消费者消费同一条消息**，因为一旦消费者拉取一条消息后，这条消息就从 List 中删除了，无法被其它消费者再次消费。

  要实现一条消息可以被多个消费者消费，那么就要将多个消费者组成一个消费组，使得多个消费者可以消费同一条消息，但是 **List 类型并不支持消费组的实现**。

  这就要说起 Redis 从 5.0 版本开始提供的 `Stream` 数据类型了，**<font color='cornflowerblue'>`Stream` 同样能够满足消息队列的三大需求，而且它还支持「消费组」形式的消息读取。</font>**

### 2.4 Hash 类型

#### 2.4.1 基本介绍

Hash是一个键值对集合，即`value=[{field1，value1}，...{fieldN，valueN}]`，就是说k-v数据库的v是一个键值对的集合。Hash很适合用来存储对象。

Hash 与 String 对象的区别如下图所示:

<img src="G:\code\study\CppStudy\docs\figures\hash.png" style="zoom:50%;" />

#### 2.4.2 内部实现

Hash类型的底层是由**<font color='red'>压缩列表或哈希表</font>**实现的：

- 如果哈希类型元素个数小于 `512` 个，所有值小于 `64` 字节的话，Redis会使用**压缩列表**作为 Hash 类型的底层数据结构；
- 如果哈希类型元素不满足上面条件，Redis 会使用**哈希表**作为 Hash 类型的 底层数据结构。

<font color='cornflowerblue'>**在 Redis 7.0 中，压缩列表（ziplist）数据结构已经废弃了，交由 listpack 数据结构来实现了**。</font>

#### 2.4.3 常用指令

```shell
# 向Hash表key中添加filed-value对
> HSET key field value
(integer) 1

# 获取Hash表key中的field键对应的值
> HGET key field
"value"

# 在一个哈希表key中存储多个键值对
> HMSET key field value [field value...] 
# 批量获取哈希表key中多个field键值
> HMGET key field [field ...]       
# 删除哈希表key中的field键值
> HDEL key field [field ...]    

# 返回哈希表key中field的数量
HLEN key       
# 返回哈希表key中所有的键值
HGETALL key 

# 为哈希表key中field键的值加上增量n
HINCRBY key field 
```

#### 2.4.4 应用场景

- **缓存对象**

  Hash 类型的 （key，field， value） 的结构与对象的（对象id， 属性， 值）的结构相似，也可以用来存储对象。如下图所示：

  <img src="G:\code\study\CppStudy\docs\figures\hash存储结构.png" style="zoom:50%;" />

  > String + Json也是存储对象的一种方式，那么存储对象时，到底用 String + json 还是用 Hash 呢？

  **一般对象用 String + Json 存储，对象中某些<font color='cornflowerblue'>频繁变化的属性</font>可以考虑抽出来用 Hash 类型存储。**

- **购物车**

  以用户 id 为 key，商品 id 为 field，商品数量为 value，恰好构成了购物车的3个要素，如下图所示。

  <img src="G:\code\study\CppStudy\docs\figures\购物车.png" style="zoom:50%;" />

### 2.5 Set 类型

#### 2.5.1 基本介绍

Set类型是一个无序且唯一的键值集合，它的存储顺序不会按照插入的先后顺序进行存储。一个集合最多可以存储 `2^32-1` 个元素。概念和数学中个的集合基本类似，可以交集，并集，差集等等，所以 Set 类型**<font color='cornflowerblue'>除了支持集合内的增删改查，同时还支持多个集合取交集、并集、差集。</font>**

![img](G:\code\study\CppStudy\docs\figures\set.png)

**Set和List的区别：**

- List可以存储重复的元素，Set只能存储非重复的元素；
- List是按照元素插入的先后顺序来存储元素的，Set是无序方式存储元素的。

#### 2.5.2 内部实现

Set类型的底层数据结构是由**<font color='red'>哈希表或整数集合</font>**实现的：

- 如果集合中的元素都是整数且元素个数小于 `512` （默认值，`set-maxintset-entries`配置）个，Redis 会使用**整数集合**作为 Set 类型的底层数据结构；
- 如果集合中的元素不满足上面条件，则 Redis 使用**哈希表**作为 Set 类型的底层数据结构。

#### 2.5.3 常用指令

**常规操作指令：**

```shell
# 向集合key中加入元素number
SADD key number
# 从集合key中删除元素number
SREM key number

# 判断member元素是否存在于集合key中
SISMEMBER key member

# 从集合key中随机选出count个元素，元素不从key中删除
SRANDMEMBER key [count]
# 从集合key中随机选出count个元素，元素从key中删除
SPOP key [count]
```

**运算操作指令：**

```shell
# 交集运算
SINTER key [key ...]
# 将交集结果存入新集合destination中
SINTERSTORE destination key [key ...]

# 并集运算
SUNION key [key ...]
# 将并集结果存入新集合destination中
SUNIONSTORE destination key [key ...]

# 差集运算
SDIFF key [key ...]
# 将差集结果存入新集合destination中
SDIFFSTORE destination key [key ...]
```

#### 2.5.4 应用场景

Set 类型比较适合用来数据去重和保障数据的唯一性，还可以用来统计多个集合的交集、错集和并集等.

**存在一个风险：**<font color='red'>**Set 的差集、并集和交集的计算复杂度较高，在数据量较大的情况下，如果直接执行这些计算，会导致 Redis 实例阻塞**。</font>

- **点赞、抽奖活动**

  Set 类型可以保证一个用户只能点一个赞，只能中奖一次。

- **共同关注**

  利用Set的取交集运算。

### 2.6 Zset 类型

#### 2.6.1 基本介绍

Zset 类型（有序集合类型）相比于 Set 类型多了一个排序属性 score（分值），对于有序集合 ZSet 来说，每个存储元素相当于有两个值组成的，一个是有序集合的元素值，一个是排序值。

![img](G:\code\study\CppStudy\docs\figures\zset.png)

#### 2.6.2 内部实现

Zset 类型的底层数据结构是由<font color='red'>**压缩列表或跳表**</font>实现的：

- 如果有序集合的元素个数小于 `128` 个，并且每个元素的值小于 `64` 字节时，Redis 会使用**压缩列表**作为 Zset 类型的底层数据结构；
- 如果有序集合的元素不满足上面的条件，Redis 会使用**跳表**作为 Zset 类型的底层数据结构；

<font color='cornflowerblue'>**在 Redis 7.0 中，压缩列表数据结构已经废弃了，交由 listpack 数据结构来实现了。**</font>

#### 2.6.3 常用指令

**常规操作指令：**

```shell
# 往有序集合key中加入带分值元素
ZADD key score member [[score member]...]   # 注意，每个元素需要指定一个分值
# 往有序集合key中删除元素
ZREM key member [member...]                 
# 返回有序集合key中元素member的分值
ZSCORE key member
# 返回有序集合key中元素个数
ZCARD key 

# 为有序集合key中元素member的分值加上increment
ZINCRBY key increment member 

# 正序获取有序集合key从start下标到stop下标的元素
ZRANGE key start stop [WITHSCORES]
# 倒序获取有序集合key从start下标到stop下标的元素
ZREVRANGE key start stop [WITHSCORES]

# 返回有序集合中指定分数区间内的成员，分数由低到高排序。
ZRANGEBYSCORE key min max [WITHSCORES] [LIMIT offset count]

# 返回指定成员区间内的成员，按字典正序排列, 分数必须相同。
ZRANGEBYLEX key min max [LIMIT offset count]
# 返回指定成员区间内的成员，按字典倒序排列, 分数必须相同
ZREVRANGEBYLEX key max min [LIMIT offset count]
```

**运算操作指令：**（相比于 Set 类型，ZSet 类型没有支持差集运算）

```shell
# 并集计算(相同元素分值相加)，numberkeys一共多少个key，WEIGHTS每个key对应的分值乘积
ZUNIONSTORE destkey numberkeys key [key...] [WEIGHTS weight [weight ...]]
# 交集计算(相同元素分值相加)，numberkeys一共多少个key，WEIGHTS每个key对应的分值乘积
ZINTERSTORE destkey numberkeys key [key...] [WEIGHTS weight [weight ...]]
```

#### 2.6.4 应用场景

Zset 类型（Sorted Set，有序集合） 可以根据元素的权重来排序，我们可以自己来决定每个元素的权重值。比如说，我们可以根据元素插入 Sorted Set 的时间确定权重值，先插入的元素权重小，后插入的元素权重大。

在面对需要展示<font color='cornflowerblue'>**最新列表、排行榜、电话排序、姓名排序**</font>等场景时，如果数据更新频繁或者需要分页显示，可以优先考虑使用 Sorted Set。

### 2.7 BitMap 类型

#### 2.7.1 基本介绍

Bitmap，即位图，是一串连续的二进制数组（0和1），可以通过偏移量（offset）定位元素。由于 bit 是计算机中最小的单位，使用它进行储存将非常节省空间，特别适合一些数据量大且使用<font color='cornflowerblue'>**二值统计的场景**</font>（例如：使用`0|1`表示状态）。

![img](G:\code\study\CppStudy\docs\figures\bitmap.png)

#### 2.7.2 内部实现

Bitmap 本身是用 **<font color='red'>String 类型</font>**作为底层数据结构实现的一种统计二值状态的数据类型。

String 类型是会保存为**二进制的字节数组**，所以，Redis 就把字节数组的**每个 bit 位利用起来**，用来表示一个元素的二值状态，你可以把 Bitmap 看作是一个 bit 数组。

#### 2.7.3 常用指令

**常规操作指令：**

```shell
# 设置值，其中offset是指位的偏移量，表示从字符串的开始位置（0）到目标位的距离，value只能是 0 和 1
SETBIT key offset value
#例如：
> SETBIT key 0 1
(integer) 0

# 获取值
GETBIT key offset

# 获取指定范围内值为 1 的个数
# start 和 end 以字节为单位，这两个参数是可选的，如果不指定，则计算整个字符串。
BITCOUNT key start end
```

**位运算操作指令：**

```shell
# 当 BITOP 处理不同长度的字符串时，较短的那个字符串所缺少的部分会被看作 0。返回值是保存到 result 的字符串的长度（以字节byte为单位），和输入 key 中最长的字符串长度相等。
BITOP [operations] [result] [key1] [keyn...]

# operations 位操作符
# AND 与运算 &
# OR 或运算 |
# XOR 异或 ^
# NOT 取反 ~
# 举例：
> BITOP & result key1 key2

# 返回指定key中的BitMap第一次出现指定value(0/1)的位置
BITPOS [key] [value]
```

#### 2.7.4 应用场景

Bitmap 类型非常适合**<font color='cornflowerblue'>二值状态统计的场景（例如：签到统计、登陆状态）</font>**，这里的二值状态就是指集合元素的取值就只有 0 和 1 两种，在记录海量数据时，Bitmap 能够有效地节省内存空间。

### 2.8 HyperLogLog 类型

#### 2.8.1 基本介绍

HyperLogLog是用于**「统计基数」的数据集合类型**。基数统计就是指统计一个集合中不重复的元素个数。但要注意，HyperLogLog 是统计规则是**基于概率完成的，不是非常准确，标准误算率是 0.81%**。

简单来说 HyperLogLog <font color='cornflowerblue'>**提供不精确的去重计数**</font>。

HyperLogLog 的优点是：**在输入元素的数量或者体积非常非常大时，计算基数所需的内存空间总是固定的、并且是很小的。**

#### 2.8.2 内部实现

请参考：[HyperLogLog (opens new window)](https://en.wikipedia.org/wiki/HyperLogLog)。

#### 2.8.3 常用指令

HyperLogLog 命令很少，就三个。

```shell
# 添加指定元素到（HyperLogLog）key中
PFADD key element

# 返回给定的key中的基数估算值
PFCOUNT key 

# 将多个HyperLogLog合并成一个
PFMERGE destkey sourcekey1 sourcekey2 ...
```

#### 2.8.4 应用场景

Redis HyperLogLog 优势在于只需要花费 12 KB 内存，就可以计算接近 2^64 个元素的基数，和元素越多就越耗费内存的 Set 和 Hash 类型相比，HyperLogLog 就非常节省空间。所以，非常适合统计**<font color='cornflowerblue'>百万级以上的网页不重复访客数量（UV） 的场景</font>**。

### 2.9 GEO 类型

#### 2.9.1 基本介绍

GEO类型主要用于**存储地理位置信息，并对存储的信息进行操作**。在日常生活中，我们越来越依赖搜索“附近的餐馆”、在打车软件上叫车，这些都离不开基于位置信息服务（Location-Based Service，LBS）的应用。GEO 就非常适合应用在 LBS 服务的场景中。

#### 2.9.2 内部实现

GEO 本身并没有设计新的底层数据结构，而是直接使用了 **<font color='cornflowerblue'>Sorted Set 集合类型（Zset）</font>**。

GEO 类型使用 **GeoHash 编码方法**实现了经纬度到 Sorted Set 中元素权重分数的转换，**「对二维地图做区间划分」和「对区间进行编码」**，一组经纬度落在某个区间后，就用区间的编码值来表示，并把编码值作为 Sorted Set 元素的权重分数。

这样一来，我们就可以把经纬度保存到 Sorted Set 中，利用 Sorted Set 提供的“**按权重进行有序范围查找**”的特性，实现 LBS 服务中频繁使用的“**搜索附近**”的需求。

#### 2.9.3 常用指令

```shell
# 存储指定的地理空间位置，将经度(longitude)、纬度(latitude)、位置名称(member)添加到指定的 key 中。
GEOADD key longitude latitude member

# 从给定的key里返回所有指定名称的经纬度
GETPOS key member

# 返回给定两个位置之间的距离，[指定单位]
GEODIST key member1 member2 [m|km|ft|mi]

# 根据用户给定的经纬度坐标来获取指定范围内的地理位置集合
GEORADIUS key longitude lattude radius m|km|ft|mi
```

#### 2.9.4 应用场景

需要用到地理位置的，如：**滴滴叫车**。

### 2.10 Stream 类型

#### 2.10.1 基本介绍

Redis 5.0 版本新增加的数据类型，Redis <font color='cornflowerblue'>**专门为消息队列设计的数据类型**</font>。

在 Redis 5.0 Stream 没出来之前，消息队列的实现方式都有着各自的缺陷，例如：

- **发布订阅模式**：不能持久化保存消息，离线重连的客户端不能读取历史消息；
- **List实现消息队列**：无法重复消费一条消息，即不支持消费组模式。

Redis 5.0 便推出了 Stream 类型也是此版本最重要的功能，用于完美地实现消息队列，它**支持消息的持久化、支持自动生成全局唯一 ID、支持 ack 确认消息的模式、支持消费组模式等，让消息队列更加的稳定和可靠**。

#### 2.10.2 常用指令

```shell
# 向流中添加数据，自动生成全局唯一ID
XADD stream_name * field1 value1

# 从一个或多个流中读取数据，可以按照ID读取数据
XREAD STREAMS stream_name id

# 从流中读取范围数据
XRANGE stream_name start end

# XLEN ：查询消息长度；
# XDEL ： 根据消息 ID 删除消息；
# DEL ：删除整个 Stream；
# XREADGROUP：按消费组形式读取消息；
# XPENDING 和 XACK：
# XPENDING 命令可以用来查询每个消费组内所有消费者「已读取、但尚未确认」的消息；
# XACK 命令用于向消息队列确认消息处理已完成；
```

#### 2.10.3 应用场景

- 消息队列

  ![](G:\code\study\CppStudy\docs\figures\Stream简易.png)

  Stream 可以以使用 **XGROUP 创建消费组**，创建消费组之后，Stream 可以使用 **<font color='cornflowerblue'>XREADGROUP 命令让消费组内的消费者读取消息</font>**。

  ![img](G:\code\study\CppStudy\docs\figures\消息确认.png)

### 2.11 数据类型总结

Redis 常见的五种数据类型：<font color='cornflowerblue'>**String（字符串），Hash（哈希），List（列表），Set（集合）及 Zset(sorted set：有序集合)**</font>。

这五种数据类型**都由多种数据结构实现的**，主要是出于时间和空间的考虑，当**数据量小的时候使用更简单的数据结构，有利于节省内存，提高性能**。

![img](G:\code\study\CppStudy\docs\figures\Redis数据类型.png)

Redis 五种数据类型的应用场景：

- String 类型的应用场景：缓存对象、常规计数、分布式锁、共享session信息等。
- List 类型的应用场景：消息队列（有两个问题：1. 生产者需要自行实现全局唯一 ID；2. 不能以消费组形式消费数据）等。
- Hash 类型：缓存对象、购物车等。
- Set 类型：聚合计算（并集、交集、差集）场景，比如点赞、共同关注、抽奖活动等。
- Zset 类型：排序场景，比如排行榜、电话和姓名排序等。

Redis 后续版本又支持四种数据类型，它们的应用场景如下：

- BitMap（2.2 版新增）：二值状态统计的场景，比如签到、判断用户登陆状态、连续签到用户总数等；
- HyperLogLog（2.8 版新增）：海量数据基数统计的场景，比如百万级网页 UV 计数等；
- GEO（3.2 版新增）：存储地理位置信息的场景，比如滴滴叫车；
- Stream（5.0 版新增）：消息队列，相比于基于 List 类型实现的消息队列，有这两个特有的特性：自动生成全局唯一消息ID，支持以消费组形式消费数据。

> **针对 Redis 是否适合做消息队列，关键看你的业务场景：**

- 如果你的业务场景足够简单，对于数据丢失不敏感，而且消息积压概率比较小的情况下，把 Redis 当作队列是完全可以的。
- 如果你的业务有海量消息，消息积压的概率比较大，并且不能接受数据丢失，那么还是用专业的消息队列中间件吧，如RabbitMQ 或 Kafka 。



## 三、Redis 线程模型

### 3.1 Redis是单线程吗？

Redis的单线程模型并不是指Redis只有单线程。Redis单线程指的是<font color='red'>**「接收客户端请求->解析请求 ->进行数据读写等操作->发送数据给客户端」**</font>这个过程是由主线程（单个线程）来完成的。

实际上，Redis不止有这一个线程，还存在**<font color='cornflowerblue'>后台线程</font>**：

- Redis 2.6版本有<font color='cornflowerblue'>**2个后台线程**</font>：关闭文件任务线程、AOF刷盘任务线程；
- Redis 4.0版本之后有**<font color='cornflowerblue'>3个后台线程</font>**：关闭文件任务线程、AOF刷盘任务线程、释放Redis内存线程；

后台线程相当于一个消费者，**生产者把耗时任务丢到任务队列中，消费者（BIO）不停轮询这个队列**，拿出任务就去执行对应的方法即可。

<img src="G:\code\study\CppStudy\docs\figures\后台线程.jpg" style="zoom:50%;" />

之所以 Redis 为「关闭文件、AOF 刷盘、释放内存」这些任务创建单独的线程来处理，是因为**这些任务的操作都是很耗时的**，如果把这些任务都放在主线程来处理，那么 Redis 主线程就很容易发生阻塞，这样就无法处理后续的请求了。

### 3.2 Redis单线程模式是怎样的？

Redis 6.0之前是单线程模式，如下图：

<img src="G:\code\study\CppStudy\docs\figures\redis单线程模型.drawio.png" style="zoom:50%;" />

其中，蓝色部分是一个事件循环，是由主线程负责的，可以看到无论是网络I/O还是命令处理都是由主线程负责的（单线程）。Redis在初始化时会进行以下工作：

- 首先，利用`epoll_create()`创建一个epoll对象，调用`socket()`创建一个服务端套接字socket；
- 然后，调用`bind()`将socket绑定到指定端口，并使socket进入监听状态；
- 最后，调用epoll_ctl()将监听事件加入到epoll中，同时注册事件触发后的处理函数。

**代码示例：**

```cpp
epoll_fd = epoll_create(max_connections);	// 创建一个 epoll 实例，返回一个 epoll 文件描述符。

// 1. 创建服务端socket，返回socket文件描述符
server_fd = socket(AF_INET, SOCK_STREAM, 0); 

// 设置 socket 选项，允许重用本地地址、允许多个套接字绑定同一端口
if (setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR | SO_REUSEPORT, &opt, sizeof(opt))) {
        handle_error("setsockopt");
}

// 配置地址和端口信息
address.sin_family = AF_INET;		// 地址族为 IPv4（AF_INET）。
address.sin_addr.s_addr = INADDR_ANY;	// 0.0.0.0 接受任何传入的网络接口的连接请求
address.sin_port = htons(PORT);		// 设置端口号，htons为转换为网络字节序
// 2. 将socket绑定端口
bind(server_fd, (struct sockaddr *)&address, sizeof(address))
    
// 使套接字进入被动监听状态，等待连接请求
if (listen(server_fd, 3) < 0) {	// 最多3个未完成连接（包括在半连接队列和全连接队列中的连接）等待处理
        handle_error("listen");
}

// 3. 创建epoll监听事件，加入到epoll实例中
ev.events = EPOLLIN;	// 读事件
ev.data.fd = server_fd;	// 套接字描述符
if (epoll_ctl(epoll_fd, EPOLL_CTL_ADD, server_fd, &ev) == -1) {		// epoll_ctl 注册事件
        handle_error("epoll_ctl: server_fd");
}
```

初始化完后，主线程就进入到一个**事件循环函数**，主要会做以下事情：

- 首先，先调用<font color='cornflowerblue'>**处理发送队列函数**</font>，看是发送队列里是否有任务，如果有发送任务，则通过 write 函数将客户端发送缓存区里的数据发送出去，如果这一轮数据没有发送完，就会注册写事件处理函数，等待 epoll_wait 发现可写后再处理 。
- 接着，调用 epoll_wait 函数等待事件的到来：
  - 如果是**<font color='cornflowerblue'>连接事件</font>**到来，则会调用**<font color='cornflowerblue'>连接事件处理函数</font>**，该函数会做这些事情：调用 accpet 获取已连接的 socket -> 调用 epoll_ctl 将已连接的 socket 加入到 epoll -> 注册「读事件」处理函数；
  - 如果是**<font color='cornflowerblue'>读事件</font>**到来，则会调用**<font color='cornflowerblue'>读事件处理函数</font>**，该函数会做这些事情：调用 read 获取客户端发送的数据 -> 解析命令 -> 处理命令 -> 将客户端对象添加到发送队列 -> 将执行结果写到发送缓存区等待发送；
  - 如果是**<font color='cornflowerblue'>写事件</font>**到来，则会调用**<font color='cornflowerblue'>写事件处理函数</font>**，该函数会做这些事情：通过 write 函数将客户端发送缓存区里的数据发送出去，如果这一轮数据没有发送完，就会继续注册写事件处理函数，等待 epoll_wait 发现可写后再处理 。
- **==注意：以上所有工作都是在主线程中完成的，因此是单线程模型。==**

###  3.3 Redis 采用单线程为什么还这么快？

**单线程的 Redis 吞吐量可以达到 10W/每秒**，如下图所示：

<img src="G:\code\study\CppStudy\docs\figures\性能.png" style="zoom:67%;" />

Redis单线程很快的原因主要有以下几点：

- **<font color='cornflowerblue'>基于内存</font>**：Redis的大多数操作都在内存中完成，且采用了高效的数据结构，因此Redis的瓶颈不是CPU（而可能是**机器内存**或**网络带宽**），因此可以直接使用单线程；
- **<font color='cornflowerblue'>避免多线程竞争</font>**：减少了多线程之间的上下文切换和互斥锁消耗；
- **<font color='cornflowerblue'>I/O多路复用</font>**：通过`epoll`方法来实现多路复用，能够在单线程中监听多个socket上的连接事件。

### 3.3 Redis 6.0版本后为什么又引入了多线程？

随着网络硬件的性能提升，Redis的性能瓶颈有时会出现在网络I/O的处理上。网络带宽和传输速度都大幅提高，可能**<font color='cornflowerblue'>有大量的网络并发请求</font>**，由于单线程在某个时刻只能去处理新的I/O请求或执行命令中的一个，**导致CPU等待I/O操作完成，进而降低整体系统的吞吐量**。

为了克服这个瓶颈，Redis 6.0引入了**<font color='cornflowerblue'>多I/O线程来处理网络请求</font>。<font color='red'>但是对于命令的执行，Redis 仍然使用单线程来处理</font>**。Redis 6.0 版本支持的 I/O 多线程特性，默认情况下 I/O 多线程**只针对发送响应数据（write client socket）**，并不会以多线程的方式处理读请求（read client socket）。

因此， Redis 6.0 版本之后，Redis 在启动的时候，默认情况下会**额外创建 6 个线程**（*这里的线程数不包括主线程*）：

- `Redis-server` ： Redis的主线程，主要负责执行命令；

- `bio_close_file、bio_aof_fsync、bio_lazy_free`：三个后台线程，分别异步处理关闭文件任务、AOF刷盘任务、释放内存任务；
- `io_thd_1、io_thd_2、io_thd_3`：三个 I/O 线程，io-threads 默认是 4 ，所以会启动 3（4-1）个 I/O 多线程，用来分担 Redis 网络 I/O 的压力。

## 四、Redis 持久化

### 4.1 Redis 如何保证数据不丢失？

由于Redis的数据是保存在内存中，而内存中的数据会在Redis重启后丢失。因此，**为了保证数据不丢失，Redis实现了数据持久化的机制**。这个机制会将内存中的数据存储到磁盘，重启后可以从磁盘中恢复。

Redis共有三种数据持久化的方式：

- **<font color='cornflowerblue'>AOF日志</font>**：每执行一条写操作命令，就把该命令以追加的方式写到一个文件里；
- **<font color='cornflowerblue'>RDB快照</font>**：将某一时刻的内存数据，以二进制的方式写入磁盘；
- **<font color='cornflowerblue'>混合持久化方式</font>**：Redis 4.0新增，集成了AOF和RBD的优点。

### 4.2 AOF 日志是如何实现的？

#### 4.2.1 AOF 日志写入

Redis在执行完一条写操作命令后，就会把该命令以追加写的方式写到一个文件中。当Redis重启后，会读取该文件记录的命令，逐个执行来恢复数据。

![img](G:\code\study\CppStudy\docs\figures\AOF.png)

> **为什么是先执行命令，然后再把命令写入日志？**

Reids 是先执行写操作命令后，才将该命令记录到 AOF 日志里的，这么做其实有两个好处：

- <font color='cornflowerblue'>避免额外的检查开销</font>：如果先写入再执行，那在写入的时候还无法确定是否能够执行。需要增加一步检查命令。
- <font color='cornflowerblue'>不会阻塞当前写操作命令的执行</font>：因为是完成写操作再去写入之日。

也会存在风险：

- <font color='cornflowerblue'>数据可能会丢失</font>：如果完成了写操作但还没来得及写入日志，服务器宕机了，会丢失这个数据；
- <font color='cornflowerblue'>可能阻塞其他操作</font>：AOF日志也是在主线程中执行的，会阻塞后续的操作等待写入完成；

#### 4.2.2 AOF 日志写回策略

Redis写入AOF日志的过程可以由下图所示：

<img src="G:\code\study\CppStudy\docs\figures\AOF写入.png" style="zoom:67%;" />

1. Redis执行完写操作命令后，会将命令追加到一个缓冲区；
2. 然后通过write()系统调用，将该缓冲区的命令写入到磁盘中的AOF文件中。**<font color='red'>实际上这一步只会让命令拷贝到了AOF文件的内核缓冲区，什么时候写入到硬盘由内核决定</font>**；

Redis提供了**3种写回硬盘的策略，控制什么时候将数据写入硬盘**：

- <font color='cornflowerblue'>**Always**</font>：每次写操作执行完成后，**直接**将AOF日志数据写回硬盘；
- **<font color='cornflowerblue'>Everysec</font>**：写操作执行完成后，数据会先存放在AOF文件的内核缓冲区，**每隔一秒**将缓冲区的内容写回到磁盘。
- **<font color='cornflowerblue'>No</font>**：Redis不控制写回的执行，完全**交给操作系统来控制**。

![img](G:\code\study\CppStudy\docs\figures\AOF写回策略.png)

#### 4.2.3 AOF 重写机制

随着执行的写操作越来越多，AOF日志文件会越来越大，在数据恢复时会越慢，影响性能。因此，Redis提供了**重写机制，来避免AOF日志文件越写越大**。

AOF重写机制是根据K-V数据中的键值对，将每个键值对形成一个命令，记录到「新的 AOF 文件」中，完成后用这个「新的 AOF 文件」替换掉原有的AOF文件。

![img](G:\code\study\CppStudy\docs\figures\AOF日志重写.png)

AOF重写机制的本质就是**<font color='red'>只保留最新的</font>**。**在使用重写机制后，就会读取 name 最新的 value（键值对） ，然后用一条 「set name xiaolincoding」命令记录到新的 AOF 文件**，之前的第一个命令就没有必要记录了，因为它属于「历史」命令，没有作用了。这样一来，**<font color='cornflowerblue'>一个键值对在重写日志中只用一条命令就行了</font>**。

> **AOF重写是怎么完成的？**

Redis是在每次需要AOF重写时，通过**<font color='cornflowerblue'>新开辟一个子进程<font color='red'>`bgrewriteaof`</font>来完成AOF重写 </font>**。这是由于：

- **不会阻塞主进程**，主进程可以继续处理命令请求；
- 如果是多线程，由于共享数据，需要通过锁机制来保证数据安全，会影响性能。而子进程是拷贝数据的副本，创建子进程时，父子进程是共享内存数据的，不过这个共享的内存只能以只读的方式，而当父子进程任意一方修改了该共享内存，就会发生**「写时复制」**，于是父子进程就有了**独立的数据副本**，**就不用加锁来保证数据安全**。

> **数据不一致问题？**

因为在重写过程中，主进程仍然可以正常处理命令。如果发生了对已有key-value的修改，由于**「写时复制」**，**<font color='red'>子进程中关于这个key-value和主进程中的数据出现了不一致</font>**。为了解决这个问题，Redis设置了一个**<font color='cornflowerblue'>AOF重写缓冲区</font>**。

在重写AOF期间，主进程对于所有的写操作命令除了将其加入到**「AOF 缓冲区」**，还会加入到**<font color='cornflowerblue'>「AOF重写缓冲区」</font>**。当子进程完成重写时，会向主进程发送一个**<font color='red'>信号</font>**（进程间的通信方式），主进程收到信号时会执行：

- 将**<font color='cornflowerblue'>「AOF重写缓冲区」</font>**中的所有内容追加到**<font color='cornflowerblue'>「新的 AOF 文件」</font>**中；
- 用**<font color='cornflowerblue'>「新的 AOF 文件」</font>**覆盖掉「旧的 AOF 文件」

截止到此，就完成了整个AOF重写的工作。

### 4.3 RDB 快照是如何实现的？

Redis使用AOF进行恢复时，如果AOF日志较多，需要逐个执行，势必会使**Redis的恢复缓慢**。为了解决这个问题，Redis增加了RDB快照。

**<font color='cornflowerblue'>RDB快照就是记录某一个瞬间的内存数据（是数据而不是命令）</font>**，由于是直接记录的数据，因此在恢复时不需要执行操作命令，**只需要将RDB文件读入内存即可，效率很高**。

#### 4.3.1 RDB快照生成方法

Redis提供了两个命令来生成RDB快照：`save`和`bgsave`，它们的区别在于生成快照**是否在「主进程」里执行**：

- `save`命令：<font color='cornflowerblue'>**在主进程中生成RDB快照**</font>，如果写入时间太长，会导致主线程阻塞，影响性能；
- `bgsave`命令：会创建一个**<font color='cornflowerblue'>子进程来生成RDB快照</font>**，避免主线程的阻塞。

Redis 的快照是<font color='red'>**全量快照**</font>，也就是说每次执行快照，都是把内存中的**「所有数据」**都记录到磁盘中。所以执行快照是一个比较重的操作**，如果频率太频繁，可能会对 Redis 性能产生影响。如果频率太低，服务器故障时，丢失的数据会更多。**

#### 4.3.2 RDB快照如何保证数据一致性？

如果是使用`bgsave`命令来执行生成快照，此时主进程仍然可以继续处理操作，但是由于**快照是在一瞬间的数据，所以不需要去关注是否数据发生了变化**。因此，**<font color='red'>RDB快照不存在数据一致性问题</font>**。

<img src="G:\code\study\CppStudy\docs\figures\RDB快照1.png" style="zoom:67%;" />

如果主线程执行写操作，则被修改的数据会复制一份副本，然后 `bgsave` 子进程会**<font color='cornflowerblue'>把原来副本数据写入 RDB 文件（不去管新的）</font>**，在这个过程中，主线程仍然可以直接修改原来的数据。因此，借助**写时复制技术（Copy-On-Write, COW）**，RDB不用关注数据一致性问题。

<img src="G:\code\study\CppStudy\docs\figures\RDB快照2.png" alt="img" style="zoom:67%;" />

### 4.3 为什么会有混合持久化？

😊 RDB 优点是数据恢复速度快，但是快照的频率不好把握。频率太低，丢失的数据就会比较多，频率太高，就会影响性能。

😊 AOF 优点是丢失数据少，但是数据恢复不快。

为了集成了两者的优点， Redis 4.0 提出了<font color='cornflowerblue'>**混合使用 AOF 日志和RDB快照**</font>，也叫**混合持久化**，既保证了 Redis 重启速度，又降低数据丢失风险。

> **混合持久化是如何实现的？**

使用了混合持久化，AOF 文件的<font color='red'>**前半部分是 RDB 格式的全量数据，后半部分是 AOF 格式的增量数据**</font>。

当开启了混合持久化，在AOF重写日志时，重写子进程就不是把数据转换成命令写入了，而是**<font color='cornflowerblue'>直接生成当时数据库的RDB快照并写入到AOF文件中</font>**。然后，主进程中记录在AOF重写缓冲区的命令**<font color='cornflowerblue'>继续以AOF日志的格式写入到AOF日志文件中</font>**，替换旧的的 AOF 文件。

​																										![img](G:\code\study\CppStudy\docs\figures\混合持久化.png)	



**混合持久化优点：**

- 混合持久化结合了 RDB 和 AOF 持久化的优点，开头为 RDB 的格式，使得 Redis 可以**更快的启动**，同时结合 AOF 的优点，有**减低了大量数据丢失的风险**。

**混合持久化缺点：**

- AOF 文件中添加了 RDB 格式的内容，使得 **AOF 文件的可读性变得很差**；
- 兼容性差，如果开启混合持久化，那么此混合持久化 AOF 文件，就**不能用在 Redis 4.0 之前版本了**。

## 五、Redis 高可用集群

### 5.1 什么是Redis集群？

如何提供一个高可用的Redis服务？  ——  **构建Redis集群**

单服务器Redis由于数据都是存储在一台服务器，如果这台服务器出现宕机或者故障，可能会**导致服务不可用甚至数据丢失**。

要避免这种**单点故障**，最好的办法是将数据被分到其他服务器上，让这些服务器也能够对外提供服务，这样即使有一台服务器发生了故障，其他服务器依然可以继续提供服务。

<img src="G:\code\study\CppStudy\docs\figures\Redis集群.png" alt="图片" style="zoom:67%;" />

> **数据一致性？**

由于多个服务器保存同一份数据，所以**<font color='red'>如何保持数据的一致性是关键问题</font>**。Redis中有三种方案来保证数据的一致性：**<font color='cornflowerblue'>主从复制、哨兵机制、切片集群</font>**。

### 5.2 主从复制模式

#### 5.2.1 什么是主从复制？

主从复制模式中，主从服务器之间采用的是**「读写分离」**的方式。**<font color='cornflowerblue'>主服务器可以进行读写操作，当发生写操作时自动将写操作同步给从服务器，而从服务器一般是只读，并接受主服务器同步过来写操作命令，然后执行这条命令。</font>**

![图片](G:\code\study\CppStudy\docs\figures\主从复制.png)

也就是说，所有的**数据修改只在主服务器上进行**，然后将最新的数据同步给从服务器，这样就使得主从服务器的数据是一致的。主从服务器之间的命令复制是**<font color='cornflowerblue'>异步</font>**进行的。

但是，与Raft算法不一致的是，主服务器并不会等到从服务器实际执行完命令后，再把结果返回给客户端，而是**<font color='cornflowerblue'>主服务器自己在本地执行完命令后，就会向客户端返回结果了</font>**。如果从服务器还没有执行主服务器同步过来的命令，主从服务器间的数据就不一致了。所以，**<font color='red'>无法实现强一致性保证（主从数据时时刻刻保持一致），数据不一致是难以避免的</font>**。

#### 5.2.2 主从复制是怎么实现的？

- **第一次同步：全量复制**

  通过`replicaof`命令可以让服务器B成为服务器A的从服务器：

  ```shell
  # 服务器 B 执行这条命令
  replicaof <服务器 A 的 IP 地址> <服务器 A 的 Redis 端口号>
  ```

  服务器B成为服务器A的从服务器后，会发生与A服务器的**第一次同步**。第一次同步可以分为三个阶段：

  - <font color='cornflowerblue'>**第一阶段**</font>：**<font color='cornflowerblue'>建立连接，协商同步</font>**；

    从服务器会向主服务器申请数据同步，然后主服务器向从服务器作出响应，表明复制方法（全量复制）等。

  - **<font color='cornflowerblue'>第二阶段</font>**：**<font color='cornflowerblue'>主服务器同步数据给从服务器（全量复制</font>**）；

    主服务器会新开辟一个子进程来生成当前的**RDB快照**，然后把这个快照文件发送给从服务器；

    从服务器收到RDB快照后，清空该服务器上的数据，然后载入这个RDB快照；

    在此期间，如果主服务器收到了新的写操作命令，将会写到一个缓冲区中**`replication buffer`**，等待后面同步。

  - **<font color='cornflowerblue'>第三阶段</font>**：**<font color='cornflowerblue'>主服务器发送新的写操作命令给从服务器</font>**。

    当从服务器载入RDB文件完成后，会发送一个确认消息给主服务器；

    主服务器会将在**`replication buffer`**新增的一些写操作命令同步给从服务器，从服务器执行这些命令，从而实现主从服务器的数据一致性。

    至此，第一次同步完成。

  

  <img src="G:\code\study\CppStudy\docs\figures\第一次同步.png" alt="图片" style="zoom:67%;" />

- **提供服务期间：基于长连接的命令传播**

  第一次同步完成后，主服务器和从服务器之间会**维护它们所建立的TCP连接，形成一个<font color='cornflowerblue'>长连接</font>**。后续主服务器在收到写操作命令时，会通过这个连接将该命令传播给从服务器执行，保持数据的一致性。这个过程被称为**<font color='cornflowerblue'>基于长连接的命令传播</font>**，通过这种方式来保证第一次同步后的主从服务器的数据一致性。

  <img src="G:\code\study\CppStudy\docs\figures\命令传播.png" alt="图片" style="zoom:67%;" />

- **出现网络断连：增量复制**

  如果主从服务器之间的网络连接断开了，则无法进行命令传播，导致数据不一致，客户端可**能从「从服务器」读到旧的数据**。**<font color='red'>当网络恢复时，该如何保证主从服务器的数据一致性？</font>**

  Redis通过**<font color='cornflowerblue'>增量复制</font>**的方式继续同步，只会把网络断开期间主服务器接收到的写操作命令，同步给从服务器。

  <img src="https://cdn.xiaolincoding.com//mysql/other/e081b470870daeb763062bb873a4477e.png" alt="图片" style="zoom:67%;" />

  ❓ <font color='cornflowerblue'>**主服务器怎么知道要将哪些增量数据发送给从服务器呢？**</font>

  主服务器中维护了一个「**环形**」缓冲区**`repl_backlog_buffer`**，主服务器在进程命令传播时，还会将这些命令写入到这个缓冲区中，因此这个缓冲区里会保存着最近传播的写命令。

  另外，主服务器和从服务器都维护了一个标记同步进度的**偏移量**offset（`master_repl_offset`和`slave_repl_offset`），当网络重连时，从服务器会将自己的偏移量`slave_repl_offset`复制给主服务器，由主服务器决定需要如何同步：

  - 如果判断出从服务器要读取的数据还在 `repl_backlog_buffer` 缓冲区里，那么主服务器将采用**<font color='cornflowerblue'>增量同步</font>**的方式；
  - 相反，如果判断出从服务器要读取的数据已经不存在 `repl_backlog_buffer` 缓冲区里，那么主服务器将采用**<font color='cornflowerblue'>全量同步</font>**的方式。

  ❓ <font color='cornflowerblue'>**增量同步具体是怎么做的？**</font>

  主服务器在 `repl_backlog_buffer` 中找到主从服务器差异（增量）的数据后，就会将增量的数据写入到 `replication buffer` 缓冲区，然后经过命令传播同步给从服务器。

​															<img src="G:\code\study\CppStudy\docs\figures\增量同步.png" style="zoom:67%;" />

#### 5.2.3 如何分担主服务器的压力？

主服务器在第一次数据同步时会有两个耗时的操作：**生成RDB快照**和**传输RDB文件**。

如果从服务器非常多，且每个都需要主服务器进行全量同步的话，会带来一些问题：

- 主服务器忙于fork()创建子进程来生成RDB，导致无法正常处理请求；
- 传输RDB文件占用了大量网络贷款，对主服务器的接收请求造成影响。

Redis为了解决这个问题，让**从服务器也可以有自己的从服务器**，从服务器当作**<font color='cornflowerblue'>“经理”</font>**的角色，与它的从服务器的数据同步都是由这个“经理”完成的，而不需要主服务器参与，缓解主服务器的压力。

<img src="G:\code\study\CppStudy\docs\figures\经理.png" alt="图片" style="zoom:67%;" />



通过这种方式，**<font color='cornflowerblue'>主服务器生成 RDB 和传输 RDB 的压力可以分摊到充当经理角色的从服务器</font>**。

### 5.3 哨兵机制

#### 5.3.1 为什么需要哨兵机制？

在 Redis 的主从架构中，由于主从模式是读写分离的，如果**<font color='red'>主节点（master）挂了，那么将没有主节点来服务客户端的写操作请求</font>**，也没有主节点给从节点（slave）进行数据同步了，即**<font color='red'>十分依赖主节点</font>**。需要**人工介入**将一个「从节点」切换为「主节点」，然后让其他从节点指向新的主节点，同时还需要通知上游那些连接 Redis 主节点的客户端，将其配置中的主节点 IP 地址更新为「新主节点」的 IP 地址。

<img src="G:\code\study\CppStudy\docs\figures\主节点依赖.png" alt="主节点挂了" style="zoom:40%;" />

Redis 2.8版本以后提供了**<font color='cornflowerblue'>哨兵（Sentinel）机制</font>**，作用是实现**<font color='cornflowerblue'>主从节点故障转移</font>**，会监测主节点是否存活，如果发现主节点挂了，它就会**选举一个从节点切换为主节点，并且把新主节点的相关信息通知给从节点和客户端**。

> 类似Raft算法中领导者选举的方式

#### 5.3.2 哨兵机制是如何工作的？

哨兵本身也是一个**（服务器）节点**，和主节点、从节点类似，但是它是一种特殊的**”观察者节点“**，观察的对象就是主从节点。

哨兵节点负责三件事情：**<font color='cornflowerblue'>监控、选主、通知</font>**。

<img src="G:\code\study\CppStudy\docs\figures\哨兵节点的任务.png" alt="哨兵的职责" style="zoom:67%;" />

所以，哨兵机制的工作内容主要有：

- 如何**<font color='cornflowerblue'>监控</font>**节点？如何判断主节点是否真的故障了？
- 根据什么规则选择一个**<font color='cornflowerblue'>从节点切换为主节点</font>**？
- 怎么把新主节点的相关信息**<font color='cornflowerblue'>通知</font>**给从节点和客户端？

#### 5.3.3 如何监控节点？ —— 心跳机制

哨兵会**每隔 1 秒**给所有主从节点发送 PING 命令，当主从节点收到 PING 命令后，会**发送一个响应命令给哨兵**，即可判断它们是否在正常运行。（心跳机制）

<img src="G:\code\study\CppStudy\docs\figures\心跳.png" alt="哨兵监控主从节点" style="zoom:67%;" />

如果主节点或者从节点**没有在规定的时间内响应哨兵的 PING 命令**，哨兵就会将它们标记为<font color='cornflowerblue'>「**主观下线**」</font>。

#### 5.3.4 如何判断主节点是否真的故障了？

当标记了<font color='cornflowerblue'>「**主观下线**」</font>，有可能主节点其实没有故障，而只是因为主节点的系统压力比较大导致网络阻塞，所以才没有在规定时间内响应哨兵的PING命令。因此，针对主节点，哨兵机制设置了<font color='cornflowerblue'>「**客观下线**」</font>状态。

一般来说，为了减少误判，一般会用多个节点部署成**<font color='cornflowerblue'>哨兵集群</font>**（<font color='cornflowerblu'>*最少需要三台机器来部署哨兵集群*</font>），<font color='cornflowerblue'>**通过多个哨兵节点一起判断，就可以就可以避免单个哨兵因为自身网络状况不好，而误判主节点下线的情况**。</font>

> 如何判断主节点为「**客观下线**」呢？

当一个哨兵判断主节点为「主观下线」后，就会向其他哨兵发起命令。其他哨兵节点收到这个命令，会根据自身和主节点的网络状况，做出赞成投票或拒绝投票的响应。当这个哨兵的**<font color='cornflowerblue'>赞同票数达到哨兵配置文件中的 `quorum` 配置项设定的值后，这时主节点就会被该哨兵标记为「客观下线」</font>**。

<img src="G:\code\study\CppStudy\docs\figures\客观下线判断.png" alt="img" style="zoom:67%;" />

> PS：quorum 的值一般设置为哨兵个数的二分之一加 1，例如 3 个哨兵就设置 2。(超过半数)

哨兵判断完主节点客观下线后，哨兵就要开始在多个「从节点」中，选出一个从节点来做新主节点。

#### 5.3.5 由哪个哨兵进行主从故障转移？  —— 领导者选举

当主节点客观下线后，需要哨兵从从节点中选出一个新的主节点，由于有多个哨兵节点，应该**<font color='red'>让哪个哨兵节点负责呢？</font>**

哨兵集群中通过**<font color='cornflowerblue'>「领导者选举」</font>**来选出一个哨兵节点作为Leader，让Leader进行主从切换。一般来说，**<font color='cornflowerblue'>哪个哨兵节点判断主节点为「客观下线」，这个哨兵节点就是候选者</font>**，所谓的候选者就是想当 Leader 的哨兵。候选者会**向其他哨兵发送命令**，表明希望成为 Leader 来执行主从切换，并让所有其他哨兵对它进行投票。

每个哨兵只有一次投票机会，如果用完后就不能参与投票了，可以投给自己或投给别人，但是**只有候选者才能把票投给自己**。任何一个「候选者」，要满足两个条件则会成为Leader：

- 第一，拿到半数以上的赞成票；
- 第二，拿到的票数同时还需要大于等于哨兵配置文件中的 `quorum` 值。

所以，为了保证Leader选举顺利，至少需要3个哨兵节点，**quorum 的值建议设置为哨兵个数的二分之一加 1**，例如 3 个哨兵就设置 2，5 个哨兵设置为 3，而且**哨兵节点的数量应该是奇数**。

#### 5.3.6 主从故障转移的过程是怎样的？

在哨兵集群中通过投票的方式，选举出了哨兵 leader 后，就可以进行主从故障转移的过程了

<img src="G:\code\study\CppStudy\docs\figures\主从故障转移.png" alt="img" style="zoom:67%;" />

主从故障转移操作包含以下四个步骤：

- **第一步**：在已下线主节点（旧主节点）属下的所有「从节点」里面，挑选出一个从节点，并将其转换为**「新主节点」**；

  （1）将网络状态不好的从节点过滤掉；

  （2）第一轮考察：节点的优先级（显式的根据每个节点的性能配置的）；

  （2）第二轮考察：复制进度最靠前的节点；

  （3）第三轮考察：ID号更小的节点；

  ​	<img src="G:\code\study\CppStudy\docs\figures\选主过程.png" alt="选主过程" style="zoom:67%;" />

- **第二步**：让已下线主节点属下的所有「从节点」修改复制目标为**「新主节点」**；

  哨兵 leader 向所有从节点发送 `SLAVEOF` ，让它们成为新主节点的从节点：

  <img src="G:\code\study\CppStudy\docs\figures\从节点指向新主节点.png" alt="从节点指向新主节点" style="zoom:50%;" />

- **第三步**：将「新主节点」的IP地址和信息，通过「发布者/订阅者机制」通知给客户端；

  **<font color='cornflowerblue'>通过 Redis 的发布者/订阅者机制来实现</font>**的。每个哨兵节点提供发布者/订阅者机制，客户端可以从哨兵订阅消息。哨兵提供的消息订阅频道有很多，不同频道包含了主从节点切换过程中的不同关键事件，几个常见的事件如下：

  <img src="G:\code\study\CppStudy\docs\figures\哨兵频道.png" alt="哨兵频道" style="zoom:33%;" />

- **第四步**：继续监视旧主节点，当其重新上线时，将它设置为新主节点的从节点；

#### 5.3.7 哨兵集群是如果组成的？

前面提到了Redis中的哨兵机制是通过一个哨兵集群（多个哨兵节点）组成的，那是**如何组成哨兵集群的，它们之间是如何感知对方的？**

**<font color='cornflowerblue'>哨兵节点之间是通过 Redis 的发布者/订阅者机制来相互发现的</font>**，主节点中有一个名为`__sentinel__:hello`的频道，不同的哨兵就是通过这个频道来相互发现和互相通信的。如下图所示，哨兵 A 把自己的 IP 地址和端口的信息发布到`__sentinel__:hello` 频道上，哨兵 B 和 C 订阅了该频道。那么此时，哨兵 B 和 C 就可以从这个频道直接获取哨兵 A 的 IP 地址和端口号。然后，哨兵 B、C 可以和哨兵 A 建立网络连接。

<img src="G:\code\study\CppStudy\docs\figures\哨兵集群组成.png" alt="哨兵集群组成" style="zoom:50%;" />

**<font color='cornflowerblue'>哨兵节点之间是通过向主节点发送`INFO`命令来获取所有「从节点」的信息</font>**，哨兵节点需要监视所有的节点，但是它并不知道从节点的信息，所以哨兵会**<font color='red'>每隔10秒</font>**向主节点来获取从节点的信息。

<img src="G:\code\study\CppStudy\docs\figures\INFO命令.png" alt="INFO命令" style="zoom:50%;" />

### 5.4 切片集群模式

#### 5.4.1 为什么需要切片集群模式？

Redis用作缓存时，如果缓存数据量很大，无法在一台服务器上缓存，需要使用**<font color='cornflowerblue'>Redis切片集群</font>**（Redis Cluster ），数据可以分布在不同的服务器上，降低系统对单主节点的依赖。

#### 5.4.2 切片集群是如何实现的？

对于切片集群，最重要的问题就是决定**<font color='cornflowerblue'>一个缓存数据要放到哪一台服务器上</font>**。Redis Cluster 采用**哈希槽（Hash Slot）**，来处理数据和节点之间的映射关系。在 Redis Cluster 方案中，**<font color='red'>一个切片集群共有 16384 个哈希槽</font>**，每个数据（键值对）都会根据它的key，被映射到一个哈希槽中：

- 根据key，使用**<font color='cornflowerblue'>CRC16</font>**算法计算一个16 bit的值；
- 将这个16 bit的值对16384取模，得到一个0~16383之间的数，这个数就代表着哈希槽的编号。

然后，需要将这些哈希槽映射到各个Redis节点中，有两种映射方案：

- **<font color='cornflowerblue'>平均分配</font>**：使用 `cluster create` 命令创建 Redis 集群时，Redis会自动把所有哈希槽平均分布到集群节点上。比如集群中有 9 个节点，则每个节点上槽的个数为 16384/9 个。
- **<font color='cornflowerblue'>手动分配</font>**：使用 `cluster meet` 命令手动建立节点间的连接，组成集群，再使用 `cluster addslots` 命令，指定每个节点上的哈希槽个数。**需要把 16384 个槽都分配完，否则 Redis 集群无法正常工作。**

<img src="G:\code\study\CppStudy\docs\figures\redis切片集群映射分布关系.jpg" alt="img" style="zoom:50%;" />

### 5.5 集群脑裂导致数据丢失怎么办？

#### 5.5.1 集群脑裂是怎么出现的，如何导致数据丢失？

> **<font color='cornflowerblue'>脑裂</font>**：是指的一个集群有多个主节点。

Redis的主从架构中，主节点提供写操作，从节点提供读操作。如果主节点由于网络问题与其他从节点失联，但是与客户端之间网络正常。此时，客户端新发送的写请求操作会被主节点存放到缓存区中。

哨兵发现了主节点失联，判断为客观下线后会在集群内布重新选择一个主节点，这时集群就有两个主节点了 — **<font color='cornflowerblue'>脑裂出现了</font>**。

如果此时网络恢复，由于在集群中，「旧主节点」已经被降级成为了从节点，所以「旧主节点」会向「新主节点」申请数据同步，并清空该节点中的数据。这样就会**<font color='cornflowerblue'>导致断联期间客户端的一些写入的数据丢失，这就是集群产生脑裂数据丢失的问题。</font>**

#### 5.5.2 该如何解决？

Redis使用的策略是：当主节点发现**从节点的连接数量少于某个阈值**或者**主从数据复制和同步的延迟超过了某个阈值**，就会禁止主节点进行写入操作，直接返回错误给客户端。

即：**<font color='cornflowerblue'>主库连接的从库中至少有 N 个从库，和主库进行数据复制时的 ACK 消息延迟不能超过 T 秒，否则，主库就不会再接收客户端的写请求了。</font>**

## 六、Redis过期键值删除

### 6.1 Redis的过期键值删除策略

#### 6.1.1 什么是过期键值删除？

Redis中是可以对key设置过期时间的，所以需要有相应的机制将已过期的键值对删除，也就是**<font color='cornflowerblue'>过期键值删除策略</font>**。Redis会用一个**<font color='cornflowerblue'>过期字典（expires dict）</font>**来存储有过期时间的所有key。当查询一个key时，Redis会首先会检查这个key是否存在于过期字典中：

- 不存在，则正常读取键值；
- 存在，那需要首先获取这个key的过期时间，如果已经过期，则不会获得值；

至于具体去删除这个过期的key，Redis采用了「<font color='cornflowerblue'>**惰性删除+定期删除**</font>」两种策略配合使用。

#### 6.1.2 什么是惰性删除策略？

**惰性删除策略**：不主动删除过期键，每当从数据库访问key时，如果检测到这个key过期了，则删除这个key。

**优点**：只有在访问时才会检测，消耗很少的系统资源，**对CPU时间友好**。

**缺点**：如果一个过期的key一直没有被访问，那么会始终最早数据库内，造成内存空间浪费，对**内存不友好**。

#### 6.1.3 什么是定期删除策略？

**定期删除策略**：每隔一段时间**「随机」从过期字典中取出一定数量的 key 进行检查，并删除其中的过期key。**

**具体流程**：

1. 从过期字典中随机选取20个key；
2. 检查这20个key是否过期，删除已过期的key；
3. 如果本轮检查的「已过期 key 的数量」占比「随机抽取 key 的数量」大于 25%，则继续重复步骤 1；否则，停止本次删除流程。

> 可以看到，定期删除是一个循环的流程。那 Redis 为了保证定期删除不会出现循环过度，导致线程卡死现象，为此增加了定期删除循环流程的时间上限，默认不会超过 25ms。

**优点**：可以限制删除执行的时长和频率，能够同时减少对CPU的影响和减少空间占用；

缺点：难以确定删除操作执行的时长和频率，执行的太频繁对CPU不友好，执行的太少对内存不友好。

可以看到，惰性删除策略和定期删除策略都有各自的优点，所以 **<font color='cornflowerblue'>Redis 选择「惰性删除+定期删除」这两种策略配和使用</font>**，以求在合理使用 CPU 时间和避免内存浪费之间取得平衡。

### 6.2 Redis持久化时，对过期键是如何处理的？

#### 6.2.1 AOF日志

- **<font color='cornflowerblue'>AOF写入阶段</font>**：当Redis以AOF模式持久化时，如果数据库内某个键值过期还没有删除，AOF**仍然会保留此键值**，等过期键值被删除后，Redis会向AOF文件**追加一条DEL命令来显式地删除该键值**。
- **<font color='cornflowerblue'>AOF重写阶段</font>**：Redis执行AOF重写时，会对键值进行检查，**过期的键值不再写入重写后的AOF文件中**。

#### 6.2.2 RDB快照

- **<font color='cornflowerblue'>RDB文件生成阶段</font>**：将内存中的数据持久化为RDB文件时，**会对key进行过期检查，过期的键值不会保存到RDB文件中**。

- **<font color='cornflowerblue'>RDB文件加载阶段</font>**：RDB加载阶段，需要看服务器是主服务器还是从服务器

  - **主服务器**：在载入 RDB 文件时，会对文件中保存的key**进行检查**，过期键「不会」被载入到数据库中。
  - **从服务器**：**不进行过期检查**，不论key是否过期，键值都会被载入到数据库中。

  > 因为从服务器每次通过RDB数据同步时，从服务器都清空本身的所有数据，安装RDB文件的，所以下次就会删掉了，可以不用单独消耗时间来检查。

### 6.3 Redis主从模式中，对过期键是如何处理的？

Redis运行在主从模式下时，**<font color='cornflowerblue'>从服务器不会主动去处理过期键</font>**。即使从库中的 key 过期了，如果有客户端访问从库时，依然可以得到 key 对应的值，像未过期的键值对一样返回。

从服务器对过期键的处理依赖于主服务器，**<font color='cornflowerblue'>当主服务器删除某个过期键时，在AOF文件中增加一条命令。同步到所有的从服务器中</font>**，从服务器执行相应的命令来删除过期键值。

## 七、Redis内存淘汰

### 7.1 Redis运行在内存中，若Redis的内存满了，会发生什么？

Redis的运行内存如果达到了某个阈值，会触发**<font color='cornflowerblue'>内存淘汰机制</font>**，这个内存就是用户设置的最大运行内存。Redis提供了多种内存淘汰策略，根据不同的策略来将部分内进行淘汰。

### 7.2 Redis有哪些内存淘汰策略？

Redis 内存淘汰策略共有八种，这八种策略大体分为**<font color='cornflowerblue'>「不进行数据淘汰」</font>**和**<font color='cornflowerblue'>「进行数据淘汰」</font>**两类策略。

#### 7.2.1 不进行内存淘汰

**<font color='cornflowerblue'>noeviction</font>**（Redis3.0之后，默认的内存淘汰策略） ：它表示当运行内存超过最大设置内存时，不淘汰任何数据，而是不再提供服务，直接返回错误。

#### 7.2.2 进行内存淘汰

针对「进行数据淘汰」这一类策略，又可以细分为**「在设置了过期时间的数据中进行淘汰」**和**「在所有数据范围内进行淘汰**」这两类策略。 

- **在设置了过期时间的数据中进行淘汰**：
  - **<font color='cornflowerblue'>volatile-random</font>**：**随机**淘汰设置了过期时间的任意键值；
  - **<font color='cornflowerblue'>volatile-lru</font>**：淘汰所有设置了过期时间的键值中，**最久未使用**的键值；
  - **<font color='cornflowerblue'>volatile-lfu</font>**：淘汰所有设置了过期时间的键值中，**最少使用**的键值；
- **在所有数据范围内进行淘汰**：
  - **<font color='cornflowerblue'>allkeys-random</font>**：随机淘汰任意键值;
  - **<font color='cornflowerblue'>allkeys-lru</font>**：淘汰整个键值中最久未使用的键值；
  - **<font color='cornflowerblue'>allkeys-lfu</font>**：淘汰整个键值中最少使用的键值；

#### 7.2.3 LRU算法：最近最少使用淘汰策略

传统 LRU 算法的实现是基于「链表」结构，链表中的元素按照操作顺序从前往后排列，最新操作的键会被移动到表头，当需要内存淘汰时，只需要删除链表尾部的元素即可，因为链表尾部的元素就代表最久未被使用的元素。

Redis 并没有使用这样的方式实现 LRU 算法，因为传统的 LRU 算法存在两个问题：

- 需要用链表管理所有的缓存数据，这会带来额外的**空间开销**；
- 当有数据被访问时，需要在链表上把该数据移动到头端，如果有大量数据被访问，就会带来**很多链表移动操作，会很耗时**，进而会降低 Redis 缓存性能。

> **Redis实现LRU算法的方式？**

Redis 实现的是一种**近似 LRU 算法**，目的是为了更好的节约内存，它的**<font color='cornflowerblue'>实现方式是在 Redis 的对象结构体中添加一个额外的字段，用于记录此数据的最后一次访问时间</font>**。

当 Redis 进行内存淘汰时，会使用**<font color='cornflowerblue'>随机采样的方式来淘汰数据</font>**，它是随机取 5 个值（此值可配置），然后**<font color='cornflowerblue'>淘汰最久没有使用的那个</font>**。

但是 LRU 算法有一个问题，**<font color='red'>无法解决缓存污染问题</font>**，比如应用一次读取了大量的数据，而这些数据只会被读取这一次，那么这些数据会留存在 Redis 缓存中很长一段时间，造成缓存污染

#### 7.2.4 LFU算法：最近最不常用淘汰策略

LFU 算法会**记录每个数据的访问次数**。当一个数据被再次访问时，就会增加该数据的访问次数。这样就解决了偶尔被访问一次之后，数据留存在缓存中很长一段时间的问题，相比于 LRU 算法也更合理一些。

> **Redis实现LFU算法的方式？**

LFU 算法相比于 LRU 算法的实现，多记录了**<font color='cornflowerblue'>「数据的访问频次」</font>**的信息。

```cpp
typedef struct redisObject {
    ...
      
    // 24 bits，用于记录对象的访问信息（在LRU中只是记录访问时间，LFU中高位16bit记录访问时间 + 低位8bit记录访问频次）
    unsigned lru:24;  
    ...
} robj;
```

**在 LFU 算法中**，Redis对象头的 24 bits 的 lru 字段被分成两段来存储，**<font color='cornflowerblue'>高 16bit 存储 ldt(Last Decrement Time)，用来记录 key 的访问时间戳；低 8bit 存储 logc(Logistic Counter)，用来记录 key 的访问频次</font>**。![img](G:\code\study\CppStudy\docs\figures\lru字段.png)

## 八、Redis缓存设计

### 8.1 为什么Redis用作缓存？

一般来说，数据库的数据都是落在磁盘上的，会导致读写速度很慢。如果用户的请求量非常大，数据库很容易崩溃。由于Redis的数据保存在内存中，读写速度很快，所以Redis一般被用作数据库的缓存层。因为 Redis 是内存数据库，我们可以将数据库的数据缓存在 Redis 里，相当于**数据缓存在内存，内存的读写速度比硬盘快好几个数量级，这样大大提高了系统性能**。

<img src="G:\code\study\CppStudy\docs\figures\redis缓存.png" alt="图片" style="zoom:66%;" />

引入了缓存层，就会有缓存异常的三个问题，分别是**<font color='red'>缓存雪崩、缓存击穿、缓存穿透</font>**。

<img src="G:\code\study\CppStudy\docs\figures\缓存雪崩击穿和穿透.png" alt="图片" style="zoom:67%;" />

<img src="G:\code\study\CppStudy\docs\figures\Redis缓存总结.png" alt="图片" style="zoom:67%;" />

### 8.2 什么是缓存雪崩？

#### 8.2.1 什么是缓存雪崩？

通常来说，为了保证缓存和数据库中的数据一致性，需要给Redis缓存中的数据**<font color='cornflowerblue'>设置过期时间</font>**。当缓存过期后，则用户的请求会从数据库读取数据，并将数据再次存入到Redis缓存中。

当**<font color='cornflowerblue'>大量缓存数据在同一时间过期（失效）或者 Redis 故障宕机</font>**时，如果此时有大量的用户请求，都无法在 Redis 中处理，于是**全部请求都直接访问数据库，从而导致数据库的压力骤增**，严重的会造成数据库宕机，从而形成一系列连锁反应，造成整个系统崩溃，这就是**缓存雪崩**的问题。

<img src="G:\code\study\CppStudy\docs\figures\缓存雪崩.png" alt="图片" style="zoom:67%;" />

发生缓存雪崩有两个原因：

- **<font color='red'>大量数据同时过期</font>**；
- **<font color='red'>Redis 故障宕机</font>**；

#### 8.2.2 大量数据同时过期解决方案？

针对大量数据同时过期而引发的缓存雪崩问题，常见的应对方法有下面这几种：

- **<font color='cornflowerblu'>均匀设置过期时间，避免大量数据同时过期</font>**

  我们可以在对缓存数据设置过期时间时，**<font color='cornflowerblue'>给这些数据的过期时间加上一个随机数</font>**，这样就保证数据不会在同一时间过期。

- **<font color='cornflowerblu'>互斥锁</font>**

  如果一个请求发现访问的数据不在Redis中，则先**<font color='cornflowerblue'>加互斥锁，保证同一时刻只有一个请求从数据库中读取数据，并更新到Redis缓存中</font>**，完成请求后再释放锁。没有拿到锁的请求则会阻塞等待后**<font color='cornflowerblue'>重新读取缓存（而不需要再去访问数据库）</font>**，或者直接返回空值或默认值。

  > 实现互斥锁的时候，最好设置**超时时间**，不然第一个请求拿到了锁，然后这个请求发生了某种意外而一直阻塞，一直不释放锁，这时其他请求也一直拿不到锁，整个系统就会出现无响应的现象。

- **<font color='cornflowerblu'>后台更新缓存</font>**

  不再给缓存设置过期时间，而是让缓存**“永久有效”**，开启一个**<font color='cornflowerblue'>后台线程定时更新缓存</font>**。事实上，当系统内存资源紧张时，对一些**<font color='cornflowerblue'>缓存数据进行“淘汰”</font>**。这就需要机制来保障数据的有效性，Redis一般通过后台线程监控（就是定时更新缓存这个线程）或者业务线程等方式来通知数据库读取数据更**新这个淘汰的缓存**。

#### 8.2.3 Redis故障宕机解决方案？

针对 Redis 故障宕机而引发的缓存雪崩问题，常见的应对方法有下面这几种：

- **<font color='cornflowerblu'>服务熔断或请求限流机制</font>**
  - 启动**<font color='cornflowerblue'>服务熔断</font>**机制：**暂停业务应用对缓存服务的访问，直接返回错误**，不用再继续访问数据库，从而降低对数据库的访问压力，保证数据库系统的正常运行，然后等到 Redis 恢复正常后，再允许业务应用访问缓存服务。
  - 启动**<font color='cornflowerblue'>请求限流</font>**机制，**只将少部分请求发送到数据库进行处理，再多的请求就在入口直接拒绝服务**，等到 Redis 恢复正常并把缓存预热完后，再解除请求限流的机制。

- **<font color='cornflowerblu'>构建Redis缓存高可靠集群</font>**

  通过<font color='cornflowerblue'>**主从节点的方式构建 Redis 缓存高可靠集群**</font>。如果 Redis 缓存的主节点故障宕机，从节点可以切换成为主节点，继续提供缓存服务，避免了由于 Redis 故障宕机而导致的缓存雪崩问题。

### 8.3 什么是缓存击穿？

我们的业务通常会有几个数据会被频繁地访问，比如秒杀活动，这类被频地访问的数据被称为**<font color='cornflowerblue'>热点数据</font>**。如果缓存中的**<font color='cornflowerblue'>某个热点数据过期</font>**了，此时大量的请求访问了该热点数据，就无法从缓存中读取，**直接访问数据库，数据库很容易就被高并发的请求冲垮**，这就是**<font color='cornflowerblue'>缓存击穿</font>**的问题。

<img src="G:\code\study\CppStudy\docs\figures\缓存击穿.png" alt="图片" style="zoom:67%;" />

可以发现缓存击穿跟缓存雪崩很相似，可以认为**<font color='cornflowerblue'>缓存击穿是缓存雪崩的一个子集</font>**。

应对缓存击穿可以采取前面说到两种方案（**除了均匀分配过期时间，因为这个是只针对热点数据，而不是大量数据的**）：

- **<font color='cornflowerblu'>互斥锁方案</font>**：保证同一时间只有一个业务线程更新缓存，未能获取互斥锁的请求，要么等待锁释放后重新读取缓存，要么就返回空值或者默认值。
- **<font color='cornflowerblu'>后台更新方案</font>**：不给热点数据设置过期时间，由后台异步更新缓存，或者在热点数据准备要过期前，提前通知后台线程更新缓存以及重新设置过期时间；

### 8.4 什么是缓存穿透？

#### 8.4.1 什么是缓存穿透？

当用户访问的数据，**<font color='red'>既不在缓存中，也不在数据库中</font>**，导致请求在访问缓存时，发现缓存缺失，再去访问数据库时，发现数据库中也没有要访问的数据，没办法构建缓存数据，来服务后续的请求。那么**当有大量这样的请求到来时，数据库的压力骤增**，这就是**<font color='cornflowerblue'>缓存穿透</font>**的问题。

<img src="G:\code\study\CppStudy\docs\figures\缓存穿透.png" alt="图片" style="zoom:67%;" />

缓存穿透的发生一般有这两种情况：

- **业务误操作**，缓存中的数据和数据库中的数据都被误删除了，所以导致缓存和数据库中都没有数据；
- **黑客恶意攻击**，故意大量访问某些读取不存在数据的业务；

#### 8.4.2 如何应对缓存穿透？

应对缓存穿透的方案，常见的方案有三种：

- **<font color='cornflowerblu'>限制非法请求</font>**

  在 API 入口处我们要判断求请求参数是否合理，请求参数是否含有非法值、请求字段是否存在，如果判断出是恶意请求就**直接返回错误**，避免进一步访问缓存和数据库。

- **<font color='cornflowerblu'>缓存空值或者默认值</font>**

  当我们线上业务发现缓存穿透的现象时，可以针对查询的数据，**在缓存中设置一个空值或者默认值**，这样后续请求就可以从缓存中读取到空值或者默认值，返回给应用，而不会继续查询数据库。

- **<font color='cornflowerblu'>布隆过滤器</font>**

  在写入数据库数据时，**使用布隆过滤器做个标记**，然后在用户请求到来时，业务线程确认缓存失效后，可以**<font color='cornflowerblue'>通过查询布隆过滤器快速判断数据是否存在</font>**，如果不存在，就不用通过查询数据库来判断数据是否存在。即使发生了缓存穿透，**<font color='cornflowerblue'>大量请求只会查询 Redis 和布隆过滤器，而不会查询数据库</font>**，保证了数据库能正常运行，Redis 自身也是支持布隆过滤器的。

  > **布隆过滤器是如何工作的呢？**

  布隆过滤器由**「初始值都为 0 的位图数组」**和**「 N 个哈希函数」**两部分组成。布隆过滤器会通过 3 个操作完成标记：

  - 第一步：使用N个哈希函数分别对数据进行运算，得到N个哈希值；
  - 第二步：将N个哈希值对BitMap的长度取模，得到每个哈希值在BitMap数组的位置；
  - 第三步：将N个哈希值所在的BitMap的对应位置的值设为1；

  ![图片](G:\code\study\CppStudy\docs\figures\布隆过滤器.png)

  **<font color='cornflowerblue'>当应用要查询数据 x 是否在数据库时，通过布隆过滤器只要查到位图数组的第 1、4、6 位置的值是否全为 1，只要有一个为 0，就认为数据 x 不在数据库中</font>**。布隆过滤器由于是基于哈希函数实现查找的，高效查找的同时**<font color='cornflowerblue'>存在哈希冲突的可能性</font>**，比如数据 x 和数据 y 可能都落在第 1、4、6 位置，而事实上，可能数据库中并不存在数据 y，存在误判的情况。

  所以，**查询布隆过滤器说数据存在，并不一定证明数据库中存在这个数据，但是<font color='cornflowerblue'>查询到数据不存在，数据库中一定就不存在这个数据</font>**。

### 8.6 如何设计可以动态缓存热点数据的缓存策略？

由于数据存储受限，系统并不是将所有数据都需要存放到缓存中的，而**<font color='cornflowerblue'>只是将其中一部分热点数据缓存起来</font>**，所以我们要设计一个热点数据动态缓存的策略。

热点数据动态缓存的策略总体思路：**<font color='cornflowerblue'>通过数据最新访问时间来做排名，并过滤掉不常访问的数据，只留下经常访问的数据</font>**。

以电商平台场景中的例子，现在要求只缓存用户经常访问的 Top 1000 的商品。具体细节如下：

- 先通过缓存系统做一个排序队列（比如存放 1000 个商品），系统会根据商品的访问时间，更新队列信息，越是最近访问的商品排名越靠前；
- 同时系统会定期过滤掉队列中排名最后的 200 个商品，然后再从数据库中随机读取出 200 个商品加入队列中；
- 这样当请求每次到达的时候，会先从队列中获取商品 ID，如果命中，就根据 ID 再从另一个缓存数据结构中读取实际的商品信息，并返回。

在 Redis 中可以用 zadd 方法和 zrange 方法来完成排序队列和获取 200 个商品的操作。

### 8.7 常见的缓存策略？

常见的缓存更新策略有3种：

- Cache Aside（旁路缓存）策略；
- Read/Write Through （读穿/写穿）策略；
- Write Back （写回）策略；

实际开发中，Redis 和 MySQL 的更新策略用的是 **<font color='cornflowerblue'>Cache Aside</font>**，另外两种策略应用不了。

#### 8.7.1 Cache Aside（旁路缓存）策略

Cache Aside（旁路缓存）策略是最常用的，应用程序**直接与「数据库、缓存」交互，并负责对缓存的维护**，该策略又可以细分为**「读策略」**和**「写策略」**。

<img src="G:\code\study\CppStudy\docs\figures\旁路缓存策略.png" alt="img" style="zoom:80%;" />

**<font color='cornflowerblue'>写策略的步骤</font>：**

- **先**更新数据库中的数据，**再**删除缓存中的数据。

**<font color='cornflowerblue'>读策略的步骤</font>：**

- 如果读取的数据命中了缓存，则直接返回数据；
- 如果读取的数据没有命中缓存，则**从数据库中读取数据，然后将数据写入到缓存**，并且返回给用户。

> **写策略能否反过来？ 先删除缓存数据，再更新数据库数据？**
>
> 不能，原因是在「读+写」并发的时候，会出现缓存和数据库的数据不一致性的问题。详情见后面分析。

**<font color='cornflowerblue'>Cache Aside 策略适合读多写少的场景，不适合写多的场景</font>**，因为当写入比较频繁时，缓存中的数据会被频繁地清理，这样会对缓存的命中率有一些影响。如果业务对缓存命中率有严格的要求，那么可以考虑两种解决方案：

- 更新数据时也更新缓存，只是在更新缓存前先**加一个分布式锁**，因为这样在**同一时间只允许一个线程更新缓存，就不会产生并发问题了**。当然这么做对于写入的性能会有一些影响；
- 更新数据时更新缓存，只是给缓存**加一个较短的过期时间**，这样即使出现**缓存不一致的情况，缓存的数据也会很快过期**，对业务的影响也是可以接受。

#### 8.7.2 Read/Write Through（读穿 / 写穿）策略

Read/Write Through（读穿 / 写穿）策略原则是应用程序**<font color='cornflowerblue'>只和缓存交互，不再和数据库交互</font>**，而是由缓存和数据库交互，相当于**更新数据库的操作由缓存自己代理了**。

**<font color='cornflowerblue'>读策略的步骤</font>：**

- 先查询缓存中数据是否存在，如果存在则直接返回，如果不存在，则**由缓存组件负责从数据库查询数据**，并将结果写入到缓存组件，最后缓存组件将数据返回给应用。

**<font color='cornflowerblue'>写策略的步骤</font>：**

当有数据更新的时候，先查询要写入的数据在缓存中是否已经存在：

- 如果缓存中数据已经存在，则更新缓存中的数据，并且由缓存组件同步更新到数据库中，然后缓存组件告知应用程序更新完成；
- 如果缓存中数据不存在，直接更新数据库，然后返回。

<img src="G:\code\study\CppStudy\docs\figures\WriteThrough.jpg" alt="img" style="zoom:50%;" />

Read Through/Write Through 策略的特点是**<font color='cornflowerblue'>由缓存节点而非应用程序来和数据库打交道</font>**，在我们开发过程中相比 Cache Aside 策略要少见一些，原因是我们经常使用的分布式缓存组件，无论是 **<font color='cornflowerblue'>Memcached 还是 Redis 都不提供写入数据库和自动加载数据库中的数据的功能</font>**。而我们在使用本地缓存的时候可以考虑使用这种策略。

#### 8.7.3 Write Back （写回）策略

Write Back（写回）策略在更新数据的时候，**只更新缓存，同时将缓存数据设置为脏的**，然后立马返回，并不会更新数据库。对于数据库的更新，会通**<font color='cornflowerblue'>过批量异步更新的方式进行</font>**。

**<font color='cornflowerblue'>Write Back 策略特别适合写多的场景</font>**，因为发生写操作的时候， 只需要更新缓存，就立马返回了。比如，写文件的时候，实际上是写入到文件系统的缓存就返回了，并不会写磁盘。

**<font color='cornflowerblue'>但是带来的问题是，数据不是强一致性的，而且会有数据丢失的风险</font>**，因为缓存一般使用内存，而内存是非持久化的，所以一旦缓存机器掉电，就会造成原本缓存中的脏数据丢失。所以你会发现系统在掉电之后，之前写入的文件会有部分丢失，就是因为 Page Cache 还没有来得及刷盘造成的。

### 8.8 如何保证缓存和数据库数据的一致性？

#### 8.8.1 缓存更新还是删除？先数据库还是先缓存？

无论是**<font color='cornflowerblue'>「先更新数据库，再更新缓存」</font>**，还是**<font color='cornflowerblue'>「先更新缓存，再更新数据库」</font>**，这两个方案都**存在并发问题**，当两个请求并发更新同一条数据的时候，可能会出现**缓存和数据库中的数据不一致的现象**。于是，Cache Aside策略中使用的是**<font color='cornflowerblue'>「先更新数据库，再删除缓存」</font>**策略，不对缓存更新，而**<font color='cornflowe'>是直接把缓存给删除，等下次读取数据时从数据库读并加入到缓存中</font>**。

**<font color='cornflowerblue'>「先删除缓存，再更新数据库」</font>**，在「读 + 写」并发的时候，还是会出现缓存和数据库的数据不一致的问题：

<img src="G:\code\study\CppStudy\docs\figures\先删缓存不一致性.png" alt="图片" style="zoom:67%;" />

因此，Cache Aside策略中使用的是**<font color='cornflowerblue'>「先更新数据库，再删除缓存」</font>**策略。理论上分析，先更新数据库，再删除缓存也是会出现数据不一致性的问题，**<font color='red'>但是在实际中，这个问题出现的概率并不高</font>**。如下图：

<img src="G:\code\study\CppStudy\docs\figures\先更新数据库不一致性.png" alt="图片" style="zoom:67%;" />

**<font color='cornflowerblue'>因为缓存的写入通常要远远快于数据库的写入</font>**，所以在实际中很难出现请求 B 已经更新了数据库并且删除了缓存，请求 A 才更新完缓存的情况。所以，**「先更新数据库 + 再删除缓存」的方案，是可以保证数据一致性的**。

#### 8.8.2 「先更新数据库 + 再删除缓存」如何确保两个操作都能执行成功？

这次用户的投诉是因为在删除缓存（第二个操作）的时候失败了，导致缓存还是旧值，而数据库是最新值，造成数据库和缓存数据不一致的问题，会对敏感业务造成影响。

<img src="G:\code\study\CppStudy\docs\figures\两个操作没有执行成功.png" alt="图片" style="zoom:67%;" />

该怎么解决呢？有两种方法：

- **<font color='cornflowerblu'>消息队列重试机制</font>**
- **<font color='cornflowerblu'>订阅 MySQL binlog，再操作缓存</font>**

## 九、Redis实战

### 9.1 Redis如何实现延迟队列？

延迟队列是把当前要做的事情，往后推迟一段时间再做，如以下常见场景：

- 在淘宝、京东等购物平台上下单，超过一定时间未付款，订单会自动取消；
- 打车的时候，在规定时间没有车主接单，平台会取消你的单并提醒你暂时没有车主接单；
- 点外卖的时候，如果商家在10分钟还没接单，就会自动取消订单；

Redis可以使用**有序集合（ZSet）**来实现延迟消息队列，ZSet中的`Score`属性可以用来存储延迟的时间。使用 `zadd score1 value1` 命令就可以一直往内存中生产消息。再利用 `zrangebysocre` 查询符合条件的所有待处理的任务， 通过循环执行队列任务即可。

![img](G:\code\study\CppStudy\docs\figures\延迟队列.png)



### 9.2 Redis的大key如何处理？

#### 9.2.1 什么是Redis大key？

大key不是指key很大，还是**<font color='cornflowerblue'>key对应的value很大</font>**。一般而言，下面这两种情况被称为大 key：

- String 类型的值大于 **10 KB**；
- Hash、List、Set、ZSet 类型的元素的个数超过 **5000个**；

#### 9.2.2 大key会造成什么问题？

- **<font color='cornflowerblue'>客户端超时阻塞</font>**：Redis单线程操作大key会比较耗时，从客户端来看，就是很久很久没有响应；
- **<font color='cornflowerblue'>引发网络阻塞</font>**：获取大key的网络流量较大，如果并发访问过多，会导致网络阻塞；
- <font color='cornflowerblue'>**阻塞工作线程**</font>：当使用DEL删除大key时，由于删除需要时间，会阻塞工作线程，无法处理后续命令；
- **<font color='cornflowerblue'>内存分布不均</font>**：集群模型会出现数据和查询倾斜情况，有大key的Redis节点占用内存较多，QPS也比大。

#### 9.2.3 如何查找大key？

- `redis-cli --bigkeys`查找大key
- `SCAN`命令
- `RdbTools`工具

#### 9.2.4 如何删除大key？

删除操作的本质是要**释放键值对占用的内存空间**，为了更加高效地管理内存空间，在应用程序释放内存时，操作系统需要把释放掉的内存块插入一个空闲内存块的链表，以便后续进行管理和再分配。**这个过程本身需要一定时间，而且会阻塞当前释放内存的应用程序。**因此，**<font color='red'>会造成 Redis 主线程的阻塞</font>**。

Redis中有两种方法删除大key：

- 分批次删除
- 异步删除（`unlink` ，Redis 4.0版本以上）

### 9.3 Redis管道有什么用？

管道技术（Pipeline）是**客户端提供的**一种批处理技术，用于一次处理多个 Redis 命令，从而提高整个交互的性能。

普通命令模式，如下图所示：

![img](G:\code\study\CppStudy\docs\figures\普通命令模式.jpg)

管道模式，如下图所示：

![img](G:\code\study\CppStudy\docs\figures\管道模式.jpg)



**<font color='cornflowerblue'>管道技术可以解决多个命令执行时的网络等待</font>**，它是把多个命令整合到一起发送给服务器端处理之后统一返回给客户端，这样就免去了每条命令执行后都要等待的情况，从而有效地提高了程序的执行效率。注意**避免发送的命令过大，**或**管道内的数据太多而导致的网络阻塞**。

> 管道技术本质上是**客户端提供的功能**，而非 Redis 服务器端的功能。

### 9.4 Redis事务支持回滚吗？

MySQL 在执行事务时，会提供**回滚机制**，当事务执行发生错误时，事务中的所有操作都会撤销，**已经修改的数据也会被恢复到事务执行前的状态**。

**<font color='cornflowerblue'>Redis 中并没有提供回滚机制</font>**，虽然 Redis 提供了 DISCARD 命令，但是这个命令**只能用来主动放弃事务执行，把暂存的命令队列清空，起不到回滚的效果**。下面是 DISCARD 命令用法：

```shell
#读取 count 的值4
127.0.0.1:6379> GET count
"1"
#开启事务
127.0.0.1:6379> MULTI 
OK
#发送事务的第一个操作，对count减1
127.0.0.1:6379> DECR count
QUEUED
#执行DISCARD命令，主动放弃事务
127.0.0.1:6379> DISCARD
OK
#再次读取a:stock的值，值没有被修改
127.0.0.1:6379> GET count
"1"
```

事务执行过程中，如果命令入队时没报错，而事务提交后，实际执行时报错了，正确的命令依然可以正常执行，所以这可以看出 **<font color='red'>Redis 并不一定保证原子性</font>**（原子性：事务中的命令要不全部成功，要不全部失败）。

比如下面这个例子：

```c
#获取name原本的值
127.0.0.1:6379> GET name
"xiaolin"
#开启事务
127.0.0.1:6379> MULTI
OK
#设置新值
127.0.0.1:6379(TX)> SET name xialincoding
QUEUED
#注意，这条命令是错误的
# expire 过期时间正确来说是数字，并不是‘10s’字符串，但是还是入队成功了
127.0.0.1:6379(TX)> EXPIRE name 10s
QUEUED
#提交事务，执行报错
#可以看到 set 执行成功，而 expire 执行错误。
127.0.0.1:6379(TX)> EXEC
1) OK
2) (error) ERR value is not an integer or out of range
#可以看到，name 还是被设置为新值了
127.0.0.1:6379> GET name
"xialincoding"
```

支持事务回滚的原因有以下两个：

- 他认为 Redis 事务的执行时，错误通常都是**编程错误造成的**，这种错误通常只会出现在开发环境中，而很少会在实际的生产环境中出现，所以他认为没有必要为 Redis 开发事务回滚功能；
- 不支持事务回滚是因为这种**复杂的功能和 Redis 追求的简单高效的设计主旨不符合**。

### 9.5 如何用Redis实现分布式锁？

#### 9.5.1 为什么Redis能用来实现分布式锁？

分布式锁是用于分布式环境下并发控制的一种机制，用于控制某个资源**在同一时刻只能被一个应用所使用**。如下图所示：

<img src="G:\code\study\CppStudy\docs\figures\分布式锁.jpg" alt="img" style="zoom:67%;" />

Redis**可以被多个客户端共享访问**，可以用来保存分布式锁，且Redis的**读写性能高**，可以应对高并发的锁操作常见。

#### 9.5.2 如何实现分布式锁？

Redis 的 SET 命令有个 **<font color='cornflowerblue'>NX 参数</font>**可以实现**「key不存在才插入」**，所以可以用它来实现分布式锁：

- 如果 key 不存在，则显示插入成功，可以用来表示加锁成功；
- 如果 key 存在，则会显示插入失败，可以用来表示加锁失败。

基于 Redis 节点实现分布式锁时，对于加锁操作，我们需要满足三个条件：

- 加锁包括了**读取锁变量、检查锁变量值和设置锁变量值**三个操作，但需要以原子操作的方式完成，所以，我们**<font color='cornflowerblue'>使用 SET 命令带上 NX 选项</font>**来实现加锁；

- 锁变量需要**设置过期时间**，以免客户端拿到锁后发生异常，导致锁一直无法释放，所以，我们在 SET 命令执行时加上 **<font color='cornflowerblue'>EX/PX 选项，设置其过期时间</font>**；

- 锁变量的值需要**能区分来自不同客户端的加锁操作**，以免在释放锁时，出现误释放操作，所以，我们使用 SET 命令设置锁变量值时，**<font color='cornflowerblue'>每个客户端设置的值是一个唯一值，用于标识客户端</font>**；

  ```shell
  SET lock_key unique_value NX PX 10000
  
  # lock_key 就是 key 键；
  # unique_value 是客户端生成的唯一的标识，区分来自不同客户端的锁操作；
  # NX 代表只在 lock_key 不存在时，才对 lock_key 进行设置操作；
  # PX 10000 表示设置 lock_key 的过期时间为 10s，这是为了避免客户端发生异常而无法释放锁。
  ```

解锁的过程就是将 lock_key 键删除，但不能乱删，要保证执行操作的客户端就是加锁的客户端，即先判断先判断锁的 unique_value 是否为加锁客户端。因此，解锁是两个步骤，**无法保证原子性。**因此Redis需要引入<font color='cornflowerblue'>**`Lua`脚本**</font>来实现原子操作，解锁的`Lua`脚本如下：

```lua
// 释放锁时，先比较 unique_value 是否相等，避免锁的误释放
if redis.call("get",KEYS[1]) == ARGV[1] then
    return redis.call("del",KEYS[1])
else
    return 0
end
```

用户通常需要**编写适当的 Lua 脚本并通过 Redis 客户端发送到 Redis 服务器执行**。这是因为分布式锁的解除需要原子操作，而 Lua 脚本可以保证在 Redis 中的操作是原子的。发送到Redis服务器方法如下：

```shell
## 后面的1是指有一个键
> EVAL "if redis.call('get',KEYS[1]) == ARGV[1] then return redis.call('del',KEYS[1]) else return 0 end" 1 lock_key unique_value
```

不过，为了简化这一过程，一些 Redis 客户端库已经内置了处理分布式锁的功能。这些库通常**会封装 Lua 脚本的编写和执行过程，使用户可以更方便地使用分布式锁**。

#### 9.5.3 基于Redis的分布式锁的优缺点？

**优点**：

- 性能高效；
- 实现方便；
- 避免单点故障；

**缺点**：

- 超时时间不好设置；
- Redis 主从复制模式中的数据是**异步复制的**，这样**<font color='red'>导致分布式锁的不可靠性</font>**。

#### 9.5.4 Redis如何解决集群情况下分布式锁的可靠性？

Redlock 算法的基本思路，**<font color='cornflowerblue'>是让客户端和多个独立的 Redis 节点依次请求申请加锁，如果客户端能够和半数以上的节点成功地完成加锁操作，那么我们就认为，客户端成功地获得分布式锁，否则加锁失败</font>**。

可以看到，加锁成功要同时满足两个条件：

- 条件一：客户端从超过半数（大于等于 N/2+1）的 Redis 节点上成功获取到了锁；
- 条件二：客户端从大多数节点获取锁的总耗时（t2-t1）小于锁设置的过期时间。



# 资料参考

内容大多参考自：[图解Redis介绍 | 小林coding (xiaolincoding.com)](https://xiaolincoding.com/redis/)

[Redis数据结构——快速列表(quicklist) - 随心所于 - 博客园 (cnblogs.com)](https://www.cnblogs.com/hunternet/p/12624691.html)