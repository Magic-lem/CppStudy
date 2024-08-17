# MySQL学习&八股文

## 一、MySQL基础

### 1.1 MySQL 执行流程是怎样的？

以执行一条 SQL 查询语句的流程为例，如下图所示：

<img src="G:\code\study\CppStudy\docs\figures\mysql查询流程.png" alt="查询语句执行流程" style="zoom:67%;" />

MySQL可以分为两层：**<font color='cornflowerblue'>Server层和存储引擎层</font>**。其中：

- **<font color='cornflowerblue'>Server层负责管理连接、分析和执行SQL</font>**。包括连接器、解析器、预处理器、优化器、执行器、各种内置函数、各种功能等都是在Server层实现。
- **<font color='cornflowerblue'>存储引擎层负责数据的存储和提取</font>**。支持InnoDB、MylSAM、Memory等多个存储引擎，默认的是InnoDB。

对于一条SQL查询语句，它的步骤包括以下流程：

#### **第一步：建立连接**

第一步肯定是要先连接 MySQL 服务，然后才能执行 SQL 语句。连接命令 ：

```mysql
# -h 指定 MySQL 服务得 IP 地址，如果是连接本地的 MySQL服务，可以不用这个参数；
# -u 指定用户名，管理员角色名为 root；
# -p 指定密码，如果命令行中不填写密码（为了密码安全，建议不要在命令行写密码），就需要在交互对话里面输入密码
mysql -h {ip} -u {user} -p
```

MySQL服务器中由Server层的**<font color='cornflowerblue'>「连接器」</font>**管理这部分工作，经过 TCP 三次握手成功后，连接器会验证用户输入的用户名和密码，并获取用户的权限，**后续用户在此链接中的所有操作都会基于这个权限进行判断**（就算中途管理员修改了用户的权限，也不会影响本次连接）。

MySQL 的连接也跟 HTTP 一样，有**短连接和长连接的概念**，一般是使用长连接。因此可能会存在长连接占用内存的问题，MySQL中有两种解决方案：

- **定期断开长连接**：MySQL有一个`wait_timeout`参数，由Server层定期清理长时间“空闲”的连接；
- **客户端主动重置连接**：当客户端执行了一个很大的操作后，会重置连接，达到释放内存的效果。

> **总结一下第一步中连接器的工作：**

- 与客户端进行 TCP 三次握手建立连接；
- 校验客户端的用户名和密码，如果用户名或密码不对，则会报错；
- 如果用户名和密码都对了，会读取该用户的权限，然后后面的权限逻辑判断都基于此时读取到的权限；

#### **第二步：查询缓存**

建立连接后，客户端就可以向MySQL服务发送SQL语句了，对于查询语句，MySQL会先去从**<font color='cornflowerblue'>查询缓存（ Query Cache ）</font>**中查找数据，看是否能够命中。若命中，则可以直接读取缓存并返回。若没有命中，则继续向下查询，并将结果加入到缓存中。

然而，对于更新频繁的表，**查询缓存会被经常情况，所以导致命中率很低**，因此在MySQL 8.0 版本这一步就被省略了，不会再查询缓存。 

#### **第三步：解析SQL**

当没有使用缓存时则需要去具体执行SQL语句，在执行之前需要对SQL语句进行解析，由Server层的**<font color='cornflowerblue'>「解析器」</font>**完成。解析过程分为两步：

1. **<font color='cornflowerblue'>词法分析</font>**

   解析器会根据输入的字符串识别关键字，如SELECT、FROM等。

   | 关键字 | 非关键字 | 关键字 | 非关键字 |
   | ------ | -------- | ------ | -------- |
   | select | username | from   | userinfo |

2. **<font color='cornflowerblue'>语法分析</font>**

   语法解析器会根据语法规则，判断SQL语句是否合法，构建出SQL语法树，便于后面执行

   <img src="G:\code\study\CppStudy\docs\figures\db-mysql-sql-parser-2.png" alt="img" style="zoom:50%;" />

#### **第四步：执行SQL**

解析完成后，一般会需要**验证用户权限（由授权管理模块负责）**，若验证通过，则进入执行SQL语句的流程，可以分为下面三个阶段：

**<font color='cornflowerblue'>1. 预处理阶段：预处理器</font>**

此阶段主要有两个任务：

- 判断查询的表或者字段**是否存在**；
- 将select `*` 中的 `*` 符号**扩展为表上的所有列**

**<font color='cornflowerblue'>2. 优化阶段：优化器</font>**

「优化器」的作用是**将SQL语句的执行方案确定下来**，比如基于查询成本该使用那个索引，决定各个表的连接顺序等。

> 要想知道优化器选择了哪个索引，我们可以在查询语句**<font color='red'>最前面加个 `explain` 命令</font>**，这样就会输出这条 SQL 语句的执行计划

**<font color='cornflowerblue'>3. 执行阶段：执行器</font>**

确定了执行方案后，「执行器」就开始真正执行语句了，在此过程中「执行器」会**和存储引擎交互**来查询数据，并返回给客户端。



### 1.2 MySQL 一行记录是怎么存储的？

#### **1.2.1 首先，MySQL的数据保存在哪个文件？ 是如何保存的？**

MySQL的存储行为是由存储引擎实现的，不同存储引擎保存的文件不同。InnoDB 是常用的存储引擎，也是 MySQL 默认的存储引擎。因此这里就InnoDB为例，对于每个数据库都会在 **<font color='cornflowerblue'>`/var/lib/mysql/` 目录里面创建一个以 database 为名的目录</font>**，然后保存**表结构和表数据的文件都会存放在这个目录里**。

在这个目录里一共会存在三个文件：

- **db.opt**，用来存储当前数据库的默认字符集和字符校验规则。
- **t_order.frm** ，t_order 的**表结构**会保存在这个文件。在 MySQL 中建立一张表都会生成一个.frm 文件，该文件是用来保存每个表的元数据信息的，主要包含表结构定义。
- **t_order.ibd**，t_order 的**表数据**会保存在这个文件。表数据既可以存在共享表空间文件（文件名：ibdata1）里，也可以存放在独占表空间文件（文件名：表名字.ibd）。这个行为是由参数 innodb_file_per_table 控制的，若设置了参数 innodb_file_per_table 为 1，则会将存储的数据、索引等信息单独存储在一个独占表空间，从 MySQL 5.6.6 版本开始，它的默认值就是 1 了，因此从这个版本之后， MySQL 中每一张表的数据都存放在一个独立的 .ibd 文件。

所以，一张数据库表的数据是保存在**<font color='cornflowerblue'>「 表名字.ibd 」</font>**的文件里的，这个文件也称为**<font color='cornflowerblue'>独占表空间文件</font>**。这个**<font color='red'>表空间的结构</font>**如下图所示：



<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/row_format/%E8%A1%A8%E7%A9%BA%E9%97%B4%E7%BB%93%E6%9E%84.drawio.png" style="zoom:80%;" />

- **行（row）**

  数据库表中的记录都是按行（row）进行存放的，每行记录根据不同的行格式，有不同的存储结构。

- **页（page）**

  记录是按照行来存储的，但是数据库的读取并不以「行」为单位，否则一次读取（也就是一次 I/O 操作）只能处理一行数据，效率会非常低。**<font color='cornflowerblue'>InnoDB 的数据是按「页」为单位来读写的</font>**，也就是说，当需要读一条记录的时候，并不是将这个行记录从磁盘读出来，而是以页为单位，将其整体读入内存。

- **区（extent）**

  InnoDB 存储引擎是用 B+ 树来组织数据的，B+ 树中每一层都是通过双向链表连接起来的，如果是以页为单位来分配存储空间，那么链表中相邻的两个页之间的物理位置并不是连续的，可能离得非常远，那么磁盘查询时就会有大量的随机I/O，随机 I/O 是非常慢的。

  解决这个问题也很简单，就是让**<font color='cornflowerblue'>链表中相邻的页的物理位置也相邻</font>**。为某个索引分配空间的时候就不再按照页为单位分配了，而是**按照区（extent）为单位分配**。**每个区的大小为 1MB，对于 16KB 的页来说，连续的 64 个页会被划为一个区，这样就使得链表中相邻的页的物理位置也相邻**，就能使用顺序 I/O 了。

- **段（segment）**

  表空间是由各个段（segment）组成的，段是由多个区（extent）组成的。段一般分为数据段、索引段和回滚段等。

  - 索引段：存放 B + 树的非叶子节点的区的集合；
  - 数据段：存放 B + 树的叶子节点的区的集合；
  - 回滚段：存放的是回滚数据的区的集合。

#### **1.2.2 InnoDB 行格式 —— COMPACT**

行格式（row_format），就是一条记录的**<font color='red'>存储结构</font>**。

InnoDB 提供了 4 种行格式，分别是 Redundant、Compact、Dynamic和 Compressed 行格式。重点介绍 Compact 行格式。

![img](G:\code\study\CppStudy\docs\figures\COMPACT.drawio.png)

一条完整的记录分为**<font color='cornflowerblue'>「记录的额外信息」</font>**和**<font color='cornflowerblue'>「记录的真实数据」</font>**两个部分。

> 😊 **记录的额外信息**

记录的额外信息包含 3 个部分：变长字段长度列表、NULL值列表、记录头信息。

- **变长字段长度列表**

  由于对于`varchar`是变长的，实际存储的数据大小不固定。因此需要把大小也记录下来，也就是村早这个变长字段长度列表中。

- **NULL值列表**

  表中的某些列可能会存储NULL值，如果直接存放NULL值到真实数据中会比较浪费空间，而是把这些值为 NULL 的列**存储到 NULL值列表中**，则**<font color='cornflowerblue'>每个允许NULL值列对应一个二进制位（bit，这样就实现了一个NULL值只占用1bit）</font>**，二进制位按照列的顺序逆序排列：

  - 二进制位的值为`1`时，代表该列的值为NULL。
  - 二进制位的值为`0`时，代表该列的值不为NULL。

  注意：NULL 值列表**<font color='red'>必须用整数个字节的位表示（1字节8位）</font>**，如果使用的二进制位个数不足整数个字节，则在字节的高位补 `0`。

- **记录头信息**

  是一些标记字段，用来表示当前记录的类型、数据是否被删除（不会立即删除，而是加标记）、下一条记录的位置（因为是双向链表组织的）

> 😊 **记录的真实数据**

记录真实数据部分有三个隐藏字段，分别为：row_id、trx_id、roll_pointer：

<img src="G:\code\study\CppStudy\docs\figures\记录的真实数据.png" alt="img" style="zoom:80%;" />



- row_id：如果**没有指定主键或唯一约束**，InnoDB会额外添加一个row_id字段，占用6个字节；
- trx_id：事务id，标识数据是哪个事务生成的，占用6个字节；
- roll_pinter：指向这个记录上一个版本的指针，占用7个字节。

### **1.3 varchar(n)中n最大取多少？**

`varchar(n)` 字段类型的 `n` 代表的是**<font color='cornflowerblue'>最多存储的字符数量</font>**。

> **<font color='cornflowerblu'>MySQL 规定除了 TEXT、BLOBs 这种大对象类型之外，其他所有的列（不包括隐藏列和记录头信息）占用的字节长度加起来不能超过 65535 个字节（64KB），其中包括「变长字段长度列表」和 「NULL 值列表」。</font>**

因此n最大取多少，取决于这个字段字符集中的字符所占用的字节大小：

**<font color='red'>字符数量 = （65535 -  「变长字段长度列表」占用字节数量 - 「NULL 值列表」占用字节数量）/ 单个字符所占用的字节数</font>**

