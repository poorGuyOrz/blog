---
title: "Postgres Executor"
date: 2022-07-20T09:26:50+08:00
draft: false
tags: ["数据库", "Postgres", "执行器"]
---


## 执行器

exec_simple_query

pg_parse_query

pg_analyze_and_rewrite_fixedparams

pg_plan_queries


CreatePortal

PortalDefineQuery

PortalStart
  ChoosePortalStrategy
    PORTAL_ONE_SELECT
      CreateQueryDesc
      ExecutorStart
        standard_ExecutorStart
    PORTAL_ONE_MOD_WITH
    PORTAL_ONE_RETURNING
      ExecCleanTypeFromTL
    PORTAL_UTIL_SELECT
      UtilityTupleDescriptor

PortalRun
  case PORTAL_ONE_SELECT:
  case PORTAL_ONE_RETURNING:
  case PORTAL_ONE_MOD_WITH:
  case PORTAL_UTIL_SELECT:
    FillPortalStore
    PortalRunSelect
      ExecutorRun
        standard_ExecutorRun -- hook，可以使用其他执行器
          ExecutePlan
            ExecProcNode -- 递归执行ExecProcNode
  case PORTAL_MULTI_QUERY:
    PortalRunMulti
      ProcessQuery
        CreateQueryDesc
        ExecutorStart
        ExecutorRun
        ExecutorFinish
        ExecutorEnd
        FreeQueryDesc
      PortalRunUtility
        ProcessUtility
          standard_ProcessUtility -- 预留钩子


PortalDrop
  PortalCleanup
    ExecutorFinish
    ExecutorEnd
      ExecEndNode
    FreeQueryDesc


 We have several execution strategies for Portals, depending on what query or queries are to be executed.  (Note: in all cases, a Portal executes just a single source-SQL query, and thus produces just a single result from the user's viewpoint.  However, the rule rewriter may expand the single source query to zero or many actual queries.)

- PORTAL_ONE_SELECT
  the portal contains one single SELECT query.  We run the Executor incrementally as results are demanded.  This strategy also supports holdable cursors (the Executor results can be dumped into a tuplestore for access after transaction completion).
  查询操作，结果可以在原子操作之后缓存，用于游标访问
- PORTAL_ONE_RETURNING
  the portal contains a single INSERT/UPDATE/DELETE query with a RETURNING clause (plus possibly auxiliary queries added by rule rewriting).  On first execution, we run the portal to completion and dump the primary query's results into the portal tuplestore; the results are then returned to the client as demanded.  (We can't support suspension of the query partway through, because the AFTER TRIGGER code can't cope, and also because we don't want to risk failing to execute all the auxiliary queries.)
- PORTAL_ONE_MOD_WITH,
- PORTAL_UTIL_SELECT,
- PORTAL_MULTI_QUERY


- 常规的DML语句，以节点树的形式表示
- 非DML语句，utility，语句执行时一系列的过程
- 其他子模块






-- 
# DEBUG CASE

```sql
CREATE TABLE tenk1 (
	unique1		int4,
	unique2		int4,
	two			int4,
	four		int4,
	ten			int4,
	twenty		int4,
	hundred		int4,
	thousand	int4,
	twothousand	int4,
	fivethous	int4,
	tenthous	int4,
	odd			int4,
	even		int4,
	stringu1	name,
	stringu2	name,
	string4		name
);

-- 回歸測試文件
\set filename :abs_srcdir '/data/tenk.data'
COPY tenk1 FROM :'filename';
VACUUM ANALYZE tenk1;


select distinct sub.unique1, stringu1
from tenk1, lateral (select tenk1.unique1 from generate_series(1, 1000)) as sub;

```

