---
title: "Postgres Optimizer"
date: 2022-07-22T09:24:24+08:00
draft: false
tags: ["数据库", "Postgres", "优化器"]
description: "本文从源码级别进行优化器的分析，对Postgres优化器代码调研，具体包括他的数据结构，以及具体的代码架构实现以及核心算法等"
---
<!-- cover: "img/hello.jpg" -->

## pg 优化器

但是从今天的视角，PostgreSQL 优化器不是一个好的实现
1. 它用C语言实现，所以扩展性不好
2. 它不是 Volcano 优化模型的，所以灵活性不好
3. 它的很多优化复杂度很高（例如Join重排是System R风格的动态规划算法），所以性能不好


源代码文件主要是在`optimizer`目录下，总共分为四个主要模块，文档中注释为  
  These directories take the Query structure returned by the parser, and generate a plan used by the executor.
The /plan directory generates the actual output plan,
the /path code generates all possible ways to join the tables,
and /prep handles various preprocessing steps for special cases.
/util is utility stuff.
/geqo is the separate "genetic optimization" planner --- it does a semi-random search through the join tree space, rather than
exhaustively considering all possible join trees.  (But each join considered
by /geqo is given to /path to create paths for, so we consider all possible
implementation paths for each specific join pair even in GEQO mode.)


当前看下来，他的主要流程是先使用prep中的方法对query先进行预处理，执行常规的基于规则的优化，之后使用path中的方法，对处理之后的语法树进行处理，找到代价最小的path。
对于path的，如果没有join，则直接查找table的path即可，如果有多表join的场景，则需要进行path的组合，使用一定的搜索策略来进行路径搜索，
当前有三种主流的优化方法
1. system-R
2. cascades
3. gen
pg使用自底向上的动态规划算法和基因算法，默认表数量大于12的时候，使用基因算法，所以这里我只关注system-R算法



### 优化器代码细节

具体的代码入口在exec_simple_query中，接受语句使用pg_parse_query编译，输出是一个list，具体的例子可以查看下面的语句，gdb中使用 `p pprint(parsetree_list)`可以打印list，对于具体的内部的数据结构，例如List,Node等，可以gdb调试pprint，可以查看他们之间的结构关系

parser出来是list，在打印的时候，使用pprint，在内部判断list类型，取出实际的数据，然后按照NodeTag进行转换，NodeTag在几乎所有的数据结构中都在第一位，目的是为了便捷eider进行转换，以select为例  
在语法文件中，selectstmt是一个结构体，具体生成如下，
```c++
simple_select:
      SELECT opt_all_clause opt_target_list
      into_clause from_clause where_clause
      group_clause having_clause window_clause
        {
          SelectStmt *n = makeNode(SelectStmt);

          n->targetList = $3;
          n->intoClause = $4;
          n->fromClause = $5;
          n->whereClause = $6;
          n->groupClause = ($7)->list;
          n->groupDistinct = ($7)->distinct;
          n->havingClause = $8;
          n->windowClause = $9;
          $$ = (Node *) n;
        }

typedef struct SelectStmt
{
  NodeTag    type;

  /*
   * These fields are used only in "leaf" SelectStmts.
   */
  List     *distinctClause; /* NULL, list of DISTINCT ON exprs, or
                 * lcons(NIL,NIL) for all (SELECT DISTINCT) */
  IntoClause *intoClause;    /* target for SELECT INTO */
  List     *targetList;    /* the target list (of ResTarget) */
  List     *fromClause;    /* the FROM clause */
  Node     *whereClause;  /* WHERE qualification */
  List     *groupClause;  /* GROUP BY clauses */
  bool    groupDistinct;  /* Is this GROUP BY DISTINCT? */
  Node     *havingClause;  /* HAVING conditional-expression */
  List     *windowClause;  /* WINDOW window_name AS (...), ... */

  /*
   * In a "leaf" node representing a VALUES list, the above fields are all
   * null, and instead this field is set.  Note that the elements of the
   * sublists are just expressions, without ResTarget decoration. Also note
   * that a list element can be DEFAULT (represented as a SetToDefault
   * node), regardless of the context of the VALUES list. It's up to parse
   * analysis to reject that where not valid.
   */
  List     *valuesLists;  /* untransformed list of expression lists */

  /*
   * These fields are used in both "leaf" SelectStmts and upper-level
   * SelectStmts.
   */
  List     *sortClause;    /* sort clause (a list of SortBy's) */
  Node     *limitOffset;  /* # of result tuples to skip */
  Node     *limitCount;    /* # of result tuples to return */
  LimitOption limitOption;  /* limit type */
  List     *lockingClause;  /* FOR UPDATE (list of LockingClause's) */
  WithClause *withClause;    /* WITH clause */

  /*
   * These fields are used only in upper-level SelectStmts.
   */
  SetOperation op;      /* type of set op */
  bool    all;      /* ALL specified? */
  struct SelectStmt *larg;  /* left child */
  struct SelectStmt *rarg;  /* right child */
  /* Eventually add fields for CORRESPONDING spec here */
} SelectStmt;

```
之后进行传递的时候使用的是list或者node类型，在print的时候，使用`switch (nodeTag(obj))`判断类型，最后选择`case T_SelectStmt:`分支，执行`_outSelectStmt`方法，具体方法如下，最终解析各个clause
```c++
static void
_outSelectStmt(StringInfo str, const SelectStmt *node)
{
  WRITE_NODE_TYPE("SELECTSTMT");

  WRITE_NODE_FIELD(distinctClause);
  WRITE_NODE_FIELD(intoClause);
  WRITE_NODE_FIELD(targetList);
  WRITE_NODE_FIELD(fromClause);
  WRITE_NODE_FIELD(whereClause);
  WRITE_NODE_FIELD(groupClause);
  WRITE_BOOL_FIELD(groupDistinct);
  WRITE_NODE_FIELD(havingClause);
  WRITE_NODE_FIELD(windowClause);
  WRITE_NODE_FIELD(valuesLists);
  WRITE_NODE_FIELD(sortClause);
  WRITE_NODE_FIELD(limitOffset);
  WRITE_NODE_FIELD(limitCount);
  WRITE_ENUM_FIELD(limitOption, LimitOption);
  WRITE_NODE_FIELD(lockingClause);
  WRITE_NODE_FIELD(withClause);
  WRITE_ENUM_FIELD(op, SetOperation);
  WRITE_BOOL_FIELD(all);
  WRITE_NODE_FIELD(larg);
  WRITE_NODE_FIELD(rarg);
}

```

> 以前阅读过oceanbase以及tidb的代码，现在想来也是似曾相识，他们的做法都是类似的，在parser阶段，语法按照stmt组织结构，现在只是暂时接触pg，尚不清楚优点是什么，后续有深入的了解之后再详细探究

stmt是特定的数据结构，后续的处理需要统一的数据结构query，所以需要进行转换，入口为`parse_analyze_fixedparams`

1. 先处理select into ，转换为create table as ,处理函数再transformOptionalSelectInto，主要是建立CreateTableAsStmt，然后把into上拉到CreateTableAsStmt中
2. 之后使用transformStmt处理stmt，`switch (nodeTag(parseTree))`选择合适的分支，这里以select为例子
3. 进入transformSelectStmt分支
  1. 先转换transformWithClause
  2. 转换transformFromClause
  ...

最终得到一个query，一个完整的结构体，包含语句的所有信息，后续进行rewrite，
对于rewrite。貌似是和之前了解的不太一杨，之前的理解是进行基于规则的重写，但是这里好像是使用自定义的rule重写，类似于触发器，其他数据库中，只有sql server有类似的实现，但是已经声明在之后的版本中会移除这个东西，所以在代码中占据相当一部分分量的功能是否是必要的，

TODO: 后续我自身会进行调研，如果确定是上述的东西，或许可以尝试移除这个功能

之后结构为query，会参与到pg_plan_queries中

### 优化
对于utility语句，直接跳过，他只是代码函数的集合，对于DML，进行pg_plan_query优化，分为几个步骤
1. 基于规则的优化
2. join 路径优化


