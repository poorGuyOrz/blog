---
title: "15445课程笔记"
date: 2022-06-10T11:07:10+08:00
draft: false
tags: ["课程", "数据库", "15445"]
---


https://15445.courses.cs.cmu.edu/fall2021/notes/02-advancedsql.pdf

* output control
  控制输出结果，例如order，limit等
* 窗口函数
* CTE
   Common Table Expressions，把一个语句的输出视为一张临时表参与下面的语句的运算
    ```sql
    WITH cte1 (col1) AS (SELECT 1), cte2 (col2) AS (SELECT 2)
    SELECT * FROM cte1, cte2;
    ```


###  Database Storage

* 数据库的存储介质当前还是磁盘，IO慢
* 数据库存储要点之一是使用缓存维护数据在磁盘和内存之间的数据交换，以实现数据的快速读写
* 顺序读写和随机读写
  1. 顺序读写的意思是需要定位到读写的位置才能操作，例如链表。
  2. 随机读写的意思是可以直接定位到读写的位置，例如数组。
  3. 由于磁盘上随机读写速度不如顺序读写，所以当前数据库还是需要想办法使用顺序读写，例如LSM，GFS等架构都是因为这个原因导致的

### 磁盘和内存中数据的组织格式
* 数据全部在磁盘上，按page组织数据，内存中使用buffer pool维护缓存，磁盘中有一个page专门维护page的位置信息，使用的时候先读取此page到内存，然后
  然后读取其他page到buffer pool，使用buffer pool维护page的置换情况，例如LRU，或者其他算法  
  可以参考lab1和slide，还是比较明显的  
  buffer pool中的page可以用于上层的数据运算

* 使用mmap可以完成类似的操作，但是实际上在使用中，如果在发生缺页中断的时候，mmap需要进行置换操作，所以会阻碍程序进程。且mmap是通用的组件，所以没有对数据库
  的使用场景进行一些优化，
  * You never want to use mmap in your DBMS if you need to write.
  * The DBMS (almost) always wants to control things itself and can do a better job at it since it knows more about the data being accessed and the queries being processed.
  * The operating system is not your friend.

  最好是需要什么就自己实现什么，

### 数据组织形式

1. 文件  
  有的数据库使用一个文件保存所有的数据，有的使用一个文件架构，数据库就是在操作这些文件

2. page
  * 文件由一系列page组成，page有不同的用途，可以保存不同的数据，例如元数据，index，原始数据等的
  可以混合存储，也可以单独存储，一些page还可以要求是否可以通过page进行自解析
  * 对与page的位置，可以使用偏移计算，在元数据中记录偏移，例如leveldb，也可以记录pageid，page是等大的，也可以直接计算
  * page的大小一般为8k，或者4k，与物理页大小是一样的，因为操作系统在读写一个物理页的时候是一个原子操作
  * 在磁盘中有两种page的组织方式
    1. 使用链表组织page
    2. 使用字典

* page的结构
  一般page有个头部记录page的元数据信息，例如page大小，crc校监码以及其他特殊信息，之后才是具体的数据，数据的组织方式有两种
  1. Slotted pages  
    在page中，首部记录数据的偏移，向后扩展，数据从尾部开始，先前扩展，当他们相遇的时候，page full  
    此时数据的唯一记录可以使用pageid + slot来表示，当有数据删除操作的时候，会调整slot偏移，postgress ctid
  2. Log-Structured  
    只会添加数据，且在记录数据的时候还会记录的数据的类型，是有效的或者是删除的，由于不会直接进行delete，所以需要定义执行compact操作已处理过期数据

* tuple的结构  
  tuple代表一行数据，由header和数据体构成

* 数据类型
  1. 定长格式，如int


### buffer pool

内存中的数据按page大小组织数据，当加载page到内存中，按算法替换一个page，内存中的page称为frame  
内存中使用page_table记录内存中的page，内存中的page同时还会记录一些其他信息，例如是否是脏页，pin、unpin以及引用数等  


