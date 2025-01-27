---
layout:     post
title:      postgresql堆表结构学习
subtitle:   HeapTableFile和HeapTuple解析
date:       2025-01-27
author:     BY CRDV7
header-img: img/post-bg-debug.jpg
catalog: true
tags:
    - postgresql源码阅读
    - postgresql存储引擎
---

## 前言

RDBMS（关系数据库管理系统）一般由存储、事务和查询三大模块组成，本系列文章计划按照这三个模块的划分逐步学习postgresql数据库的源码。首先从数据库最底层的存储引擎开始学习。



## 正文

数据库集簇，数据库，数据表是PostgreSQL存储数据的层次结构（如图1所示），本期主要阅读学习这个结构中最基本的单元-数据表（pg采用的堆表HeapTable结构）以及存储在其中的元组（HeapTuple）在pg中是如何组织的。

<img src="/img/db-cluster.png" style="zoom:50%;" />

### 堆表文件（Heap Table File）

数据文件（堆表、索引，也包括空闲空间映射和可见性映射）内部被划分为固定长度的页，或者叫区块，大小默认为8192B（8KB）。每个文件中的页从0开始按顺序编号，这些数字称为区块号。如果文件已填满，PostgreSQL就通过在文件末尾追加一个新的空页来增加文件长度。页面内部的布局取决于数据文件的类型。本节会描述表的页面布局, 图2就是堆表文件的页面布局。

![](/img/heapfile.png)

表的页面包含了三种类型的数据：

1. **堆元组**（Heap Tuples）——即数据记录本身。它们从页面底部开始依序堆叠。HeapTuple的内部结构后续博客补充
2. **行指针** (Line pointers)——每个行指针占4B，保存着指向堆元组的指针。它们也被称为项目指针。行指针形成一个简单的数组，扮演了元组索引的角色。每个索引项从1开始依次编号，称为偏移号。当向页面中添加新元组时，一个相应的新行指针也会被放入数组中，并指向新添加的元组。
3. **首部数据** (Header info)——页面的起始位置分配了由结构PageHeaderData 定义的首部数据。它的大小为24B，包含关于页面的元数据。该结构的主要成员变量如下：

- **pd_lsn(PageXLogRecPtr, 8 bytes)**——表示WAL（Write-Ahead Logging，预写日志）记录中最后更改此页面的最后一个字节之后的下一个字节的位置。这用于崩溃恢复和复制。

- **pd_checksum**——本页面的校验和值，，用于检测由于硬件故障等原因导致的页面损坏。并非所有平台或配置都启用校验和。（注意，只有在9.3或更高版本中才有此变量，早期版本中该字段用于存储页面的时间线标识。）

- **pd_flags**——标志位，包含有关页面状态的信息，如是否为新分配的页面等。

  具体的标志位如下：

  - **PD_HAS_FREE_LINES (0x0001)**: 如果在`pd_lower`之前存在任何未使用的行指针（LP_UNUSED），则设置此标志。这意味着页面上可能有可用于新记录的空间。需要注意的是，这个标志被认为是一个提示而非绝对事实，因为它不会被写入WAL（预写日志）中，所以它的状态可能不是最新的。
  - **PD_PAGE_FULL (0x0002)**: 当更新操作在页面中找不到足够的空闲空间来存放新的元组版本时，会设置此标志。这表明可能需要执行一次修剪（prune）操作来释放空间。同样地，这也只是一个提示，并不保证准确性。
  - **PD_ALL_VISIBLE (0x0004)**: 如果页面上的所有元组对所有事务都是可见的，则设置此标志。这对于避免不必要的锁定和检查非常有用，因为它允许系统假设页面中的数据对于所有并发事务都是安全可读的。
  - **PD_VALID_FLAG_BITS (0x0007)**: 这个宏定义表示所有有效`pd_flags`位的或值（OR）。即，当前定义的所有合法标志位的总和（在这个例子中是`0x0001 | 0x0002 | 0x0004 = 0x0007`），确保未来添加的新标志位不会与现有位冲突。

- **pd_lower、pd_upper**——pd_lower 指向行指针的末尾，pd_upper指向最新堆元组的起始位置。

- **pd_special**——在索引页中会用到该字段，在堆表页中它指向页尾。（在索引页中它指向特殊空间的起始位置，特殊空间是仅由索引使用的特殊数据区域，包含特定的数据，具体内容依索引的类型而定，如B树、GiST、GiN等。）