如果有多个字段的话，要保证**所有字段的长度** + 变长字段字节数列表所占用的字节数 + NULL值列表所占用的字节数 <= 65535。

### **1.4 MySQL是怎么知道varchar(n)实际占用大小的？**

MySQL中每一行记录中会有一个**变长字段长度列表**，这个列表中记录了**这个记录中这个变长字段的实际大小**。

### **1.5 MySQL中的NULL值是怎么存放的？**

MySQL会**使用一个NULL值列表来存储NULL值**，而不是把NULL值存在记录的真实数据中。每个可能含有NULL值的列都在NULL值列表中对应一个bit（逆序排列），如果这个bit值为1，则说明这个列的值是NULL，否则不是NULL，这样可以节省内存。如果所有的字段都不含NULL，则不会有这个NULL值列表。

### 1.6 行溢出后，MySQL 是怎么处理的？

MySQL中磁盘和内存交换的本质是页，一个页大小为`16K`，即`16384`字节。而MySQL中`varchar`最大可以是`65535`字节，一些大对象可能存储更多的数据，此时一个页可能存不了一条记录，会发生**<font color='cornflowerblue'>行溢出，多余的数据会存到另外的「溢出页」中</font>**。

Compact行格式在发生行溢出时，在记录的真实数据处只会保存一部分的数据，通过留出**<font color='cornflowerblue'>20字节的空间保存溢出页的地址，从而可以找到剩余数据所在的页，即溢出页（单个溢出页内存不足时则通过链式）</font>**，如下图所示：

<img src="G:\code\study\CppStudy\docs\figures\行溢出.png" alt="img" style="zoom:80%;" />

Compressed 和 Dynamic 这两个行格式和 Compact 非常类似，主要的区别在于处理行溢出数据时有些区别。这两种格式采用完全的行溢出方式，**记录的真实数据处不会存储该列的一部分数据，只存储 20 个字节的指针来指向溢出页**。而实际的数据都存储在溢出页中，看起来就像下面这样：

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/row_format/%E8%A1%8C%E6%BA%A2%E5%87%BA2.png" style="zoom:80%;" />

## 二、MySQL索引

### 2.1 什么是索引？

**<font color='cornflowerblue'>索引的定义就是帮助存储引擎快速获取数据的一种数据结构</font>，索引是数据的目录**。利用空间换取时间，能够极大加速数据库的查询效率。

索引和数据一样，位于MySQL的**存储引擎层**。

### 2.2 索引的分类

- 按「数据结构」分类：**B+tree索引、Hash索引、Full-text索引**。
- 按「物理存储」分类：**聚簇索引（主键索引）、二级索引（辅助索引）**。
- 按「字段特性」分类：**主键索引、唯一索引、普通索引、前缀索引**。
- 按「字段个数」分类：**单列索引、联合索引**。

#### 2.2.1 按数据结构分类

索引本身是通过一种数据结构来实现的，根据数据结构的不同，可以分为：**<font color='cornflowerblue'>B+树索引、Hash索引、Full-Text索引</font>**等。每种存储引擎支持的索引类型也不同，InnoDB中默认使用**B+树**作为索引的数据结构。

#### 2.2.2 按物理存储分类

从物理存储的角度，索引分为**<font color='cornflowerblue'>聚簇索引（主键索引）、二级索引（辅助索引）</font>**，这两个的主要区别在于：

- 主键索引的B+树叶子节点中存放的是**实际数据**，所有的记录都存放在主键索引的B+树叶子节点中；
- 二级索引的B+树的叶子节点存放的是**主键值**，需要再通过主键去查找实际数据。

#### 2.2.3 按字段特性分类

从字段特性分类，索引可以分为：**<font color='cornflowerblue'>主键索引、唯一索引、普通索引、前缀索引</font>**。

- **主键索引**：建立在主键上的索引。一张表只能有一个主键索引（逐渐只能有一个），不能重复，不允许空值NULL
- **唯一索引**：UNIQUE字段上建立的索引。一张表可以有多个唯一索引，允许存在空值

- **普通索引**：在普通字段上建立的索引，对字段没什么要求
- **前缀索引**：对字符类型（char、varchar、binary、varbinary）的字段的前几个字符建立的索引，而不是在整个字段上建立的索引

#### 2.2.4 按字段个数分类

从字段个数上来看，索引分为**单列索引、联合索引**。

- **单列索引**：建立在单个列上的索引，如主键索引；
- **联合索引**：建立在多个列上的索引

### 2.3 为什么使用B+树作为索引？

#### 2.3.1 B+Tree 索引的存储和查询的过程

B+树是一种**<font color='cornflowerblue'>多叉搜索树</font>**，只有**叶子节点才会存放实际数据，非叶子节点只会存放索引**，每个节点中数据是按照主键的顺序存放的。每一层父节点的索引值都会出现在下层子节点的索引值中，因此在**<font color='cornflowerblue'>叶子节点中，包括了所有的索引值信息，并且每一个叶子节点都有两个指针，分别指向下一个叶子节点和上一个叶子节点，形成一个双向链表</font>**。

**<font color='red'>主键索引</font>**的 B+Tree 如图所示（图中叶子节点之间画了单向链表，但是实际上是双向链表，脑补成双向链表就行）：

<img src="G:\code\study\CppStudy\docs\figures\btree.drawio.png" alt="主键索引 B+Tree" style="zoom:80%;" />



数据库的索引和数据都是存储在硬盘的，我们可以把**读取一个节点当作一次磁盘 I/O 操作**。B+Tree 存储千万级的数据只需要 3-4 层高度就可以满足，这意味着从千万级的表查询目标数据最多需要 3-4 次磁盘 I/O，所以**<font color='cornflowerblue'>B+Tree 相比于 B 树和二叉树来说，最大的优势在于查询效率很高，因为即使在数据量很大的情况，查询一个数据的磁盘 I/O 依然维持在 3-4次。</font>**

**<font color='red'>二级索引</font>**的 B+Tree 如下图（（图中叶子节点之间画了单向链表，但是实际上是双向链表，脑补成双向链表就行））：

<img src="G:\code\study\CppStudy\docs\figures\二级索引btree.drawio.png" alt="二级索引 B+Tree" style="zoom:80%;" />

会先检二级索引中的 B+Tree 的索引值，找到对应的叶子节点，然后获取主键值，然后再通过主键索引中的 B+Tree 树查询到对应的叶子节点，然后获取整行数据。**<font color='red'>这个过程叫「回表」，也就是说要查两个 B+Tree 才能查到数据</font>**。

不过，**当查询的数据是能在二级索引的 B+Tree 的叶子节点里查询到（说明查询的数据就是索引）**，这时就不用再查主键索引查，**<font color='cornflowerblue'>这种在二级索引的 B+Tree 就能查询到结果的过程就叫作「覆盖索引」，也就是只需要查一个 B+Tree 就能找到数据</font>**。

#### 2.3.2 InnoDB为什么使用B+树作为索引？

B+树相对于B树、二叉树、Hash表的优势：

- **<font color='cornflowerblue'>B+树 vs B树</font>**

  与B树相比，B+树只在叶子节点中存放数据，因此B+树单个节点可以存放更多的索引数据，可以使得**树的层数更小**。即能够**在相同的I/O次数下，查询到更多的节点**。

  另外，B+树的叶子节点通过**双链表连接**，很适合MySQL中的范围查询，B树无法做到这一点。

- **<font color='cornflowerblue'>B+树 vs 二叉树</font>**

  二叉树的一个父节点只能有两个子节点，搜索复杂度为`O(logN)`，当节点很多时会导致二叉树的层数很高，检索到目标数据所经历的磁盘I/O次数更多。

  对于B+树最大子节点个数为d个，实际应用中就保证了**即使数据达到千万级，B+树的高度依然在3~4层左右**。

- **<font color='cornflowerblue'>B+树 vs Hash表</font>**

  Hash表在做等值查询时效率非常快，搜索复杂度为O(1)。但是Hash表很难做范围查询。因此B+Tree 索引要比 Hash 表索引有着更广泛的适用场景。

### 2.4 联合索引是怎么发挥作用的？

#### 2.4.1 什么是联合索引？

通过将**多个字段组合成一个索引**，这个索引就叫做**联合索引**。联合索引`(product_no, name)` 的 B+Tree 示意图如下（图中叶子节点之间我画了单向链表，但是实际上是双向链表，原图我找不到了，修改不了，偷个懒我不重画了，大家脑补成双向链表就行）：

<img src="G:\code\study\CppStudy\docs\figures\联合索引.drawio.png" alt="联合![联合索引](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/%E7%B4%A2%E5%BC%95/%E8%81%94%E5%90%88%E7%B4%A2%E5%BC%95.drawio.png)索引" style="zoom:80%;" />

联合索引的非叶子节点**用两个字段的值作为 B+Tree 的 key 值**。当在联合索引查询数据时，先按 product_no 字段比较，在 product_no 相同的情况下再按 name 字段比较。这是**<font color='cornflowerblue'>最左匹配原则</font>**，按照最左优先的方式进行索引的匹配。在使用联合索引进行查询的时候，**<font color='red'>如果不遵循「最左匹配原则」，联合索引会失效</font>**，这样就无法利用到索引快速查询的特性了。

#### 2.4.2 最左匹配原则

创建了一个 `(a, b, c)` 联合索引，如果查询条件是以下这几种，就可以匹配上联合索引：

```mysql
where a=1；
where a=1 and b=2 and c=3；
where a=1 and b=2；
```

> 因为有查询优化器，所以 a 字段在 where 子句的顺序并不重要，但在理论上需要用标准化写法：按照顺序。

但是，如果查询条件是以下这几种，因为不符合最左匹配原则，所以就无法匹配上联合索引，联合索引就会失效:

```mysql
where b=2；
where c=3；
where b=2 and c=3；
```

这是因为**<font color='cornflowerblue'>利用索引的前提是索引里的 key 是有序的</font>**，对于联合索引，在保证 `a` 字段相等时，才能使得`b`有序。

#### 2.4.3 联合索引的范围查询

当联合索引中对某个字段使用了范围查询时，会导致联合索引到这个「范围查询」就会停止匹配。**<font color='cornflowerblue'>也就是范围查询的字段（及之前）可以用到联合索引，但是在范围查询字段的后面的字段无法用到联合索引</font>**。这也是由于**<font color='cornflowerblu'>利用索引的前提是索引里的 key 是有序的</font>**决定的。

> **联合索引的最左匹配原则，在遇到范围查询（如 >、<）的时候，就会停止匹配，也就是范围查询的字段可以用到联合索引，但是在范围查询字段的后面的字段无法用到联合索引。注意，对于 >=、<=、BETWEEN、like 前缀匹配的范围查询，并不会停止匹配**

#### 2.4.4 联合索引的索引下推

由于无法继续联合索引的匹配，在MySQL早期版本，需要<font color='red'>**「回表」**</font>。通过联合索引中**范围查询之前的字段**快速定位到符合条件的索引条目对应的主键值（如果是覆盖索引可以直接得到数据）。 使用获取到的主键值**回到原始数据表中，进一步读取需要的其他列数据**。这个步骤需要再次访问数据表，称为 **"回表"**。

显然，回表会带来一定的性能损失。因此在 MySQL 5.6 引入了**<font color='cornflowerblue'>索引下推优化</font>**，**可以在联合索引遍历过程中，对联合索引中包含的字段先做判断，直接过滤掉不满足条件的记录，减少回表次数**。

#### 2.4.5 联合索引的索引区分度

由于「最左匹配原则」，建立联合索引的顺序对索引效率有很大的影响。实际开发工作中，需要**把区分度大的字段排在前面**，可以很早地过滤掉大部分不符合的数据。区分度就是**某个字段 column 不同值的个数「除以」表的总行数**，计算公式如下：

