---
title: OneData数据治理方法论-阿里OneData实施方案
date: '2022/09/26 11:22:49'
categories:
  - 数据仓库
tags:
  - 维度建模
  - OneData

---

# 阿里OneData实施方案

## 简介

​		文章重点讲解了怎么使用OneData体系及与之相配套的工具在阿里的具体业务中实施落地。

<!--More-->

## 指导方针

- ### 数仓建设的基石
  - 充分的业务调研
  - 需求分析

- ### 数据总体架构设计
  - 数据域的划分
  - 按照维度建模理论，构建总线矩阵、抽象出业务过程和维度。
  - 对报表需求进行抽象整理，梳理出相关指标体系。

- ### 使用OneData工具完成指标规范定义和模型设计

## 实施工作流

1. 数据调研

   - 业务调研：阿里集团业务面比较广，涉及电商、数字娱乐、导航、移动互联网服务等领域。各个领域又涵盖多条业务线，如电商领域涵盖toC和toB的业务。数据仓库是要涵盖所有领域，还是各个领域自建？在各个领域内的不同业务线也存在这种问题。因此要构建大数据仓库，就需要了解各个业务领域、业务线的业务有什么异同点，业务线的模块划分、业务流程。业务调研是否充分，直接决定了数据仓库建设是否会成功。

     ![实施工作流](images/实施工作流.png)

     阿里由于各业务领域内业务比较相似、关联性大，因此在各业务领域内独自建设数据仓库，进行统一集中建设。

     ![阿里C类电商业务调研](images/C类电商业务调研.png)

   - 需求调研：

     - 目标：数据分析师、业务运营人员
     - 途径：
       1. 与数据分析师、业务运营人员的沟通。
       2. 对现有报表系统的报表进行研究分析。

2. 架构设计

   - 数据域划分

     数据域是面向业务分析时，对业务或维度进行抽象的集合。业务则是由一个个不可拆分的业务动作构成。

     ![业务与业务动作构成](images/业务与业务动作构成.png)

     从上图即可看出业务与业务动作之间的关系。在此基础上，对业务过程进行归纳，可抽象出数据域。

     ![数据域](images/数据域.png)

   - 构建总线矩阵

     - 明确每个数据域下有哪些业务过程。
     - 业务过程与哪些维度相关。

     分析清楚上述信息后，就可以构建总线矩阵了。

     ![供应链管理业务过程示例](images/供应链管理业务过程示例.png)

3. 规范定义

   主要定义指标体系，包括原子指标、修饰词、时间周期、派生指标。

4. 模型设计

   主要包括维度及维度属性的规范定义、维表、明细事实表和汇总事实表的模型设计。

**OneData体系的实施过程是一个高度迭代和动态的过程，一般采用螺旋式实施方法。在总体架构设计完成后，开始根据数据域进行迭代模型设计和评审。在各个实施过程中，都需要引入评审机制，以确保实施过程的正确性。**