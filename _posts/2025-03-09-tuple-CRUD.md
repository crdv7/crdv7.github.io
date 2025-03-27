---
layout:     post
title:      postgresql元组操作
subtitle:   HeapTuple的创建与读取
date:       2025-03-09
author:     BY CRDV7
header-img: img/post-bg-debug.png
catalog: true
tags:
    - postgresql源码阅读
    - postgresql存储引擎
---

## 前言

前一篇文章介绍了PostgreSQL中heatuple的基本构造，本文继续围绕heaptuple主题，介绍数据库对其插入、更新以及删除这些基本操作的过程。



## 正文

对元组的操作包括插入、删除和更新三种基本操作，这三种操作都是把元组当作一个整体进行处理。除些之外，在 heaptuple. c 这个文件中还实现了元组内部结构的相关操作，包括元组的构造、修改、分解、复制、释放等操作 。一个完擎的元组信息对应一个 HeapTupleData 结构和一个 TupleDesc 结构，在 HeapTupleData中还包含一个前面介绍过的 HeapTupleHeaderData 结构。TupleDesc 是关系结构 RelationData 的 一部分，也称为元组描述符 ，它记录了与该元组相关的全部属性模式信息 。 通过元组描述符可以读取磁盘中存储的无格式数据，并根据元组描述符构造出元组的各个属性值， 元组描述符的结构如下：

````C
typedef struct PageHeaderData
{
    /* XXX LSN is member of *any* block, not only page-organized ones */
    PageXLogRecPtr pd_lsn;      /* LSN: next byte after last byte of xlog
                                 * record for last change to this page */
    uint16      pd_checksum;    /* checksum */
    uint16      pd_flags;       /* flag bits, see below */
    LocationIndex pd_lower;     /* offset to start of free space */
    LocationIndex pd_upper;     /* offset to end of free space */
    LocationIndex pd_special;   /* offset to start of special space */
    uint16      pd_pagesize_version;
    TransactionId pd_prune_xid; /* oldest prunable XID, or zero if none */
    ItemIdData  pd_linp[FLEXIBLE_ARRAY_MEMBER]; /* line pointer array */
} PageHeaderData;
````

1. **`natts`**
   - 表示元组中的属性数量（列数）。
   - 这个数字告诉你元组中有多少个字段。
2. **`tdtypeid`**
   - 复合类型ID，用来标识元组类型的OID（对象标识符）。
   - 如果元组对应于一个命名的行类型（例如一个表的行类型），则此字段标识那个类型。
   - 对于匿名行类型（比如查询结果），这个值被设置为 `RECORDOID`。
3. **`tdtypmod`**
   - 类型修饰符，提供有关元组类型的一些额外信息。
   - 对于命名的行类型，通常这个值是 `-1`。
   - 对于匿名行类型，它可以是 `-1`（完全匿名），或者是一个非负值，允许通过 `typcache.c` 中的类型缓存查找行类型。
4. **`tdrefcount`**
   - 元组描述符的引用计数。
   - 如果描述符是在缓存（如 `relcache` 或 `typcache`）中，则该值大于等于0。
   - 如果不是引用计数（例如由执行器创建并绑定到特定内存上下文的描述符），则设置为 `-1`。
5. **`constr`**
   - 指向 `TupleConstr` 结构，包含约束信息（例如 NOT NULL, CHECK 约束等）。
   - 如果没有约束，这个字段可以是 `NULL`。
6. **`compact_attrs`**
   - 一个可变长度的 `CompactAttribute` 数组。
   - 它是 `FormData_pg_attribute` 数组的一个简化版本，旨在提高性能关键代码的效率。
   - 每个 `CompactAttribute` 包含对应属性（列）的紧凑元数据。

HeapTupleData 是元组在内存中的拷贝，它是磁盘格式的元组读入内存后的存在方式， HeapTu-
pleData 的结构如下所示：