<img src="G:\code\study\CppStudy\docs\figures\区分度.png" alt="区分度计算公式" style="zoom:50%;" />

### 2.5 什么时候需要/不需要索引？

索引主要目的是加快查询，但也不可避免地增加了内存占用和维护数据库表的难度，应有选择的使用。

#### 2.5.1 索引的优缺点

**优点**：

- 可以大幅度加快数据的查询速度（大部分情况下比全表扫描快得多）；
- 唯一性索引可能保证该字段中数据的唯一性；

**缺点**：

- 带来了额外的空间消耗。一个索引就是一颗B+树，B+树的每一个节点就是一个16KB的页，占用内存大；
- 索引的创建和维护会耗费很多时间，对表进行增删改时对应需要修改索引中的内容，降低了效率。

#### 2.5.2 什么时候需要创建索引

- 表很大，数据很多，全表扫描性能差；
- 频繁用于 `WHERE` 查询条件的字段；
- 频繁使用排序（`order by`）或分组（`group by`）的字段；
- 需要保证唯一性的字段。

#### 2.5.3 什么时候不需要创建索引

- 表数据比较小，全局扫描也不会有太大延迟；
- 很少用于 `WHERE` 查询条件、排序（`order by`）或分组（`group by`）的字段；
- 经常需要更新的字段最好不要创建索引，带来维护的性能瓶颈；
- 存在大量重复数据的字段（区分度不大，如性别等）。

### 2.6 优化索引的方法？

- **前缀索引优化**：对于字符串，可以通过字符串的前几个字符建立索引而不是整个，可以减小索引字段的大小；
- **覆盖索引优化**：尽量让查询的字段都在索引中，这样可以避免回表，减少大量的I/O操作；
- **主键索引最好是自增的**：每次插入一条新记录，都是追加操作，不需要移动数据，不会产生页分裂；
- **索引最好是NOT NULL**：索引列存在NULL值时会导致优化器更加难以优化查询的执行，且NULL值会占用物理空间

- **防止索引失效**：避免写出索引失效的查询语句

> 常见扫描类型的**执行效率从低到高的顺序为**：
>
> - `All`（全表扫描）；
> - `index`（全索引扫描）；
> - `range`（索引范围扫描）；
> - `ref`（非唯一索引扫描）；
> - `eq_ref`（唯一索引扫描）；
> - `const`（结果只有一条的主键或唯一索引扫描）。

### 2.6 索引失效有哪些？

查询条件用上了索引列，**查询过程不一定都用上索引**，接下来再一起看看哪些情况会导致索引失效，而发生全表扫描。

#### 2.6.1 对索引使用左模糊或左右模糊匹配

使用左或者左右模糊匹配的时候，也就是 `like %xx` 或者 `like %xx%` 这两种方式都会造成索引失效。**<font color='cornflowerblue'>因为索引 B+ 树是按照「索引值」有序排列存储的，只能根据前缀进行比较。</font>**使用带有左模糊的匹配时，无法确定前缀，则无法根据顺序查找，只能全局扫描。

#### 2.6.2 对索引使用函数或表达式计算

在查询条件中对索引进行表达式计算，也是无法走索引的。例如：

```mysql
select * from user where length(name)=6;  # 函数
selecr * from user where id + 1 = 10;	  # 表达式，改成 where id = 10 - 1，这样就不是在索引字段进行表达式计算了，于是就可以走索引查询了。
```

原因是建立的**<font color='cornflowerblue'>索引是对字段的原始值的</font>**，而不是对函数值或表达式计算后的值（`id + 1`），后面得到的值当然无法使用索引匹配。

#### 2.6.3 对索引隐式类型转换

例如，在表中phone的类型是`varchar`，但是查询语句如下：

```mysql
select * from user where phone = 1300000001;
```

这样就**<font color='cornflowerblue'>导致字段phone发生了隐式类型转换，则无法通过索引匹配</font>**，而是全局扫描。

> 特例：从字符串转换成整型不会。由于MySQL 在遇到字符串和数字比较的时候，会自动把字符串转为数字，然后再进行比较。所以其作用对象不是字段，而是整型。

#### 2.6.4 使用联合索引但不满足最左匹配

创建了联合索引，在使用时需要满足最左匹配原则，否则索引会失效，进行全局扫描。

#### 2.6.5 WHERE字句中的OR

在 WHERE 子句中，如果在 **OR 前的条件列是索引列，而在 OR 后的条件列不是索引列，那么索引会失效**。

这是因为 OR 的含义就是**<font color='cornflowerblue'>两个只要满足一个即可，因此只有一个条件列是索引列是没有意义的</font>**，只要有条件列不是索引列，就会进行全表扫描。

### 2.7 MySQL 单表不要超过 2000W 行，靠谱吗？

> - 非叶子节点内指向其他页的数量为 x
> - 叶子节点内能容纳的数据行数为 y
> - B+ 数的层数为 z

MySQL的**表数据**是以页（记录是行）的形式存放的，页在磁盘中不一定连续。 页的空间是 `16K` ,并不是所有的空间都是用来存放数据的，会有一些固定的信息，如，页头，页尾，页码，校验码等等，大概需要`1K`左右的空间，剩下的`15K`用来存放数据。

索引页中主要记录的是主键与页号，主键我们假设是 Bigint (8 byte), 而页号也是固定的（4Byte）, 那么索引页中的一条数据也就是 12byte。

**<font color='cornflowerblue'>所以 x=15*1024/12≈1280 行</font>。**

叶子节点和非叶子节点的结构是一样的，同理，能放数据的空间也是 15k。但是叶子节点中存放的是真正的行数据，这个影响的因素就会多很多，比如，**字段的类型，字段的数量。每行数据占用空间越大，页中所放的行数量就会越少**。

暂时按一条行数据 1k 来算，那一页就能存下 15 条，**<font color='cornflowerblue'>y = 15*1024/1000 ≈15</font>**。

Total =x^(z-1) *y，已知 x=1280，y=15：

- 假设 B+ 树是两层，那就是 z = 2， Total = （1280 ^1 ）*15 = 19200
- 假设 B+ 树是三层，那就是 z = 3， Total = （1280 ^2） *15 = 24576000 （约 2.45kw）

一般 B+ 数的层级最多也就是 3 层，**<font color='cornflowerblue'>能够存储两千多万行的数据</font>**。如果数据过多，导致三层的B+树无法完全存储，就会导致**<font color='red'>B+树的层数增加，则在查询时会导致多一次的I/O操作，造成性能下降</font>**。

### 2.8 count(*) 和 count(1) 有什么区别？哪个性能最好？

#### 2.8.1 count()聚合函数比较

<img src="G:\code\study\CppStudy\docs\figures\count性能.png" alt="图片" style="zoom:50%;" />

count() 是一个聚合函数，函数的参数不仅可以是字段名，也可以是其他任意表达式，该函数作用是**<font color='cornflowerblue'>统计符合查询条件的记录中，函数指定的参数不为 NULL 的记录有多少个</font>**。

- **count(\*) 执行过程跟 count(1) 执行过程基本一样的**，性能没有什么差异。
- **count(主键字段)需要遍历索引读取主键值**，根据主键值是否为NULL来增加计数；而count(\*) 和 count(1) 虽然也遍历索引，不需要读取任何字段的值，所以速度更快。
- **count(普通字段)** 由于该字段不是索引，所以必须通过全局扫描的方式，性能最差。

#### 2.8.2 为什么需要遍历来计数？

MyISAM 引擎时，执行 count 函数**<font color='cornflowerblue'>只需要 O(1 )复杂度</font>**，这是因为每张 MyISAM 的数据表都有一个 meta 信息有存储了row_count值，由表级锁保证一致性，所以直接读取 row_count 值就是 count 函数的执行结果。

InnoDB存储引擎是支持事务的，同一个时刻的多个查询，由于多版本并发控制（MVCC）的原因，InnoDB 表**“<font color='red'>应该返回多少行”也是不确定的（不同版本结果不同）</font>**，所以无法像 MyISAM一样，只维护一个 row_count 变量。

#### 2.8.2 如何优化count(*)？

- 近似值：使用 show table status 或者 explain 命令来表进行估算
- 额外维护一个表，来统计计数；

## 三、MySQL事务

### 3.1 什么是事务？ 有哪些特性？

如何保障数据库的某些操作是不可分割的，要么全部执行成功 ，要么全部失败，不允许出现中间状态的数据？ ——  「**<font color='cornflowerblue'>事务（Transaction）</font>**」

事务是**一组操作单元的集合**，能够使数据从一种状态转变为另一种状态。事务是由存储引擎中实现的，MylSAM不支持事务，InnoDB支持。

实现事务必须要遵守**<font color='cornflowerblue'> 4 个特性`ACID`</font>**：

- **<font color='cornflowerblue'>原子性（Atomicity，A）</font>**：一个事务中的操作要么全都完成，要么全都不完成，不会出现中间状态。若事务在执行的过程中出现错误，会**回滚到事务开始前的状态**（**通过`undo log`实现**）；
- **<font color='cornflowerblue'>一致性（Consistency，C）</font>**：事务操作前后，数据库保持一致性状态，即满足数据库的完整性约束和规则（由**持久性+原子性+隔离性，实现保证一致性**）；
- **<font color='cornflowerblue'>隔离性（Isolation，I）</font>**：数据库允许多个事务并发，多个事务并发同时使用相同的数据时，不会互相干扰，对于其他事务隔离的，这是通过**MVCC（多版本并发控制）**实现的；
- **<font color='cornflowerblue'>持久性（Durability，D）</font>**：事务处理结束后，对数据的修改是永久的，即使系统故障也不会丢失，持久性是**通过`redo log`来保证**的。

### 3.2 并发事务会出现什么问题？

MySQL 服务端是允许多个客户端连接的，这意味着 MySQL 会出现**同时处理多个事务的情况**。**<font color='red'>在同时处理多个事务的时候，就可能出现脏读（dirty read）、不可重复读（non-repeatable read）、幻读（phantom read）的问题</font>**。这是事务的**<font color='cornflowerblue'>隔离性</font>**的相关问题，也是为什么要引入隔离性的主要原因。

#### 3.2.1 什么是脏读？

**<font color='cornflowerblue'>如果一个事务「读到」另一个未提交事务所修改过的数据，就意味着发生了「脏读」现象。</font>**

因为这个未提交的事务有可能**执行失败而发生回滚**，那么这个事务读取到的数据就是过期的数据，于是产生了**「脏读」**

<img src="G:\code\study\CppStudy\docs\figures\脏读.png" alt="图片" style="zoom:80%;" />

#### 3.2.2 什么是不可重复读？

**<font color='cornflowerblue'>在一个事务内多次读取同一个数据，如果出现前后两次读到的数据不一样的情况，就意味着发生了「不可重复读」现象。</font>**

出现这种情况的主要原因是，事务B在事务A两次读取之间提交了，从而**修改了数据库**。

<img src="G:\code\study\CppStudy\docs\figures\不可重复读.png" alt="图片" style="zoom:80%;" />

#### 3.2.3 什么是幻读？

**<font color='cornflowerblue'>在一个事务内多次查询某个符合查询条件的「记录数量」，如果出现前后两次查询到的记录数量不一样的情况，就意味着发生了「幻读」现象。</font>**

<img src="G:\code\study\CppStudy\docs\figures\幻读.png" alt="图片" style="zoom:80%;" />

幻读与不可重复的区别在于满足查询条件的**记录数量发生了变化**，而不是记录本身发生变化。

#### 3.2.4 总结

