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



## 二、Redis 数据结构

### 2.1 Redis有哪些数据结构？

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







# 资料参考

[图解Redis介绍 | 小林coding (xiaolincoding.com)](https://xiaolincoding.com/redis/)

