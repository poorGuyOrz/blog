---
title: "Postgres Optimizer"
date: 2022-07-22T09:24:24+08:00
draft: false
tags: ["数据库", "Postgres", "优化器"]
summary: "Postgres优化器代码调研"
---

## pg 优化器

但是从今天的视角，PostgreSQL 优化器不是一个好的实现
1. 它用C语言实现，所以扩展性不好
2. 它不是 Volcano 优化模型的，所以灵活性不好
3. 它的很多优化复杂度很高（例如Join重排是System R风格的动态规划算法），所以性能不好



公式块:



$c = \pm\sqrt{a^2 + b^2}$ 和 \\(f(x)=\int_{-\infty}^{\infty} \hat{f}(\xi) e^{2 \pi i \xi x} d \xi\\)


$$ c = \pm\sqrt{a^2 + b^2} $$

\\[ f(x)=\int_{-\infty}^{\infty} \hat{f}(\xi) e^{2 \pi i \xi x} d \xi \\]

\begin{equation*}
  \rho \frac{\mathrm{D} \mathbf{v}}{\mathrm{D} t}=\nabla \cdot \mathbb{P}+\rho \mathbf{f}
\end{equation*}

\begin{equation}
  \mathbf{E}=\sum_{i} \mathbf{E}\_{i}=\mathbf{E}\_{1}+\mathbf{E}\_{2}+\mathbf{E}_{3}+\cdots
\end{equation}

\begin{align}
  a&=b+c \\\\
  d+e&=f
\end{align}

\begin{alignat}{2}
   10&x+&3&y = 2 \\\\
   3&x+&13&y = 4
\end{alignat}

\begin{gather}
   a=b \\\\
   e=b+c
\end{gather}

\begin{CD}
   A @>a\>> B \\\\
@VbVV @AAcA \\\\
   C @= D
\end{CD}


$$ \ce{CO2 + C -> 2 CO} $$

$$ \ce{Hg^2+ ->[I-] HgI2 ->[I-] [Hg^{II}I4]^2-} $$


[Hugo]^(一个开源的静态网站生成工具)


[浅色]/[深色]

[99]/[100]


去露营啦! :(fas fa-campground fa-fw): 很快就回来.

真开心! :(far fa-grin-tears):

{{< figure src="/images/lighthouse.jpg" title="Lighthouse (figure)" >}}

{{< gist spf13 7896402 >}}


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



### case

```sql

create table sales          (product_id int , b int , c int);
create table product        (product_id int , b int , c int);
create table product_class  (product_id int , b int , c int);

insert into sales values(1,1,1),(2,2,2),(3,3,3),(4,4,4),(5,5,5),(6,6,6),(7,7,7),(8,8,8),(9,9,9),(10,10,10),(11,11,11);
insert into product values(1,1,1),(2,2,2),(3,3,3),(4,4,4),(5,5,5),(6,6,6),(7,7,7),(8,8,8),(9,9,9),(10,10,10),(11,11,11);
insert into product_class values(1,1,1),(2,2,2),(3,3,3),(4,4,4),(5,5,5),(6,6,6),(7,7,7),(8,8,8),(9,9,9),(10,10,10),(11,11,11);


select s.product_id, pc.product_id from sales as s 
 join product as p 
  on s.product_id = p.product_id 
 join product_class pc 
  on s.product_id = pc.product_id;

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