当多个事务并发执行时可能会遇到「脏读、不可重复读、幻读」的现象，这些现象会**对事务的一致性产生不同程序的影响**。

- **<font color='cornflowerblue'>脏读</font>**：读到其他事务未提交的数据；
- **<font color='cornflowerblue'>不可重复读</font>**：前后读取的数据不一致；
- **<font color='cornflowerblue'>幻读</font>**：前后读取的记录数量不一致。

这三个现象的严重性排序如下：

<img src="G:\code\study\CppStudy\docs\figures\脏读不可重复幻读.png" style="zoom:80%;" />



MySQL通过**<font color='cornflowerblue'>隔离性</font>**来规避这三种现象，SQL标准提出了**<font color='cornflowerblue'>四种隔离级别</font>**，隔离级别越高，越能够避免上述现象的发生，事务的一致性更强，但同时也会带来并发性能的损失。

### 3.3 事务的隔离级别有哪些？ 怎么实现的？

#### 3.3.1 事务的四种隔离级别

MySQL中存在四种隔离级别，隔离级别越高，并发性能效率越低，但能够减少不一致性：

- **<font color='cornflowerblue'>读未提交</font>**：一个事务还没有提交时，对数据的修改可以被其他事务看到。会存在脏读、不可重复读、幻读现象；
- <font color='cornflowerblue'>**读已提交**</font>：只有事务被提交后，其他事务才可以看到它做的修改。存在不可重复读、幻读现象；
- <font color='cornflowerblue'>**可重复读**</font>：一个事务在其执行过程中多次执行相同的查询，始终看到相同的数据。存在幻读现象；**<font color='cornflowerblue'>（InnoDB的默认隔离级别）</font>**
- <font color='cornflowerblue'>**串行化**</font>：最高隔离级别，若事务访问相同的记录，则通过**加锁**实现同一时刻只有一个事务执行。

按隔离水平高低排序如下：

<img src="G:\code\study\CppStudy\docs\figures\隔离级别.png" alt="图片" style="zoom:80%;" />

针对不同的隔离级别，并发事务时可能发生的现象也会不同。

<img src="G:\code\study\CppStudy\docs\figures\各个隔离级别的现象.png" alt="图片" style="zoom:80%;" />

#### 3.3.2 隔离级别是如何实现的？

- **<font color='cornflowerblue'>读未提交</font>**：无需处理，直接读取最新的就行了；
- **<font color='cornflowerblue'>读已提交</font>**：MVCC机制 + 读写锁（只对查询的记录加锁，且读取完后立即释放），每次读取数据时会生成一个新的Read View，在这个Read View中，只能看到在生成前已提交的事务修改的数据；
- **<font color='cornflowerblue'>可重复读</font>**：MVCC机制 + 读写锁 + 间隙锁（锁会持续到事务结束），在启动事务时生成一个Read View，然后整个事务期间都是用这个Read View；
- <font color='cornflowerblue'>**串行化**</font>：通过加**读写锁**的方式实现互斥。

#### 3.3.3 什么是MVCC？

**MVCC**（多版本并发控制，Multiversion Concurrency Control）是一种用于管理数据库并发访问的方法。通过**<font color='cornflowerblue'>保存数据的多个版本</font>**来实现不同事务对数据的并发访问，从而避免了加锁导致的性能瓶颈，并减少了事务冲突。

一些关键概念：

- **版本链**：每条记录会存储多个版本，每个版本对应一次数据的修改操作。这些版本通常通过链表的方式链接在一起。
- **事务ID**：每个事务在启动时会获取一个唯一的事务ID。这个ID用于标识事务的先后顺序，并帮助MVCC判断哪些版本对当前事务可见。
- **快照（Snapshot）**：在某个事务读取数据时，数据库会生成一个快照，即当前可见的数据版本集合。这个快照包括了所有在事务开始之前已经提交的修改，但不包括事务开始之后的未提交修改。
- **Read View**：在可重复读（Repeatable Read）和读已提交（Read Committed）等隔离级别下，**<font color='cornflowerblue'>MVCC会为每个事务创建一个Read View</font>**。Read View决定了当前事务在访问数据时可以看到哪些版本。对于可重复读隔离级别，一个事务的Read View会在事务开始时生成，并在整个事务期间保持不变；对于读已提交隔离级别，每次读取数据时都会生成新的Read View。

#### 3.3.4 Read View在MVCC中是如何工作的？

Read View 有四个重要的字段：

- m_ids ：指的是在创建 Read View 时，当前数据库中「活跃事务」的**事务 id 列表**，注意是一个列表，**“活跃事务”指的就是，启动了但还没提交的事务**。
- min_trx_id ：指的是在创建 Read View 时，当前数据库中「活跃事务」中事务 **id 最小的事务**，也就是 m_ids 的最小值。
- max_trx_id ：这个并不是 m_ids 的最大值，而是**创建 Read View 时当前数据库中应该给下一个事务的 id 值**，也就是全局事务中最大的事务 id 值 + 1；
- creator_trx_id ：指的是**创建该 Read View 的事务的事务 id**。

<img src="G:\code\study\CppStudy\docs\figures\readview结构.drawio.png" alt="img" style="zoom:80%;" />

同时，对于聚簇索引中的记录行，其行结构中有隐藏列，其中存在：

- `trx_id`，当一个事务对某条聚簇索引记录进行改动时，就会**把该事务的事务 id 记录在 trx_id 隐藏列里**；
- `roll_pointer`，每次对某条聚簇索引记录进行改动时，都会把旧版本的记录写入到 undo 日志中，然后**这个隐藏列是个指针，指向每一个旧版本记录**，于是就可以通过它找到修改前的记录。

<img src="G:\code\study\CppStudy\docs\figures\隐藏列.png" alt="图片" style="zoom:80%;" />

于是，通过**<font color='cornflowerblue'>对比事务的Read View中和要查询的记录的`trx_id`就可以知道哪些记录是当前事务可见的</font>**。

- `trx_id < min_trx_id`：说明这条记录是创建这个Read View之前的事务提交的，可见。
- `trx_id >= max_trx_id`：说明这条记录是这个Read View创建之后的事务生成的，不可见。
-  `min_trx_id <= trx_id < max_trx_id`：
  - `trx_id`在`m_ids`中：说明还未提交，不可见；
  - `trd_id`不在`m_ids`中，说明已经提交，可见。

如果当前事务需要访问的记录在事务的Read View中不可见，数据库会**<font color='cornflowerblue'>尝试读取该记录的前一个可见版本</font>**。这是因为每条记录都有一个版本链，保存了历史版本。数据库通过遍历版本链，找到事务可以看到的最新版本来返回给用户。

**<font color='red'>这种通过「版本链」来控制并发事务访问同一个记录时的行为就叫 MVCC（多版本并发控制）。</font>**

### 3.4 可重复读是怎么尽量解决幻读的？

可重复读无法彻底解决幻读问题，但是串行化的隔离级别会导致并发性能很差。因此InnoDB使用「可重复读」作为默认隔离级别，因为**<font color='cornflowerblue'>「可重复读」可以很大程度上避免幻读现象</font>**。

#### 3.4.1 快照读和当前读

MySQL 里除了**<font color='cornflowerblue'>普通查询（select）是快照读，其他都是当前读</font>**，比如 update、insert、delete，这些语句执行前都会**查询最新版本的数据**，再做进一步的操作。

针对快照读，由于「可重复读」会在事务启动是就创建了Read View，后续的查询语句都是通过这个Read View来查询可见的数据。因此，**<font color='cornflowerblue'>「可重复读」隔离级别下的快照读不会产生幻读现象，MVCC方式可以解决幻读问题</font>**。

#### 3.4.2 当前读如何避免幻读？

对于当前读，因为每次执行的时候需要读取最新的数据，包括其他事务新提交的数据。所以可能会出现两次查询结果满足条件的记录数量不一致问题（幻读）。

> **为什么不会出现不可重复读现象？**
>
> 因为可重复读隔离级别会在事务执行期间加记录锁，避免了记录被修改，可以完全解决不可重复读现象。但是无法完全解决幻读现象。

因此InnoDB引擎引出了**<font color='cornflowerblue'>间隙锁</font>**来解决这个问题。事务执行期间，除了对记录本身加锁外，还会对前面的间隙加锁，防止新的记录插入。例如：表中有一个范围 id 为（3，5）间隙锁，那么其他事务就无法插入 id = 4 这条记录了，这样就有效的防止幻读现象的发生。

<img src="G:\code\study\CppStudy\docs\figures\gap锁.drawio.png" alt="img" style="zoom:80%;" />

总体来说，**<font color='red'>可重复读通过next-key锁（记录锁 + 间隙锁）来保证不会出现不可重复读现象，并尽量防止幻读现象的出现。</font>**

#### 3.4.3 幻读被完全解决了吗？

**<font color='cornflowerblue'>可重复读隔离级别下虽然很大程度上避免了幻读，但是还是没有能完全解决幻读</font>**。

下面这个场景也会发生幻读现象。

- T1 时刻：事务 A 先执行「快照读语句」：select * from t_test where id > 100 得到了 3 条记录。
- T2 时刻：事务 B 往插入一个 id= 200 的记录并提交；
- T3 时刻：事务 A 再执行「当前读语句」 select * from t_test where id > 100 for update 就会得到 4 条记录，此时也发生了幻读现象。

原因是T1时刻是快照读，不会加锁，直接通过MVCC实现的读取。而T3时刻是当前读，读取了最新的数据。

## 四、MySQL锁

### 4.1 MySQL有哪些锁？

#### 4.1.1 全局锁

全局锁就是**<font color='cornflowerblue'>对整个数据库实例加锁</font>**，主要用于**全库逻辑备份**等场景。

```mysql
flush tables with read lock # 加全局锁

unlock tables   # 解锁
```

加上全局（读）锁后，整个数据库都是**只读状态**。若数据库的数据较多，导致整个处理流程较慢，数据库长时间为只读状态，造成业务停滞、服务长时间不可用。

因此，由于**InnoDB支持事务，支持可重复读的隔离级别**，在备份数据库之前先开启事务。整个事务执行期间都在用这个 Read View，而且由于 MVCC 的支持，备份期间业务依然可以对数据进行更新操作。而如**MylSAM等不支持事务的引擎，就只能通过全局锁的方式**。

#### 4.1.2 表级锁

MySQL中存在多种表级锁：

- **表锁**

  锁住整张表，命令如下：

  ```mysql
  # 对表t_student加锁
  lock tables t_student read; 	# 表读锁
  lock tables t_student write; 	# 表读锁
  
  unlock tablse # 释放锁
  ```

  **<font color='red'>表锁除了会限制别的线程的读写外，也会限制本线程接下来的读写操作。</font>**

  如果本线程对学生表加了「共享表锁」，那么本线程接下来如果要对学生表执行**写操作的语句，是会被阻塞的**，当然其他线程对学生表进行写操作时也会被阻塞，直到锁被释放。

  > 尽量避免在使用 InnoDB 引擎的表使用表锁，因为表锁的颗粒度太大，会影响并发性能，**InnoDB 牛逼的地方在于实现了颗粒度更细的行级锁**。

- **元数据锁（meta data lock，MDL）**

  无需显示地调用MDL，每次访问一张表时会自动加上。MDL 是为了**保证当用户对表执行 CRUD（增删改查） 操作时，<font color='cornflowerblue'>防止其他线程对这个表结构做了变更</font>**。MDL 是在事务提交后才会释放，这意味着**<font color='cornflowerblue'>事务执行期间，MDL 是一直持有的</font>**。

  > 因此如果存在一个长事务对某个表加上了MDL读锁，如果此时有线程尝试修改这个表的结构，但无法拿到MDL写锁，则陷入阻塞。此后任何想要读或者写的线程都无法执行而是阻塞等待（因为申请 MDL 锁的操作会形成一个队列，队列中**写锁获取优先级高于读锁**）。
  >
  > 所以为了能安全的对表结构进行变更，在对表结构变更前，先要看看数据库中的长事务，是否**有长事务已经对表加上了 MDL 读锁**，如果可以考虑 kill 掉这个长事务，然后再做表结构的变更。

