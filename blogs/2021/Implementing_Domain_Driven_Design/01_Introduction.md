---
title: 实现领域驱动设计-引言
date: 2021-07-17
tags:
 - ABP
 - DDD
categories:
 - ABP
 - DDD
sidebar: 'auto'
sticky: 1
prev: false
next: ./02_What_Is_DDD
---
::: tip
本系列文章，翻译至[Implementing Domain Driven Design](https://abp.io/books/implementing-domain-driven-design)
:::
<!-- more -->

# 引言

这是实现领域驱动设计 (DDD) 的**实用指南**。虽然实现细节依赖于 ABP 框架基础设施，但核心概念、原则和模式适用于任何类型的解决方案，即使它不是 .NET 解决方案。



## 目标

本书的目标是；

- 介绍和解释 DDD 架构、概念、原理、模式和构建块。

- 解释 ABP 框架提供的分层架构和解决方案结构。
- 通过给出具体示例，引入明确的规则来实现 DDD 模式和最佳实践。
- 展示 ABP 框架为您提供了什么基础设施来以适当方式实现 DDD 的。
- 最后，根据软件开发最佳实践和我们的经验提供建议，以创建可维护的代码库。



## 简单的代码

::: tip
踢足球很简单，但踢简单的足球才是最难的。 —— 约翰克鲁伊夫
:::

如果我们将这句名言用于编程，我们可以说：

::: tip
写代码很简单，但写简单的代码是最难的。 ——？？？
:::

在本文档中，我们将介绍**简单的规则**，这些规则**很容易实现**。

一旦您的**应用程序增长**，将**很难遵循**这些规则。有时您会发现**打破规则**会在短期内为您节省时间。但是，短期节省的时间会在中长期带来**更多的时间损失**。您的代码库变得**复杂**且难以维护。大多数业务应用程序被**重写**只是因为您**无法再维护**它。

如果您**遵循规则和最佳实践**，您的代码库将更简单、更易于维护。您的应用程序**对变化的反应**更快。





::: warning
作者：Gerry Ge

出处：[https://gerryge.com/blogs/2021/Implementing_Domain_Driven_Design/01_Introduction.html](https://gerryge.com/blogs/2021/Implementing_Domain_Driven_Design/01_Introduction.html)

版权：本作品采用「[署名-非商业性使用-相同方式共享 4.0 国际](https://creativecommons.org/licenses/by-nc-sa/4.0/)」许可协议进行许可。

**转载请注明出处**
:::