- **pd_pagesize_version**—— 包含页面大小和布局版本号的信息，页面大小是256的倍数，低8位用于版本号。这对于支持不同版本的PostgreSQL之间的兼容性非常重要。

  - **版本0**：用于PostgreSQL 7.3之前的版本。
  - **版本1**：PostgreSQL 7.3和7.4使用，表示堆元组头（HeapTupleHeader）布局的更新。
  - **版本2**：PostgreSQL 8.0使用，再次更改了堆元组头布局。
  - **版本3**：PostgreSQL 8.1使用，重新定义了堆元组头信息掩码（infomask）位。
  - **版本4**：PostgreSQL 8.3引入，再次修改了堆元组头布局，并添加了`pd_flags`字段（通过借用部分`pd_tli`字段的位），同时也增加了`pd_prune_xid`字段，这实际上扩大了页面头部的大小。

- **pd_prune_xid**——页面上最旧未修剪的XMAX事务ID，或者是0如果没有这样的事务。这个字段有助于优化VACUUM操作。

- **pd_linp**——行指针数组，用于存储行指针，指向页面中实际存储的数据项。

  源码：

```C
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
```

通过前述部分可以看出，pg通过pageheader记录页面的元数据信息，其中包括行指针数组，行指针则指向了页面中无序的堆元组数据，页面中的堆有空闲空间时，就可以根据pageheader里面记录的空闲空间起始位置信息写入元组，并更新相应的pageheader以及行指针信息。对于这个组织结构，熟悉数据库的同学就会想到，页面的写入删除，索引的构建等问题，关于这一部分，会在之后研读事务，索引和pg特有的vacuum机制时解释。了解完页面的组织结构后，我们再简单了解下其中堆元组的具体信息。

### 堆元组（HeapTuple）

堆元组（Heap Tuple）是表中一行数据的内部表示。每个堆元组都存储在一个页面（通常为8KB）内，并且由三个主要部分组成：HeapTupleHeaderData 结构、空值位图以及用户数据。

![](/img/heaptuple.png)

#### **HeapTupleHeaderData** 结构

在src/include/access/htup_details.h 中定义。

```C
typedef struct HeapTupleFields
{
    TransactionId t_xmin;       /* inserting xact ID */
    TransactionId t_xmax;       /* deleting or locking xact ID */
 
    union
    {
        CommandId   t_cid;      /* inserting or deleting command ID, or both */
        TransactionId t_xvac;   /* old-style VACUUM FULL xact ID */
    }           t_field3;
} HeapTupleFields;
 
typedef struct DatumTupleFields
{
    int32       datum_len_;     /* varlena header (do not touch directly!) */

    int32       datum_typmod;   /* -1, or identifier of a record type */
 
    Oid         datum_typeid;   /* composite type OID, or RECORDOID */
 
    /*
     * datum_typeid cannot be a domain over composite, only plain composite,
     * even if the datum is meant as a value of a domain-over-composite type.
     * This is in line with the general principle that CoerceToDomain does not
     * change the physical representation of the base type value.
     *
     * Note: field ordering is chosen with thought that Oid might someday
     * widen to 64 bits.
     */
} DatumTupleFields;
struct HeapTupleHeaderData
{
    union
    {
        HeapTupleFields t_heap;
        DatumTupleFields t_datum;
    }           t_choice;
 
    ItemPointerData t_ctid;     /* current TID of this or newer tuple (or a
                                 * speculative insertion token) */
 
    /* Fields below here must match MinimalTupleData! */
 
#define FIELDNO_HEAPTUPLEHEADERDATA_INFOMASK2 2
    uint16      t_infomask2;    /* number of attributes + various flags */
 
#define FIELDNO_HEAPTUPLEHEADERDATA_INFOMASK 3
    uint16      t_infomask;     /* various flag bits, see below */
 
#define FIELDNO_HEAPTUPLEHEADERDATA_HOFF 4
    uint8       t_hoff;         /* sizeof header incl. bitmap, padding */
 
    /* ^ - 23 bytes - ^ */
 
#define FIELDNO_HEAPTUPLEHEADERDATA_BITS 5
    bits8       t_bits[FLEXIBLE_ARRAY_MEMBER];  /* bitmap of NULLs */
 
    /* MORE DATA FOLLOWS AT END OF STRUCT */
};
```