standard_planner
  subquery_planner
    SS_process_ctes
    transform_MERGE_to_join
    replace_empty_jointree
    pull_up_sublinks
    preprocess_function_rtes
    pull_up_subqueries
    flatten_simple_union_all
    preprocess_rowmarks
    preprocess_expression
    preprocess_qual_conditions
    remove_useless_groupby_columns
    reduce_outer_joins
    remove_useless_result_rtes
    grouping_planner
      query_planner
    SS_identify_outer_params
  fetch_upper_rel
  get_cheapest_fractional_path
  create_plan 

* subquery_planner  
  使用一些规则对树进行变换，调整为规则上更优的执行计划，这里处理的数据结构主要是Query，使用PlannerInfo包装Query，保存一些上下文信息，所以他的处理还是过程式的，按照具体的语句场景进行处理，对于不是太熟悉的他数据结构的人来说，需要大量的记忆时间，个人看来，这种优化方式不是太友好，如果想要进行一些扩展，对人员的要求很高

* path    
  make_one_rel    
    set_base_rel_pathlists
    make_rel_from_joinlist
      standard_join_search
  自底向上的动态规划，先生成基础表的path，path是当前局部最优的，会考虑启动最优，总代价最优，最好的排序结果，其他代价较大的path直接丢弃，这类似动态规划的第一层，然后进行组合，生成join的path，使用后的是make_rel_from_joinlist，

* get_cheapest_fractional_path  
  使用选择率tuple_fraction，对RelOptInfo->cheapest_total_path进行二次调整

* create_plan   
  按照Path，使用PlannerInfo生成plan，plan的结构是二叉树的形式，执行的时候按照节点依次执行



> planner_hook使用cascade实现的可行性分析
------

* 输入
  * Query  
  * const char *
  * int
  * ParamListInfo
* 输出
  * PlannedStmt
* 功能点  
  * 实现SQL优化，输入为query类型的数据结构，输出为树形结构的plan，
* 数据结构


cascade
  group
  expression
  node
  cost
  mem

------


### case