```c++

postgres=# explain select distinct sub.unique1, stringu1
from tenk1, lateral (select tenk1.unique1 from generate_series(1, 1000)) as sub;
                                     QUERY PLAN                                     
------------------------------------------------------------------------------------
 HashAggregate  (cost=175480.00..175580.00 rows=10000 width=68)
   Group Key: tenk1.unique1, tenk1.stringu1
   ->  Nested Loop  (cost=0.00..125480.00 rows=10000000 width=68)
         ->  Function Scan on generate_series  (cost=0.00..10.00 rows=1000 width=0)
         ->  Materialize  (cost=0.00..495.00 rows=10000 width=68)
               ->  Seq Scan on tenk1  (cost=0.00..445.00 rows=10000 width=68)
(6 rows)

#0  exec_simple_query (query_string=0x55ef2315fe40 "select distinct sub.unique1, stringu1\nfrom tenk1, lateral (select tenk1.unique1 from generate_series(1, 1000)) as sub;")
    at postgres.c:1243
#1  0x000055ef22314d31 in PostgresMain (dbname=0x55ef2315c598 "postgres", username=0x55ef2318c218 "wen") at postgres.c:4482
#2  0x000055ef2223aaff in BackendRun (port=0x55ef23185110) at postmaster.c:4490
#3  0x000055ef2223a3e4 in BackendStartup (port=0x55ef23185110) at postmaster.c:4218
#4  0x000055ef222366da in ServerLoop () at postmaster.c:1808
#5  0x000055ef22235e73 in PostmasterMain (argc=3, argv=0x55ef2315a510) at postmaster.c:1480
#6  0x000055ef220fb9dd in main (argc=3, argv=0x55ef2315a510) at main.c:197


#0  standard_ExecutorRun (queryDesc=0x55ef23184340, direction=ForwardScanDirection, count=0, execute_once=true) at execMain.c:321
#1  0x000055ef22074ada in ExecutorRun (queryDesc=0x55ef23184340, direction=ForwardScanDirection, count=0, execute_once=true) at execMain.c:307
#2  0x000055ef22317259 in PortalRunSelect (portal=0x55ef231d23d0, forward=true, count=0, dest=0x55ef23242a10) at pquery.c:924
#3  0x000055ef22316e8c in PortalRun (portal=0x55ef231d23d0, count=9223372036854775807, isTopLevel=true, run_once=true, dest=0x55ef23242a10, altdest=0x55ef23242a10, qc=0x7fffcc2f5d50)
    at pquery.c:768
#4  0x000055ef223101bb in exec_simple_query (
    query_string=0x55ef2315fe40 "select distinct sub.unique1, stringu1\nfrom tenk1, lateral (select tenk1.unique1 from generate_series(1, 1000)) as sub;") at postgres.c:1243
#5  0x000055ef22314d31 in PostgresMain (dbname=0x55ef2315c598 "postgres", username=0x55ef2318c218 "wen") at postgres.c:4482
#6  0x000055ef2223aaff in BackendRun (port=0x55ef23185110) at postmaster.c:4490
#7  0x000055ef2223a3e4 in BackendStartup (port=0x55ef23185110) at postmaster.c:4218
#8  0x000055ef222366da in ServerLoop () at postmaster.c:1808
#9  0x000055ef22235e73 in PostmasterMain (argc=3, argv=0x55ef2315a510) at postmaster.c:1480
#10 0x000055ef220fb9dd in main (argc=3, argv=0x55ef2315a510) at main.c:197

#0  ExecProcNodeFirst (node=0x55ef232386e0) at execProcnode.c:451
#1  0x000055ef220745f1 in ExecProcNode (node=0x55ef232386e0) at ../../../src/include/executor/executor.h:259
#2  0x000055ef220772c9 in ExecutePlan (estate=0x55ef23238490, planstate=0x55ef232386e0, use_parallel_mode=false, operation=CMD_SELECT, sendTuples=true, numberTuples=0,
    direction=ForwardScanDirection, dest=0x55ef23242a10, execute_once=true) at execMain.c:1636
#3  0x000055ef22074ccb in standard_ExecutorRun (queryDesc=0x55ef23184340, direction=ForwardScanDirection, count=0, execute_once=true) at execMain.c:363
#4  0x000055ef22074ada in ExecutorRun (queryDesc=0x55ef23184340, direction=ForwardScanDirection, count=0, execute_once=true) at execMain.c:307
#5  0x000055ef22317259 in PortalRunSelect (portal=0x55ef231d23d0, forward=true, count=0, dest=0x55ef23242a10) at pquery.c:924
#6  0x000055ef22316e8c in PortalRun (portal=0x55ef231d23d0, count=9223372036854775807, isTopLevel=true, run_once=true, dest=0x55ef23242a10, altdest=0x55ef23242a10, qc=0x7fffcc2f5d50)
    at pquery.c:768

#0  agg_retrieve_hash_table (aggstate=0x55ef232386e0) at nodeAgg.c:2730
#1  0x000055ef22093f1c in ExecAgg (pstate=0x55ef232386e0) at nodeAgg.c:2157
#2  0x000055ef22080dd8 in ExecProcNodeFirst (node=0x55ef232386e0) at execProcnode.c:463
#3  0x000055ef220745f1 in ExecProcNode (node=0x55ef232386e0) at ../../../src/include/executor/executor.h:259
#4  0x000055ef220772c9 in ExecutePlan (estate=0x55ef23238490, planstate=0x55ef232386e0, use_parallel_mode=false, operation=CMD_SELECT, sendTuples=true, numberTuples=0,
    direction=ForwardScanDirection, dest=0x55ef23242a10, execute_once=true) at execMain.c:1636
#5  0x000055ef22074ccb in standard_ExecutorRun (queryDesc=0x55ef23184340, direction=ForwardScanDirection, count=0, execute_once=true) at execMain.c:363
#6  0x000055ef22074ada in ExecutorRun (queryDesc=0x55ef23184340, direction=ForwardScanDirection, count=0, execute_once=true) at execMain.c:307
#7  0x000055ef22317259 in PortalRunSelect (portal=0x55ef231d23d0, forward=true, count=0, dest=0x55ef23242a10) at pquery.c:924
#8  0x000055ef22316e8c in PortalRun (portal=0x55ef231d23d0, count=9223372036854775807, isTopLevel=true, run_once=true, dest=0x55ef23242a10, altdest=0x55ef23242a10, qc=0x7fffcc2f5d50)
    at pquery.c:768


#0  ExecScan (node=0x55ef23238f80, accessMtd=0x55ef220a0171 <FunctionNext>, recheckMtd=0x55ef220a056d <FunctionRecheck>) at execScan.c:169
#1  0x000055ef220a05c3 in ExecFunctionScan (pstate=0x55ef23238f80) at nodeFunctionscan.c:270
#2  0x000055ef22080dd8 in ExecProcNodeFirst (node=0x55ef23238f80) at execProcnode.c:463
#3  0x000055ef220c4214 in ExecProcNode (node=0x55ef23238f80) at ../../../src/include/executor/executor.h:259
#4  0x000055ef220c4497 in ExecNestLoop (pstate=0x55ef23238dd0) at nodeNestloop.c:109
#5  0x000055ef22080dd8 in ExecProcNodeFirst (node=0x55ef23238dd0) at execProcnode.c:463
#6  0x000055ef220909a0 in ExecProcNode (node=0x55ef23238dd0) at ../../../src/include/executor/executor.h:259
#7  0x000055ef22090efa in fetch_input_tuple (aggstate=0x55ef232386e0) at nodeAgg.c:563
#8  0x000055ef220946d6 in agg_fill_hash_table (aggstate=0x55ef232386e0) at nodeAgg.c:2533
#9  0x000055ef22093f10 in ExecAgg (pstate=0x55ef232386e0) at nodeAgg.c:2154
#10 0x000055ef22080dd8 in ExecProcNodeFirst (node=0x55ef232386e0) at execProcnode.c:463

#0  ExecEvalFuncArgs (fcinfo=0x55ef23257e30, argList=0x55ef23239868, econtext=0x55ef23239198) at execSRF.c:841
#1  0x000055ef22083604 in ExecMakeTableFunctionResult (setexpr=0x55ef23239320, econtext=0x55ef23239198, argContext=0x55ef23257d10, expectedDesc=0x55ef23239dc8, randomAccess=false)
    at execSRF.c:181
#2  0x000055ef220a0226 in FunctionNext (node=0x55ef23238f80) at nodeFunctionscan.c:95
#3  0x000055ef22085167 in ExecScanFetch (node=0x55ef23238f80, accessMtd=0x55ef220a0171 <FunctionNext>, recheckMtd=0x55ef220a056d <FunctionRecheck>) at execScan.c:133
#4  0x000055ef220851e0 in ExecScan (node=0x55ef23238f80, accessMtd=0x55ef220a0171 <FunctionNext>, recheckMtd=0x55ef220a056d <FunctionRecheck>) at execScan.c:182
#5  0x000055ef220a05c3 in ExecFunctionScan (pstate=0x55ef23238f80) at nodeFunctionscan.c:270
#6  0x000055ef22080dd8 in ExecProcNodeFirst (node=0x55ef23238f80) at execProcnode.c:463
#7  0x000055ef220c4214 in ExecProcNode (node=0x55ef23238f80) at ../../../src/include/executor/executor.h:259
#8  0x000055ef220c4497 in ExecNestLoop (pstate=0x55ef23238dd0) at nodeNestloop.c:109
#9  0x000055ef22080dd8 in ExecProcNodeFirst (node=0x55ef23238dd0) at execProcnode.c:463
#10 0x000055ef220909a0 in ExecProcNode (node=0x55ef23238dd0) at ../../../src/include/executor/executor.h:259
#11 0x000055ef22090efa in fetch_input_tuple (aggstate=0x55ef232386e0) at nodeAgg.c:563
#12 0x000055ef220946d6 in agg_fill_hash_table (aggstate=0x55ef232386e0) at nodeAgg.c:2533

#0  ExecJustConst (state=0x55ef232393b8, econtext=0x55ef23239198, isnull=0x55ef23257e58) at execExprInterp.c:2209
#1  0x000055ef22069449 in ExecInterpExprStillValid (state=0x55ef232393b8, econtext=0x55ef23239198, isNull=0x55ef23257e58) at execExprInterp.c:1858
#2  0x000055ef220832e3 in ExecEvalExpr (state=0x55ef232393b8, econtext=0x55ef23239198, isNull=0x55ef23257e58) at ../../../src/include/executor/executor.h:324
#3  0x000055ef22084853 in ExecEvalFuncArgs (fcinfo=0x55ef23257e30, argList=0x55ef23239868, econtext=0x55ef23239198) at execSRF.c:846
#4  0x000055ef22083604 in ExecMakeTableFunctionResult (setexpr=0x55ef23239320, econtext=0x55ef23239198, argContext=0x55ef23257d10, expectedDesc=0x55ef23239dc8, randomAccess=false)
    at execSRF.c:181
#5  0x000055ef220a0226 in FunctionNext (node=0x55ef23238f80) at nodeFunctionscan.c:95
#6  0x000055ef22085167 in ExecScanFetch (node=0x55ef23238f80, accessMtd=0x55ef220a0171 <FunctionNext>, recheckMtd=0x55ef220a056d <FunctionRecheck>) at execScan.c:133
#7  0x000055ef220851e0 in ExecScan (node=0x55ef23238f80, accessMtd=0x55ef220a0171 <FunctionNext>, recheckMtd=0x55ef220a056d <FunctionRecheck>) at execScan.c:182
#8  0x000055ef220a05c3 in ExecFunctionScan (pstate=0x55ef23238f80) at nodeFunctionscan.c:270
#9  0x000055ef22080dd8 in ExecProcNodeFirst (node=0x55ef23238f80) at execProcnode.c:463
#10 0x000055ef220c4214 in ExecProcNode (node=0x55ef23238f80) at ../../../src/include/executor/executor.h:259
#11 0x000055ef220c4497 in ExecNestLoop (pstate=0x55ef23238dd0) at nodeNestloop.c:109
#12 0x000055ef22080dd8 in ExecProcNodeFirst (node=0x55ef23238dd0) at execProcnode.c:463
#13 0x000055ef220909a0 in ExecProcNode (node=0x55ef23238dd0) at ../../../src/include/executor/executor.h:259

#0  heap_getnextslot (sscan=0x55ef232d29a0, direction=ForwardScanDirection, slot=0x55ef2322fb00) at heapam.c:1347
#1  0x000055ef220c7065 in table_scan_getnextslot (sscan=0x55ef232d29a0, direction=ForwardScanDirection, slot=0x55ef2322fb00) at ../../../src/include/access/tableam.h:1046
#2  0x000055ef220c7131 in SeqNext (node=0x55ef2322f950) at nodeSeqscan.c:80
#3  0x000055ef22085167 in ExecScanFetch (node=0x55ef2322f950, accessMtd=0x55ef220c7099 <SeqNext>, recheckMtd=0x55ef220c7142 <SeqRecheck>) at execScan.c:133
#4  0x000055ef2208520c in ExecScan (node=0x55ef2322f950, accessMtd=0x55ef220c7099 <SeqNext>, recheckMtd=0x55ef220c7142 <SeqRecheck>) at execScan.c:199
#5  0x000055ef220c7198 in ExecSeqScan (pstate=0x55ef2322f950) at nodeSeqscan.c:112
#6  0x000055ef220b53f1 in ExecProcNode (node=0x55ef2322f950) at ../../../src/include/executor/executor.h:259
#7  0x000055ef220b5601 in ExecMaterial (pstate=0x55ef2323a200) at nodeMaterial.c:134
#8  0x000055ef220c4214 in ExecProcNode (node=0x55ef2323a200) at ../../../src/include/executor/executor.h:259
#9  0x000055ef220c4673 in ExecNestLoop (pstate=0x55ef23238dd0) at nodeNestloop.c:160
#10 0x000055ef220909a0 in ExecProcNode (node=0x55ef23238dd0) at ../../../src/include/executor/executor.h:259
#11 0x000055ef22090efa in fetch_input_tuple (aggstate=0x55ef232386e0) at nodeAgg.c:563
#12 0x000055ef220946d6 in agg_fill_hash_table (aggstate=0x55ef232386e0) at nodeAgg.c:2533
```