```C
typedef struct HeapTupleData
{
    uint32      t_len;          /* length of *t_data */
    ItemPointerData t_self;     /* SelfItemPointer */
    Oid         t_tableOid;     /* table the tuple came from */
#define FIELDNO_HEAPTUPLEDATA_DATA 3
    HeapTupleHeader t_data;     /* -> tuple header and data */
} HeapTupleData;
```



1. **`t_len`**
   - 表示元组数据的实际长度（以字节为单位）。
   - 这个字段记录了 `t_data` 指针所指向的数据区域的大小。
   - 在指针为空的情况下（即 `t_data` 为 `NULL`），这个字段无效。
2. **`t_self`**
   - 类型为 `ItemPointerData`，表示该元组在磁盘上的物理位置（SelfItemPointer）。
   - 它包含两个部分：
     - **块号**：元组所在的磁盘页面的编号。
     - **偏移量**：元组在页面中的偏移。
   - 如果 `HeapTupleData` 指向一个磁盘缓冲区中的元组，或者它是从磁盘复制的元组，则 `t_self` 应该是有效的。
   - 对于人工构造的元组（例如通过代码生成的元组），`t_self` 应显式设置为无效。
3. **`t_tableOid`**
   - 表示元组所属表的 OID（对象标识符）。
   - 如果 `HeapTupleData` 指向磁盘缓冲区中的元组，或者它是从磁盘复制的元组，则 `t_tableOid` 应该是有效的。
   - 对于人工构造的元组，`t_tableOid` 应显式设置为无效。
4. **`t_data`**
   - 指向元组的实际数据区域（包括元组头和数据内容）。
   - 根据不同使用场景， **`t_data`** 的指向方式会有所不同：
     - **指向磁盘缓冲区中的元组**：直接指向缓冲区中的数据。
     - **指向空值**：`t_data` 为 `NULL`，通常用于表示失败或无效状态。
     - **与 `HeapTupleData` 合并分配**：`t_data` 指向 `HeapTupleData` 结构之后的内存区域（偏移量为 `HEAPTUPLESIZE`）。这是 `heap_form_tuple` 和相关函数输出的标准格式。
     - **单独分配的元组**：`t_data` 指向一个独立分配的内存块，不与 `HeapTupleData` 相邻（这种用法已被弃用）。
     - **最小化元组**：`t_data` 指向一个 `MinimalTuple` 数据的前 `MINIMAL_TUPLE_OFFSET` 字节位置。

### HeapTuple的插入

#### **写入堆元组**

为了理解 PostgreSQL 中堆元组（heap tuple）的写入过程，我们以一个简单的场景为例：假设有一个表由单个页面组成，该页面最初只包含一个堆元组。随着新元组的插入，页面中的数据布局会发生变化。以下是详细的过程说明。

------

#### **初始状态**

- **页面结构**：

  - 页面的 `pd_lower` 指向第一个行指针（line pointer）。
  - 行指针和 `pd_upper` 都指向第一个堆元组。
  - 此时，页面中只有一个元组存在。

- **图示**：如下图a所示

  <img src="/img/heaptuplewrite.png" style="zoom:50%;" />

- ```C
  +---------------------------+
  | Page Header               |
  | - pd_lower (→ line ptr 1) |
  | - pd_upper (→ tuple 1)    |
  +---------------------------+
  | Line Pointer 1 → Tuple 1  |
  +---------------------------+
  ```

------

#### **插入第二个元组**

当第二个元组被插入时，PostgreSQL 的存储机制会按照以下步骤更新页面：

1. **放置元组数据**：

   - 第二个元组被放置在第一个元组之后的空闲空间中（即从 `pd_upper` 开始向下扩展）。
   - `pd_upper` 被更新为指向新的元组位置。

2. **添加行指针**：

   - 第二个行指针被追加到第一个行指针之后。
   - 新的行指针指向第二个元组的位置。
   - `pd_lower` 被更新为指向新的行指针位置。

3. **更新页面头部信息**：

   - 页面头部的其他字段（如 `pd_lsn` `pg_checksum`和 `pg_flag`）会被更新为适当的值。
     - `pd_lsn` 记录了最新的日志序列号（LSN），用于 WAL（Write-Ahead Logging）机制。
     - `pg_checksum` 是页面的校验和，用于检测页面损坏。
     - `pg_flag` 包含页面的状态标志。