- **意向锁**

  意向锁是指示一个事务**<font color='cornflowerblue'>在未来可能会请求对某些资源的锁定</font>**。

  - 在使用 InnoDB 引擎的表里对**某些记录**加上「共享锁」之前，需要先在**<font color='cornflowerblue'>表级别</font>**加上一个「意向共享锁」；
  - 在使用 InnoDB 引擎的表里对**某些纪录**加上「独占锁」之前，需要先在**<font color='cornflowerblue'>表级别</font>**加上一个「意向独占锁」；

  当执行插入、更新、删除操作，需要**先对表加上「意向独占锁」，然后对该记录加独占锁**。查询不需要，因为查询是通过MVCC实现一致性读的，无需加锁。

  > **意向共享锁和意向独占锁是表级锁，不会和行级的共享锁和独占锁发生冲突，而且意向锁之间也不会发生冲突，只会和共享表锁（`lock tables ... read`）和独占表锁（`lock tables ... write`）发生冲突。**

  如果没有「意向锁」，那么加「独占表锁」时，就需要**遍历表里所有记录，查看是否有记录存在独占锁**，这样效率会很慢。那么有了「意向锁」，由于在对记录加独占锁前，先会加上表级别的意向独占锁，那么在加「独占表锁」时，直接查该表是否有意向独占锁，如果有就意味着表里已经有记录被加了独占锁，这样就不用去遍历表里的记录。**<font color='cornflowerblue'>意向锁的目的是为了快速判断表里是否有记录被加锁</font>**。

- **AUTO-INC锁（自增锁）**

  一个表中的主键通常是自增的，在插入数据时，**数据库会自动给主键赋值递增的值，这主要是通过AUTO-INC锁来实现的**。**<font color='cornflowerblue'>在插入数据时，会加一个表级别的 AUTO-INC 锁</font>**，然后为被 `AUTO_INCREMENT` 修饰的字段赋值**递增的值**，等插入语句执行完成后，才会把 AUTO-INC 锁释放掉。

  AUTO-INC 锁是特殊的表锁机制，锁**不是再一个事务提交后才释放，而是再<font color='cornflowerblue'>执行完插入语句后就会立即释放</font>**。

#### 4.1.3 行级锁

InnoDB 引擎是支持行级锁的，而 MyISAM 引擎并不支持行级锁。行级锁的类型主要有三类：

- **记录锁（Record Lock）**

  将一条记录加锁，又分为**共享锁（读锁、S锁）**和**排他锁（读锁、X锁）**，不同事物之间可能存在冲突关系。

- **间隙锁（Gap Lock）**

  锁定一个范围，不包含记录，间隙锁**不存在互斥关系**（因为不涉及到具体记录，于是不同事务可以同时包含共同范围的间隙锁），只存在于可重复读隔离级别下，用来防止可重复读隔离级别下的幻读现象。

  ![gap锁.drawio](G:\code\study\CppStudy\docs\figures\gap锁.drawio.png)

  

- **临键锁（Next-Key Lock）**

  就是Record Lock和Gap Lock的组合，锁定记录本身和一个范围。即能**<font color='cornflowerblue'>保护该记录，又能阻止其他事务将新纪录插入到被保护记录前面的间隙中</font>**。因此，虽然间隙锁是多个事务相互兼容的，但记录锁会存在冲突关系。

  ![img](G:\code\study\CppStudy\docs\figures\临键锁.drawio.png)



> 还有一种**特殊的间隙锁**：**插入意向锁**
>
> 它是指当一个插入操作发现插入的位置被加了间隙锁，那么这个线程想要向这个区域加上一个“插入意向锁”，只能阻塞等待，直到间隙锁被释放。

#### 4.1.4 乐观锁与悲观锁

- **乐观锁**

  一种思想，认为对同一个数据的并发操作发生概率较小，不需要每次都对数据上锁。常通过时间戳、版本号机制来实现。适用于**读操作比较多**的场景。

- **悲观锁**

  一种思想，认为总是会发生并发冲突，具有强烈独占性和排他性，通过锁机制来保护数据。适用于**写操作比较多**的场景。

### 4.2 MySQL是如何加行级锁的？

#### 4.2.1 什么SQL语句会加行级锁？

InnoDB 引擎支持行级锁，与表级锁相比，行级锁的并发性能要好很多。

普通的 select 语句是**不会对记录加锁的**（除了串行化隔离级别），因为它属于快照读，是通过 **<font color='cornflowerblue'>MVCC（多版本并发控制）实现</font>**的。但是可以通过显式指定的方式给select语句加行级锁：

```mysql
select ... lock in shared mode;    # 对读取的记录加共享锁
select ... for update;			   # 对读取的记录加排他锁
```

**<font color='cornflowerblue'>update 和 delete 操作都会加行级锁，且锁的类型都是独占锁(X型锁)</font>**。

#### 4.2.2 InnoDB两阶段锁协议

在InnoDB引擎中，可重复读隔离级别下，行级锁遵顼两阶段锁协议：**<font color='cornflowerblue'>在需要的时候加上，事务提交或出现回滚时才会释放</font>**（而不是操作完成了立即释放）。

#### 4.2.2 MySQL行级锁加锁规则

MySQL中（InnoDB引擎）行级锁**<font color='cornflowerblue'>加锁的对象是索引</font>**，加锁的**<font color='cornflowerblue'>基本单位是Next-Key Lock</font>**。在一些场景下，Next-Key Lock会退化成记录锁或间隙锁。

> **在能使用记录锁或者间隙锁就能避免幻读现象的场景下， next-key lock 就会退化成记录锁或间隙锁**。

- **唯一索引等值查询**

  若记录「存在」，在索引树上定位到这一条记录后，将该记录的索引中的 next-key lock 会**退化成「记录锁」**。

  若记录「不存在」，在索引树找到第一条大于该查询记录的记录后，将该记录的索引中的 next-key lock 会**退化成「间隙锁」**。

- **唯一索引范围查询**

  当唯一索引进行范围查询时，**会对每一个扫描到的索引加 next-key 锁**，在一些情况下的索引的next-key锁会退化为记录锁或间隙锁。

- **非唯一索引等值查询**

  当我们用非唯一索引进行等值查询的时候，因为存在两个索引，一个是主键索引，一个是非唯一索引（二级索引），所以在加锁时，**同时会对这两个索引都加锁**，但是对主键索引加锁的时候，**<font color='cornflowerblue'>只有满足查询条件的记录才会对它们的主键索引加锁</font>**。针对非唯一索引等值查询时，对于**扫描到的二级索引记录加 next-key 锁**，在某些情况下的二级索引的锁会退化为间隙锁。

- **非唯一索引范围查询**

  非唯一索引和主键索引的范围查询的加锁也有所不同，不同之处在于**非唯一索引范围查询，索引的 next-key lock 不会有退化为间隙锁和记录锁的情况**，也就是非唯一索引进行范围查询时，**<font color='cornflowerblue'>对二级索引记录加锁都是加 next-key 锁</font>**。

#### 4.2.3 没有索引的查询会发生什么？

对于存在索引的查询，查询语句都有使用索引查询，也就是查询记录的时候，是通过索引扫描的方式查询的，然后**对扫描出来的记录进行加锁**。

如果**<font color='red'>锁定读查询语句</font>**，没有使用索引列作为查询条件，或者查询语句没有走索引查询，导致扫描是**<font color='cornflowerblue'>全表扫描</font>**。那么，**<font color='cornflowerblue'>每一条记录的索引上都会加 next-key 锁，这样就相当于锁住的全表，这时如果其他事务对该表进行增、删、改操作的时候，都会被阻塞</font>**。

> **注意：这里是说对整张表的索引都加锁，而不是对表加锁。**

#### 4.2.4 可重复读隔离级别中Next-Key Lock可以防止删除操作导致的幻读吗？

前面说到，在可重复读隔离级别中，通过使用「记录锁+间隙锁」可以很大概率上避免新插入数据带来的幻读现象。

> **这种方案可以防止删除操作带来的幻读现象吗？**

可以大概率避免，因为**<font color='cornflowerblue'>当前读</font>**的语句会对索引加Next-Key Lock，其他事务对被加锁的记录和间隙上增、删、改的操作都会被阻塞。

#### 4.2.5 MySQL中加了什么锁会导致死锁？

死锁的四个条件：**<font color='cornflowerblue'>互斥、占有且等待、不可强占用、循环等待</font>**

由于间隙锁不会互斥，而当想要插入数据时，如果某个范围正好存在间隙锁，那么这个插入事务会向这个区域加上一个**<font color='cornflowerblue'>“插入意向锁”并等待，直到间隙锁被释放</font>**。

<img src="G:\code\study\CppStudy\docs\figures\ab事务死锁.drawio.png" alt="img" style="zoom:80%;" />

如上图所示，事务A和事务B在分别执行完update语句后，都含有一个间隙锁（事务结束后才会释放锁）。从下图的表中数据可以分析得到，**<font color='cornflowerblue'>两个事务的间隙锁的范围都是（20，30）</font>**。

![img](G:\code\study\CppStudy\docs\figures\t_student.png)

因此，两个事务分别向对方持有的间隙锁范围内插入一条记录，而**<font color='cornflowerblue'>插入操作为了获取到插入意向锁，都在等待对方事务的间隙锁释放</font>**，于是就造成了循环等待，满足了死锁的四个条件，因此发生了死锁。

#### 4.2.6 如何避免死锁？

- **破坏死锁条件**

  **<font color='cornflowerblue'>互斥、占有且等待、不可强占用、循环等待</font>**。只要系统发生死锁，这些条件必然成立，但是只要破坏任意一个条件就死锁就不会成立。

- **设置事务等待锁超时时间**

  当一个事务阻塞等待时间超过阈值后，直接对该事务进行回滚，避免死锁。

- **开启死锁检测**

  MySQL支持死锁检测，开启后当发现死锁时，会主动回滚死锁链条中的某个事务，解除死锁。

### 4.3 InnoDB使用表锁还是行锁？

为了保证并发性能，InnoDB引擎**在绝大部分情况使用行级锁**。

使用表级锁的情况：

- 表比较大，需要对表中全部或大部分数据进行更新；
- 事务涉及到多个表，比较复杂，行锁可能会引起死锁，导致事务大量回滚。

## 五、MySQL日志

### 5.1 MySQL中有哪些日志？

MySQL中主要有三种日志：undo log（回滚日志）、redo log（重做日志）、binlog（归档日志），简单介绍：

- **undo log（回滚日志）**：InnoDB存储引擎层生成的日志，保证事务中的**原子性**，用于**事务回滚和MVCC**；
- **redo log（重做日志）**：InnoDB存储引擎层生成的日志，保证事务中的**持久性**，用于**掉电等故障恢复**；
- **binlog（归档日志）**：Server层生成的日志，用于**数据备份和主从复制**。

### 5.2 为什么需要undo log？

#### 5.2.1 undo log是什么？

当事务对数据库进行更新（插入、修改、删除）时，MySQL会记录**<font color='cornflowerblue'>更新前的数据到undo log文件中</font>**，当事务回滚时，可以利用undo log来实现。如图：

<img src="G:\code\study\CppStudy\docs\figures\回滚事务.png" alt="回滚事务" style="zoom:80%;" />