dml的执行，最后生成节点树，每个节点通过ExecProcNode来递归调用

-- ProcessQuery

```c++

CreatePortal

PortalDefineQuery

PortalStart
  CreateQueryDesc   
  ExecutorStart
    standard_ExecutorStart
      CreateExecutorState
        InitPlan
          ExecInitNode  按类型执行node的初始化，设置执行过程中的关键参数，比如函数指针ExecProcNode。执行模型是火山模型，由于没有c++中的虚函数，所以pg中大部分所有的算子都会设置ExecProcNe，以达到在后续的ExecProcNode函数中，可以直接根据具体的节点类型递归的执行tuple的拉取，后续的run和end也是类似的思路

PortalRun
  ExecutorRun
    standard_ExecutorRun
      ExecutePlan
        ExecProcNode

PortalDrop
  PortalCleanup
    ExecutorFinish
    ExecutorEnd
      ExecEndNode
    FreeQueryDesc

```

大致的过程，ProcessQuery中有完整的过程，在exec_simple_query中是一个大的循环，和其他类型的语句揉在一起




-- case 2 ddl执行， utility语句，按照语句类型选择具体的执行函数，以函数为主体执行功能

```sql
create table course (
  no serial,
  name varchar,
  credit int,
  constraint cons check (credit >= 0 and name <> ''),
  primary key(no)
);
```

