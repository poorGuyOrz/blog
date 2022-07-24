---
title: "[大数据]Spark"
date: 2022-03-14T19:57:11+08:00
draft: true
---

# 大数据--spark

学习文档参考[这里](https://www.bookstack.cn/read/BigData-Notes/notes-%E5%A4%A7%E6%95%B0%E6%8D%AE%E5%AD%A6%E4%B9%A0%E8%B7%AF%E7%BA%BF.md)，现在因为工作的要求，需要学习sprk的开发


核心组件为Spark core，对上层提供了使用的API



## 核心组件
核心组件为三个
1. RDD
    弹性分布数据集
2. 累加器
    分布式共享只写变量
3. 广播器
    分布式共享只读变量

RDD 有两中方法
* 转换
    * 单值类型
        * map
    * 双值类型
    * keyvalue 类型
* 操作