每当 InnoDB 引擎对一条记录进行操作（修改、删除、新增）时，要把回滚时需要的信息都记录到 undo log 里，比如：

- 在**插入**一条记录时，要把这条记录的主键值记下来，这样之后回滚时只需要把这个主键值对应的记录**删掉**就好了；
- 在**删除**一条记录时，要把这条记录中的内容都记下来，这样之后回滚时再把由这些内容组成的记录**插入**到表中就好了；
- 在**更新**一条记录时，要把被更新的列的旧值记下来，这样之后回滚时再把这些列**更新为旧值**就好了。

不同的操作，需要记录的内容也是不同的，所以不同类型的操作（修改、删除、新增）产生的 undo log 的格式也是不同的。

#### 5.2.2 undo log的两大作用？

除了事务回滚作用。前面提到，MySQL中一个记录的真实数据中有几个隐藏字段，包括 roll_pointer 指针和一个 trx_id 事务id。

- 通过 trx_id 可以知道该记录是被哪个事务修改的；
- 通过 roll_pointer 指针可以将这些 undo log 串成一个链表，**这个链表就被称为<font color='cornflowerblue'>版本链</font>**；

<img src="G:\code\study\CppStudy\docs\figures\版本链.png" style="zoom:80%;" />

通过trx_id 和版本链，利用**<font color='cornflowerblue'>ReadView + undo log 可以实现MVCC（多版本并发控制）</font>**。

「事务的 Read View 里的字段」和「记录中的两个隐藏列（trx_id 和 roll_pointer）」的比对，如果不满足可见性，就会**<font color='cornflowerblue'>顺着 undo log 版本链里找到满足其可见性的记录</font>**，从而控制并发事务访问同一个记录时的行为，这就叫 MVCC（多版本并发控制）。

因此，undo log 两大作用：

- <font color='cornflowerblue'>**实现事务回滚，保障事务的原子性**</font>。事务处理过程中，如果出现了错误或者用户执 行了 ROLLBACK 语句，MySQL 可以利用 undo log 中的历史数据将数据恢复到事务开始之前的状态。
- **<font color='cornflowerblue'>实现 MVCC（多版本并发控制）关键因素之一</font>**。MVCC 是通过 ReadView + undo log 实现的。undo log 为每条记录保存多份历史数据，MySQL 在执行快照读（普通 select 语句）的时候，会根据事务的 Read View 里的信息，顺着 undo log 的版本链找到满足其可见性的记录。

### 5.3 为什么需要redo log？

#### 5.3.1 认识Buffer Pool

MySQL的数据是保存在磁盘的，但是频繁的进行磁盘I/O会影响性能。因此，InnoDB存储引擎设计了一个**<font color='cornflowerblue'>缓冲池（Buffer Pool）</font>**，提高数据库的读写性能。

<img src="G:\code\study\CppStudy\docs\figures\缓冲池.drawio.png" alt="Buffer Poo" style="zoom:60%;" />

有了 Buffer Poo 后：

- 当读取数据时，如果数据存在于 Buffer Pool 中，客户端就会**直接读取 Buffer Pool 中的数据**，否则再去磁盘中读取。
- 当修改数据时，如果数据存在于 Buffer Pool 中，那直接修改 Buffer Pool 中数据所在的页，然后将其页设置为**<font color='cornflowerblue'>脏页（该页的内存数据和磁盘上的数据已经不一致）</font>**，为了减少磁盘I/O，不会立即将脏页写入磁盘，后续**由后台线程选择一个合适的时机将脏页写入到磁盘**。

这样可以显著提高数据库的性能。

#### 5.3.2 Buffer Pool缓存什么？

由于InnoDB存储引擎中，使用**<font color='cornflowerblue'>「页」作为磁盘与内存交互的基本单位</font>**。因此Buffer Pool中也是由页组成的，在 MySQL 启动的时候，**InnoDB 会为 Buffer Pool 申请一片连续的内存空间，然后按照默认的`16KB`的大小划分出一个个的页， Buffer Pool 中的页就叫做<font color='cornflowerblue'>缓存页</font>**。初始时，这些缓存页都是空闲的，之后随着程序的运行，才会有磁盘上的页被缓存到 Buffer Pool 中。

<img src="G:\code\study\CppStudy\docs\figures\bufferpool内容.drawio.png" alt="img" style="zoom:80%;" />

Buffer Pool 除了缓存**<font color='cornflowerblue'>「索引页」和「数据页」</font>**，还包括了 **<font color='cornflowerblue'>Undo 页，插入缓存页、自适应哈希索引、锁信息</font>**等等。

> **数据库更新时，undo log 就是写入 Buffer Pool 中的 Undo 页面中的。**

Buffer Pool 和磁盘交换的**单位是页**，当我们查询一条记录时，InnoDB 是会把整个页的数据加载到 Buffer Pool 中，将页加载到 Buffer Pool 后，再通过页里的「页目录」去定位到某条具体的记录。

#### 5.3.3 redo log是什么？

Buffer Pool 是基于内存的，而内存总是不可靠，万一**断电重启，还没来得及落盘的脏页数据就会丢失**。因此，当有一条记录需要更新时，InnoDB首先更新内存，并将这个Buffer Pool页面标记为脏页，然后**<font color='cornflowerblue'>把本次对这个页的修改以redo log的形式记录下来，此时已经算是完成更新了</font>**（但实际上还没有存储到磁盘）。

后续，InnoDB 引擎会在适当的时候，由后台线程将缓存在 Buffer Pool 的脏页刷新到磁盘里，这就是 **<font color='cornflowerblue'>WAL （Write-Ahead Logging）技术： MySQL 的写操作并不是立刻写到磁盘上，而是先写日志，然后在合适的时间再写到磁盘上</font>**。

<img src="G:\code\study\CppStudy\docs\figures\wal.png" alt="img" style="zoom:60%;" />

redo log 是**物理日志**，记录了某个数据页做了什么修改，比如**对 XXX 表空间中的 YYY 数据页 ZZZ 偏移量的地方做了AAA 更新**，每当执行一个事务就会产生这样的一条或者多条物理日志。在事务提交时，只要**<font color='cornflowerblue'>先将 redo log 持久化到磁盘即可</font>**，可以不需要等到将缓存在 Buffer Pool 里的脏页数据持久化到磁盘。

当系统崩溃时，虽然脏页数据没有持久化，但是 redo log 已经持久化，接着 MySQL 重启后，可以根据 redo log 的内容，将所有数据恢复到最新的状态。

#### 5.3.4 redo log和undo log的区别

这两种日志是属于 InnoDB 存储引擎的日志，它们的区别在于：

- redo log 记录了此次事务「**<font color='cornflowerblue'>完成后</font>**」的数据状态，记录的是更新**之后**的值；
- undo log 记录了此次事务「**<font color='cornflowerblue'>开始前</font>**」的数据状态，记录的是更新**之前**的值；

事实上，当InnoDB层更新记录时，会生成一条undo log，这个undo log会写入到Buffer Pool中的Undo页，此时这个写入过程也会被记录对应的redo log。所以，可以说：**<font color='cornflowerblue'>undo log 和数据页的刷盘（持久化到磁盘）策略是一样的，都需要通过 redo log 保证持久化。</font>**

<img src="G:\code\study\CppStudy\docs\figures\事务恢复.png" alt="事务恢复" style="zoom:80%;" />

#### 5.3.5 redo log 要写到磁盘，数据也要写磁盘，为什么要多此一举？

写入 redo log 的方式使用了追加操作， 所以磁盘操作是**<font color='cornflowerblue'>顺序写</font>**，而写入数据需要先找到写入位置，然后才写到磁盘，所以磁盘操作是**<font color='cornflowerblue'>随机写</font>**。

**磁盘的「顺序写 」比「随机写」 高效的多**，因此 redo log 写入磁盘的开销更小。

至此， 针对为什么需要 redo log 这个问题我两个答案：

- **<font color='cornflowerblue'>实现事务的持久性，让 MySQL 有 crash-safe 的能力</font>**，能够保证 MySQL 在任何时间段突然崩溃，重启后之前已提交的记录都不会丢失；
- **<font color='cornflowerblue'>将写操作从「随机写」变成了「顺序写」</font>**，提升 MySQL 写入磁盘的性能。

#### 5.3.6 redo log是怎么写入磁盘的？ 是直接写入吗？

redo log也不是直接写入磁盘的，而是有自己的缓存：**<font color='cornflowerblue'>redo log buffer</font>**。每当产生一条 redo log 时，会先写入到 redo log buffer，后续在持久化到磁盘。redo log buffer 默认大小 16 MB，可以通过 `innodb_log_Buffer_size` 参数动态的调整大小，增大它的大小可以让 MySQL 处理「大事务」是不必写入磁盘，进而提升写 IO 性能。

> **缓存在 redo log buffer 里的 redo log 还是在内存中，它什么时候刷新到磁盘？**

主要有下面几个时机：

- MySQL **正常关闭时**；
- 当 redo log buffer 中记录的写入量大于 redo log buffer **内存空间的一半时**，会触发落盘；
- InnoDB 的**后台线程每隔 1 秒**，将 redo log buffer 持久化到磁盘。
- 每次**事务提交时**都将缓存在 redo log buffer 里的 redo log 直接持久化到磁盘（由参数 `innodb_flush_log_at_trx_commit` 参数控制）。

#### 5.3.7 redo log文件写满了怎么办？

