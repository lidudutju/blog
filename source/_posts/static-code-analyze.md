---
title: 静态代码检测工具对比
author: lidu
date: 2018-12-16 12：29
tags: Android
---

## 对比结论

对比项| Checkstyle | FindBugs | Android Lint | Facebook Infer
--- | --- | --- | --- | ---
内置检查规则 | 针对代码风格，100+检测规则 | 针对Java工程，200+检测规则 | 针对Android项目，300+检测规则 | 不限定编程语言和工程方式，18项检测规则
检查文件类型 | Java源代码 | Java字节码 | Java、Kotlin、XML源代码和JVM 字节码 | Java、C/C++、OC源代码
实现原理 | 抽象语法树（Antlr库）| ASM+字节码缺陷模式匹配 | 抽象语法树（Lombok -> PSI -> UAST）| Separation logic、Bi-abduction、Control Flow Graph
优势 | 轻量级、速度快、针对代码风格 | 操作字节码，对JDK定制化程度高 | 检测项多、拓展性强、官方支持 | 准确程度高、抽象程度高、与语言无关
问题与不足 | 无法检查潜在Bug | 对字节码依赖程度很高，学习门槛高 | 全量检查耗时、API变动较大 | 用OCaml实现，学习门槛高

<!--more-->

## Checkstyle

内置的检查规则主要是针对**编程规范**的一些约束规则，比如命名规约、Import限制、尺寸超标（最大文件行数、单行最大长度、方法最大行数...）等一些编码风格和命名规则的约束。

详细的内置规则分为14大类，供100多项可选择配置的规则。

可参考规则配置：