```sql

create table sales          (product_id int , b int , c int);
create table product        (product_id int , b int , c int);
create table product_class  (product_id int , b int , c int);

insert into sales values(1,1,1),(2,2,2),(3,3,3),(4,4,4),(5,5,5),(6,6,6),(7,7,7),(8,8,8),(9,9,9),(10,10,10),(11,11,11);
insert into product values(1,1,1),(2,2,2),(3,3,3),(4,4,4),(5,5,5),(6,6,6),(7,7,7),(8,8,8),(9,9,9),(10,10,10),(11,11,11);
insert into product_class values(1,1,1),(2,2,2),(3,3,3),(4,4,4),(5,5,5),(6,6,6),(7,7,7),(8,8,8),(9,9,9),(10,10,10),(11,11,11);

select s.product_id, pc.product_id from sales as s 
-- left join product as p 
--   on s.product_id = p.product_id 
left join product_class pc 
  on s.product_id = pc.product_id;


select s.product_id, pc.product_id from sales as s 
left join product as p 
  on s.product_id = p.product_id 
left join product_class pc 
  on s.product_id = pc.product_id;

                                   QUERY PLAN                                   
--------------------------------------------------------------------------------
 Hash Join  (cost=41.82..114.24 rows=1142 width=8)
   Hash Cond: (pc.product_id = s.product_id)
   ->  Seq Scan on product_class pc  (cost=0.00..30.40 rows=2040 width=4)
   ->  Hash  (cost=40.42..40.42 rows=112 width=8)
         ->  Hash Join  (cost=1.25..40.42 rows=112 width=8)
               Hash Cond: (p.product_id = s.product_id)
               ->  Seq Scan on product p  (cost=0.00..30.40 rows=2040 width=4)
               ->  Hash  (cost=1.11..1.11 rows=11 width=4)
                     ->  Seq Scan on sales s  (cost=0.00..1.11 rows=11 width=4)


                                   QUERY PLAN                                   
--------------------------------------------------------------------------------
 Hash Right Join  (cost=41.82..114.24 rows=1142 width=8)
   Hash Cond: (pc.product_id = s.product_id)
   ->  Seq Scan on product_class pc  (cost=0.00..30.40 rows=2040 width=4)
   ->  Hash  (cost=40.42..40.42 rows=112 width=4)
         ->  Hash Right Join  (cost=1.25..40.42 rows=112 width=4)
               Hash Cond: (p.product_id = s.product_id)
               ->  Seq Scan on product p  (cost=0.00..30.40 rows=2040 width=4)
               ->  Hash  (cost=1.11..1.11 rows=11 width=4)
                     ->  Seq Scan on sales s  (cost=0.00..1.11 rows=11 width=4)

parsetree_list
{type = T_List, length = 1, max_length = 5, elements = 0x561614a8c810, initial_elements = 0x561614a8c810}

(
   {RAWSTMT 
   :stmt 
      {SELECTSTMT 
      :distinctClause <> 
      :intoClause <> 
      :targetList (
         {RESTARGET 
         :name <> 
         :indirection <> 
         :val {COLUMNREF :fields ("s" "product_id") :location 7 }
         :location 7
         }
         {RESTARGET 
         :name <> 
         :indirection <> 
         :val {COLUMNREF :fields ("pc" "product_id") :location 21 }
         :location 21
         }
      )
      :fromClause (
         {JOINEXPR 
         :jointype 0 
         :isNatural false 
         :larg 
            {JOINEXPR 
            :jointype 0 
            :isNatural false 
            :larg 
               {RANGEVAR 
               :schemaname <> 
               :relname sales 
               :inh true 
               :relpersistence p
               :alias {ALIAS :aliasname s :colnames <>}
               :location 40
               }
            :rarg 
               {RANGEVAR 
               :schemaname <> 
               :relname product 
               :inh true 
               :relpersistence p
               :alias {ALIAS :aliasname p :colnames <>}
               :location 58
               }
            :usingClause <> 
            :join_using_alias <> 
            :quals 
               {A_EXPR  
               :name ("=")
               :lexpr 
                  {COLUMNREF 
                  :fields ("s" "product_id")
                  :location 77
                  }
               :rexpr 
                  {COLUMNREF 
                  :fields ("p" "product_id")
                  :location 92
                  }
               :location 90
               }
            :alias <> 
            :rtindex 0
            }
         :rarg 
            {RANGEVAR 
            :schemaname <> 
            :relname product_class 
            :inh true 
            :relpersistence p 
            :alias 
               {ALIAS 
               :aliasname pc 
               :colnames <>
               }
            :location 112
            }
         :usingClause <> 
         :join_using_alias <> 
         :quals 
            {A_EXPR  
            :name ("=")
            :lexpr 
               {COLUMNREF 
               :fields ("s" "product_id")
               :location 135
               }
            :rexpr 
               {COLUMNREF 
               :fields ("pc" "product_id")
               :location 150
               }
            :location 148
            }
         :alias <> 
         :rtindex 0
         }
      )
      :whereClause <> 
      :groupClause <> 
      :groupDistinct false 
      :havingClause <> 
      :windowClause <> 
      :valuesLists <> 
      :sortClause <> 
      :limitOffset <> 
      :limitCount <> 
      :limitOption 0 
      :lockingClause <> 
      :withClause <> 
      :op 0 
      :all false 
      :larg <> 
      :rarg <>
      }
   :stmt_location 0 
   :stmt_len 163
   }
)

pg_analyze_and_rewrite_fixedparams
(
   {QUERY 
   :commandType 1 
   :querySource 0 
   :canSetTag true 
   :utilityStmt <> 
   :resultRelation 0 
   :hasAggs false 
   :hasWindowFuncs false 
   :hasTargetSRFs false 
   :hasSubLinks false 
   :hasDistinctOn false 
   :hasRecursive false 
   :hasModifyingCTE false 
   :hasForUpdate false 
   :hasRowSecurity false 
   :isReturn false 
   :cteList <> 
   :rtable (
      {RANGETBLENTRY 
      :alias 
         {ALIAS :aliasname s :colnames <>}
      :eref 
         {ALIAS :aliasname s :colnames ("product_id" "b" "c")}
      :rtekind 0 
      :relid 83295 
      :relkind r 
      :rellockmode 1 
      :tablesample <> 
      :lateral false 
      :inh true 
      :inFromCl true 
      :requiredPerms 2 
      :checkAsUser 0 
      :selectedCols (b 8)
      :insertedCols (b)
      :updatedCols (b)
      :extraUpdatedCols (b)
      :securityQuals <>
      }
      {RANGETBLENTRY 
      :alias 
         {ALIAS :aliasname p :colnames <>}
      :eref 
         {ALIAS :aliasname p :colnames ("product_id" "b" "c")}
      :rtekind 0 
      :relid 83298 
      :relkind r 
      :rellockmode 1 
      :tablesample <> 
      :lateral false 
      :inh true 
      :inFromCl true 
      :requiredPerms 2 
      :checkAsUser 0 
      :selectedCols (b 8)
      :insertedCols (b)
      :updatedCols (b)
      :extraUpdatedCols (b)
      :securityQuals <>
      }
      {RANGETBLENTRY 
      :alias <> 
      :eref 
         {ALIAS :aliasname unnamed_join :colnames ("product_id" "b" "c" "product_id" "b" "c")}
      :rtekind 2 
      :jointype 0 
      :joinmergedcols 0 
      :joinaliasvars (
         {VAR :varno 1 :varattno 1 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 1 :varattnosyn 1 :location -1}
         {VAR :varno 1 :varattno 2 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 1 :varattnosyn 2 :location -1}
         {VAR :varno 1 :varattno 3 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 1 :varattnosyn 3 :location -1}
         {VAR :varno 2 :varattno 1 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 2 :varattnosyn 1 :location -1}
         {VAR :varno 2 :varattno 2 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 2 :varattnosyn 2 :location -1}
         {VAR :varno 2 :varattno 3 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 2 :varattnosyn 3 :location -1}
      )
      :joinleftcols (i 1 2 3)
      :joinrightcols (i 1 2 3)
      :join_using_alias <> 
      :lateral false 
      :inh false 
      :inFromCl true 
      :requiredPerms 0 
      :checkAsUser 0 
      :selectedCols (b)
      :insertedCols (b)
      :updatedCols (b)
      :extraUpdatedCols (b)
      :securityQuals <>
      }
      {RANGETBLENTRY 
      :alias 
         {ALIAS :aliasname pc :colnames <>}
      :eref 
         {ALIAS :aliasname pc :colnames ("product_id" "b" "c")}
      :rtekind 0 
      :relid 83301 
      :relkind r 
      :rellockmode 1 
      :tablesample <> 
      :lateral false 
      :inh true 
      :inFromCl true 
      :requiredPerms 2 
      :checkAsUser 0 
      :selectedCols (b 8)
      :insertedCols (b)
      :updatedCols (b)
      :extraUpdatedCols (b)
      :securityQuals <>
      }
      {RANGETBLENTRY 
      :alias <> 
      :eref 
         {ALIAS :aliasname unnamed_join :colnames ("product_id" "b" "c" "product_id" "b" "c" "product_id" "b" "c") }
      :rtekind 2 
      :jointype 0 
      :joinmergedcols 0 
      :joinaliasvars (
         {VAR :varno 1 :varattno 1 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 1 :varattnosyn 1 :location -1}
         {VAR :varno 1 :varattno 2 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 1 :varattnosyn 2 :location -1}
         {VAR :varno 1 :varattno 3 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 1 :varattnosyn 3 :location -1}
         {VAR :varno 2 :varattno 1 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 2 :varattnosyn 1 :location -1}
         {VAR :varno 2 :varattno 2 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 2 :varattnosyn 2 :location -1}
         {VAR :varno 2 :varattno 3 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 2 :varattnosyn 3 :location -1}
         {VAR :varno 4 :varattno 1 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 4 :varattnosyn 1 :location -1}
         {VAR :varno 4 :varattno 2 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 4 :varattnosyn 2 :location -1}
         {VAR :varno 4 :varattno 3 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 4 :varattnosyn 3 :location -1}
      )
      :joinleftcols (i 1 2 3 4 5 6)
      :joinrightcols (i 1 2 3)
      :join_using_alias <> 
      :lateral false 
      :inh false 
      :inFromCl true 
      :requiredPerms 0 
      :checkAsUser 0 
      :selectedCols (b)
      :insertedCols (b)
      :updatedCols (b)
      :extraUpdatedCols (b)
      :securityQuals <>
      }
   )
   :jointree 
      {FROMEXPR 
      :fromlist (
         {JOINEXPR 
         :jointype 0 
         :isNatural false 
         :larg 
            {JOINEXPR 
            :jointype 0 
            :isNatural false 
            :larg 
               {RANGETBLREF :rtindex 1}
            :rarg 
               {RANGETBLREF :rtindex 2}
            :usingClause <> 
            :join_using_alias <> 
            :quals 
               {OPEXPR 
               :opno 96 
               :opfuncid 65 
               :opresulttype 16 
               :opretset false 
               :opcollid 0 
               :inputcollid 0 
               :args (
                  {VAR :varno 1 :varattno 1 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 1 :varattnosyn 1 :location 77}
                  {VAR :varno 2 :varattno 1 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 2 :varattnosyn 1 :location 92}
               )
               :location 90
               }
            :alias <> 
            :rtindex 3
            }
         :rarg 
            {RANGETBLREF 
            :rtindex 4
            }
         :usingClause <> 
         :join_using_alias <> 
         :quals 
            {OPEXPR 
            :opno 96 
            :opfuncid 65 
            :opresulttype 16 
            :opretset false 
            :opcollid 0 
            :inputcollid 0 
            :args (
               {VAR :varno 1 :varattno 1 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 1 :varattnosyn 1 :location 135}
               {VAR :varno 4 :varattno 1 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 4 :varattnosyn 1 :location 150}
            )
            :location 148
            }
         :alias <> 
         :rtindex 5
         }
      )
      :quals <>
      }
   :mergeActionList <> 
   :mergeUseOuterJoin false 
   :targetList (
      {TARGETENTRY 
      :expr 
         {VAR :varno 1 :varattno 1 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 1 :varattnosyn 1 :location 7}
      :resno 1 
      :resname product_id 
      :ressortgroupref 0 
      :resorigtbl 83295 
      :resorigcol 1 
      :resjunk false
      }
      {TARGETENTRY 
      :expr 
         {VAR :varno 4 :varattno 1 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 4 :varattnosyn 1 :location 21}
      :resno 2 
      :resname product_id 
      :ressortgroupref 0 
      :resorigtbl 83301 
      :resorigcol 1 
      :resjunk false
      }
   )
   :override 0 
   :onConflict <> 
   :returningList <> 
   :groupClause <> 
   :groupDistinct false 
   :groupingSets <> 
   :havingQual <> 
   :windowClause <> 
   :distinctClause <> 
   :sortClause <> 
   :limitOffset <> 
   :limitCount <> 
   :limitOption 0 
   :rowMarks <> 
   :setOperations <> 
   :constraintDeps <> 
   :withCheckOptions <> 
   :stmt_location 0 
   :stmt_len 163
   }
)


p pprint (best_path) at get_cheapest_fractional_path
   {HASHPATH 
   :jpath.path.pathtype 349 
   :parent_relids (b 1 2 4)
   :jpath.path.pathtarget 
      {PATHTARGET 
      :exprs (
         {VAR :varno 1 :varattno 1 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 1 :varattnosyn 1 :location 7}
         {VAR :varno 4 :varattno 1 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 4 :varattnosyn 1 :location 21}
      )
      :sortgrouprefs  0 0 
      :cost.startup 0.00 
      :cost.per_tuple 0.00 
      :width 8 
      :has_volatile_expr 0
      }
   :required_outer (b)
   :jpath.path.parallel_aware false 
   :jpath.path.parallel_safe true 
   :jpath.path.parallel_workers 0 
   :jpath.path.rows 1142 
   :jpath.path.startup_cost 41.82 
   :jpath.path.total_cost 114.24 
   :jpath.path.pathkeys <> 
   :jpath.jointype 3 
   :jpath.inner_unique false 
   :jpath.outerjoinpath 
      {PATH 
      :pathtype 329 
      :parent_relids (b 4)
      :required_outer (b)
      :parallel_aware false 
      :parallel_safe true 
      :parallel_workers 0 
      :rows 2040 
      :startup_cost 0.00 
      :total_cost 30.40 
      :pathkeys <>
      }
   :jpath.innerjoinpath 
      {HASHPATH 
      :jpath.path.pathtype 349 
      :parent_relids (b 1 2)
      :required_outer (b)
      :jpath.path.parallel_aware false 
      :jpath.path.parallel_safe true 
      :jpath.path.parallel_workers 0 
      :jpath.path.rows 112 
      :jpath.path.startup_cost 1.25 
      :jpath.path.total_cost 40.42 
      :jpath.path.pathkeys <> 
      :jpath.jointype 3 
      :jpath.inner_unique false 
      :jpath.outerjoinpath 
         {PATH 
         :pathtype 329 
         :parent_relids (b 2)
         :required_outer (b)
         :parallel_aware false 
         :parallel_safe true 
         :parallel_workers 0 
         :rows 2040 
         :startup_cost 0.00 
         :total_cost 30.40 
         :pathkeys <>
         }
      :jpath.innerjoinpath 
         {PATH 
         :pathtype 329 
         :parent_relids (b 1)
         :required_outer (b)
         :parallel_aware false 
         :parallel_safe true 
         :parallel_workers 0 
         :rows 11 
         :startup_cost 0.00 
         :total_cost 1.11 
         :pathkeys <>
         }
      :jpath.joinrestrictinfo (
         {RESTRICTINFO 
         :clause 
            {OPEXPR 
            :opno 96 
            :opfuncid 65 
            :opresulttype 16 
            :opretset false 
            :opcollid 0 
            :inputcollid 0 
            :args (
               {VAR :varno 1 :varattno 1 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 1 :varattnosyn 1 :location 81}
               {VAR :varno 2 :varattno 1 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 2 :varattnosyn 1 :location 96}
            )
            :location 94
            }
         :is_pushed_down false 
         :outerjoin_delayed false 
         :can_join true 
         :pseudoconstant false 
         :leakproof false 
         :has_volatile 2 
         :security_level 0 
         :clause_relids (b 1 2)
         :required_relids (b 1 2)
         :outer_relids (b 1)
         :nullable_relids (b)
         :left_relids (b 1)
         :right_relids (b 2)
         :orclause <> 
         :eval_cost.startup 0.00 
         :eval_cost.per_tuple 0.00 
         :norm_selec 0.0050 
         :outer_selec 0.0050 
         :mergeopfamilies (o 1976)
         :left_em <> 
         :right_em <> 
         :outer_is_left false 
         :hashjoinoperator 96 
         :left_bucketsize 0.0909 
         :right_bucketsize 0.1000 
         :left_mcvfreq 0.0000 
         :right_mcvfreq 0.0000 
         :left_hasheqoperator 96 
         :right_hasheqoperator 96
         }
      )
      :path_hashclauses (
         {RESTRICTINFO 
         :clause 
            {OPEXPR 
            :opno 96 
            :opfuncid 65 
            :opresulttype 16 
            :opretset false 
            :opcollid 0 
            :inputcollid 0 
            :args (
               {VAR :varno 1 :varattno 1 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 1 :varattnosyn 1 :location 81}
               {VAR :varno 2 :varattno 1 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 2 :varattnosyn 1 :location 96}
            )
            :location 94
            }
         :is_pushed_down false 
         :outerjoin_delayed false 
         :can_join true 
         :pseudoconstant false 
         :leakproof false 
         :has_volatile 2 
         :security_level 0 
         :clause_relids (b 1 2)
         :required_relids (b 1 2)
         :outer_relids (b 1)
         :nullable_relids (b)
         :left_relids (b 1)
         :right_relids (b 2)
         :orclause <> 
         :eval_cost.startup 0.00 
         :eval_cost.per_tuple 0.00 
         :norm_selec 0.0050 
         :outer_selec 0.0050 
         :mergeopfamilies (o 1976)
         :left_em <> 
         :right_em <> 
         :outer_is_left false 
         :hashjoinoperator 96 
         :left_bucketsize 0.0909 
         :right_bucketsize 0.1000 
         :left_mcvfreq 0.0000 
         :right_mcvfreq 0.0000 
         :left_hasheqoperator 96 
         :right_hasheqoperator 96
         }
      )
      :num_batches 1 
      :inner_rows_total 11
      }
   :jpath.joinrestrictinfo (
      {RESTRICTINFO 
      :clause 
         {OPEXPR 
         :opno 96 
         :opfuncid 65 
         :opresulttype 16 
         :opretset false 
         :opcollid 0 
         :inputcollid 0 
         :args (
            {VAR :varno 1 :varattno 1 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 1 :varattnosyn 1 :location 143}
            {VAR :varno 4 :varattno 1 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 4 :varattnosyn 1 :location 158}
         )
         :location 156
         }
      :is_pushed_down false 
      :outerjoin_delayed false 
      :can_join true 
      :pseudoconstant false 
      :leakproof false 
      :has_volatile 2 
      :security_level 0 
      :clause_relids (b 1 4)
      :required_relids (b 1 4)
      :outer_relids (b 1 2)
      :nullable_relids (b)
      :left_relids (b 1)
      :right_relids (b 4)
      :orclause <> 
      :eval_cost.startup 0.00 
      :eval_cost.per_tuple 0.00 
      :norm_selec 0.0050 
      :outer_selec 0.0050 
      :mergeopfamilies (o 1976)
      :left_em <> 
      :right_em <> 
      :outer_is_left false 
      :hashjoinoperator 96 
      :left_bucketsize 0.0909 
      :right_bucketsize 0.1000 
      :left_mcvfreq 0.0000 
      :right_mcvfreq 0.0000 
      :left_hasheqoperator 96 
      :right_hasheqoperator 96
      }
   )
   :path_hashclauses (
      {RESTRICTINFO 
      :clause 
         {OPEXPR 
         :opno 96 
         :opfuncid 65 
         :opresulttype 16 
         :opretset false 
         :opcollid 0 
         :inputcollid 0 
         :args (
            {VAR :varno 1 :varattno 1 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 1 :varattnosyn 1 :location 143}
            {VAR :varno 4 :varattno 1 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 4 :varattnosyn 1 :location 158}
         )
         :location 156
         }
      :is_pushed_down false 
      :outerjoin_delayed false 
      :can_join true 
      :pseudoconstant false 
      :leakproof false 
      :has_volatile 2 
      :security_level 0 
      :clause_relids (b 1 4)
      :required_relids (b 1 4)
      :outer_relids (b 1 2)
      :nullable_relids (b)
      :left_relids (b 1)
      :right_relids (b 4)
      :orclause <> 
      :eval_cost.startup 0.00 
      :eval_cost.per_tuple 0.00 
      :norm_selec 0.0050 
      :outer_selec 0.0050 
      :mergeopfamilies (o 1976)
      :left_em <> 
      :right_em <> 
      :outer_is_left false 
      :hashjoinoperator 96 
      :left_bucketsize 0.0909 
      :right_bucketsize 0.1000 
      :left_mcvfreq 0.0000 
      :right_mcvfreq 0.0000 
      :left_hasheqoperator 96 
      :right_hasheqoperator 96
      }
   )
   :num_batches 1 
   :inner_rows_total 112
   }


select s.product_id, pc.product_id from sales as s 
 join product_class pc 
  on s.product_id = pc.product_id;


CREATE TABLE emp AS
SELECT * FROM (VALUES
 (7369, 'SMITH',  'CLERK',     7902, DATE '1980-12-17', 800.00,     null, 20),
 (7499, 'ALLEN',  'SALESMAN',  7698, DATE '1981-02-20', 1600.00,  300.00, 30),
 (7521, 'WARD',   'SALESMAN',  7698, DATE '1981-02-22', 1250.00,  500.00, 30),
 (7566, 'JONES',  'MANAGER',   7839, DATE '1981-02-04', 2975.00,    null, 20),
 (7654, 'MARTIN', 'SALESMAN',  7698, DATE '1981-09-28', 1250.00, 1400.00, 30),
 (7698, 'BLAKE',  'MANAGER',   7839, DATE '1981-01-05', 2850.00,    null, 30),
 (7782, 'CLARK',  'MANAGER',   7839, DATE '1981-06-09', 2450.00,    null, 10),
 (7788, 'SCOTT',  'ANALYST',   7566, DATE '1987-04-19', 3000.00,    null, 20),
 (7839, 'KING',   'PRESIDENT', null, DATE '1981-11-17', 5000.00,    null, 10),
 (7844, 'TURNER', 'SALESMAN',  7698, DATE '1981-09-08', 1500.00,    0.00, 30),
 (7876, 'ADAMS',  'CLERK',     7788, DATE '1987-05-23', 1100.00,    null, 20),
 (7900, 'JAMES',  'CLERK',     7698, DATE '1981-12-03',  950.00,    null, 30),
 (7902, 'FORD',   'ANALYST',   7566, DATE '1981-12-03', 3000.00,    null, 20),
 (7934, 'MILLER', 'CLERK',     7782, DATE '1982-01-23', 1300.00,    null, 10)
) AS emp (EMPNO, ENAME, JOB, MGR, HIREDATE, SAL, COMM, DEPTNO);

CREATE TABLE dept AS
SELECT * FROM (VALUES
 (10, 'ACCOUNTING', 'NEW YORK'),
 (20, 'RESEARCH',   'DALLAS'),
 (30, 'SALES',      'CHICAGO'),
 (40, 'OPERATIONS', 'BOSTON')
) AS dept (DEPTNO, DNAME, LOC);

CREATE TABLE bonus (
  ENAME VARCHAR(10),
  JOB  VARCHAR(9),
  SAL  DECIMAL(6, 2),
  COMM DECIMAL(6, 2));

CREATE TABLE salgrade AS
SELECT * FROM (VALUES
  (1,  700, 1200),
  (2, 1201, 1400),
  (3, 1401, 2000),
  (4, 2001, 3000),
  (5, 3001, 9999)
) AS salgrade (GRADE, LOSAL, HISAL);

CREATE TABLE dummy AS
SELECT * FROM (VALUES
  (0)
) AS dummy (DUMMY);


select * from sales.dept d where d.deptno in (select e.deptno from sales.emp e) order by d.deptno


```