默认情况下， InnoDB 存储引擎有 1 个重做日志文件组( redo log Group），「重做日志文件组」由有 2 个 redo log 文件组成。在重做日志组中，每个 **redo log File 的大小是固定且一致的**，假设每个 redo log File 设置的上限是 1 GB，那么总共就可以记录 2GB 的操作。

重做日志文件组是以**<font color='cornflowerblue'>循环写</font>**的方式工作的，从头开始写，写到末尾就又回到开头，相当于一个环形。所以 InnoDB 存储引擎会先写 ib_logfile0 文件，当 ib_logfile0 文件被写满的时候，会切换至 ib_logfile1 文件，当 ib_logfile1 文件也被写满时，会切换回 ib_logfile0 文件。

![重做日志文件组写入过程](G:\code\study\CppStudy\docs\figures\重做日志文件组写入过程.drawio.png)

redo log 是循环写的方式，相当于一个环形，InnoDB 用 **write pos 表示 redo log 当前记录写到的位置，用 checkpoint 表示当前要擦除的位置**，如下图：

<img src="G:\code\study\CppStudy\docs\figures\checkpoint.png" alt="img" style="zoom:50%;" />

write pos 追上了 checkpoint，就意味着 **<font color='cornflowerblue'>redo log 文件满了，这时 MySQL 不能再执行新的更新操作，也就是说 MySQL 会被阻塞</font>**。

此时**会停下来<font color='cornflowerblue'>将 Buffer Pool 中的脏页刷新到磁盘中，然后标记 redo log 哪些记录可以被擦除，接着对旧的 redo log 记录进行擦除，等擦除完旧记录腾出了空间，checkpoint 就会往后移动（图中顺时针）</font>**，然后 MySQL 恢复正常运行，继续执行新的更新操作。

### 5.4 为什么需要binlog？

#### 5.4.1 什么是binlog？

除了InnoDB存储引擎层生成的undo log和redo log，MySQL在完成一条更新操作后，Server层还会生成一条binlog（二进制日志，binary log），**事务提交时会将该事务执行过程中产生的binlog统一写入binlog 文件**。

最开始 MySQL 里并没有 InnoDB 引擎，MySQL 自带的引擎是 MyISAM，但是 MyISAM 没有 crash-safe 的能力，binlog 日志只能用于归档。InnoDB 是另一个公司以插件形式引入 MySQL 的，既然只依靠 binlog 是没有 crash-safe 能力的，所以 InnoDB 使用 redo log 来实现 crash-safe 能力。

#### 5.4.2 redo log和binlog的区别？

- **<font color='cornflowerblue'>适用对象不同</font>**：binlog是Server层实现的，所以引擎都可以使用；redo log是InnoDB引擎实现的。
- **<font color='cornflowerblue'>文件格式不同</font>**：binlog有三种格式`statement`、`row`、`mixed`，区别如下：
  - statement：记录修改数据的SQL语句，相当于记录了逻辑操作，因此binlog可以成为逻辑日志；
  - row：记录数据最终被修改成什么样了，此时就不是逻辑日志了；
  - mixed：根据不同情况自动选择使用statement模式或row模式。

- **<font color='cornflowerblue'>写入方式不同</font>**：binlog 是追加写，写满一个文件就新创建一个文件继续写；redo log是循环追加写，日志空间大小固定，写满了就从头开始。
- **<font color='cornflowerblue'>用途不同</font>**：binlog主要用于备份恢复（因为是全量记录的）、主从复制；redo log用于掉电等故障恢复（只能存一部分）

> **如果不小心整个数据库的数据被删除了，能使用 redo log 文件恢复数据吗？**
>
> 不可以使用 redo log 文件恢复，只能使用 binlog 文件恢复。
>
> 因为 redo log 文件是循环写，是会边写边擦除日志的，只记录未被刷入磁盘的数据的物理日志，已经刷入磁盘的数据都会从 redo log 文件里擦除。
>
> binlog 文件保存的是全量的日志，也就是保存了所有数据变更的情况，理论上只要记录在 binlog 上的数据，都可以恢复，所以如果不小心整个数据库的数据被删除了，得用 binlog 文件恢复数据。

#### 5.4.3 binlog什么时候刷盘？

事务执行过程中，先把日志写到 binlog cache（Server 层的 cache），**事务提交的时候**，再把 binlog cache 写到 binlog 文件中。对于一个事务，**<font color='cornflowerblue'>无论这个事务有多大（比如有很多条语句），也要保证一次性写入</font>**。这是为了保证事务的原子性。

**<font color='cornflowerblue'>在事务提交的时候，执行器把 binlog cache 里的完整事务写入到 binlog 文件中，并清空 binlog cache</font>**。如下图：

<img src="G:\code\study\CppStudy\docs\figures\binlogcache.drawio.png" alt="binlog cach" style="zoom:80%;" />

虽然每个线程有自己 binlog cache，但是最终**都写到同一个 binlog 文件**：

- 图中的 write，指的就是指把日志写入到 binlog 文件，但是并没有把数据持久化到磁盘，因为数据还缓存在文件系统的 page cache 里，write 的写入速度还是比较快的，因为不涉及磁盘 I/O。
- 图中的 fsync，才是**将数据持久化到磁盘的操作（刷盘）**，这里就会涉及磁盘 I/O，所以频繁的 fsync 会导致磁盘的 I/O 升高。

MySQL提供一个 sync_binlog 参数来控制数据库的 binlog 刷到磁盘上的频率：

- sync_binlog = 0 的时候，表示每次提交事务都只 write，不 fsync，后续交**<font color='cornflowerblue'>由操作系统决定何时将数据持久化到磁盘</font>**；
- sync_binlog = 1 的时候，表示每次**<font color='cornflowerblue'>提交事务都会 write，然后马上执行 fsync</font>**；
- sync_binlog =N(N>1) 的时候，表示每次提交事务都 write，但**<font color='cornflowerblue'>累积 N 个事务后才 fsync</font>**。

### 5.5 为什么需要两阶段提交？

#### 5.5.1 为什么需要两阶段提交？

事务提交后，redo log和binlog都需要持久化到磁盘，但是它们是独立的逻辑，可能出现**半成功（只有一个成功持久化了）**的状态，于是两份日志之间的逻辑不一致，存在两种情况：

- **如果在将 redo log 刷入到磁盘之后， MySQL 突然宕机了，而 binlog 还没有来得及写入**。
- **如果在将 binlog 刷入到磁盘之后， MySQL 突然宕机了，而 redo log 还没有来得及写入**。

 redo log 影响主库的数据，binlog 影响从库的数据，所以 redo log 和 binlog 不一致会导致**<font color='cornflowerblue'>主从环境中的数据不一致性</font>**。

MySQL 为了避免出现两份日志之间的逻辑不一致的问题，使用了「两阶段提交」来解决，**<font color='cornflowerblue'>两阶段提交其实是分布式事务一致性协议，它可以保证多个逻辑操作要不全部成功，要不全部失败，不会出现半成功的状态</font>**。

#### 5.5.2 两阶段提交的过程是怎样的？

**两阶段提交把单个事务的提交拆分成了 2 个阶段，分别是「准备（Prepare）阶段」和「提交（Commit）阶段」**，每个阶段都由协调者（Coordinator）和参与者（Participant）共同完成。在 MySQL 的 InnoDB 存储引擎中，开启 binlog 的情况下，MySQL 会同时维护 binlog 日志与 InnoDB 的 redo log，为了保证这两个日志的一致性，MySQL 使用了**内部 XA 事务**，内部 XA 事务由 **<font color='cornflowerblue'>binlog 作为协调者，存储引擎是参与者</font>**。

当客户端执行 commit 语句或者在自动提交的情况下，MySQL 内部开启一个 **<font color='cornflowerblue'>XA 事务</font>**，**分两阶段来完成 XA 事务的提交**，如下图：

<img src="G:\code\study\CppStudy\docs\figures\两阶段提交.drawio.png" alt="两阶段提交" style="zoom:60%;" />

事务的提交包含两个阶段，本质上就是**<font color='cornflowerblue'>将 redo log 的写入拆成了两个步骤：prepare 和 commit，中间再穿插写入binlog</font>**：

- **<font color='cornflowerblue'>准备阶段（prepare）</font>**：将内部XA事务的ID写入到redo log，同时将redo log对应的事务状态设置为prepare，然后将redo log持久化到磁盘；
- <font color='cornflowerblue'>**提交阶段（commit）**</font>：将内部XA事务的ID写入到binlog，然后将binlog持久化到磁盘。将redo log的状态设置为commit，此时该状态不需要立马持久化到磁盘中，写入到缓存区就够了。

#### 5.5.3 两阶段提交如何保证数据一致的？

下图中有时刻 A 和时刻 B 都有可能发生崩溃，不管是时刻 A（redo log 已经写入磁盘， binlog 还没写入磁盘），还是时刻 B （redo log 和 binlog 都已经写入磁盘，还没写入 commit 标识）崩溃，**<font color='cornflowerblue'>此时的 redo log 都处于 prepare 状态</font>**。

<img src="G:\code\study\CppStudy\docs\figures\两阶段提交崩溃点.drawio.png" alt="时刻 A 与时刻 B" style="zoom:60%;" />

MySQL 重启后会按顺序扫描 redo log 文件，碰到处于 prepare 状态的 redo log，就拿着 redo log 中的 XID 去 binlog 查看是否存在此 XID：

- **<font color='cornflowerblue'>如果 binlog 中没有当前内部 XA 事务的 XID，说明 redolog 完成刷盘，但是 binlog 还没有刷盘，则回滚事务</font>**。对应时刻 A 崩溃恢复的情况。
- **<font color='cornflowerblue'>如果 binlog 中有当前内部 XA 事务的 XID，说明 redolog 和 binlog 都已经完成了刷盘，则提交事务</font>**。对应时刻 B 崩溃恢复的情况。

这样就可以保证 redo log 和 binlog 这两份日志的一致性了。

#### 5.5.4 两阶段提交存在什么问题？

两阶段提交虽然保证了两个日志文件的数据一致性，但是**性能很差**，主要有两个方面的影响：

- **<font color='red'>磁盘I/O次数高</font>**：每个事务提交都会进行两次刷盘，redo log刷盘和binlog刷盘；
- **<font color='red'>锁竞争激烈</font>**：在「多事务」的情况下，两阶段提交不能保证两者的提交顺序一致，因此，在两阶段提交的流程基础上，还需要**加一个锁**来保证提交的原子性，从而保证多事务的情况下，两个日志的提交顺序一致。

#### 5.5.5 MySQL 磁盘 I/O 很高，有什么优化的方法？

事务在提交的时候，需要将 binlog 和 redo log 持久化到磁盘，那么如果出现 MySQL 磁盘 I/O 很高的现象，我们可以通过**<font color='cornflowerblue'>控制一些参数（主要是指定redo log和binlog何时进行刷盘的参数）</font>**，来 **<font color='cornflowerblue'>“延迟” binlog 和 redo log 刷盘的时机</font>**，从而降低磁盘 I/O 的频率。

## 六、MySQL主从复制和读写分离

### 6.1 什么是主从复制？

数据量都比较大，而单台数据库在数据存储、安全性和高并发方面都无法满足实际的需求，所以需要配置多台主从数据服务器。

MySQL 主从复制是数据库**高可用性架构**中的一种常见实现方式，将数据从**MySQL数据库服务器（主服务器）**复制到**一个或多个MySQL数据库服务器（从服务器）**，目的是能够实现数据的备份、读写分离、负载均衡等，提高MySQL服务的可用性。

MySQL 主从复制通常采用一主多从的拓扑结构，所有的数据**<font color='cornflowerblue'>写操作都在主服务器上执行</font>**，而从服务器主要用于处理**<font color='cornflowerblue'>读操作和备份</font>**。

### 6.2 主从复制是怎么实现的？

MySQL的主从复制依赖于binlog，该日志将MySQL的所有更新以二进制的形式保存在磁盘上，复制的过程就是**将binlog从主库传输到从库上**。

这个过程**<font color='cornflowerblue'>一般是异步</font>**的，也就是主库上执行事务操作的线程不会等待复制 binlog 的线程同步完成。

<img src="G:\code\study\CppStudy\docs\figures\主从复制过程.drawio.png" alt="MySQL 主从复制过程" style="zoom:80%;" />

MySQL集群的主从复制过程可以划分为下面几个阶段：

- **<font color='cornflowerblue'>写入Binlog</font>**：主库在收到提交事务的请求之后，会先写入binlog，再提交事务，更新存储引擎中的数据，返回客户端操作成功的响应；
- **<font color='cornflowerblue'>请求Binlog</font>**：从库中有一个专门的I/O线程，用于向主库请求从指定位置之后的binlog日志内容；
- **<font color='cornflowerblue'>同步Binlog</font>**：主库中维护了一个log dump线程，在收到binlog同步的请求后，向从库传输相应的日志数据，从库中的I/O线程收到这些数据，会将其写入到本地的**<font color='red'>relay log（中继日志）</font>**中，并返回主库接收成功的响应；
- **<font color='cornflowerblue'>回放Binlog</font>**：从库中I/O线程接收了binlog只是放到了relay log，还维护了一个SQL线程从relay log读取新同步过来的数据，解析成SQL语句逐一执行，最终保证主从数据的一致性。

<img src="G:\code\study\CppStudy\docs\figures\主从复制2.png" alt="[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-dPrvD0SI-1620876295915)(E:/笔记/JAVA/Java复习框架-数据库/Mysql/temp2/5.png)]" style="zoom:80%;" />

### 6.3 主从复制的几种模式？

#### 6.3.1 异步模式（async-mode）

MySQL主从复制**默认的模式就是异步模式**，也就是上文中的过程。主库中不会主动向从库推送binlog，并且主库自身完成客户端提交的事务后会立马响应给客户端，并**<font color='cornflowerblue'>不关心从库是否接收到新的同步日志</font>**。

**<font color='red'>如果主节点突然崩溃了，可能导致更新的数据没有同步到从节点上，如果将一个从库提升为主库，会导致数据的丢失，从而产生不一致性</font>**。

<img src="G:\code\study\CppStudy\docs\figures\异步模式.png" alt="[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-LiYZE5yK-1620876295918)(E:/笔记/JAVA/Java复习框架-数据库/Mysql/temp2/6.png)]" style="zoom:80%;" />

#### 6.3.2 同步模式 （sync-mode）

同步模式中MySQL主库中**<font color='cornflowerblue'>提交事务的线程需要阻塞等待所有从库的复制成功响应，才会将结果返回给客户端</font>**。这种模式可以保证更新的数据被同步到了所有的从库中，但是在实际情况中，一般无法使用：

- **性能很差**，需要等待所有从库同步完成才会响应；
- 有一个主库或从库不可用时，**整体的服务就不可用**。

#### 6.3.3 半同步模式（semi-sync）

MySQL 5.7版本之后新增加了一种模式：半同步模式，介于异步模式和同步模式之间。事务线程不需要等待所有的从库都完成同步，只需要**一部分复制成功即可**。这种**<font color='cornflowerblue'>半同步复制的方式，兼顾了异步复制和同步复制的优点，即使出现主库宕机，至少还有一个从库有最新的数据，不存在数据丢失的风险</font>**。

<img src="G:\code\study\CppStudy\docs\figures\半同步模式" alt="[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-W6DVdDkG-1620876295920)(E:/笔记/JAVA/Java复习框架-数据库/Mysql/temp2/7.png)]" style="zoom:80%;" />

#### 6.3.3 GTID模式

> **什么是GTID？**

GTID（Global Transaction ID，全局事务标识符）复制模式是 MySQL 中用于实现主从复制的一种**增强机制**。相比于传统的基于二进制日志位置（Binary Log Position）的复制方式，**<font color='cornflowerblue'>GTID 提供了一种更加简洁、可靠的方式来管理和跟踪主从复制的事务状态。</font>**

**全局事务标识符（GTID）**：全局唯一的标识符，标记在 MySQL 主服务器上执行的每一个事务，一般来说**：<font color='cornflowerblue'>`GTID = source_id:transaction_id`</font>**，即有两个部分组成：

-  `source_id` ：主服务器的唯一标识（一般是服务器的UUID）；
- `transaction_id`：在主服务器上执行的事务的序号

通过GTID，**每个事务在MySQL集群中都有唯一的GTID**，无论事务被复制到多少个从服务器，GTID总是不变的。

> **GTID的工作原理？**

- **事务执行**：当主服务器在执行一个事务时，会给这个事务生成一个唯一的GTID，将这GTID和事务一起记录到binlog中；
- **日志同步**：从服务器向主服务器请求日志同步时，从服务器会向主服务器发送它已经处理的最后一个 GTID 及其 GTID 集（GTID Set），主服务器可以通过这个GTID集来确定哪些事务没有发送给从服务器；
- **故障恢复**：在 GTID 模式下，从服务器已经记录了所有执行过的 GTID 集。当主服务器故障时，只需将从服务器提升为主服务器，无需关心具体的 binlog 文件名或位置。新的主服务器会基于其 GTID 集继续生成 binlog，而其他从服务器可以自动继续复制，无需复杂的手动介入。

### 6.4 怎么实现读写分离？

**<font color='cornflowerblue'>主从复制的目的是为了实现数据库的读写分离</font>**：写操作和实时性较强的读操作则访问主数据库；读操作则访问从数据库。从而使数据库具有更强大的访问负载能力，支撑更多的用户访问。





### 6.5 主从复制有哪些问题？



### 6.6 MySQL主从复制主流架构模型





## 七、MySQL面试问题

### 7.1 基础知识

#### 7.1.1 关系型和非关系型数据库的区别？

**关系型数据库（RDBMS）**：

- 使用表来存储数据，由行（记录）和列（字段）组成，每个表有预定义的模式；
- 具有严格的关系模型，适合结构化数据、高度一致性和复杂查询的场景；
- 使用**结构化查询语言（SQL）**进行数据的插入、更新、查询和删除，支持复杂的查询、处理操作；
- 示例：MySQL、Oracle、SQL Server。

**非关系型数据库（NoSQL）**

- 数据存储在**文档、键值对、图片**等多种数据模型中，通常不需要预定义的模式；
- 没有固定的关系模型，数据结构可以灵活变化，适用于**大数据应用、实时数据处理、内容管理、社交媒体**等不需要严格关系模型和事务一致性的场景
- 使用数据库特定的**查询语法**或API来操作数据，查询方式多样化，有时更直观和高效，读写性能更高，但不如SQL标准化
- 示例：Redis、MongoDB。

> 所谓的关系模型就是指表，因为对于一张表其结构是固定的，每一个记录都会有固定的字段值。这就是关系

#### 7.1.2 什么是主键？ 什么是外键？

- **主键**：是一个或多个字段（复合主键），可以**<font color='cornflowerblue'>唯一标识</font>**表中的每一行数据。每个表只能有一个主键，主键值必须唯一，不能包含NULL值。

  ```mysql
  # 创建一个名为person的表，有一个id字段，约束该字段是INT型且不能为NULL，并通过PRIMARY KEY约束其为主键字段
  CREATE TABLE persons (
  	id INT NOT NULL PRIMARY KEY
  );
  ```

  对于InnoDB引擎，如果没有显式指定主键会自动创建一个隐藏的 6 字节的行 ID 作为主键。但这种隐式主键无法直接访问或使用，还是会进行全表扫描。

- **外键**：用于在两个表之间建立关联的数据库约束，是另一个表的主键，可以重复，也可以是NULL，用来和其他表建立联系用的。

  ```mysql
  # FOREIGN KEY表时通过定义约束来创建外键
  CREATE TABLE employees (
      emp_id INT NOT NULL PRIMARY KEY,
      id INT,
      FOREIGN KEY (id) REFERENCES persons(id)
  );
  ```

#### 7.1.3 为什么InnoDB使用自增id作为主键？

如果不使用自增的主键，当插入新数据时，新纪录可能被查到现有索引页的中间某个位置，**<font color='cornflowerblue'>导致大量数据的频繁移动，且可能会导致页分裂，频繁的移动、分页会造成很多内存碎片</font>**。而使用自增的主键，每次**新添加的记录都是直接追加在后面的**，就不会导致记录的移动，如果空间不足，就自动新开辟一个页。

#### 7.1.4 什么是表的连接？

**表的连接**（Join）是SQL中用于从两个或多个表中组合数据的操作。通过连接，可以在查询中基于特定条件获取来自不同表的相关数据。表连接是**<font color='cornflowerblue'>关系型数据库的核心概念之一</font>**，主要有以下几种类型：

- **内连接（JOIN、INNER JOIN）**：返回两个表中**满足连接条件**的所有匹配行。如果某个表中没有匹配行，则该行不会出现在结果集中。
- **左连接（LEFT JOIN）**：返回左表中的所有行，以及右表中**匹配连接条件**的行。如果右表中没有匹配的行，则结果集中的该行会显示 `NULL`。
- **右连接（RIGHT JOIN）**：返回右表中的所有行，以及左表中**匹配连接条件**的行。如果左表中没有匹配的行，则结果集中显示 `NULL`。
- **全连接（FULL JOIN）**：（在MySQL中不直接支持，可以通过 `UNION` 实现）返回两个表中**所有满足条件和不满足条件**的行。如果某个表中没有匹配的行，则结果集中显示 `NULL`。
- **交叉连接（CROSS JOIN）**：返回两个表的**笛卡尔积**，即将左表的每一行与右表的每一行组合在一起。

#### 7.1.5 MySQL的内部构造？ 分为几部分？

MySQL可以分为两层：**<font color='cornflowerblue'>Server层和存储引擎层</font>**。其中：

- **<font color='cornflowerblue'>Server层负责管理连接、分析和执行SQL</font>**。包括连接器、解析器、预处理器、优化器、执行器、各种内置函数、各种功能等都是在Server层实现。
- **<font color='cornflowerblue'>存储引擎层负责数据的存储和提取</font>**。支持InnoDB、MylSAM、Memory等多个存储引擎，默认的是InnoDB。

#### 7.1.6 MySQL执行普通select语句的过程？

1. **<font color='cornflowerblue'>Server层的连接器</font>**负责与客户端通过三次握手建立TCP连接，验证用户名和密码，确认权限；
2. 对于收到的查询SQL语句，MySQL会首先去**<font color='cornflowerblue'>查询缓存（ Query Cache ）</font>**中查找，若命中则直接返回；
3. 若没有命中缓存则需要具体执行SQL语句，首先由**<font color='cornflowerblue'>Server层的解析器</font>**对SQL语句进行解析，包括**词法解析（识别关键词）**和**语法解析（构建出SQL语法树，判断语句是否合法）**两个步骤；
4. 解析完成后就可以去执行SQL，这一步之前需要先**<font color='cornflowerblue'>验证是否存在权限</font>**，又分为三个阶段：**预处理阶段、优化阶段和执行阶段**，也是在**Server层**完成；
5. **<font color='cornflowerblue'>预处理阶段</font>**主要判断查询的表和字段是否存在，同时将 `*` 符号扩展为表上的所有列；
6. **<font color='cornflowerblue'>优化阶段</font>**的目的是**将SQL语句的执行方案确定下来**，比如基于查询成本该使用那个索引，决定各个表的连接顺序等；
7. **<font color='cornflowerblue'>执行阶段</font>**就是和存储引擎层进行交流，在数据库中执行查询方案，获得结果，并返回给客户端。

#### 7.1.7 MySQL执行更新语句的过程？

涉及到记录更新的SQL语句与普通的select语句执行过程在前面均一致，唯一的区别在于**<font color='cornflowerblue'>执行阶段</font>**。因为更新记录的SQL语句的执行涉及到了**redo log和binlog的日志写入和提交**。单独针对两个日志具体来说：

1. 存储引擎层（InnoDB）在更新记录前，先记录**undo log**，同时将**内存中的数据更新并标记为脏页，然后把更新写到redo lo**g中，此时更新语句算是执行完毕了，因为WAL技术的存在，刷盘是异步执行的；
2. 更新语句执行完成后，同时也在将更新内容写入到**binlog中**；
3. 当更新事务提交，此时需要将redo log和binlog刷盘，即**两阶段提交**。

### 7.2 SQL语句相关

#### 7.2.1 Drop、Delete与Truncate的共同点和区别？

- **Drop用来删除表**，所有的数据和元数据都会被删除，操作不能回滚；
- **Delete用于删除表中的数据**，可以是所有记录或者是满足条件的部分记录（where），操作可以回滚；
- **Truncate用于清空表中的所有记录**，但表结构、字段、约束、索引等保持不变，操作不能回滚，与DELETE相比，它更快并且使用的系统资源更少。

### 7.3 MySQL索引相关



### 7.4 MySQL事务相关



### 7.5 MySQL锁相关



### 7.6 MySQL日志相关



### 7.5 MySQL分布式相关

#### 7.5.1 分布式数据库为什么不能使用自增ID或UUID做主键？





# 资料参考

内容大多参考自：[图解MySQL介绍 | 小林coding (xiaolincoding.com)](https://xiaolincoding.com/mysql/)

[【Mysql面试高频】- Mysql主从复制相关的面试知识点_从库读取主库的binlog线程为啥只有一个-CSDN博客](https://blog.csdn.net/Mind_programmonkey/article/details/116742748)