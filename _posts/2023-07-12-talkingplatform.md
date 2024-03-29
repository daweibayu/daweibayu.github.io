---
layout: post
title:  "中台漫谈"
author: "daweibayu"
tags: 设计模式
excerpt_separator: <!--more-->
---

<!--more-->

## 中台是什么

举个例子，在战争中，有前线部队，也有后勤部队，前线部队可以对应为“前台”，后勤部队可以理解为“中台”。
再举个例子，在公司中，有前台（就公司门口负责接待的前台），有财务，有行政，这些其实也都可以归为中台，行政中台嘛。因为这些角色服务的人群是全公司，并不属于某个具体部门（他们本身就是一个部门），

常见的有业务中台、数据中台，还有其他五花八门的中台，这里不赘述。在我看来，所有 **一对多** 关系的部门都能归到中台。

## 中台该不该存在

我也没法直接回答这个问题，因为不同公司对中台的定义其实是不同的，有些该存在，有些则应该消失。按照我的理解，就是个名字嘛，或者说一种组织架构，这个组织架构可不可以不存在？当然可以，可不可以存在，自然也可以。但是不论这个组织结构存在与否，到干活的时候，该是谁的活还是谁的活，`你需要的不是中台，而是一名合格的架构师`。

以公司举例：

* 就算没有`行政中台`这个部门，也一定有财务、前台、人力资源等部门；
* 就算没有`业务中台`这个部门，也一定有人些 common、base 相关的代码；
* 就算没有`数据中台`这个部门，也一定有 BI 工程师。

能看到这篇文章的应该都是技术人员，我想向各位看客提一个问题“诸君写过的代码中有没有 base、common 开头的？”，反正我是写过，而且不少，虽然很多代码规范中会推荐尽量减少 base、common 命名形式，因为这些词很多时候表意不明，但是吧，实践中，我真的不认为有谁能避开这些词，因为很多时候这就是最佳命名。

那什么时候该存在？很简单，就看规模，如果只需要一个人或者两个人写 common、base 的代码，那就没必要搞这个组织架构。如果需要的人数多了，那自然用一个单独的部门管理会更好一点。

## 中台该干什么

1. 中台不应该成为前台的约束，仍然以战争举例，后勤部队是为了服务前线部队的，如果前线部队处处要看后勤部队脸色，那这仗估计是没法打了。

前线一定要拥有前线该有的机动灵活性，针对突发敌情，前线就是要快准狠，换到公司角度，前台一定要拥有直接操纵资源的能力，中台是重要的一环，但不应该也绝不能成为必不可少的一环。

### 中台的天然滞后性

中台天然就是有滞后性的，我这不是为了中台从业人员开脱，而是中台本身就是这样的生态位。你什么时候见过（非极端状态下）后勤部队扛起枪去一线打仗的？同样工作中也没见过谁把具体业务代码命名加上 common 和 base 的。

本身的合理形式就是前台将公共业务逐步沉淀到中台，由中台维护并扩展至全平台使用，这就是中台的滞后性。因为中台的滞后性而裁撤中台的，可能是真心没明白中台的责任是什么。当然安于这种状态的中台也只能算是差强人意。

### 中台的前瞻性

如果安于“滞后”，那其实被裁撤了也没什么说不过去。就像后勤部队如果缺武器了你才知道送过去，那这仗八成也不用打了。

就像我们常说产品要走在用户前面，这话就像是有先天正义性的废话，嘴炮嘛，又不用付啥责任。

那具体怎么做到前瞻性呢？

#### 提升架构设计能力

不多说，可以参看 [设计模式漫谈](/2023-06-26/talkingpattern)，好的设计在支持前台的时候会如鱼得水，这个其实是好的工程师都应该具有的能力，实际和中台无关，但前台工程师缺乏点设计能力对于工作应该不会太大，但是对于中台，如果没有架构能力，那就有的头疼了。

#### `工程师一定要理解业务`

所有的前瞻都是基于业务的，如果没有对产品的感知能力，那根本就没有前瞻的方向。其实就像“中台”只是个名字，“程序员”其实也只是个名字，如果被这个名字框住，只写代码，那就真的很悲哀了。至于具体怎么做：

1. 像 [设计模式漫谈](/2023-06-26/talkingpattern) 中说到的，通过代码架构的合理性去推导产品的合理性是一个很好的从 0 - 1 的方式
2. 多与 pm 交流，甚至可以让你熟悉的 pm 对你进行面试（我是真的通过被 pm 面试学到了很多），进一步对产品有更整体的认知
3. 尝试操刀设计具体页面和 app

#### 提前做技术储备

很多时候公司或部门都是有季度会议的，这种会议一般都有总结与前瞻的，在熟悉业务的基础上，对于管理层中长期的一个规划其实就应该有大概的预判了。这个时候中台部门就可以提前对一些可行性和技术做调研了。虽说不要高估管理层的前瞻能力，但是针对这种可行性和技术方案的初步预判，一般不会花太长时间。有了初步预判后，就根据复杂度和优先级提前做了安排。当下在写代码的时候就也要尽量留出可扩展空间。

当然，主动跟 Leader 和前台各种部门多聊一聊，需求基本就可以早前台一些开始了。

#### 中心位置的优势

对比于中台，反而是前台更容易形成信息孤岛，中台可以接触多个前台部门，收集到更多的信息，在此基础上，反而应该更容易了解前沿需求，这也是前瞻性必不可少的信息源。而且也可以将这些信息反馈给前台部门，形成公司级别跨部门的信息流通的正向反馈。

## 总结

`佛说第一波罗蜜，既非第一波罗蜜，是名第一波罗蜜`。中台就是个名字，大家赋予了这个名词不同含义。不要注重形式，不要注重名字，也不要被各种名词绕混，从真实业务出发，从具体公司文化出发，锻炼发现问题的能力，提升解决问题的能力，才能不变应万变。中台不中台的，由它去吧。

（你要是还问我，你怎么看阿里拆分中台？我会建议你再看一遍上边这段话）

ps：把此文归入“设计模式”，其实也是个人感觉企业架构也好，管理体系也罢，其实都是设计模式在不同维度的体现。
