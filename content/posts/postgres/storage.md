---
title: "Postgres Storage"
date: 2022-07-21T09:15:05+08:00
draft: false
tags: ["数据库", "Postgres", "存储"]
---


## 存储

### 内存

* 共享内存
* 本地内存
* 缓存
* 内存上下文

* 缓存空间管理  
  数据块的缓存，减少磁盘IO，有共享缓存和进程缓存
* Cache  
  数据块之外的缓存，例如系统表
  * 系统表缓存不会缓存整个表，是以block为单位缓存？
* 虚拟文件描述符  
  系统中文件有打开的上限，使用VFD可以突破这种限制，本质上是一个LRU缓存
* 空闲空间定位  
  快速定位磁盘中的空闲空间以插入数据
* 进程间通信 
  使用共享内存或者信号量通信



#### 读取过程
1. 从系统表中读取表的元数据信息构造元组信息
2. 尝试从缓存读取数据
3. 使用SMGR从磁盘读取数据到缓存中，SMGR是一个抽象层，用于实现不同存储介质的管理
4. SMGR和存储介质之间使用VFD来管理文件描述符，以突破系统的FD限制

* 标记删除，vacuum清理数据
* FSM记录空闲空间



### 磁盘

* 表文件

* SMGR
* VFD
* FSM


#### Page 结构

```c++

typedef struct PageHeaderData
{
  /* XXX LSN is member of *any* block, not only page-organized ones */
  PageXLogRecPtr pd_lsn;        /* LSN: next byte after last byte of xlog record for last change to this page */
  uint16    pd_checksum;        /* checksum */
  uint16    pd_flags;           /* flag bits, see below */
  LocationIndex pd_lower;       /* offset to start of free space */
  LocationIndex pd_upper;       /* offset to end of free space */
  LocationIndex pd_special;     /* offset to start of special space */
  uint16 pd_pagesize_version;
  TransactionId pd_prune_xid;   /* oldest prunable XID, or zero if none */
  ItemIdData  pd_linp[FLEXIBLE_ARRAY_MEMBER]; /* line pointer array */
} PageHeaderData;

void
PageInit(Page page, Size pageSize, Size specialSize)
{
  PageHeader  p = (PageHeader) page;

  specialSize = MAXALIGN(specialSize);

  Assert(pageSize == BLCKSZ);
  Assert(pageSize > specialSize + SizeOfPageHeaderData);

  /* Make sure all fields of page are zero, as well as unused space */
  MemSet(p, 0, pageSize);

  p->pd_flags = 0;
  p->pd_lower = SizeOfPageHeaderData;
  p->pd_upper = pageSize - specialSize;
  p->pd_special = pageSize - specialSize;
  PageSetPageSizeAndVersion(page, pageSize, PG_PAGE_LAYOUT_VERSION);
  /* p->pd_prune_xid = InvalidTransactionId;    done by above MemSet */
}
```
大小为pageSize，默认为8k，最开始是PageHeader，
RelationPutHeapTuple insert tuple到page中，根据head中的信息，计算insert的位置，其中page中的数据从首位向中兴靠齐，尾部为数据，前面为offset


* 数据的变动以page为单位，不直接和存储交互，先把数据块读到缓存，然后在进行insert或者update或者delete，具体的函数有
  * heap_update
  * heap_multi_insert
  * heap_insert

tuple结构如下
```c++
struct HeapTupleHeaderData
{
  union
  {
    HeapTupleFields t_heap;
    DatumTupleFields t_datum;
  }      t_choice;
  // 一个union，在内存中的时候t_heap记录事务相关的信息，在磁盘中的时候，事务信息不在使用，此时转换为数据长度等信息

  ItemPointerData t_ctid;    /* current TID of this or newer tuple (or a * speculative insertion token) */

  /* Fields below here must match MinimalTupleData! */

#define FIELDNO_HEAPTUPLEHEADERDATA_INFOMASK2 2
  // hot相关信息，
  //   hot指的是数据在更新的时候，如果有index,会同事更新index，即使没有修改到index属性的数据，此时index中也会有一条数据的链表，为了节约空间，在满足
  //   1. 没有修改到index属性数据
  //   2. 数组修改限于在同一个page内，此时index不会有额外的数据，查找的时候从index找到最老的数据，按照数据链查找到最新数据即可
  uint16    t_infomask2;  /* number of attributes + various flags */

#define FIELDNO_HEAPTUPLEHEADERDATA_INFOMASK 3
  uint16    t_infomask;    /* various flag bits, see below */

#define FIELDNO_HEAPTUPLEHEADERDATA_HOFF 4
  uint8    t_hoff;      /* sizeof header incl. bitmap, padding */

  /* ^ - 23 bytes - ^ */

#define FIELDNO_HEAPTUPLEHEADERDATA_BITS 5
  bits8    t_bits[FLEXIBLE_ARRAY_MEMBER];  /* bitmap of NULLs */

  /* MORE DATA FOLLOWS AT END OF STRUCT */
};
```