* lock && latch
  * lock  
    比较上层的架构，在事务中保持资源的并发，可以用于管理tuple，table，page，database等，并且需要回滚等操作
  * latch  
    操作系统上的锁，用于管理多线程之间的数据竞争

* page directory    
  用于维护磁盘文件中数据的位置关系，在读取page的时候，可以从directory定位page位置
* page table    
  内存中维护frame和page的映射关系，frame在内存中是一个数组

* 替换算法
  * LRU  
    顺序读取的时候， 需要读取全部的数据，此时LRU优化为LRU-K，目的是处理偶发数据对缓存的污染
  * 时钟算法

### hash
  * hash 算法
    任意输入产出相同长度的数字，要求是低碰撞
  * 静态hash
    *  Linear Probe Hashing  
      简单的hansh结构，处理冲突的时候直接向后查找空闲位置，然后进行插入，但是在删除的时候，不能直接删除，需要设置墓碑，墓碑在使用过一次之后调整位置擦除墓碑。
      对于相同key的处理，由两种解决方案
      1. hash中存储指针，数据单独存放，相同的key公用一个slot  
      2. hash存放多个相同的key，重复的key按照冲突规则处理
    *  Robin Hood Hashing  
      劫富济贫，在冲突的时候，冲突的slot会记录此是数据和原位置偏移的大小，如果由连续冲突的时候，按照偏移大小排序的，确保在查找的时候可以在最短时间找到的数据，具体例子可以查看[ppt](https://15445.courses.cs.cmu.edu/fall2021/notes/06-hashtables.pdf)
    * Cuckoo Hashing  
      使用两个hash表存储数据，当insert的时候，同时探测两个hash表，如果都有空位，直接随机选择一个，如果只有一个空位，则insert，如果两个都冲突了，则此时
      随机选择一个hash表中的数据替换，原来的数据存储到另一个hash表中，如果还是有冲突，继续替换冲突的数据，由于两个hash表使用不同的hash函数，所以在没有full之前
      理论上查找时间都为1，因为insert的时候始终会进行位置的调整，具体例子查看ppt

    * 如果full了，则需要rehash
  * 动态hash
    * Chained Hashing  
      bucket中存储的是链表的其实地址的，对于重复和冲突的key，直接插入链表，链表可以使用其他数据结构替换
    * Extendible Hashing  
      动态扩展的hash 表，来源是前缀树，论文在[这里](https://www.alexdelis.eu/M149/p315-fagin.pdf)，他的dir是一个大小为k的数组，此时可以记录2^k中状态
      类似地址，此时dir和buket各自记录deep，代表的是对地址位数的使用，例如deep为2，则使用前2位计算index，具体例子查看ppt，需要完成lab，需要注意insert和remove操作中hash table的动态变化


### index

index是表的子集，用于快速的精准查询数据，index 的数据是和表自动同步的，用户无需额外的操作

index 一般是B+树。特殊的例如位图索引等例外

前置要求。熟悉B+tree的常规概率，例如insert，delete等，这里简单的总结

* 自平衡树，时间复杂度为(logn)
* node 按 page 大小组织数据，所以有多个孩子
* 要求叶子节点等高
* 叶子节点之间有指针连接，可以顺序scan

按key排序，value可以携带数据，或者是数据的位置，位置可以类似pg 的pict，使用pageid + offset设置。

node的大小和存储的读取成正相关，如果是在内存中，在设计node是设计可以小一点，如果是在磁盘中，则可以大一点，因为顺序读写在磁盘上快于随机读写

自平衡过程不一定需要严格执行，可以允许一定的额外空间，在业务空闲的时候进行

* 优化
  * 前缀压缩
  * 批量操作
  * 去重 
    允许重复的key，此时可以把重复的key的数据存在一起，进行key的压缩
  