- **图示**：如下图b所示

  <img src="/img/heaptuplewrite.png" style="zoom:50%;" />

- ```C
  +---------------------------+
  | Page Header               |
  | - pd_lower (→ line ptr 2) |
  | - pd_upper (→ tuple 2)    |
  +---------------------------+
  | Line Pointer 1 → Tuple 1  |
  | Line Pointer 2 → Tuple 2  |
  +---------------------------+
  ```

------

#### **关键点解析**

1. **`pd_lower` 和 `pd_upper` 的作用**：
   - `pd_lower` 指向页面中第一个未使用的行指针位置。
   - `pd_upper` 指向页面中第一个可用的空闲空间位置。
   - 随着元组的插入，`pd_lower` 向下移动（增加），而 `pd_upper` 向上移动（减少）。
2. **行指针的作用**：
   - 行指针是一个固定大小的结构，用于快速定位元组的实际存储位置。
   - 它允许 PostgreSQL 在页面中高效地管理元组，即使元组的物理存储位置发生变化。
3. **页面头部信息的更新**：
   - 插入元组后，页面头部的相关字段（如 `pd_lsn` 和 `pg_checksum`）需要更新，以确保数据一致性和可靠性。
   - 这些字段的具体细节将在元组可见性的博客中分析。

### HeapTuple的读取

在PostgreSQL中，有两种典型的访问方法用于读取堆元组（heap tuple）：顺序扫描和B树索引扫描。这两种方法各有特点，适用于不同的场景。

------

#### **顺序扫描（Sequential Scan）**

- **定义**：顺序扫描通过逐页扫描每个页面中的所有行指针来依次读取所有元组。

- 过程：

  - PostgreSQL会从表的第一个页面开始，依次读取每个页面的所有行指针。
  - 根据行指针指向的位置读取对应的堆元组数据。
  - 这种方式不会跳过任何元组，因此称为“顺序”扫描。

- 图示：参考下图(a)：

  <img src="/img/heaptuplescan.png" style="zoom:50%;" />

  - 在顺序扫描中，PostgreSQL会检查每一个页面的每一个行指针，并根据这些指针找到并读取相应的堆元组。

这种方式虽然简单直接，但在处理大型表时效率较低，因为它需要遍历整个表或至少是大部分页面，即使最终只需要少数几个元组的数据。

------

### **B树索引扫描（B-tree Index Scan）**

- **定义**：B树索引扫描通过读取包含索引元组的索引文件来定位目标堆元组。每个索引元组由一个索引键和一个TID（Tuple Identifier）组成，后者指向目标堆元组。

- 过程：

  - 首先，在索引中查找具有特定键值的索引元组。
  - 找到匹配的索引元组后，使用其TID值来直接定位并读取对应的堆元组。
  - TID值包括块号和偏移量信息，例如`(block = 7, Offset = 2)`表示目标堆元组位于表的第7个页面中的第2个元组位置。

- 图示：参考下图(b)：

  <img src="/img/heaptuplescan.png" style="zoom:50%;" />

  - 假设查询条件找到了索引中的某个元组，其TID为`(block = 7, Offset = 2)`，那么PostgreSQL可以直接跳转到第7个页面，并准确地读取该页面中的第2个元组，而无需进行不必要的页面扫描。

这种方法特别适合于当知道确切查询条件时，能够快速定位所需元组，极大地提高了查询效率，尤其是在处理大量数据时。

PostgreSQL 还支持多种扫描方法（access methods），除了顺序扫描（Sequential Scan）和 B 树索引扫描（B-tree Index Scan）外，还包括 **TID 扫描**、**位图扫描（Bitmap Scan）** 和 **仅索引扫描（Index-Only Scan）**。这些扫描方法各有其特点和适用场景，下面逐一进行详细介绍：

------

#### **TID 扫描（TID-Scan）**

- **定义**：
  - TID 扫描直接使用元组标识符（Tuple Identifier, TID）来定位堆元组。
  - TID 是一个 `(block number, offset)` 对，用于唯一标识表中的某个元组。
