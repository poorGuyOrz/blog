---
title: "VolcanoOptimizer"
date: 2022-04-04T22:10:50+08:00
draft: false
---


# NOTE
论文阅读笔记[The Volcano Optimizer Generator: Extensibility and Efficient Search](http://www.cse.iitb.ac.in/infolab/Data/Courses/CS632/Papers/Volcano-graefe.pdf)


* 可扩展
* 面向对象
* 自顶向下
* 剪枝
* 原型是EXODUS， Volcano是对他的改进
    - 可以单独使用的优化器
    - 优化搜索时间和搜索空间
    - 可扩展
    - 可以使用启发式算法和有效的代价模型来扩展和减少搜索空间，【剪枝】
    - 灵活的成本计算模型

* 一个框架，优化器生成器，可以由“optimizer implementor”自行实现关键函数，整个优化器框架的输入是AST，输出是一个执行计划，算子的集合

* SQL是基于关系代数，Volcano把关系表达式分为逻辑表达式和物理表达式，逻辑表达式之间使用transformation进行转换，物理表达式使用基于代价的implementation和逻辑表达式映射的，关系不一定是意义对应的，例如scan可以同时一起实现projection
* 在表达式进行转换的时候可以使用condition进行模式判断，满足条件的时候可以进行变换
* 表达式使用特征描述输出，
* enforcers会强制添加某属性，用于指导优化器进行优化，例如指定表的scan方式


Logical Operator  
Operator set，也就是可以描述在目标data model上可以执行的代数运算合  
Transformation rules + Condition，对每条等价变换规则，在满足condition时才可以应用  
Logical properties : 逻辑属性，用来描述代数运算的输出所具有的一些特征，这些特征与运算的具体执行方式无关，是逻辑的，例如输出的行数，结果集的schema等    

Physical Operator  
Algorithm + Enforcer set，即可应用的物理实现算法 + 可添加的Enforcer集合  
Implementation rules + Condition，满足Condition的前提下，可以尝试该物理算法  
Cost model + Cost formula，基于cost选择最优的物理算法  
Physical property，与logical property对应，物理属性是选择了特定的算法实现后，输出的数据所具有的物理上的特性，例如数据是否有序、是否具有唯一性，数据在多机上的分布情况等，不同的物理算法，会决定执行该operator产生的物理属性，例如sort merge join会在join key上产生有序属性  
Applicability function : 决定一个物理算法，其输出是否可以满足要求父算子对自身的physical property要求，且决定它对自身的输入具有什么样的physical property要求  
Enforcer是search engine中一个重要的概念，它用来强制产生某种物理属性。例如上层是join算子，在枚举时会考虑使用sort merge join的物理执行方式(Implementation），但当递归到下层时，子节点可以选择table scan（无序输出），或者index scan（目标序输出），当选择table scan时，由于输出不满足父算子对自身输出的物理属性要求，就可以通过Order Enforcer来产生目标输出，Enforcer表示了排序这个操作，同时也包含了排序操作会产生的代价。


## The  Search  Engine

搜索实现

```c++
// PhysProp：： 此LogExpr锁具有的物理属性的要求
FindBestPlan (LogExpr, PhysProp, Limit) 
    // 如果可以在look-up table找到满足的计划，则代表以及算过，直接返回
    if the pair LogExpr and PhysProp is in the look-up table 
      if the cost in the look-up table < Limit 
          return Plan and Cost 
      else 
          return failure 

    /* else: optimization required */
    // 否则进行优化，由三种优化方式
    //  1. 使用转换方法转换为等价的逻辑表达式，且使用相同物理属性计算
    //  2. 使用实现方法生成物理表达式，此处是本层的物理表达式，所以会递归调用，且计算本层代价TotalCost，此时是使用
    //        代价模型计算代价且使用剪枝
    //        Applicability functio决定Implementation是否满足PhysProp
    //  3. 使用enforcer，计算enforcer的代价，且修改PhysProp为满足enforcer的PhysProp，使用新的PhysProp计算LogExpr
    create the set of possible "moves" from 
        - applicable transformations 
        - algorithms that give the required PhysProp 
        - enforcers for required PhysProp 
    order the set of moves by promise 
    for the most promising moves 
        if the move uses a  transformation 
            apply the transformation creating NewLogExpr 
            call FindBestPlan (NewLogExpr, PhysProp, Limit) 
        else if the move uses an algorithm
            TotalCost := cost of the algorithm 
            for each input I while TotalCost < Limit 
                determine required physical properties PP for I 
                Cost = FindBestPlan (I, PP, Limit -  TotalCost) 
                add Cost to TotalCost 
        else/* move uses an enforcer */ 
            TotalCost := cost of the enforcer 
            modify PhysProp for enforced property 
            call FindBestPlan for LogExpr with new PhysProp 
        /* maintain the look-up table of explored facts */ 

    //  如果没有在look-up table找到满足的计划，则insert
    if LogExpr is not in the look-up table 
        insert LogExpr into the look-up table 

    //  记录左右执行计划
    insert PhysProp and best plan found into look-up table 
    return best Plan and Cost

```


自定向下的搜索，使用三种规则向下扩展搜索空间，使用look-up table记录已搜索的表达式，在表达式没有新的扩展方法，或者达到代价阈值的时候停止搜索，

## 自顶向下和自底向上

* 自顶向下
    volcano，cascades等
    1. 在等价变形的范围不仅仅限于join，关系代数的等价变换都可以实现，所以理论上，这里可以使用一些启发式的算法，而不必要进行分层
    2. 可以在优化的过程中基于代价模型进行剪枝
* 自底向上
    几乎会枚举所有的执行计划，由更大的搜索空间，且由于在优化的过程中更多的只是为了搜索join order 的执行计划，所以需要提前进行一些启发式的算法，或优化语句结构便于优化器优化，或提前搜索到执行计划结束优化过程。




# [The Cascades Framework for Query Optimization]

对于volcano的改进和优化
* 把原来的搜索流程拆分为单个的task，使用栈来实现，task之间使用一个DGA来维护调度关系
* 不区分逻辑表达式和物理表达式是
* rule抽象为task，是具体的对象，apply 的时候区分表达类型


group：
    等价表达式的集合
expr：
    表达式，等价表达式使用特征输入和特征输出来表示，表达式的输入可以是表达式，也可以是group，可以视为执行计划中的一个片段。
rule
    抽象实现，分为逻辑表达式的转换rule和逻辑表达式转换到物理表达式的Implementation rule
    是否可以适用某个rule需要判断表达式可以满足rule的某些要求的规则，这个规则抽象为pattern，如果满足pattern，则可以使用此转换方法转换表达式，转换之后的表达式称为Substitution


* OptimizeGroupTask
    优化任务的入口，初始化的时候是整个ast的集合
* OptimizeExprTask
    对于group中的所有的表达式，一个表达式可能只是单纯的表达式，也有可能是由其他group作为输入的表达式，所以
    1. 都表达式使用ApplyRuleTask入栈
    2. 对于所有表达式中的group，使用ExploreGroupTask入栈，此时对于此节点，会先完成ExploreGroupTask，然后再使用ApplyRuleTask
    3. 如果是物理表达式，则使用CreatePlanTask
* ApplyRuleTask
    对表达式使用rule转换，使用nextSubstitute之后得到Substitution，可以对新的表达式使用
    1. ExploreExprTask
    2. OptimizeExprTask
    3. CreatePlanTask   如果是逻辑表达式，则使用CreatePlanTask
* ExploreGroupTask
    对group中的所有表达式调用ExploreExprTask
* ExploreExprTask
    对所有的表达式，调用ApplyRuleTask，在表达式使用ApplyRuleTask之前，会先使用ExploreGroupTask扩展表达式的group
* CreatePlanTask
    对应的是implementation rule，转换表达式为物理表达式

其中ApplyRuleTask某个表达式的时候，会先递归的使用ExploreGroupTask处理表达式的所有group，所以在transformation某个规则的时候，是整体的引用到整个表达式上