- **t_xmin**: 存储插入该元组的事务的事务ID（txid）。每当一个新的元组被插入到数据库中时，`t_xmin`就会记录执行此操作的事务的ID。
- **t_xmax**: 存储删除或更新该元组的事务的事务ID。如果该元组尚未被删除或更新，则`t_xmax`设置为0，表示无效。在PostgreSQL中，更新操作实际上是通过在新位置创建一个元组的新版本并标记旧版本为已删除（通过设置`t_xmax`）来实现的。
- **t_cid**: 存储命令ID（cid），即在同一事务内当前命令之前执行的SQL命令的数量，从0开始计数。例如，在一个事务中连续执行三个`INSERT`命令：`BEGIN; INSERT; INSERT; INSERT; COMMIT;`。如果第一个命令插入了这个元组，则`t_cid`设为0；如果是第二个命令插入的，则`t_cid`设为1，以此类推。需要注意的是，自PostgreSQL 13起，由于性能优化的原因，默认情况下不再存储`cmin`/`cmax`值（它们与`cid`相关），除非使用了`SET LOCAL txid_snapshot = 'xmin:xmax'`等特定配置。
- **t_xvac**: 旧版本VACUUM FULL操作使用的事务ID。
- **t_ctid**: 存储指向自身或新元组的元组标识符（tid）。`tid`用于在表内识别一个元组。当元组被更新时，原始元组的`t_ctid`会指向新版本的元组；如果没有更新，`t_ctid`则指向自己。这种方式使得PostgreSQL能够追踪元组的不同版本，并支持多版本并发控制（MVCC）。
- **t_infomask2**：包含属性数量和各种标志位的信息，如是否有NULL值、是否为HOT更新等。
  - **HEAP_NATTS_MASK**: 存储属性的数量。
  - **HEAP_KEYS_UPDATED, HEAP_HOT_UPDATED, HEAP_ONLY_TUPLE**: 描述元组是否被更新且键列被修改、是否为HOT更新（Heap Only Tuple Update）以及是否仅为堆元组。

- **t_infomask**： 包含各种标志位，如事务状态（提交、无效）、锁信息（排他锁、共享锁）等。
  - **HEAP_HASNULL, HEAP_HASVARWIDTH, HEAP_HASEXTERNAL**: 表示元组是否包含NULL值、可变宽度属性或外部存储属性。
  - **HEAP_XMIN_COMMITTED, HEAP_XMIN_INVALID**: 描述插入事务的状态，如已提交或无效。
  - **HEAP_XMAX_EXCL_LOCK, HEAP_XMAX_KEYSHR_LOCK, HEAP_XMAX_LOCK_ONLY**: 描述最大事务ID (`t_xmax`) 的锁状态。
  - **HEAP_XMAX_IS_MULTI**: 表示`t_xmax`是一个多事务ID（MultiXactId）。
  - **HEAP_UPDATED, HEAP_MOVED_OFF, HEAP_MOVED_IN**: 描述元组是否被更新或因VACUUM FULL操作而移动。

- **t_hoff**：包括位图和填充在内的头部大小，也就是用户数据的起始位置

#### **空置位图**

仅仅在t_infomask中HEAP_HASNULL位被设置时存在。每个数据列对应一位，1表示非空，0表示空。读取元组数据时，会首先检查NULL位图，如果字段为空，则跳过该字段。

#### 用户数据

- 实际的列数据从t_hoff指定的偏移开始。
- 必须始终是平台MAXALIGN距离的倍数。

这些标志位和结构共同作用，使得PostgreSQL能够高效地管理事务、锁以及元组的状态，支持复杂的查询操作，并确保数据库系统的高并发性能和数据一致性。通过这种方式，PostgreSQL不仅实现了高效的多版本并发控制（MVCC），还提供了对复杂数据类型的支持和优化的数据访问模式。本期就到这里了，后续文章会介绍这些标志位是如何实现MVCC以及如何空置元组可见性等复杂操作。

### 参考
- [bufpage.h](https://doxygen.postgresql.org/bufpage_8c_source.html)
- [PostgreSQL数据库内核分析](https://book.douban.com/subject/6971366/)
- [htup_details.h](https://doxygen.postgresql.org/htup__details_8h_source.html)