## case 2

```sql
create table student(sno int primary key, sname varchar(10), ssex int);

insert into student values(1, 'tom', 1);
insert into student values(2, 'bob', 0);
insert into student values(3, 'andy', 0);
insert into student values(4, 'anven', 1);
insert into student values(5, 'snona', 1);

create table course(cno int primary key, cname varchar(10), tno int);
insert into course values(1, 'math', 1);
insert into course values(2, 'chinese', 3);
insert into course values(3, 'english', 2);
insert into course values(4, 'music', 5);
insert into course values(5, 'art', 4);

create table score(sno int, cno int, degree int);
insert into score values(1, 1, 70);
insert into score values(1, 2, 70);
insert into score values(1, 3, 70);
insert into score values(1, 4, 70);
insert into score values(1, 5, 70);
insert into score values(2, 1, 80);
insert into score values(2, 2, 80);
insert into score values(2, 3, 80);
insert into score values(2, 4, 80);
insert into score values(2, 5, 80);
insert into score values(3, 1, 80);
insert into score values(3, 2, 80);

create table teacher(tno int primary key, tname varchar(10), tsex int);
insert into teacher values(1, 'zhao', 1);
insert into teacher values(2, 'qian', 0);
insert into teacher values(3, 'sun', 0);
insert into teacher values(4, 'li', 1);
insert into teacher values(5, 'zhou', 1);

```