page具体的操作代码在`storage/page`目录下，建议进行15445的实验，可以更好的理解page相关的操作，这里page的操作总体类似，主要是具体的数据结构和某些特定的方法需要时间进行记忆，但是大的方向上似曾相识。  



Each table and index is stored in a separate file.  For ordinary relations, these files are named after the table or index's <firstterm>filenode</firstterm> number,
which can be found in <structname>pg_class</structname>.<structfield>relfilenode</structfield>. But
for temporary relations, the file name is of the form
<literal>t<replaceable>BBB</replaceable>_<replaceable>FFF</replaceable></literal>, where <replaceable>BBB</replaceable>
is the backend ID of the backend which created the file, and <replaceable>FFF</replaceable>
is the filenode number.  In either case, in addition to the main file (a/k/a
main fork), each table and index has a <firstterm>free space map</firstterm> (see <xref
linkend="storage-fsm"/>), which stores information about free space available in
the relation.  The free space map is stored in a file named with the filenode
number plus the suffix <literal>_fsm</literal>.  Tables also have a
<firstterm>visibility map</firstterm>, stored in a fork with the suffix <literal>_vm</literal>,
to track which pages are known to have no dead tuples.  The visibility map is
described further in <xref linkend="storage-vm"/>.  Unlogged tables and indexes
have a third fork, known as the initialization fork, which is stored in a fork
with the suffix <literal>_init</literal> (see <xref linkend="storage-init"/>).


* these files are named after the table or index’s filenode number, which can be found in pg_class.relfilenode.
* for temporary relations, the file name is of the form tBBB_FFF, where BBB is the backend ID of the backend which created the file, and FFF is the filenode number.
* each table and index has a free space map (see ), which stores information about free space available in the relation.
* Tables also have a visibility map, to track which pages are known to have no dead tuples.   