1. [English](http://checkstyle.sourceforge.net/checks.html)
2. [中文简体](https://blog.csdn.net/yang1982_0907/article/details/18086693)

### 原理

![](https://ws1.sinaimg.cn/large/dc50da5fly1fv7xwirz35j20ce0godfx.jpg)

### 优势

1. Antlr分析器和checkstyle的发展时间都比较长，技术方案很成熟
2. 业内广泛使用，如Google和Sun公司都有已配置好的开源checksyle.xml
3. 对规范类检查规则检查很细致，不依赖编译，检测速度快

### 问题和不足

1. 目前仅支持对java源代码进行检测，无法对xml、kotlin代码等进行检测，因为未集成其他语言的词法分析器
2. 检测的规则比较简单，仅限一些规范类检查，无法有效检查代码潜在Bug

## FindBugs

FindBugs是由马里兰大学开发的用于查找Java代码中的潜在错误的开源软件。最新的稳定版于2015年发布，支持JDK版本1.0-1.8。

与其他静态代码分析工具不同，FindBugs主要是从字节码层面出发，而非抽象语法树。Findbugs检查.class文件，主要检查字节码中的bug pattern。

### 原理

FindBugs项目代码主要由Java组成，运行于JDK1.7+环境中。

FindBugs从字节码层面对代码进行静态分析，主要用到技术是**缺陷模式匹配**和**数据流分析**:

1. 缺陷模式匹配：事先从代码分析经验中收集足够多的共性缺陷模式，将待分析代码与已有的共性缺陷模式进行模式匹配，从而完成软件的安全分析。这种方式的优点是简单方便，但是要求内置足够多缺陷模式，且容易产生误报。

2. 数据流分析：通过收集代码中引用到的变量信息，从而分析变量在程序中的赋值、引用以及传递等情况。对数据流进行分析可以确定变量的定义以及在代码中被引用的情况，同时还能够检查代码数据流异常，如引用在前赋值在后、只赋值无引用等。

### 优势

1. 检测思路与众不同，将字节码与一组缺陷模式进行对比以发现可能的问题

### 问题和不足

1. 需要编译源码，检测相对耗时
2. 自定义规则需要熟悉字节码相关知识，开发成本较大

## Android Lint

Lint是Google开发的与Android平台深度结合的静态代码检查工具，可以检测代码中存在的性能安全问题或一些潜在的Bug，目前内置300余项检查项，涵盖CORRECTNESS、
SECURITY、PERFORMANCE、USABILITY、A11Y、I18N、ICONS、TYPOGRAPHY、MESSAGES、CHROME_OS、RTL等11个类别。

### 原理

![](https://ws1.sinaimg.cn/large/dc50da5fly1fv7yifzl6qj20gs07z74t.jpg)

Lint Tool内部实现的大致原理是：

![](https://ws1.sinaimg.cn/large/dc50da5fly1fv957ahxvvj20i60goaai.jpg)

另外，如果我们要实现自定义Lint规则，需针对不同Lint版本实现不同接口：

![](https://ws1.sinaimg.cn/large/dc50da5fly1fv95etcjhkj20nh0gedg8.jpg)

### 优势

1. 功能强大，Lint支持Java源文件、class文件、资源文件、Gradle等文件的检查
2. 扩展性强，支持开发自定义Lint规则
3. 配套工具完善，Android Studio、Android Gradle插件原生支持Lint工具
4. Lint专为Android设计，原生提供了几百个实用的Android相关检查规则
5. 有Google官方的支持，会和Android开发工具一起升级完善

### 问题和不足

1. Lint执行全量检查比较耗时，需要编译源码
2. Lint API变动比较大，导致适配不同Lint版本开发自定义规则成本很大

## Infer

Infer是由Facebook开发，用Ocaml编写的一个可以对Java、C/C++、OC代码进行静态分析的工具。该工具主要是对一些严重的性能问题和Bug进行检测，例如空指针、资源泄露、死锁等问题。

Facebook将infer广泛应用于facebook、messenger、instagram等移动应用中。目前该项目在Github社区中Star数量8800+。

### 原理

Infer最初由Monoidics根据最新的学术研究开发而成，于2013年被Facebook收购。

该静态分析工具最大的不同是结合了大量的前沿学术研究成果：

1. [霍尔逻辑](https://zh.wikipedia.org/wiki/%E9%9C%8D%E5%B0%94%E9%80%BB%E8%BE%91)使用严格的数理逻辑来替计算机程序的正确性提供一组逻辑规则
2. [抽象解释](https://en.wikipedia.org/wiki/Abstract_interpretation)用于测度程序语义的逼近结果
3. [Separation logic](http://fbinfer.com/docs/separation-logic-and-bi-abduction.html#separation-logic)
4. [Bi-abduction](http://fbinfer.com/docs/separation-logic-and-bi-abduction.html#bi-abduction)

Infer实现了与语言无关的静态检查，通过分析源代码将其抽象为[控制流程图](https://en.wikipedia.org/wiki/Control_flow_graph)，然后Infer的检查器对其进行检查。

### 优势

1. 实现了与语言无关的静态检查，适用于多种编程语言
2. 对空指针检测的准确程度很高
3. 支持增量和非增量分析

### 问题和不足

1. 执行检测比较耗时，需要编译源码
2. 自定义检测规则开发成本很高，需要学习Ocaml语言

## 最终接入工具

对比Checkstyle、FindBugs、Android Lint和Facebook Infer之后，发现每个工具都各有特点，最终确认全部接入，但是会根据不同场景和时机适当执行部分工具。


最终对各个工具的定位是：


1. Checkstyle：**注重代码风格的统一和规范**，减轻CR负担，使得Reviewer将精力更加集中在逻辑实现方面，而非代码格式

2. Android Lint：**全方位Android项目质量检测**，内置的300+检测规则和我们自定义的规则保证Android项目中无论是资源文件还是源代码中的潜在问题及时暴露出来

3. Facebook Infer：**注重单项检测的深度和准确率**，通过结合前沿的学术领域成果，可以让我们实现语言无关的检测工具，同时保证很高的准确率，目前Infer在Null Dereference检测中准确程度非常高

4. FindBugs：**关注Java代码的常见错误**

## 检查时机和方式

### （1）编码阶段IDE实时检查

IDE Lint原生支持，在出现Warning和Error时会在代码中分别标注黄色和红色波浪线来提醒开发者


### （2）本地编译期间

1. 执行增量Lint检查，但仅检查高优先级项目
2. 执行增量Infer检查

两项检查全部通过后才可编译成功


### （3）本地Commit期间

1. 执行增量checkstyle检查
2. 执行增量Lint检查

两项检查全部通过后才可提交成功，否则会reset本次提交，建议修复后再次提交


### （4）CI打包期间

1. 执行全量checkstyle检查
2. 执行全量Lint检查
3. 执行全量Infer检查
4. 执行全量FindBugs检查

检查全部通过才可打包成功，否则发送邮件通知

## 参考资料

1. [checkstyle原理简介](http://runfriends.iteye.com/blog/1121966)
2. [使用Lint改善您的代码](https://developer.android.com/studio/write/lint?hl=zh-cn)
3. [AST介绍](http://www.eclipse.org/articles/Article-JavaCodeManipulation_AST/)
4. [Lint工具解析](https://hujiaweibujidao.github.io/blog/2016/11/19/lint-tool-analysis-3/)
5. [Infer项目主页](http://fbinfer.com/)
6. [自定义Infer规则简介](http://fbinfer.com/docs/adding-checkers.html)