`SELECT st.sname FROM STUDENT st WHERE st.sno =ANY (SELECT sno FROM SCORE WHERE st.sno = sno);`

```sql

explain SELECT st.sname FROM STUDENT st WHERE st.sno =ANY (SELECT sno FROM SCORE WHERE st.sno = sno);
                          QUERY PLAN                           
---------------------------------------------------------------
 Seq Scan on student st  (cost=0.00..661.75 rows=550 width=38)
   Filter: (SubPlan 1)
   SubPlan 1
     ->  Seq Scan on score  (cost=0.00..1.15 rows=4 width=4)
           Filter: (st.sno = sno)

   {QUERY 
   :commandType 1 
   :querySource 0 
   :canSetTag true 
   :utilityStmt <> 
   :resultRelation 0 
   :hasAggs false 
   :hasWindowFuncs false 
   :hasTargetSRFs false 
   :hasSubLinks true 
   :hasDistinctOn false 
   :hasRecursive false 
   :hasModifyingCTE false 
   :hasForUpdate false 
   :hasRowSecurity false 
   :isReturn false 
   :cteList <> 
   :rtable (
      {RANGETBLENTRY 
      :alias 
         {ALIAS :aliasname st :colnames <>}
      :eref 
         {ALIAS :aliasname st :colnames ("sno" "sname" "ssex")}
      :rtekind 0 
      :relid 91474 
      :relkind r 
      :rellockmode 1 
      :tablesample <> 
      :lateral false 
      :inh true 
      :inFromCl true 
      :requiredPerms 2 
      :checkAsUser 0 
      :selectedCols (b 8 9)
      :insertedCols (b)
      :updatedCols (b)
      :extraUpdatedCols (b)
      :securityQuals <>
      }
   )
   :jointree 
      {FROMEXPR 
      :fromlist (
         {RANGETBLREF 
         :rtindex 1
         }
      )
      :quals 
         {SUBLINK 
         :subLinkType 2 
         :subLinkId 0 
         :testexpr 
            {OPEXPR 
            :opno 96 
            :opfuncid 65 
            :opresulttype 16 
            :opretset false 
            :opcollid 0 
            :inputcollid 0 
            :args (
               {VAR :varno 1 :varattno 1 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 1 :varattnosyn 1 :location 38}
               {PARAM 
               :paramkind 2 
               :paramid 1 
               :paramtype 23 
               :paramtypmod -1 
               :paramcollid 0 
               :location -1
               }
            )
            :location 45
            }
         :operName ("=")
         :subselect 
            {QUERY 
            :commandType 1 
            :querySource 0 
            :canSetTag true 
            :utilityStmt <> 
            :resultRelation 0 
            :hasAggs false 
            :hasWindowFuncs false 
            :hasTargetSRFs false 
            :hasSubLinks false 
            :hasDistinctOn false 
            :hasRecursive false 
            :hasModifyingCTE false 
            :hasForUpdate false 
            :hasRowSecurity false 
            :isReturn false 
            :cteList <> 
            :rtable (
               {RANGETBLENTRY 
               :alias <> 
               :eref 
                  {ALIAS :aliasname score :colnames ("sno" "cno" "degree")}
               :rtekind 0 
               :relid 91466 
               :relkind r 
               :rellockmode 1 
               :tablesample <> 
               :lateral false 
               :inh true 
               :inFromCl true 
               :requiredPerms 2 
               :checkAsUser 0 
               :selectedCols (b 8)
               :insertedCols (b)
               :updatedCols (b)
               :extraUpdatedCols (b)
               :securityQuals <>
               }
            )
            :jointree 
               {FROMEXPR 
               :fromlist (
                  {RANGETBLREF 
                  :rtindex 1
                  }
               )
               :quals 
                  {OPEXPR 
                  :opno 96 
                  :opfuncid 65 
                  :opresulttype 16 
                  :opretset false 
                  :opcollid 0 
                  :inputcollid 0 
                  :args (
                     {VAR :varno 1 :varattno 1 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 1 :varnosyn 1 :varattnosyn 1 :location 79}
                     {VAR :varno 1 :varattno 1 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 1 :varattnosyn 1 :location 88}
                  )
                  :location 86
                  }
               }
            :mergeActionList <> 
            :mergeUseOuterJoin false 
            :targetList (
               {TARGETENTRY 
               :expr 
                  {VAR :varno 1 :varattno 1 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 1 :varattnosyn 1 :location 58}
               :resno 1 
               :resname sno 
               :ressortgroupref 0 
               :resorigtbl 91466 
               :resorigcol 1 
               :resjunk false
               }
            )
            :override 0 
            :onConflict <> 
            :returningList <> 
            :groupClause <> 
            :groupDistinct false 
            :groupingSets <> 
            :havingQual <> 
            :windowClause <> 
            :distinctClause <> 
            :sortClause <> 
            :limitOffset <> 
            :limitCount <> 
            :limitOption 0 
            :rowMarks <> 
            :setOperations <> 
            :constraintDeps <> 
            :withCheckOptions <> 
            :stmt_location 0 
            :stmt_len 0
            }
         :location 45
         }
      }
   :mergeActionList <> 
   :mergeUseOuterJoin false 
   :targetList (
      {TARGETENTRY 
      :expr 
         {VAR :varno 1 :varattno 2 :vartype 1043 :vartypmod 14 :varcollid 100 :varlevelsup 0 :varnosyn 1 :varattnosyn 2 :location 7}
      :resno 1 
      :resname sname 
      :ressortgroupref 0 
      :resorigtbl 91474 
      :resorigcol 2 
      :resjunk false
      }
   )
   :override 0 
   :onConflict <> 
   :returningList <> 
   :groupClause <> 
   :groupDistinct false 
   :groupingSets <> 
   :havingQual <> 
   :windowClause <> 
   :distinctClause <> 
   :sortClause <> 
   :limitOffset <> 
   :limitCount <> 
   :limitOption 0 
   :rowMarks <> 
   :setOperations <> 
   :constraintDeps <> 
   :withCheckOptions <> 
   :stmt_location 0 
   :stmt_len 92
   }

```