<para>  
Tablespaces make the scenario more complicated.  Each user-defined tablespace
has a symbolic link inside the <varname>PGDATA</varname><filename>/pg_tblspc</filename>
directory, which points to the physical tablespace directory (i.e., the
location specified in the tablespace's <command>CREATE TABLESPACE</command> command).
This symbolic link is named after
the tablespace's OID.  Inside the physical tablespace directory there is
a subdirectory with a name that depends on the <productname>PostgreSQL</productname>
server version, such as <literal>PG_9.0_201008051</literal>.  (The reason for using
this subdirectory is so that successive versions of the database can use
the same <command>CREATE TABLESPACE</command> location value without conflicts.)
Within the version-specific subdirectory, there is
a subdirectory for each database that has elements in the tablespace, named
after the database's OID.  Tables and indexes are stored within that
directory, using the filenode naming scheme.
The <literal>pg_default</literal> tablespace is not accessed through
<filename>pg_tblspc</filename>, but corresponds to
<varname>PGDATA</varname><filename>/base</filename>.  Similarly, the <literal>pg_global</literal>
tablespace is not accessed through <filename>pg_tblspc</filename>, but corresponds to
<varname>PGDATA</varname><filename>/global</filename>.
</para>

* pg_relation_filepath(oid)


<para>
<productname>PostgreSQL</productname> uses a fixed page size (commonly
8 kB), and does not allow tuples to span multiple pages.  Therefore, it is
not possible to store very large field values directly.  To overcome
this limitation, large field values are compressed and/or broken up into
multiple physical rows.  This happens transparently to the user, with only
small impact on most of the backend code.  The technique is affectionately
known as <acronym>TOAST</acronym> (or <quote>the best thing since sliced bread</quote>).
The <acronym>TOAST</acronym> infrastructure is also used to improve handling of
large data values in-memory.
</para>


* TOAST  
  大字段数据，在字段数据大于2k的时候，会触发相应的机制，把数据按2k切分，存储到TOAST表中，原表使用专门的指针指向数据  


```c++

#0  GetPageWithFreeSpace (rel=0x61951eb6c3c4, spaceNeeded=3280) at freespace.c:134
#1  0x000055d41e4a9a8c in RelationGetBufferForTuple (relation=0x7f3d795c2160, len=32, otherBuffer=0, options=0, bistate=0x0, vmbuffer=0x7ffeab8f6d54, vmbuffer_other=0x0) at hio.c:409
#2  0x000055d41e49078b in heap_insert (relation=0x7f3d795c2160, tup=0x55d41fd92818, cid=0, options=0, bistate=0x0) at heapam.c:2051
#3  0x000055d41e4a039d in heapam_tuple_insert (relation=0x7f3d795c2160, slot=0x55d41fd92780, cid=0, options=0, bistate=0x0) at heapam_handler.c:252
#4  0x000055d41e7402d6 in table_tuple_insert (rel=0x7f3d795c2160, slot=0x55d41fd92780, cid=0, options=0, bistate=0x0) at ../../../src/include/access/tableam.h:1376
#5  0x000055d41e742093 in ExecInsert (context=0x7ffeab8f6fe0, resultRelInfo=0x55d41fd91b70, slot=0x55d41fd92780, canSetTag=true, inserted_tuple=0x0, insert_destrel=0x0)
    at nodeModifyTable.c:1058
#6  0x000055d41e746203 in ExecModifyTable (pstate=0x55d41fd91958) at nodeModifyTable.c:3700
#7  0x000055d41e704dd8 in ExecProcNodeFirst (node=0x55d41fd91958) at execProcnode.c:463
#8  0x000055d41e6f85f1 in ExecProcNode (node=0x55d41fd91958) at ../../../src/include/executor/executor.h:259
#9  0x000055d41e6fb2c9 in ExecutePlan (estate=0x55d41fd916e0, planstate=0x55d41fd91958, use_parallel_mode=false, operation=CMD_INSERT, sendTuples=false, numberTuples=0,
    direction=ForwardScanDirection, dest=0x55d41fd7a718, execute_once=true) at execMain.c:1636
#10 0x000055d41e6f8ccb in standard_ExecutorRun (queryDesc=0x55d41fd742c0, direction=ForwardScanDirection, count=0, execute_once=true) at execMain.c:363
#11 0x000055d41e6f8ada in ExecutorRun (queryDesc=0x55d41fd742c0, direction=ForwardScanDirection, count=0, execute_once=true) at execMain.c:307
#12 0x000055d41e999ed0 in ProcessQuery (plan=0x55d41fd7a628, sourceText=0x55d41fc53e40 "insert into t values(1);", params=0x0, queryEnv=0x0, dest=0x55d41fd7a718, qc=0x7ffeab8f7440)
    at pquery.c:160
#13 0x000055d41e99ba25 in PortalRunMulti (portal=0x55d41fcc1bb0, isTopLevel=true, setHoldSnapshot=false, dest=0x55d41fd7a718, altdest=0x55d41fd7a718, qc=0x7ffeab8f7440)
    at pquery.c:1277

```



## SMGR


```c++

/*
 * This struct of function pointers defines the API between smgr.c and
 * any individual storage manager module.  Note that smgr subfunctions are
 * generally expected to report problems via elog(ERROR).  An exception is
 * that smgr_unlink should use elog(WARNING), rather than erroring out,
 * because we normally unlink relations during post-commit/abort cleanup,
 * and so it's too late to raise an error.  Also, various conditions that
 * would normally be errors should be allowed during bootstrap and/or WAL
 * recovery --- see comments in md.c for details.
 */
typedef struct f_smgr
{
  void    (*smgr_init) (void);  /* may be NULL */
  void    (*smgr_shutdown) (void);  /* may be NULL */
  void    (*smgr_open) (SMgrRelation reln);
  void    (*smgr_close) (SMgrRelation reln, ForkNumber forknum);
  void    (*smgr_create) (SMgrRelation reln, ForkNumber forknum,
                bool isRedo);
  bool    (*smgr_exists) (SMgrRelation reln, ForkNumber forknum);
  void    (*smgr_unlink) (RelFileLocatorBackend rlocator, ForkNumber forknum,
                bool isRedo);
  void    (*smgr_extend) (SMgrRelation reln, ForkNumber forknum,
                BlockNumber blocknum, char *buffer, bool skipFsync);
  bool    (*smgr_prefetch) (SMgrRelation reln, ForkNumber forknum,
                  BlockNumber blocknum);
  void    (*smgr_read) (SMgrRelation reln, ForkNumber forknum,
                BlockNumber blocknum, char *buffer);
  void    (*smgr_write) (SMgrRelation reln, ForkNumber forknum,
                 BlockNumber blocknum, char *buffer, bool skipFsync);
  void    (*smgr_writeback) (SMgrRelation reln, ForkNumber forknum,
                   BlockNumber blocknum, BlockNumber nblocks);
  BlockNumber (*smgr_nblocks) (SMgrRelation reln, ForkNumber forknum);
  void    (*smgr_truncate) (SMgrRelation reln, ForkNumber forknum,
                  BlockNumber nblocks);
  void    (*smgr_immedsync) (SMgrRelation reln, ForkNumber forknum);
} f_smgr;

static const f_smgr smgrsw[] = {
  /* magnetic disk */
  {
    .smgr_init = mdinit,
    .smgr_shutdown = NULL,
    .smgr_open = mdopen,
    .smgr_close = mdclose,
    .smgr_create = mdcreate,
    .smgr_exists = mdexists,
    .smgr_unlink = mdunlink,
    .smgr_extend = mdextend,
    .smgr_prefetch = mdprefetch,
    .smgr_read = mdread,
    .smgr_write = mdwrite,
    .smgr_writeback = mdwriteback,
    .smgr_nblocks = mdnblocks,
    .smgr_truncate = mdtruncate,
    .smgr_immedsync = mdimmedsync,
  }
};
```

操作不同介质的文件抽象层，在f_smgr中定义了操作的接口，当前默认实现为smgrsw中的对磁盘操作的函数，理论上支持其他存储介质，只需要实现对应的接口即可，当前只是作为一个简单的中转，具体的文件操作在`smgr/md.c`中

### VFD

使用LRU缓存维护的fd，管理打开的文件描述符，主要代码在`file/fd.c`中，主要结构为
```c++
typedef struct vfd
{
  int      fd;        /* current FD, or VFD_CLOSED if none */
  unsigned short fdstate;    /* bitflags for VFD's state */
  ResourceOwner resowner;    /* owner, for automatic cleanup */
  File    nextFree;    /* link to next free VFD, if in freelist */
  File    lruMoreRecently;  /* doubly linked recency-of-use list */
  File    lruLessRecently;
  off_t    fileSize;    /* current size of file (0 if not temporary) */
  char     *fileName;    /* name of file, or NULL for unused VFD */
  /* NB: fileName is malloc'd, and must be free'd when closing the VFD */
  int      fileFlags;    /* open(2) flags for (re)opening the file */
  mode_t    fileMode;    /* mode to pass to open(2) */
} Vfd;
```


### FSM
free space map，page中的数据在删除且vacuum之后，会有数据空洞，此时为了节约空间，在后续insert的时候，会尝试把数据insert到之前的page中，但是遍历page查找空间类似全表扫，所以为了加快这个过程，使用额外的数据结构记录空闲page的大小，insert的时候直接定位page
1. 空闲空间不是精确统计，page默认大小为8k，把8k划分为255份，一份大小为32字节，此时使用一个字节就可以记录page中大约空闲的空间，
2. 为了快速定位，fsm使用二叉树来维护，一个page大小为8192，除了header剩下的空间约为8000字节，使用叶子节点标记page大小，此时一个page大约可以记录4000个page，
3. 一般使用三层fsm，此时前两层为辅助块，最后层page数目为4000^3，完全可以记录所有的数据块的使用情况，所以最初的时候fsm文件大小为 8192* 3，此时只使用到3个块记录大小，后期数据扩展的时候，此时fsm文件大小也会进行扩展

It is important to keep the map small so that it can be searched rapidly.
Therefore, we don't attempt to record the exact free space on a page.
We allocate one map byte to each page, allowing us to record free space
at a granularity of 1/256th of a page.  Another way to say it is that
the stored value is the free space divided by BLCKSZ/256 (rounding down).
We assume that the free space must always be less than BLCKSZ, since
all pages have some overhead; so the maximum map value is 255.

The binary tree is stored on each FSM page as an array. Because the page
header takes some space on a page, the binary tree isn't perfect. That is,
a few right-most leaf nodes are missing, and there are some useless non-leaf
nodes at the right. So the tree looks something like this:

```
       0
   1       2
 3   4   5   6
7 8 9 A B
```

```c++
/*
 * Structure of a FSM page. See src/backend/storage/freespace/README for
 * details.
 */
typedef struct
{
  /*
   * fsm_search_avail() tries to spread the load of multiple backends by
   * returning different pages to different backends in a round-robin
   * fashion. fp_next_slot points to the next slot to be returned (assuming
   * there's enough space on it for the request). It's defined as an int,
   * because it's updated without an exclusive lock. uint16 would be more
   * appropriate, but int is more likely to be atomically
   * fetchable/storable.
   */
  int      fp_next_slot;

  /*
   * fp_nodes contains the binary tree, stored in array. The first
   * NonLeafNodesPerPage elements are upper nodes, and the following
   * LeafNodesPerPage elements are leaf nodes. Unused nodes are zero.
   */
  uint8    fp_nodes[FLEXIBLE_ARRAY_MEMBER];
} FSMPageData;
```

fp_next_slot记录下一此使用的节点，目的是
1. 在多进程写同一个表时候，避免对同一个page的竞争
2. 记录下一个page，在顺序读写的时候可以有更高的性能

具体的查找算法为

```c++

  /*----------
   * Start the search from the target slot.  At every step, move one
   * node to the right, then climb up to the parent.  Stop when we reach
   * a node with enough free space (as we must, since the root has enough
   * space).
   *
   * The idea is to gradually expand our "search triangle", that is, all
   * nodes covered by the current node, and to be sure we search to the
   * right from the start point.  At the first step, only the target slot
   * is examined.  When we move up from a left child to its parent, we are
   * adding the right-hand subtree of that parent to the search triangle.
   * When we move right then up from a right child, we are dropping the
   * current search triangle (which we know doesn't contain any suitable
   * page) and instead looking at the next-larger-size triangle to its
   * right.  So we never look left from our original start point, and at
   * each step the size of the search triangle doubles, ensuring it takes
   * only log2(N) work to search N pages.
   *
   * The "move right" operation will wrap around if it hits the right edge
   * of the tree, so the behavior is still good if we start near the right.
   * Note also that the move-and-climb behavior ensures that we can't end
   * up on one of the missing nodes at the right of the leaf level.
   *
   * For example, consider this tree:
   *
   *       7
   *     7     6
   *   5   7   6   5
   *  4 5 5 7 2 6 5 2
   *        T
   *
   * Assume that the target node is the node indicated by the letter T,
   * and we're searching for a node with value of 6 or higher. The search
   * begins at T. At the first iteration, we move to the right, then to the
   * parent, arriving at the rightmost 5. At the second iteration, we move
   * to the right, wrapping around, then climb up, arriving at the 7 on the
   * third level.  7 satisfies our search, so we descend down to the bottom,
   * following the path of sevens.  This is in fact the first suitable page
   * to the right of (allowing for wraparound) our start point.
   *----------
   */
```
结合代码理解
1. 先判断root节点
2. 从fp_next_slot或者midel node开始，向右上查找，直到找到复合的node
3. 从这里开始向下查找

代码就是简单的对二叉树的操作，所有代码在`freespace/fsmpage.c`中，vacuum的时候会触发sfm的操作，具体的代码在`freespce/freespace.c`中。


### VM
可见性映射表，记录数据变动的page，pg支持多版本，在数据变动的时候不会立即清除数据，而指挥打上tag，等待后续的vacuum进程进行数据的清理，vm记录数据的变动，让vacuum可以快速的清理数据，vacuum有两种模式，一种是lazy vacuum，一种是full vacuum，lazy 的时候不会跨page清理，此时可以使用vm文件，但是full vacuum的时候一般需要全表扫描，基本不会有太大的最用，
vm是简单的bit位，0代表有数据变动，


## 内存


内存上下文，简单的理解就是一定范围内的内存池？

https://smartkeyerror.com/PostgreSQL-MemoryContext