```sh

#0  ProcessUtility (pstmt=0x55ef23161a28, queryString=0x55ef2315fe40 "create table course (\n  no serial,\n  name varchar,\n  credit int,\n  constraint cons check (credit >= 0 and name <> ''),\n  primary key(no)\n);", readOnlyTree=false, context=PROCESS_UTILITY_TOPLEVEL, params=0x0, queryEnv=0x0, dest=0x55ef23161b18, qc=0x7fffcc2f5d50) at utility.c:525
#1  0x000055ef2231785a in PortalRunUtility (portal=0x55ef231d23d0, pstmt=0x55ef23161a28, isTopLevel=true, setHoldSnapshot=false, dest=0x55ef23161b18, qc=0x7fffcc2f5d50) at pquery.c:1158
#2  0x000055ef22317ad0 in PortalRunMulti (portal=0x55ef231d23d0, isTopLevel=true, setHoldSnapshot=false, dest=0x55ef23161b18, altdest=0x55ef23161b18, qc=0x7fffcc2f5d50) at pquery.c:1315
#3  0x000055ef22316f3d in PortalRun (portal=0x55ef231d23d0, count=9223372036854775807, isTopLevel=true, run_once=true, dest=0x55ef23161b18, altdest=0x55ef23161b18, qc=0x7fffcc2f5d50) at pquery.c:791
#4  0x000055ef223101bb in exec_simple_query (query_string=0x55ef2315fe40 "create table course (\n  no serial,\n  name varchar,\n  credit int,\n  constraint cons check (credit >= 0 and name <> ''),\n  primary key(no)\n);") at postgres.c:1243


#0  transformColumnDefinition (cxt=0x7fffcc2f54d0, column=0x55ef23160aa8) at parse_utilcmd.c:529
#1  0x000055ef21f815b7 in transformCreateStmt (stmt=0x55ef23161678,
    queryString=0x55ef2315fe40 "create table course (\n  no serial,\n  name varchar,\n  credit int,\n  constraint cons check (credit >= 0 and name <> ''),\n  primary key(no)\n);")
    at parse_utilcmd.c:270
#2  0x000055ef2231a04c in ProcessUtilitySlow (pstate=0x55ef23270be0, pstmt=0x55ef23161a28,
    queryString=0x55ef2315fe40 "create table course (\n  no serial,\n  name varchar,\n  credit int,\n  constraint cons check (credit >= 0 and name <> ''),\n  primary key(no)\n);",
    context=PROCESS_UTILITY_TOPLEVEL, params=0x0, queryEnv=0x0, dest=0x55ef23161b18, qc=0x7fffcc2f5d50) at utility.c:1147


#0  ProcessUtilitySlow (pstate=0x55ef232cf220, pstmt=0x55ef232b0fa8,
    queryString=0x55ef2315fe40 "create table course (\n  no serial,\n  name varchar,\n  credit int,\n  constraint cons check (credit >= 0 and name <> ''),\n  primary key(no)\n);",
    context=PROCESS_UTILITY_SUBCOMMAND, params=0x0, queryEnv=0x0, dest=0x55ef2282e060 <donothingDR>, qc=0x0) at utility.c:1660
#1  0x000055ef22319e3b in standard_ProcessUtility (pstmt=0x55ef232b0fa8,
    queryString=0x55ef2315fe40 "create table course (\n  no serial,\n  name varchar,\n  credit int,\n  constraint cons check (credit >= 0 and name <> ''),\n  primary key(no)\n);",
    readOnlyTree=false, context=PROCESS_UTILITY_SUBCOMMAND, params=0x0, queryEnv=0x0, dest=0x55ef2282e060 <donothingDR>, qc=0x0) at utility.c:1074
#2  0x000055ef22318db1 in ProcessUtility (pstmt=0x55ef232b0fa8,
    queryString=0x55ef2315fe40 "create table course (\n  no serial,\n  name varchar,\n  credit int,\n  constraint cons check (credit >= 0 and name <> ''),\n  primary key(no)\n);",
    readOnlyTree=false, context=PROCESS_UTILITY_SUBCOMMAND, params=0x0, queryEnv=0x0, dest=0x55ef2282e060 <donothingDR>, qc=0x0) at utility.c:530
#3  0x000055ef2231a3ae in ProcessUtilitySlow (pstate=0x55ef23270be0, pstmt=0x55ef23161a28,
    queryString=0x55ef2315fe40 "create table course (\n  no serial,\n  name varchar,\n  credit int,\n  constraint cons check (credit >= 0 and name <> ''),\n  primary key(no)\n);",
    context=PROCESS_UTILITY_TOPLEVEL, params=0x0, queryEnv=0x0, dest=0x55ef23161b18, qc=0x7fffcc2f5d50) at utility.c:1252
#4  0x000055ef22319e3b in standard_ProcessUtility (pstmt=0x55ef23161a28,
    queryString=0x55ef2315fe40 "create table course (\n  no serial,\n  name varchar,\n  credit int,\n  constraint cons check (credit >= 0 and name <> ''),\n  primary key(no)\n);",
    readOnlyTree=false, context=PROCESS_UTILITY_TOPLEVEL, params=0x0, queryEnv=0x0, dest=0x55ef23161b18, qc=0x7fffcc2f5d50) at utility.c:1074


#0  DefineSequence (pstate=0x55ef232cf220, seq=0x55ef232cf508) at sequence.c:221
#1  0x000055ef2231b3c9 in ProcessUtilitySlow (pstate=0x55ef232cf220, pstmt=0x55ef232b0fa8,
    queryString=0x55ef2315fe40 "create table course (\n  no serial,\n  name varchar,\n  credit int,\n  constraint cons check (credit >= 0 and name <> ''),\n  primary key(no)\n);",
    context=PROCESS_UTILITY_SUBCOMMAND, params=0x0, queryEnv=0x0, dest=0x55ef2282e060 <donothingDR>, qc=0x0) at utility.c:1660
#2  0x000055ef22319e3b in standard_ProcessUtility (pstmt=0x55ef232b0fa8,
    queryString=0x55ef2315fe40 "create table course (\n  no serial,\n  name varchar,\n  credit int,\n  constraint cons check (credit >= 0 and name <> ''),\n  primary key(no)\n);",
    readOnlyTree=false, context=PROCESS_UTILITY_SUBCOMMAND, params=0x0, queryEnv=0x0, dest=0x55ef2282e060 <donothingDR>, qc=0x0) at utility.c:1074
#3  0x000055ef22318db1 in ProcessUtility (pstmt=0x55ef232b0fa8,
    queryString=0x55ef2315fe40 "create table course (\n  no serial,\n  name varchar,\n  credit int,\n  constraint cons check (credit >= 0 and name <> ''),\n  primary key(no)\n);",
    readOnlyTree=false, context=PROCESS_UTILITY_SUBCOMMAND, params=0x0, queryEnv=0x0, dest=0x55ef2282e060 <donothingDR>, qc=0x0) at utility.c:530
#4  0x000055ef2231a3ae in ProcessUtilitySlow (pstate=0x55ef23270be0, pstmt=0x55ef23161a28,
    queryString=0x55ef2315fe40 "create table course (\n  no serial,\n  name varchar,\n  credit int,\n  constraint cons check (credit >= 0 and name <> ''),\n  primary key(no)\n);",
    context=PROCESS_UTILITY_TOPLEVEL, params=0x0, queryEnv=0x0, dest=0x55ef23161b18, qc=0x7fffcc2f5d50) at utility.c:1252


#0  AlterSequence (pstate=0x55ef232b7df8, stmt=0x55ef232cf6a0) at sequence.c:484
#1  0x000055ef2231b3ff in ProcessUtilitySlow (pstate=0x55ef232b7df8, pstmt=0x55ef232c0f18,
    queryString=0x55ef2315fe40 "create table course (\n  no serial,\n  name varchar,\n  credit int,\n  constraint cons check (credit >= 0 and name <> ''),\n  primary key(no)\n);",
    context=PROCESS_UTILITY_SUBCOMMAND, params=0x0, queryEnv=0x0, dest=0x55ef2282e060 <donothingDR>, qc=0x0) at utility.c:1664
#2  0x000055ef22319e3b in standard_ProcessUtility (pstmt=0x55ef232c0f18,
    queryString=0x55ef2315fe40 "create table course (\n  no serial,\n  name varchar,\n  credit int,\n  constraint cons check (credit >= 0 and name <> ''),\n  primary key(no)\n);",
    readOnlyTree=false, context=PROCESS_UTILITY_SUBCOMMAND, params=0x0, queryEnv=0x0, dest=0x55ef2282e060 <donothingDR>, qc=0x0) at utility.c:1074
#3  0x000055ef22318db1 in ProcessUtility (pstmt=0x55ef232c0f18,
    queryString=0x55ef2315fe40 "create table course (\n  no serial,\n  name varchar,\n  credit int,\n  constraint cons check (credit >= 0 and name <> ''),\n  primary key(no)\n);",
    readOnlyTree=false, context=PROCESS_UTILITY_SUBCOMMAND, params=0x0, queryEnv=0x0, dest=0x55ef2282e060 <donothingDR>, qc=0x0) at utility.c:530
```