`SELECT * FROM STUDENT LEFT JOIN SCORE ON TRUE, (SELECT * FROM TEACHER) AS t, COURSE, (VALUES (1, 1)) AS NUM(x, y), GENERATE_SERIES(1, 10) AS GS(z); `

```sql
                                           QUERY PLAN                                           
------------------------------------------------------------------------------------------------
 Nested Loop  (cost=0.00..2179816861.25 rows=174240000000 width=156)
   ->  Nested Loop  (cost=0.00..1816836.25 rows=145200000 width=108)
         ->  Nested Loop Left Join  (cost=0.00..1812.50 rows=132000 width=62)
               ->  Nested Loop  (cost=0.00..161.35 rows=11000 width=50)
                     ->  Function Scan on generate_series gs  (cost=0.00..0.10 rows=10 width=4)
                     ->  Materialize  (cost=0.00..26.50 rows=1100 width=46)
                           ->  Seq Scan on student  (cost=0.00..21.00 rows=1100 width=46)
               ->  Materialize  (cost=0.00..1.18 rows=12 width=12)
                     ->  Seq Scan on score  (cost=0.00..1.12 rows=12 width=12)
         ->  Materialize  (cost=0.00..26.50 rows=1100 width=46)
               ->  Seq Scan on teacher  (cost=0.00..21.00 rows=1100 width=46)
   ->  Materialize  (cost=0.00..28.00 rows=1200 width=40)
         ->  Seq Scan on course  (cost=0.00..22.00 rows=1200 width=40)


                                                                                     QUERY PLAN                                                                                      
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Nested Loop  (cost=0.00..2179816861.25 rows=174240000000 width=156)
   Output: student.sno, student.sname, student.ssex, score.sno, score.cno, score.degree, teacher.tno, teacher.tname, teacher.tsex, course.no, course.name, course.credit, 1, 1, gs.z
   ->  Nested Loop  (cost=0.00..1816836.25 rows=145200000 width=108)
         Output: student.sno, student.sname, student.ssex, score.sno, score.cno, score.degree, teacher.tno, teacher.tname, teacher.tsex, gs.z
         ->  Nested Loop Left Join  (cost=0.00..1812.50 rows=132000 width=62)
               Output: student.sno, student.sname, student.ssex, score.sno, score.cno, score.degree, gs.z
               ->  Nested Loop  (cost=0.00..161.35 rows=11000 width=50)
                     Output: gs.z, student.sno, student.sname, student.ssex
                     ->  Function Scan on pg_catalog.generate_series gs  (cost=0.00..0.10 rows=10 width=4)
                           Output: gs.z
                           Function Call: generate_series(1, 10)
                     ->  Materialize  (cost=0.00..26.50 rows=1100 width=46)
                           Output: student.sno, student.sname, student.ssex
                           ->  Seq Scan on public.student  (cost=0.00..21.00 rows=1100 width=46)
                                 Output: student.sno, student.sname, student.ssex
               ->  Materialize  (cost=0.00..1.18 rows=12 width=12)
                     Output: score.sno, score.cno, score.degree
                     ->  Seq Scan on public.score  (cost=0.00..1.12 rows=12 width=12)
                           Output: score.sno, score.cno, score.degree
         ->  Materialize  (cost=0.00..26.50 rows=1100 width=46)
               Output: teacher.tno, teacher.tname, teacher.tsex
               ->  Seq Scan on public.teacher  (cost=0.00..21.00 rows=1100 width=46)
                     Output: teacher.tno, teacher.tname, teacher.tsex
   ->  Materialize  (cost=0.00..28.00 rows=1200 width=40)
         Output: course.no, course.name, course.credit
         ->  Seq Scan on public.course  (cost=0.00..22.00 rows=1200 width=40)
               Output: course.no, course.name, course.credit

   {QUERY 
   :commandType 1 
   :querySource 0 
   :canSetTag true 
   :utilityStmt <> 
   :resultRelation 0 
   :hasAggs false 
   :hasWindowFuncs false 
   :hasTargetSRFs false 
   :hasSubLinks false 
   :hasDistinctOn false 
   :hasRecursive false 
   :hasModifyingCTE false 
   :hasForUpdate false 
   :hasRowSecurity false 
   :isReturn false 
   :cteList <> 
   :rtable (
      {RANGETBLENTRY 
      :alias <> 
      :eref 
         {ALIAS :aliasname student :colnames ("sno" "sname" "ssex")}
      :rtekind 0 
      :relid 91474 
      :relkind r 
      :rellockmode 1 
      :tablesample <> 
      :lateral false 
      :inh true 
      :inFromCl true 
      :requiredPerms 2 
      :checkAsUser 0 
      :selectedCols (b 8 9 10)
      :insertedCols (b)
      :updatedCols (b)
      :extraUpdatedCols (b)
      :securityQuals <>
      }
      {RANGETBLENTRY 
      :alias <> 
      :eref 
         {ALIAS :aliasname score :colnames ("sno" "cno" "degree")}
      :rtekind 0 
      :relid 91466 
      :relkind r 
      :rellockmode 1 
      :tablesample <> 
      :lateral false 
      :inh true 
      :inFromCl true 
      :requiredPerms 2 
      :checkAsUser 0 
      :selectedCols (b 8 9 10)
      :insertedCols (b)
      :updatedCols (b)
      :extraUpdatedCols (b)
      :securityQuals <>
      }
      {RANGETBLENTRY 
      :alias <> 
      :eref 
         {ALIAS :aliasname unnamed_join :colnames ("sno" "sname" "ssex" "sno" "cno" "degree")}
      :rtekind 2 
      :jointype 1 
      :joinmergedcols 0 
      :joinaliasvars (
         {VAR :varno 1 :varattno 1 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 1 :varattnosyn 1 :location -1}
         {VAR :varno 1 :varattno 2 :vartype 1043 :vartypmod 14 :varcollid 100 :varlevelsup 0 :varnosyn 1 :varattnosyn 2 :location -1}
         {VAR :varno 1 :varattno 3 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 1 :varattnosyn 3 :location -1}
         {VAR :varno 2 :varattno 1 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 2 :varattnosyn 1 :location -1}
         {VAR :varno 2 :varattno 2 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 2 :varattnosyn 2 :location -1}
         {VAR :varno 2 :varattno 3 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 2 :varattnosyn 3 :location -1}
      )
      :joinleftcols (i 1 2 3)
      :joinrightcols (i 1 2 3)
      :join_using_alias <> 
      :lateral false 
      :inh false 
      :inFromCl true 
      :requiredPerms 0 
      :checkAsUser 0 
      :selectedCols (b)
      :insertedCols (b)
      :updatedCols (b)
      :extraUpdatedCols (b)
      :securityQuals <>
      }
      {RANGETBLENTRY 
      :alias 
         {ALIAS :aliasname t :colnames <>}
      :eref 
         {ALIAS :aliasname t :colnames ("tno" "tname" "tsex")}
      :rtekind 1 
      :subquery 
         {QUERY 
         :commandType 1 
         :querySource 0 
         :canSetTag true 
         :utilityStmt <> 
         :resultRelation 0 
         :hasAggs false 
         :hasWindowFuncs false 
         :hasTargetSRFs false 
         :hasSubLinks false 
         :hasDistinctOn false 
         :hasRecursive false 
         :hasModifyingCTE false 
         :hasForUpdate false 
         :hasRowSecurity false 
         :isReturn false 
         :cteList <> 
         :rtable (
            {RANGETBLENTRY 
            :alias <> 
            :eref 
               {ALIAS :aliasname teacher :colnames ("tno" "tname" "tsex")}
            :rtekind 0 
            :relid 91469 
            :relkind r 
            :rellockmode 1 
            :tablesample <> 
            :lateral false 
            :inh true 
            :inFromCl true 
            :requiredPerms 2 
            :checkAsUser 0 
            :selectedCols (b 8 9 10)
            :insertedCols (b)
            :updatedCols (b)
            :extraUpdatedCols (b)
            :securityQuals <>
            }
         )
         :jointree 
            {FROMEXPR 
            :fromlist (
               {RANGETBLREF 
               :rtindex 1
               }
            )
            :quals <>
            }
         :mergeActionList <> 
         :mergeUseOuterJoin false 
         :targetList (
            {TARGETENTRY 
            :expr 
               {VAR :varno 1 :varattno 1 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 1 :varattnosyn 1 :location 55}
            :resno 1 
            :resname tno 
            :ressortgroupref 0 
            :resorigtbl 91469 
            :resorigcol 1 
            :resjunk false
            }
            {TARGETENTRY 
            :expr 
               {VAR :varno 1 :varattno 2 :vartype 1043 :vartypmod 14 :varcollid 100 :varlevelsup 0 :varnosyn 1 :varattnosyn 2 :location 55}
            :resno 2 
            :resname tname 
            :ressortgroupref 0 
            :resorigtbl 91469 
            :resorigcol 2 
            :resjunk false
            }
            {TARGETENTRY 
            :expr 
               {VAR :varno 1 :varattno 3 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 1 :varattnosyn 3 :location 55}
            :resno 3 
            :resname tsex 
            :ressortgroupref 0 
            :resorigtbl 91469 
            :resorigcol 3 
            :resjunk false
            }
         )
         :override 0 
         :onConflict <> 
         :returningList <> 
         :groupClause <> 
         :groupDistinct false 
         :groupingSets <> 
         :havingQual <> 
         :windowClause <> 
         :distinctClause <> 
         :sortClause <> 
         :limitOffset <> 
         :limitCount <> 
         :limitOption 0 
         :rowMarks <> 
         :setOperations <> 
         :constraintDeps <> 
         :withCheckOptions <> 
         :stmt_location 0 
         :stmt_len 0
         }
      :security_barrier false 
      :lateral false 
      :inh false 
      :inFromCl true 
      :requiredPerms 0 
      :checkAsUser 0 
      :selectedCols (b)
      :insertedCols (b)
      :updatedCols (b)
      :extraUpdatedCols (b)
      :securityQuals <>
      }
      {RANGETBLENTRY 
      :alias <> 
      :eref 
         {ALIAS :aliasname course :colnames ("no" "name" "credit")}
      :rtekind 0 
      :relid 66909 
      :relkind r 
      :rellockmode 1 
      :tablesample <> 
      :lateral false 
      :inh true 
      :inFromCl true 
      :requiredPerms 2 
      :checkAsUser 0 
      :selectedCols (b 8 9 10)
      :insertedCols (b)
      :updatedCols (b)
      :extraUpdatedCols (b)
      :securityQuals <>
      }
      {RANGETBLENTRY 
      :alias 
         {ALIAS :aliasname num :colnames ("x" "y")}
      :eref 
         {ALIAS :aliasname num :colnames ("x" "y")}
      :rtekind 1 
      :subquery 
         {QUERY 
         :commandType 1 
         :querySource 0 
         :canSetTag true 
         :utilityStmt <> 
         :resultRelation 0 
         :hasAggs false 
         :hasWindowFuncs false 
         :hasTargetSRFs false 
         :hasSubLinks false 
         :hasDistinctOn false 
         :hasRecursive false 
         :hasModifyingCTE false 
         :hasForUpdate false 
         :hasRowSecurity false 
         :isReturn false 
         :cteList <> 
         :rtable (
            {RANGETBLENTRY 
            :alias <> 
            :eref 
               {ALIAS :aliasname *VALUES* :colnames ("column1" "column2")}
            :rtekind 5 
            :values_lists ((
               {CONST 
               :consttype 23 
               :consttypmod -1 
               :constcollid 0 
               :constlen 4 
               :constbyval true 
               :constisnull false 
               :location 94 
               :constvalue 4 [ 1 0 0 0 0 0 0 0 ]
               }
               {CONST 
               :consttype 23 
               :consttypmod -1 
               :constcollid 0 
               :constlen 4 
               :constbyval true 
               :constisnull false 
               :location 97 
               :constvalue 4 [ 1 0 0 0 0 0 0 0 ]
               }
            ))
            :coltypes (o 23 23)
            :coltypmods (i -1 -1)
            :colcollations (o 0 0)
            :lateral false 
            :inh false 
            :inFromCl true 
            :requiredPerms 0 
            :checkAsUser 0 
            :selectedCols (b)
            :insertedCols (b)
            :updatedCols (b)
            :extraUpdatedCols (b)
            :securityQuals <>
            }
         )
         :jointree 
            {FROMEXPR 
            :fromlist (
               {RANGETBLREF 
               :rtindex 1
               }
            )
            :quals <>
            }
         :mergeActionList <> 
         :mergeUseOuterJoin false 
         :targetList (
            {TARGETENTRY 
            :expr 
               {VAR :varno 1 :varattno 1 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 1 :varattnosyn 1 :location -1}
            :resno 1 
            :resname column1 
            :ressortgroupref 0 
            :resorigtbl 0 
            :resorigcol 0 
            :resjunk false
            }
            {TARGETENTRY 
            :expr 
               {VAR :varno 1 :varattno 2 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 1 :varattnosyn 2 :location -1}
            :resno 2 
            :resname column2 
            :ressortgroupref 0 
            :resorigtbl 0 
            :resorigcol 0 
            :resjunk false
            }
         )
         :override 0 
         :onConflict <> 
         :returningList <> 
         :groupClause <> 
         :groupDistinct false 
         :groupingSets <> 
         :havingQual <> 
         :windowClause <> 
         :distinctClause <> 
         :sortClause <> 
         :limitOffset <> 
         :limitCount <> 
         :limitOption 0 
         :rowMarks <> 
         :setOperations <> 
         :constraintDeps <> 
         :withCheckOptions <> 
         :stmt_location 0 
         :stmt_len 0
         }
      :security_barrier false 
      :lateral false 
      :inh false 
      :inFromCl true 
      :requiredPerms 0 
      :checkAsUser 0 
      :selectedCols (b)
      :insertedCols (b)
      :updatedCols (b)
      :extraUpdatedCols (b)
      :securityQuals <>
      }
      {RANGETBLENTRY 
      :alias 
         {ALIAS 
         :aliasname gs 
         :colnames ("z")
         }
      :eref 
         {ALIAS 
         :aliasname gs 
         :colnames ("z")
         }
      :rtekind 3 
      :functions (
         {RANGETBLFUNCTION 
         :funcexpr 
            {FUNCEXPR 
            :funcid 1067 
            :funcresulttype 23 
            :funcretset true 
            :funcvariadic false 
            :funcformat 0 
            :funccollid 0 
            :inputcollid 0 
            :args (
               {CONST 
               :consttype 23 
               :consttypmod -1 
               :constcollid 0 
               :constlen 4 
               :constbyval true 
               :constisnull false 
               :location 131 
               :constvalue 4 [ 1 0 0 0 0 0 0 0 ]
               }
               {CONST 
               :consttype 23 
               :consttypmod -1 
               :constcollid 0 
               :constlen 4 
               :constbyval true 
               :constisnull false 
               :location 134 
               :constvalue 4 [ 10 0 0 0 0 0 0 0 ]
               }
            )
            :location 115
            }
         :funccolcount 1 
         :funccolnames <> 
         :funccoltypes <> 
         :funccoltypmods <> 
         :funccolcollations <> 
         :funcparams (b)
         }
      )
      :funcordinality false 
      :lateral false 
      :inh false 
      :inFromCl true 
      :requiredPerms 0 
      :checkAsUser 0 
      :selectedCols (b)
      :insertedCols (b)
      :updatedCols (b)
      :extraUpdatedCols (b)
      :securityQuals <>
      }
   )
   :jointree 
      {FROMEXPR 
      :fromlist (
         {JOINEXPR 
         :jointype 1 
         :isNatural false 
         :larg 
            {RANGETBLREF 
            :rtindex 1
            }
         :rarg 
            {RANGETBLREF 
            :rtindex 2
            }
         :usingClause <> 
         :join_using_alias <> 
         :quals 
            {CONST 
            :consttype 16 
            :consttypmod -1 
            :constcollid 0 
            :constlen 1 
            :constbyval true 
            :constisnull false 
            :location 41 
            :constvalue 1 [ 1 0 0 0 0 0 0 0 ]
            }
         :alias <> 
         :rtindex 3
         }
         {RANGETBLREF 
         :rtindex 4
         }
         {RANGETBLREF 
         :rtindex 5
         }
         {RANGETBLREF 
         :rtindex 6
         }
         {RANGETBLREF 
         :rtindex 7
         }
      )
      :quals <>
      }
   :mergeActionList <> 
   :mergeUseOuterJoin false 
   :targetList (
      {TARGETENTRY 
      :expr 
         {VAR :varno 1 :varattno 1 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 1 :varattnosyn 1 :location 7}
      :resno 1 
      :resname sno 
      :ressortgroupref 0 
      :resorigtbl 91474 
      :resorigcol 1 
      :resjunk false
      }
      {TARGETENTRY 
      :expr 
         {VAR :varno 1 :varattno 2 :vartype 1043 :vartypmod 14 :varcollid 100 :varlevelsup 0 :varnosyn 1 :varattnosyn 2 :location 7}
      :resno 2 
      :resname sname 
      :ressortgroupref 0 
      :resorigtbl 91474 
      :resorigcol 2 
      :resjunk false
      }
      {TARGETENTRY 
      :expr 
         {VAR :varno 1 :varattno 3 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 1 :varattnosyn 3 :location 7}
      :resno 3 
      :resname ssex 
      :ressortgroupref 0 
      :resorigtbl 91474 
      :resorigcol 3 
      :resjunk false
      }
      {TARGETENTRY 
      :expr 
         {VAR :varno 2 :varattno 1 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 2 :varattnosyn 1 :location 7}
      :resno 4 
      :resname sno 
      :ressortgroupref 0 
      :resorigtbl 91466 
      :resorigcol 1 
      :resjunk false
      }
      {TARGETENTRY 
      :expr 
         {VAR :varno 2 :varattno 2 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 2 :varattnosyn 2 :location 7}
      :resno 5 
      :resname cno 
      :ressortgroupref 0 
      :resorigtbl 91466 
      :resorigcol 2 
      :resjunk false
      }
      {TARGETENTRY 
      :expr 
         {VAR :varno 2 :varattno 3 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 2 :varattnosyn 3 :location 7}
      :resno 6 
      :resname degree 
      :ressortgroupref 0 
      :resorigtbl 91466 
      :resorigcol 3 
      :resjunk false
      }
      {TARGETENTRY 
      :expr 
         {VAR :varno 4 :varattno 1 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 4 :varattnosyn 1 :location 7}
      :resno 7 
      :resname tno 
      :ressortgroupref 0 
      :resorigtbl 91469 
      :resorigcol 1 
      :resjunk false
      }
      {TARGETENTRY 
      :expr 
         {VAR :varno 4 :varattno 2 :vartype 1043 :vartypmod 14 :varcollid 100 :varlevelsup 0 :varnosyn 4 :varattnosyn 2 :location 7}
      :resno 8 
      :resname tname 
      :ressortgroupref 0 
      :resorigtbl 91469 
      :resorigcol 2 
      :resjunk false
      }
      {TARGETENTRY 
      :expr 
         {VAR :varno 4 :varattno 3 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 4 :varattnosyn 3 :location 7}
      :resno 9 
      :resname tsex 
      :ressortgroupref 0 
      :resorigtbl 91469 
      :resorigcol 3 
      :resjunk false
      }
      {TARGETENTRY 
      :expr 
         {VAR :varno 5 :varattno 1 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 5 :varattnosyn 1 :location 7}
      :resno 10 
      :resname no 
      :ressortgroupref 0 
      :resorigtbl 66909 
      :resorigcol 1 
      :resjunk false
      }
      {TARGETENTRY 
      :expr 
         {VAR :varno 5 :varattno 2 :vartype 1043 :vartypmod -1 :varcollid 100 :varlevelsup 0 :varnosyn 5 :varattnosyn 2 :location 7}
      :resno 11 
      :resname name 
      :ressortgroupref 0 
      :resorigtbl 66909 
      :resorigcol 2 
      :resjunk false
      }
      {TARGETENTRY 
      :expr 
         {VAR :varno 5 :varattno 3 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 5 :varattnosyn 3 :location 7}
      :resno 12 
      :resname credit 
      :ressortgroupref 0 
      :resorigtbl 66909 
      :resorigcol 3 
      :resjunk false
      }
      {TARGETENTRY 
      :expr 
         {VAR :varno 6 :varattno 1 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 6 :varattnosyn 1 :location 7}
      :resno 13 
      :resname x 
      :ressortgroupref 0 
      :resorigtbl 0 
      :resorigcol 0 
      :resjunk false
      }
      {TARGETENTRY 
      :expr 
         {VAR :varno 6 :varattno 2 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 6 :varattnosyn 2 :location 7}
      :resno 14 
      :resname y 
      :ressortgroupref 0 
      :resorigtbl 0 
      :resorigcol 0 
      :resjunk false
      }
      {TARGETENTRY 
      :expr 
         {VAR :varno 7 :varattno 1 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 7 :varattnosyn 1 :location 7}
      :resno 15 
      :resname z 
      :ressortgroupref 0 
      :resorigtbl 0 
      :resorigcol 0 
      :resjunk false
      }
   )
   :override 0 
   :onConflict <> 
   :returningList <> 
   :groupClause <> 
   :groupDistinct false 
   :groupingSets <> 
   :havingQual <> 
   :windowClause <> 
   :distinctClause <> 
   :sortClause <> 
   :limitOffset <> 
   :limitCount <> 
   :limitOption 0 
   :rowMarks <> 
   :setOperations <> 
   :constraintDeps <> 
   :withCheckOptions <> 
   :stmt_location 0 
   :stmt_len 146
   }

```


语法树的表示
1. node
2. var
3. rangetableentry
4. rangetableref
5. joinexpr
6. fromexpr
7. query
从上述的表和对应的打印的数据结构，分析具体的结构体的实际操作方法

语法树的遍历
1. 