- **工作原理**：
  - 查询中明确指定了某些元组的 TID 值（例如通过 `ctid` 列）。
  - PostgreSQL 直接根据 TID 值跳转到对应的页面和偏移位置，读取目标元组。
- **优点**：
  - 非常高效，因为不需要扫描整个表或索引。
  - 适用于需要精确访问少量元组的场景。
- **缺点**：
  - 只能用于明确知道 TID 的查询场景，应用范围有限。
- **典型场景**：
  - 使用 `ctid` 进行调试或特定操作。
  - 示例：`SELECT * FROM table WHERE ctid = '(0,1)'`。

------

#### **位图扫描（Bitmap Scan）**

- **定义**：
  - 位图扫描是一种结合了索引和顺序扫描的混合扫描方式。
  - 它首先通过索引构建一个“位图”（bitmap），表示哪些页面和元组满足查询条件，然后对这些页面进行顺序扫描。
- **工作原理**：
  1. 构建位图：
     - 使用索引找到所有符合条件的 TID。
     - 将这些 TID 按照页面分组，并标记每个页面中符合条件的元组。
  2. 顺序扫描页面：
     - 根据位图逐页读取数据，避免重复访问同一个页面。
     - 在每个页面内，只读取位图中标记的元组。
- **优点**：
  - 减少了随机 I/O 操作，提高了性能（特别是当返回的元组分布在多个页面时）。
  - 能很好地处理返回大量元组的查询。
- **缺点**：
  - 构建位图会消耗额外的内存。
  - 如果返回的元组数量非常大，位图可能变得昂贵。
- **典型场景**：
  - 索引选择性较低时（即索引匹配的元组较多）。
  - 示例：`SELECT * FROM table WHERE indexed_column BETWEEN 10 AND 1000`。

------

#### **仅索引扫描（Index-Only Scan）**

- **定义**：
  - 仅索引扫描是一种高效的扫描方式，它完全依赖索引来获取查询结果，而无需访问实际的堆元组。
  - 这种方法适用于查询只需要索引中存储的数据的情况。
- **工作原理**：
  - 如果查询所需的列全部包含在索引中（覆盖索引，covering index），则可以直接从索引中读取数据。
  - 如果索引中没有包含 `visibility map` 中的信息（用于确定元组是否对当前事务可见），PostgreSQL 仍需检查堆元组的可见性。
- **优点**：
  - 避免了访问堆元组的开销，显著提高查询性能。
  - 特别适合于小索引键和高选择性的查询。
- **缺点**：
  - 如果索引中不包含查询所需的所有列，则无法使用此方法。
  - 可见性检查可能会引入额外开销。
- **典型场景**：
  - 查询只需要索引中的列。
  - 示例：`SELECT indexed_column FROM table WHERE indexed_column = 'value'`。

### 删除元组

在 PostgreSQL中，使用标记删除的方式来删除元组，这对于多版本并发控制( MVCC) 是有好处的，其 Undo 和 Redo 速度是相当高速的，因为只需重新设置标记即可 。 被标记删除的磁盘空间会通过运行 VACUUM (清理数据库命令，通常每天运行一次)收回 。

### 更新元组

元组的更新操作实际上是删除和插入操作的结合，即先标记删除旧元组，再插入新元组。元组的更新由函数 heap_update 实现 。值得注意的是. PostgreSQL 中进行删除和更新操作时，被删除或修改的元组并不会从物理文件中删除，而是在事务标记中被标记为无效。因此，当进行过大量的删除和更新操作之后，数据库数据文件中由于有大量的无效元组，其尺寸会变得异常庞大，此时需要对数据库进行一定的清理操作，这就需要用到之后要介绍的 VACUUM 机制。

### 参考
- [heaptuple. c](https://doxygen.postgresql.org/heaptuple_8c.html)
- [PostgreSQL数据库内核分析](https://book.douban.com/subject/6971366/)
- [htup_details.h](https://doxygen.postgresql.org/htup__details_8h_source.html)

