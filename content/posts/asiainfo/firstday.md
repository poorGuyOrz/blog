---
title: "Firstday"
date: 2022-07-19T11:33:44+08:00
draft: true
---


## 环境准备

pg源码编译，VScode调试
1. 源码下载
2. 修改配置文件中的-O为-g
3. ./configure --enable-depend --enable-cassert --enable-debug
4. make
5. make install，此时可执行文件可以参考编译输出
6. ./initdb -D /home/wen/postgres/data 。初始化文件数据目录
7. ./pg_ctl -D /home/wen/postgres/data  start
8. ./pg_ctl -D /home/wen/postgres/data  stop
9. psql -p 5432 链接
10. gdb 调试



### 系统表
元数据信息的表和试图，在/home/wen/postgres/src/backend/catalog下面
可以使用\d xx查看元数据表结构，

初始化的时候创建模板数据库template1，优先级最高，之后的所有数据库从template1复制。 

initdb.c

To create the database cluster, we create the directory that contains all its data, create the files that hold the global tables, create a few other control files for it, and create three databases: the template databases "template0" and "template1", and a default user database "postgres".
 The template databases are ordinary PostgreSQL databases.  template0 is never supposed to change after initdb, whereas template1 can be changed to add site-local standard data.  Either one can be copied to produce a new database. For largely-historical reasons, the template1 database is the one built by the basic bootstrap process.  After it is complete, template0 and the default database, postgres, are made just by copying template1.
 To create template1, we run the postgres (backend) program in bootstrap mode and feed it data from the postgres.bki library file.  After this initial bootstrap phase, some additional stuff is created by normal SQL commands fed to a standalone backend.  Some of those commands are just embedded into this program (yeah, it's ugly), but larger chunks are taken from script files.

所有的对象在磁盘上使用oid编号替代，具体的oid和对象之间的联系使用元数据信息维护
在初始化的文件中，在base目录下使用oid建立对应的数据库的目录，目录中使用oid保存对应对象的文件，具体为relfilenode
```sql
-- 测试
-- 建表
create table a (a int);
-- 查看表oid
select * from pg_class where relname = 'a';
-- 在目录中查找对应的文件，修改表数据，查看文件大小的改变
```
文件默认大小为1G，超过之后文件分裂，新的文件的命名为 relfilenode.1 relfilenode.2 relfilenode.3等

postgres.bki文件，initdb调用的元数据信息文件，使用catalog目录下的文件建立的，是生成的SQL语句，initdb中会使用bootstrap 模式建表建数据。


* 理解pg元数据表的建立过程
* 知道重点元数据表的含义
* 会从元数据表中查询想要的信息
* 理解oid



### 运行时进程

一个守护进程后台运行，在有链接的时候会fork一个进程处理链接，本身还负责一些共有的资源，后台进程和master使用共享内存或者信号量通信，
```c++
#0  0x00007f0bdd4d0faa in __GI___select (nfds=7, readfds=0x7fff53551ca0, writefds=0x0, exceptfds=0x0, timeout=0x7fff53551c10) at ../sysdeps/unix/sysv/linux/select.c:41
#1  0x000055a89d821556 in ServerLoop () at postmaster.c:1772
#2  0x000055a89d820e73 in PostmasterMain (argc=3, argv=0x55a89ea354f0) at postmaster.c:1480
#3  0x000055a89d6e69dd in main (argc=3, argv=0x55a89ea354f0) at main.c:197
```

CS结构

客户端

首先客户端会在一个MainLoop方法中循环就收语句，接受的语句简单的处理之后使用SendQuery里面的PQexec传输语句到服务端，后端接受语句之后处理完成之后返回result给客户端，类似的方法还有PQexecParams，这个可以传递参数，而且这个方法可以使用占位符指定参数的时候，可以使用OID来指定参数类型，否则参数的类型就需要上下文来确定，这个是个好想法，因为这样有利于数据的处理，PQprepare可以编译语句生成执行计划，后续使用PQexecPrepared可以重复的传递参数而无需进行重复编译，这是PG自带的PSQL客户端根据，也可以自己实现，只需要实现特殊接口即可。

服务端

[Postgres中postmaster代码解析(上) - 非我在 - 博客园 (cnblogs.com)](https://www.cnblogs.com/flying-tiger/p/8245527.html)

服务端主要方法是PostmasterMain，这里会进行一些环境的配置，包括共享库的加载，参数配置，配置文件的读取等，所有的前期工作准备完成之后启动的数据库StartupDataBase→StartChildProcess，启动之后就进入一个循环ServerLoop，他这里会接受客户端的消息并执行。他有一个死循环，使用select监听端口，PG_SETMASK在select的时候是非阻塞的，成功之后阻塞

```cpp
			PG_SETMASK(&UnBlockSig);

			selres = select(nSockets, &rmask, NULL, NULL, &timeout);

			PG_SETMASK(&BlockSig);
```

如果有新的客户端链接，此时会使用BackendStartup中的BackendRun创建一个子进程用来对接一个客户端

BackendRun中主要就是PostgresMain方法

PostgresMain首先会进行子进程的状态设置，然后进入一个死循环，他使用ReadCommand方法读取客户端使用PQsendQuery输入的语句，然后按语句的分类进行处理

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/e9c76e99-7a8d-40c7-9f4b-6557c8705d62/Untitled.png)

上述就是语句到执行之前的具体阶段，这只是简单的语句的处理过程，接下来就是具体的SQL的处理阶段，具体为parser，bind，opt，exe

PostgresMain中的循环包括sigsetjmp和**siglongjmp**，只要内部SQL出错了，直接使用pg_re_throw中的**siglongjmp**跳转到此处，这也是PG的错误处理机制之一

- pg_parse_query
    - 语句编译，结果为LIST，是语法树的集合，只生成语法树，不进行语义检查

对每一个语法树执行下面操作

- pg_analyze_and_rewrite
    - parse_analyze
        - 分析语句，进行参数绑定
    - pg_rewrite_query
        - 重写，例如视图的展开
- pg_plan_queries

逻辑查询优化阶段

逻辑优化的核心是在SQL结果不变的情况下，最大限度的使用选择和投影，尽可能的从数据源开始就减少读取的数据，

物理查询优化阶段

- 子查询优化
- 视图重写
- 等价谓词重写
- 条件简化
- 外连接消除
- 嵌套连接消除



## 存储

### 内存

* 本地内存
* 共享内存
  * IPC
  * 锁
  * 共享缓存
* 内存上下文
* 缓冲区


### 外存
* 文件管理

按页维护数据，页的大小默认为8192
虚拟文件描述符-VFD
VM
FSM






















-----------

代码开发流程
  bug
  git
  
QA



