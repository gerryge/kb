---
title: 实现领域驱动设计-实现：构建块
date: 2021-07-18
tags:
 - ABP
 - DDD
categories:
 - ABP
 - DDD
sidebar: 'auto'
sticky: 4
prev: ./03_Implementation_The_Big_Picture
next: ./05_Example_User_Cases
---

本系列文章，翻译至[Implementing Domain Driven Design](https://abp.io/books/implementing-domain-driven-design)
<!-- more -->

# 实现：构建块

这是本指南的重要部分。我们将通过示例介绍和解释一些**显式规则**。在实施领域驱动设计时，您可以遵循这些规则并应用到您的解决方案中。



## 领域示例

示例将使用 GitHub 使用的一些耳熟能详的概念，例如`Issue`、`Repository`、`Label`和`User`。下图显示了一些聚合、聚合根、实体、值对象以及它们之间的关系：



![image-20210719213146321](./04_Implementation_The_Building_Blocks.assets/image-20210719213146321.png)

`Issue Aggregate` 由一个 `Issue Aggregate Root` 组成，其中包含 `Comment` 和 `IssueLabel` 集合。其他聚合只简单显示，因为我们将关注`Issue Aggregate`：



![image-20210719213000930](./04_Implementation_The_Building_Blocks.assets/image-20210719213000930.png)



## 聚合

如前所述，[聚合](https://docs.abp.io/en/abp/latest/Entities)是由聚合根对象绑定在一起的一组对象（实体和值对象）。本节将介绍与聚合相关的原则和规则。



::: tip
除非我们明确编写聚合根或子集合实体，否则我们将术语实体称为聚合根和子集合实体。
:::



## 聚合/聚合根原则



### 业务规则

实体负责执行与其自身属性相关的业务规则。聚合根实体还负责其子集合实体。

聚合应该通过实施领域规则和约束来保持其**自我完整性**和**有效性**。这意味着，与 DTO 不同，实体具有**实现某些业务逻辑的方法**。实际上，我们应该尽可能地在实体中实现业务规则。



### 单个单元

聚合被检索并保存为一个单元，包含所有子集合和属性。例如，如果您想对`Issue`添加`Comment`，则需要；

- 从数据库中获取`Issue`，包括所有子集合（`Comments` 和 `IssueLabels`）。
- 使用`Issue` 类上的方法添加新评论，如`Issue.AddComment(...)`;。
- 将`Issue`（包括所有子集合）作为单个数据库操作（更新）保存到数据库。



对于以前使用 **EF Core 和关系数据库**的开发人员来说，这似乎很奇怪。获取所有`Issue`和细节似乎没有必要且效率低下。为什么我们不直接对数据库执行 SQL Insert 命令而不查询任何数据呢？

答案是我们应该在**代码**中**实现业务**规则并保持数据的**一致性**和**完整性**。如果我们有一个像“用户不能对锁定的问题发表评论”这样的业务规则，我们如何在不从数据库中检索的情况下检查问题的锁定状态？因此，只有在应用程序代码中相关对象可用时，我们才能执行业务规则。

另一方面，**MongoDB** 开发人员会发现这个规则很自然。在 MongoDB 中，聚合对象（带有子集合）保存在数据库中的**单个集合**中（而它分布在关系数据库中的多个表中）。因此，当您获得聚合时，所有子集合都已作为查询的一部分进行检索，无需任何额外配置。

ABP 框架有助于在您的应用程序中实现这一原则。



**示例：向问题中添加评论**

```csharp
public class IssueAppService : ApplicationService, IIssueAppService
{
    private readonly IRepository<Issue, Guid> _issueRepository;

    public IssueAppService(IRepository<Issue, Guid> issueRepository)
    {
        _issueRepository = issueRepository;
    }

    [Authorize]
    public async Task CreateCommentAsync(CreateCommentDto input)
    {
        var issue = await _issueRepository.GetAsync(input.IssueId);
        issue.AddComment(CurrentUser.GetId(), input.Text);
        await _issueRepository.UpdateAsynce(issue);
    }
}
```

`_issueRepository.GetAsync` 方法默认将`Issue`及其所有详细信息（子集合）作为一个单元来检索。虽然这对 MongoDB 来说是开箱即用的，但您需要为 EF Core 配置聚合详细信息。但是，一旦您配置好，存储库就会自动处理它。 `_issueRepository.GetAsync` 方法获取一个可选参数 `includeDetails`，您可以在需要时传递 `false` 以禁用此行为。



::: tip
有关配置和替代方案，请参阅 [EF Core 文档](https://docs.abp.io/en/abp/latest/Entity-Framework-Core)的加载相关实体部分。
:::



`Issue.AddComment` 获取用户 ID 和评论文本，实现必要的业务规则并将评论添加到问题的 Comments 集合中。

最后，我们使用 `_issueRepository.UpdateAsync` 来保存对数据库的更改。



::: tip
EF Core 具有**更改跟踪**功能。因此，您实际上不需要调用 `_issueRepository.UpdateAsync`。由于 ABP 的工作单元系统会在方法结束时自动调用 `DbContext.SaveChanges()`，它将自动保存。但是，对于 MongoDB，您需要显式更新更改的实体。

因此，如果您想编写与数据库提供程序独立无关的代码，则应始终为更改的实体调用 `UpdateAsync` 方法。

:::



### 事务边界

聚合通常被视为事务边界。如果用例使用单个聚合，读取它并将其保存为单个单元，则对聚合对象所做的所有更改都将作为原子操作一起保存，并且您不需要显式数据库事务。

但是，在现实生活中，您可能需要在单个用例中更改**多个聚合实例**，并且需要使用数据库事务来确保**原子更新**和**数据一致性**。因此，ABP 框架针对用例（应用程序服务方法边界）使用显式数据库事务。有关更多信息，请参阅[工作单元](https://docs.abp.io/en/abp/latest/Unit-Of-Work)文档。



### 可序列化

聚合（带有根实体和子集合）应该作为一个单元在网络上可序列化和传输。例如，MongoDB 在保存到数据库时将聚合序列化为 JSON 文档，在从数据库读取时从 JSON 反序列化。



::: tip
当您使用关系数据库和 ORM 时，此要求不是必需的。然而，它是领域驱动设计的一个重要实践。
:::



以下规则已经带来了可序列化性。



### 聚合/聚合根规则和最佳实践

以下规则确保实施上述原则。



### 仅通过 ID 引用其他聚合

第一条规则说一个聚合应该只通过它们的 Id 引用其他聚合。这意味着您不能将导航属性添加到其他聚合。

- 该规则使得实现可序列化原则成为可能。
- 它还可以防止不同的聚合相互操纵以及将聚合的业务逻辑相互泄露。

在下面的示例中，您会看到两个聚合根，`GitRepository` 和 `Issue`；



![image-20210719223641022](./04_Implementation_The_Building_Blocks.assets/image-20210719223641022.png)

- `GitRepository` 不应该有`Issue`的集合，因为它们是不同的聚合。
- `Issue`不应具有相关 `GitRepository` 的导航属性，因为它是不同的聚合。
- `Issue` 可以有 `RepositoryId`（作为 Guid）。

所以，当你有一个`Issue`并且需要有与这个问题相关的 `GitRepository` 时，你需要通过 `RepositoryId` 从数据库中显式查询它。



### 对于 EF Core 和关系数据库

在MongoDB中，自然不适合拥有这样的导航属性/集合。如果这样做，您会在源聚合的数据库集合中找到目标聚合对象的副本，因为它在保存时被序列化为 JSON。

但是，EF Core 和关系数据库开发人员可能会发现此限制性规则是不必要的，因为 EF Core 可以在数据库读取和写入时处理它。我们认为这是一项重要规则，有助于降低领域的复杂性，防止潜在问题，我们强烈建议实施此规则。但是，如果您认为忽略此规则是可行的，请参阅上面*关于数据库独立性原则*的讨论部分。



### 保持聚合小规模

一个好的做法是保持聚合简单和小规模。这是因为聚合将作为单个单元加载和保存，并且读取/写入大对象存在性能问题。请参阅下面的示例：

![image-20210719224706910](./04_Implementation_The_Building_Blocks.assets/image-20210719224706910.png)

角色聚合具有一组 `UserRole` 值对象，用于跟踪为此角色分配的用户。请注意， `UserRole` 不是另一个聚合，并且它对于规则 *仅按 ID 引用其他聚合* 来说不是问题。然而，这在实践中是一个问题。在现实生活场景中，一个角色可能会分配给数千（甚至数百万）个用户，每当您从数据库中查询一个角色时，加载数千个项目是一个重大的性能问题（请记住：聚合由它们的子集合作为一个单元加载）。

另一方面， `User` 可能有这样一个 `Roles` 集合，因为用户实际上没有太多角色，并且在使用 User Aggregate 时拥有一个角色列表会很有用。

如果仔细想想，当使用非关系型数据库（如 MongoDB）时 Role 和 User 都有关系列表时，还有一个问题。在这种情况下，相同的信息会在不同的集合中重复，并且很难保持数据的一致性（每当您将项目添加到 `User.Roles` 时，您也需要将其添加到 `Role.Users`）。

因此，请根据以下考虑确定聚合边界和大小；

- 一起使用的对象。
- 查询（加载/保存）性能和内存消耗。
- 数据完整性、有效性和一致性。

在实践中；

- 大多数聚合根**不会有子集合**。
- 在大多数情况下，一个子集合中的项目不应超过 **100-150** 个。如果您认为一个集合可能包含更多项，请不要将集合定义为聚合的一部分，而是考虑为集合内的实体提取另一个聚合根。



### 聚合根/实体上的主键

- 聚合根通常具有单个 `Id` 属性作为其标识符（Primark Key：PK）。我们更喜欢 `Guid` 作为聚合根实体的 PK（请参阅 [Guid 生成文档](https://docs.abp.io/en/abp/latest/Guid-Generation)以了解原因）。
- 聚合中的实体（不是聚合根）可以使用复合主键

例如，请参阅下面的聚合根和实体：



![image-20210719225910436](./04_Implementation_The_Building_Blocks.assets/image-20210719225910436.png)

- `Organization` 有一个 `Guid` 标识符 (`Id`)。
- `OrganizationUser` 是一个 `Organization` 的子集合，并且有一个由 `OrganizationId` 和 `UserId` 组成的复合主键。

这并不意味着子集合实体应该始终具有复合 PK。需要时，它们可能具有单个 Id 属性。



::: tip
复合 PK 实际上是关系数据库的一个概念，因为子集合实体有自己的表，需要一个 PK。另一方面，例如，在 MongoDB 中，您根本不需要为子集合实体定义 PK，因为它们作为聚合根的一部分存储。
:::



### 聚合根/实体的构造函数

构造函数位于实体生命周期开始的地方。设计良好的构造函数有一些责任：

- 获取**所需的实体属性**作为参数以**创建有效实体**。应该强制仅传递必需的参数，并且可能会获取非必需的属性作为可选参数。
- 检查参数的**有效性**。
- 初始化**子集合**。

**示例`Issue`（聚合根）构造函数**

```csharp
using System;
using System.Collections.Generic;
using System.Collections.ObjectModel;
using Volo.Abp;
using Volo.Abp.Domain.Entities;

namespace IssueTracking.Issues
{
    public class Issue : AggregateRoot<Guid>
    {
        public Guid RepositoryId { get; set; }
        public string Title { get; set; }
        public string Text { get; set; }
        public Guid? AssignedUserId { get; set; }
        public bool IsClosed { get; set; }
        public IssueCloseReason? CloseReason { get; set; } //枚举

        public ICollection<IssueLabel> Labels { get; set; }

        public Issue(
            Guid id,
            Guid repositoryId,
            string title,
            string text = null,
            Guid? assignedUserId = null
            ) : base(id)
        {
            RepositoryId = repositoryId;
            Title = Check.NotNullOrWhiteSpace(title, nameof(title));

            Text = text;
            AssignedUserId = assignedUserId;

            Labels = new Collection<IssueLabel>();
        }
        private Issue() {/*反序列化和ORMs需要*/}
    }
}
```



- `Issue`类通过在其构造函数中获取最少必需的属性作为参数来正确**强制创建有效实体**。
- 构造函数验证输入（如果给定值为空，`Check.NotNullOrWhiteSpace(...)` 抛出 `ArgumentException`）。
- 它会**初始化子集合**，因此当您在创建`Issue`后尝试使用 `Labels` 集合时，不会出现空引用异常。
- 构造函数也**接受** `id` 并传递给基类。我们不会在构造函数中生成 `Guid`，以便能够将此责任委托给另一个服务（请参阅 [生成Guid](https://docs.abp.io/en/abp/latest/Guid-Generation)）。
- ORM 需要私有的**空构造函数**。我们将其设为`private`以防止在我们自己的代码中意外使用它。




::: tip
请参阅[实体](https://docs.abp.io/en/abp/latest/Entities)文档以了解有关使用 ABP 框架创建实体的更多信息。
:::



### 实体属性访问器和方法

你可能觉得上面的例子很奇怪！例如，我们强制在构造函数中传递一个非空的 `Title`。但是，开发人员随后可以在没有任何控制的情况下将 `Title` 属性设置为 `null`。这是因为上面的示例代码只关注构造函数。

如果我们使用公共 设置器声明所有属性（如上面的示例 `Issue` 类），我们无法强制实体在其生命周期中的有效性和完整性。所以;

- 当您在设置该属性时需要执行任何逻辑时，请为该属性使用**私有设置器**。
- 定义公共方法来操作这些属性。



**示例：以受控方式更改属性的方法**

```csharp
using System;
using Volo.Abp;
using Volo.Abp.Domain.Entities;

namespace IssueTracking.Issues
{    
    public class Issue : AggregateRoot<Guid>
    {
        public Guid RepositoryId { get; private set; } //永不可变
        public string Title { get; private set; } //需要验证
        public string Text { get; set; } //无需验证
        public Guid? AssignedUserId { get; set; } //无需验证
        public bool IsClosed { get; private set; } //需要和 CloseReason 一起更改
        public IssueCloseReason? CloseReason { get; private set; } //需要和 IsClosed 一起更改

        //...

        public void SetTitle(string title)
        {
            Title = Check.NotNullOrWhiteSpace(title, nameof(title));
        }

        public void Close(IssueCloseReason reason)
        {
            IsClosed = true;
            CloseReason = reason;
        }

        public void ReOpen()
        {
            IsClosed = false;
            CloseReason = null;
        }
    }
}
```



- `RepositoryId` 设置器设为私有，创建`Issue`后无法更改它，因为这是我们在此领域中想要的：问题不能移动到另一个存储库。
- 如果您想稍后以受控方式更改它，则将`Title` 设置器设为私有并创建 SetTitle 方法。
- `Text` 和 `AssignedUserId` 具有公共设置器，因为对它们没有限制。它们可以为 null 或任何其他值。我们认为没有必要定义单独的方法来设置它们。如果我们稍后需要，我们可以添加方法并使 设置器成为私有。领域层中的重大更改不是问题，因为领域层是一个内部项目，它不会暴露给客户端。
- `IsClosed` 和 `IssueCloseReason` 是对属性。定义 `Close` 和 `ReOpen` 方法来一起改变它们。通过这种方式，我们可以防止无故关闭问题。



### 实体中的业务逻辑和异常

当您在实体中实现验证和业务逻辑时，您经常需要管理异常情况。在这些情况下；

- 创建**领域特定的异常**
- 必要时在实体方法中**抛出这些异常**。



**示例：**

```csharp
public class Issue : AggregateRoot<Guid>
{
    //..

    public bool IsLocked { get; private set; }
    public bool IsClosed { get; private set; }
    public IssueCloseReason? CloseReason { get; private set; }

    public void Close(IssueCloseReason reason)
    {
        IsClose = true;
        CloseReason = reason;
    }

    public void ReOpen()
    {
        if (IsLocked)
        {
            throw new IssueStateException("Can not open a locked issue! Unlock it first.");
        }
        IsClosed = false;
        CloseReason = null;
    }

    public void Lock()
    {
        if (!IsClosed)
        {
            throw new IssueStateException("Can not open a locked issue! Unlock it first.");
        }

        IsLocked = true;
    }

    public void Unlock()
    {
        IsLocked = false;
    }
}
```

这里有两个业务规则；

- 无法重新打开锁定的问题。
- 您无法锁定未解决的问题。

在这些情况下，`Issue`类会抛出一个 `IssueStateException` 以强制执行业务规则：

```csharp
using System;

namespace IssueTracking.Issues
{
    public class IssueStateException : Exception
    {
        public IssueStateException(string message)
        : base(message)
        {
        }
    }
}
```

抛出这样的异常有两个潜在的问题；

1. 如果出现此类异常，**最终用户**是否应该看到异常（错误）消息？如果是这样，您如何**本地化**异常消息？您不能使用[本地化系统](https://docs.abp.io/en/abp/latest/Localization)，因为您不能在实体中注入和使用 `IStringLocalizer`。
2. 对于 Web 应用程序或 HTTP API，应该向客户端返回什么 **HTTP 状态码**？

ABP 的[异常处理系统](https://docs.abp.io/en/abp/latest/Exception-Handling)解决了这些和类似的问题。



**示例：使用代码抛出业务异常**

```csharp
using Volo.Abp;

namespace IssuTracking.Issues
{
    public class IssueStateException : BuisinessException
    {
        public IssueStateExcetipn(string code)
            : base(code)
        {
        }
    }
}
```



- `IssueStateException` 类继承了 `BusinessException` 类。对于从 `BusinessException` 派生的异常，ABP 默认返回 403（禁止）HTTP 状态代码（而不是 500 - 内部服务器错误）。
- 该`code`用作本地化资源文件中的关键字，用于查找本地化消息。



现在，我们可以更改 `ReOpen` 方法，如下所示：

```csharp
public void ReOpen()
{
    if (IsLocked)
    {
        throw new IssueStateException("IssueTracking:CanNotOpenLockedIssue");
    }
    IsClosed = false;
    CloseReason = null;
}
```



::: tip
使用常量而不是魔术字符串。
:::



并向本地化资源添加一个条目，如下所示：

```json
"IssueTracking:CanNotOpenLockedIssue": "Can not open a locked issue! Unlock it first."
```

- 当您抛出异常时，ABP 会自动使用此本地化消息（基于当前语言）向最终用户显示。
- 异常代码（这里是IssueTracking:CanNotOpenLockedIssue）也被发送到客户端，所以它可以以编程方式处理错误情况。



::: tip
对于此示例，您可以直接抛出 `BusinessException` 而不是定义专门的 `IssueStateException`。结果将是相同的。有关所有详细信息，请参阅[异常处理文档](https://docs.abp.io/en/abp/latest/Exception-Handling)。
:::



### 实体中的业务逻辑依赖外部服务

当业务逻辑仅使用该实体的属性时，在实体方法中实现业务规则很简单。如果业务逻辑需要**查询数据库**或使用任何应该从[依赖注入](https://docs.abp.io/en/abp/latest/Dependency-Injection)系统解析的**外部服务**怎么办。记住;实体不能注入服务！

有两种常见的方式来实现这样的业务逻辑：

- 在实体方法上实现业务逻辑并**获取外部依赖项作为方法的参数**。
- 创建**领域服务**。

领域服务将在后面解释。但是，现在让我们看看它是如何在实体类中实现的。

**示例：业务规则：不能同时为用户分配超过 3 个未解决的问题**

```csharp
public class Issue : AggregateRoot<Guid>
{
    //...
    public Guid? AssignedUserId { get; private set; }
    public async Task AssignToAsync(AppUser user, IUserIssueService userIssueService)
    {
        var openIssueCount = await userIssueService.GetOpenIssueCountAsync(user.Id);

        if (openIssueCount >= 3)
        {
            throw new BusinessException("IssueTracking:ConcurrentOpenIssueLimit");
        }

        AssignedUserId = user.Id;
    }

    public void CleanAssignment()
    {
        AssignedUserId = null;
    }
}
```

- `AssignedUserId` 属性设置器设为私有。因此，只能使用 `AssignToAsync` 和 `CleanAssignment` 方法更改它。
- `AssignToAsync` 获取一个 `AppUser` 实体。实际上，它只使用 `user.Id`，因此您可以获得 `Guid` 值，例如 `userId`。但是，这种方式可以确保 `Guid` 值是现有用户的 `Id` 而不是随机的 `Guid` 值。
- `IUserIssueService` 是一种任意服务，用于获取用户的未解决问题计数。代码部分（调用`AssignToAsync`）负责解析IUserIssueService 并传递到这里。
- 如果业务规则不满足，`AssignToAsync` 会抛出异常。
- 最后，如果一切正确，则设置 `AssignedUserId` 属性。



当您想将问题分配给用户时，此方法完美地保证了应用业务逻辑。但是，它有一些问题；

- 它使实体类依**赖于外部服务**这将使实体变得**复杂**。
- 它让实体变得难以使用。使用实体的代码现在需要注入`IUserIssueService` 并传递给 `AssignToAsync` 方法。


实现此业务逻辑的另一种方法是引入**领域服务**，稍后将对此进行解释。



### 存储库

[存储库](https://docs.abp.io/en/abp/latest/Repositories)是一个类似集合的接口，领域和应用层使用它来访问数据持久性系统（数据库）以读取和写入业务对象，通常是聚合。

常见的存储库原则是；

- 在**领域层定义一个存储库接口**（因为它将在领域和应用层中使用），**在基础设施层实现**（启动模板中*EntityFrameworkCore*项目）。
- 不要在存储库中包含业务逻辑。
- 存储库接口应该独立于**数据库提供者/ORM**。例如，不要从存储库方法返回 `DbSet`。 `DbSet` 是 EF Core 提供的对象。
- **为聚合根创建存储库**，而不是为所有实体创建存储库。因为，应该通过聚合根访问子集合实体（聚合的）。



**不要在存储库中包含领域逻辑**

虽然这条规则一开始看起来很明显，但很容易将业务逻辑泄漏到存储库中。

**示例：从存储库中获取非活动问题**

```csharp
using System;
using System.Collections.Generic;
using System.Threading.Tasks;
using Volo.Abp.Domain.Repositories;

namespace IssueTracking.Issues
{
	public interface IIssueRepository : IRepository<Issue, Guid>
    {
    	Task<List<Issue>> GetInActiveIssuesAsync();
    }
}
```

`IIssueRepository` 通过添加 `GetInActiveIssuesAsync` 方法扩展了标准 `IRepository<...>` 接口。这个存储库与下面这样一个问题类一起工作：

```csharp
public class Issue : AggregateRoot<Guid>, IHasCreationTime
{
    public bool IsClosed { get; private set; }
    public Guid? AssignedUserId { get; private set; }
    public DateTime CreationTime { get; private set; }
    public DateTime? LastCommentTime { get; private set; }
    //...
}
```

（代码只显示了我们在这个例子中需要的属性）

规则说存储库不应该知道业务规则。这里的问题是“**什么是非活动问题**？它是一个业务规则定义吗？”

让我们看一下实现来理解它：

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using IssueTracking.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore;
using Volo.Abp.Domain.Repositories.EntityFrameworkCore;
using Volo.Abp.EntityFrameworkCore;

namespace IssueTracking.Issues
{
    public class EfCoreIssueRepository :
        EfCoreRepository<IssueTrackingDbContext, Issue, Guid>,
        IIssueRepository
    {
        public EfCoreIssueRepository(
            IDbContextProvider<IssueTrackingDbContext> dbContextProvider)
            : base(dbContextProvider)
        {
        }

        public async Task<List<Issue>> GetInActiveIssuesAsync()
        {
            var daysAgo30 = DateTime.Now.Subtract(TimeSpan.FromDays(30));

            var dbSet = await GetDbSetAsync();
            return await dbSet.Where(i =>

                //Open
                !i.IsClosed &&

                //Assigned to nobody
                i.AssignedUserId == null &&

                //Created 30+ days ago
                i.CreationTime < daysAgo30 &&

                //No comment or the last comment was 30+ days ago
                (i.LastCommentTime == null || i.LastCommentTime < daysAgo30)
            ).ToListAsync();
        }
    }
}
```

（使用 EF Core 进行实现。请参阅 [EF Core 集成文档](https://docs.abp.io/en/abp/latest/Entity-Framework-Core)以了解如何使用 EF Core 创建自定义存储库。）

当我们检查 `GetInActiveIssuesAsync` 实现时，我们看到一个定义非活动问题的业务规则：问题应该是开放的，没有分配给任何人，在 30 天以前创建并且在过去 30 天内没有评论。

这是隐藏在存储库方法中的业务规则的隐式定义。当我们需要重用这个业务逻辑时，就会出现问题。

例如，假设我们要在 `Issue` 实体上添加一个 `bool IsInActive()` 方法。这样，当我们有一个问题实体时，我们可以检查活跃度。

让我们看看实现：

```csharp
public class Issue : AggregateRoot<Guid>, IHasCreationTime
{
    public bool IsClosed { get; private set; }
    public Guid? AssignedUserId { get; private set; }
    public DateTime CreationTime { get; private set; }
    public DateTime? LastCommentTime { get; private set; }
    //...

    public bool IsInActive()
    {
        var daysAgo30 = DateTime.Now.Subtract(TimeSpan.FromDays(30));
        return
            //Open
            !IsClosed &&

            //Assigned to nobody
            AssignedUserId == null &&

            //Created 30+ days ago
            CreationTime < daysAgo30 &&

            //No comment or the last comment was 30+ days ago
            (LastCommentTime == null || LastCommentTime < daysAgo30);
    }
}
```

我们不得不复制/粘贴/修改代码。如果活跃度的定义发生变化怎么办？我们不应该忘记更新这两个地方。这是业务逻辑的重复，非常危险。

这个问题的一个很好的解决方案是*规约模式*！



### 规约

[规约](https://docs.abp.io/en/abp/latest/Specifications)是一个**命名的**、**可重用的**、**可组合的**和**可测试**的类，用于根据业务规则过滤领域对象。

ABP 框架提供了必要的基础设施来轻松创建规约类并在您的应用程序代码中使用它们。让我们将非活动问题过滤器实现为规约类：

```csharp
using System;
using System.Linq.Expressions;
using Volo.Abp.Specifications;

namespace IssueTracking.Issues
{
    public class InActiveIssueSpecification : Specification<Issue>
    {
        public override Expression<Func<Issue, bool>> ToExpression()
        {
            var daysAgo30 = DateTime.Now.Subtract(TimeSpan.FromDays(30));
            return i =>
                //Open
                !i.IsClosed &&

                //Assigned to nobody
                i.AssignedUserId == null &&

                //Created 30+ days ago
                i.CreationTime < daysAgo30 &&

                //No comment or the last comment was 30+ days ago
                (i.LastCommentTime == null || i.LastCommentTime < daysAgo30);
        }
    }
}
```

`Specification<T>` 基类通过定义表达式简化了创建规约类的过程。仅仅从存储库中将表达式移到此处。

现在，我们可以重用 `Issue` 实体和 `EfCoreIssueRepository` 类中的 `InActiveIssueSpecification`。



**在实体中使用规约**

规约类提供了一个 `IsSatisfiedBy` 方法，如果给定的对象（实体）满足规约，则该方法返回 `true`。

```csharp
public class Issue : AggregateRoot<Guid>, IHasCreationTime
{
    public bool IsClosed { get; private set; }
    public Guid? AssignedUserId { get; private set; }
    public DateTime CreationTime { get; private set; }
    public DateTime? LastCommentTime { get; private set; }
    //...

    public bool IsInActive()
    {
        return new InActiveIssueSpecification().IsSatisfiedBy(this);
    }
}
```

仅仅创建了 `InActiveIssueSpecification` 的一个新实例，并使用其 `IsSatisfiedBy` 方法重新使用了规约定义的表达式。



**在存储库中使用规约**

首先，从存储库接口开始：

```csharp
public interface IIssueRepository : IRepository<Issue, Guid>
{
	Task<List<Issue>> GetIssuesAsync(ISpecification<Issue> spec);
}
```

通过获取规约对象，将 `GetInActiveIssuesAsync` 重命名为简单的 `GetIssuesAsync`。由于**规约（过滤器）已移出存储库**，我们不再需要创建不同的方法来获取具有不同条件的问题（例如 `GetAssignedIssues(...)`、`GetLockedIssues(...)` 等）



更新后的存储库的实现如下：

```csharp
public class EfCoreIssueRepository :
    EfCoreRepository<IssueTrackingDbContext, Issue, Guid>,
    IIssueRepository
{
    public EfCoreIssueRepository(
        IDbContextProvider<IssueTrackingDbContext> dbContextProvider)
        : base(dbContextProvider)
    {
    }

    public async Task<List<Issue>> GetIssuesAsync(ISpecification<Issue> spec)
    {
        var dbSet = await GetDbSetAsync();
        return await dbSet
            .Where(spec.ToExpression())
            .ToListAsync();
    }
}
```

由于 `ToExpression()` 方法返回一个表达式，因此可以直接传递给 `Where` 方法来过滤实体。

最后，我们可以将任何规约实例传递给 `GetIssuesAsync` 方法：

```csharp
public class IssueAppService : ApplicationService, IIssueAppService
{
    private readonly IIssueRepository _issueRepository;

    public IssueAppService(IIssueRepository issueRepository)
    {
        _issueRepository = issueRepository;
    }

    public async Task DoItAsync()
    {
        var issues = await _issueRepository.GetIssuesAsync(
            new InActiveIssueSpecification()
        );
    }
}
```



**使用默认存储库**

实际上，您不必创建自定义存储库即可使用规约。标准的 `IRepository` 已经扩展了 `IQueryable`，所以你可以在它上面使用标准的 LINQ 扩展方法：

```csharp
public class IssueAppService : ApplicationService, IIssueAppService
{
    private readonly IRepository<Issue, Guid> _issueRepository;

    public IssueAppService(IRepository<Issue, Guid> issueRepository)
    {
        _issueRepository = issueRepository;
    }

    public async Task DoItAsync()
    {
        var queryable = await _issueRepository.GetQueryableAsync();
        var issues = AsyncExecuter.ToListAsync(
            queryable.Where(new InActiveIssueSpecification())
        );
    }
}
```

`AsyncExecuter` 是 ABP 框架提供的实用程序，用于在不依赖于 EF Core NuGet 包的情况下使用异步 LINQ 扩展方法（如此处的 `ToListAsync`）。有关更多信息，请参阅[存储库文档](https://docs.abp.io/en/abp/latest/Repositories)。



**组合规约**

规约的一个强大方面是它们是可组合的。假设我们有另一个规约，仅当`Issue`在里程碑中时才返回 `true`：

```csharp
public class MilestoneSpecification : Specification<Issue>
{
    public Guid MilestoneId { get; }

    public MilestoneSpecification(Guid milestoneId)
    {
        MilestoneId = milestoneId;
    }

    public override Expression<Func<Issue, bool>> ToExpression()
    {
        return i => i.MilestoneId == MilestoneId;
    }
}
```

该规约是有参数的，这与 `InActiveIssueSpecification` 有所不同。我们可以组合这两个规约来获取特定里程碑中的非活动问题列表：

```csharp
public class IssueAppService : ApplicationService, IIssueAppService
{
    private readonly IRepository<Issue, Guid> _issueRepository;

    public IssueAppService(IRepository<Issue, Guid> issueRepository)
    {
        _issueRepository = issueRepository;
    }

    public async Task DoItAsync(Guid milestoneId)
    {
        var queryable = await _issueRepository.GetQueryableAsync();
        var issues = AsyncExecuter.ToListAsync(
            queryable
                .Where(
                    new InActiveIssueSpecification()
                        .And(new MilestoneSpecification(milestoneId))
                        .ToExpression()
                )
        );
    }
}
```

上面的示例使用 `And` 扩展方法来组合规约。还有更多的组合方法可用，例如 `Or(...)` 和 `AndNot(...)`。



::: tip
有关 ABP 框架提供的规约基础结构的更多详细信息，请参阅[规约文档]()。
:::



## 领域服务

领域服务实现领域逻辑，其中它；

- 依赖于**服务**和**存储库**。
- 需要使用**多个聚合**，因此逻辑不能正确地适合任何聚合。

领域服务与领域对象一起工作。它们的方法可以**获取和返回实体、值对象、原始类型等**。但是，**它们不获取/返回 DTOs**。 DTOs 是应用层的一部分。



**示例：将问题分配给用户**

回想一下，我们之前如何在`Issue`实体中实现问题分配的：

```csharp
public class Issue : AggregateRoot<Guid>
{
    //...
    public Guid? AssignedUserId { get; private set; }
    public async Task AssignToAsync(AppUser user, IUserIssueService userIssueService)
    {
        var openIssueCount = await userIssueService.GetOpenIssueCountAsync(user.Id);

        if (openIssueCount >= 3)
        {
            throw new BusinessException("IssueTracking:ConcurrentOpenIssueLimit");
        }

        AssignedUserId = user.Id;
    }

    public void CleanAssignment()
    {
        AssignedUserId = null;
    }
}
```

在这里，我们将把这个逻辑移到领域服务中。

首先，更改`Issue`类：

```csharp
public class Issue : AggregateRoot<Guid>
{
    //...
    public Guid? AssignedUserId { get; internal set; }
}
```

- 删除了与分配相关的方法。
- 将 `AssignUserId` 属性设置器从`private`更改为
  `internal`，以允许从领域服务设置它。

下一步是创建一个名为 `IssueManager` 的领域服务，它具有 `AssignToAsync` 以将给定的问题分配给给定的用户。

```csharp
public class IssueManager : DomainService
{
    private readonly IRepository<Issue, Guid> _issueRepository;

    public IssueManager(IRepository<Issue, Guid> issueRepository)
    {
        _issueRepository = issueRepository;
    }

    public async Task AssignToAsync(Issue issue, AppUser user)
    {
        var openIssueCount = await _issueRepository.CountAsync(
            i => i.AssingedUserId == user.Id && !i.IsClosed
        );

        if (openIssueCount >= 3)
        {
            throw new BusinessException("IssueTracking:ConcurrentOpenIssueLimit");
        }

        issue.AssignedUserId = user.Id;
    }
}
```

`IssueManager` 可以注入任何服务依赖项并用于查询用户的未解决问题计数。



::: tip
我们更喜欢并建议对领域服务使用 `Manager` 后缀。
:::



这种设计的唯一问题是 `Issue.AssignedUserId` 现在是开放的，可以在类的外部设置。但是，它不是`public`。它是`internal`，并且只能在同一个程序集中进行更改，此示例解决方案的 `IssueTracking.Domain` 项目。我们认为这是合理的；

- 开发⼈员已经清楚领域层的开发规则，他们会使`⽤IssueManager`来执⾏业务逻辑。
- 应⽤层开发⼈员只能使用`IssueManager`，因为他们⽆法直接修改实体属性。

虽然两种方法之间存在权衡，但当业务逻辑需要使用外部服务时，我们更喜欢创建领域服务。



::: tip
如果您没有充分的理由，我们认为没有必要为领域服务创建接口（如 `IssueManager` 的 `IIssueManager`）。
:::



## 应用服务

[应用程序](https://docs.abp.io/en/abp/latest/Application-Services)服务是实现应用程序**用例**的无状态服务。应用程序服务通常获取并返回 DTOs。它由表示层使用。它**使用和协调领域对象**（实体、存储库等）来实现用例。



应用服务的通用原则是；

- 实现特定于当前用例的**应用层逻辑**。不要在应用服务内部实现核心领域逻辑。我们将介绍它和领域层逻辑的差异。
- **永远不要**为应用程序服务方法**获取或返回实体**。这打破了领域层的封装。应始终获取和返回 DTOs。



**示例：将问题分配给用户**

```csharp
using System;
using System.Threading.Tasks;
using IssueTracking.Users;
using Microsoft.AspNetCore.Authorization;
using Volo.Abp.Application.Services;
using Volo.Abp.Domain.Repositories;

namespace IssueTracking.Issues
{
    public class IssueAppService : ApplicationService, IIssueAppService
    {
        private readonly IssueManager _issueManager;
        private readonly IRepository<Issue, Guid> _issueRepository;
        private readonly IRepository<AppUser, Guid> _userRepository;

        public IssueAppService(
            IssueManager issueManager,
            IRepository<Issue, Guid> issueRepository,
            IRepository<AppUser, Guid> userRepository
        )
        {
            _issueManager = issueManager;
            _issueRepository = issueRepository;
            _userRepository = userRepository;
        }

        [Authorize]
        public async Task AssignAsync(IssueAssignDto input)
        {
            var issue = await _issueRepository.GetAsync(input.IssueId);
            var user = await _userRepository.GetAsync(inpu.UserId);

            await _issueManager.AssignToAsync(issue, user);

            await _issueRepository.UpdateAsync(issue);
        }
    }
}
```

一个应用服务方法通常具有三个步骤，这里实现了这些步骤；

1. 从数据库中获取相关领域对象以实现用例。
2. 使用领域对象（域服务、实体等）来执行实际操作。
3. 更新数据库中已更改的实体。

本例中的 `IssueAssignDto` 是一个简单的 DTO 类：



::: tip
如果您使用的是 EF Core，则不需要最后一步*更新*操作，因为它具有更改跟踪系统。如果您想利用此 EF Core 功能，请参阅上面关于*数据库独立原则*的讨论部分。
:::



本例中的 `IssueAssignDto` 是一个简单的 DTO 类：

```csharp
using System;

namespace IssueTracking.Issues
{
    public class IssueAssignDto
    {
        public Guid IssueId{get;set;}
        public Guid UserId{get;set;}
    }
}
```



## 数据传输对象

[DTO](https://docs.abp.io/en/abp/latest/Data-Transfer-Objects) 是一个简单的对象，用于在应用程序层和表示层之间传输状态（数据）。因此，应用程序服务方法获取并返回 DTOs。



**通用 DTO 原则和最佳实践**

- 就其性质而言，DTO **应该是可序列化的**。因为，大多数时候它是通过网络传输的。因此，它应该有一个**无参数（空）构造函数**。
- 不应包含任何**业务逻辑**。
- **永远**不要继承或引用**实体**。

**输入 DTOs**（传递给应用服务方法的那些）与**输出 DTOs**（那些从应用服务方法返回的）具有不同的性质。因此，他们将受到不同的对待。



### 输入 DTO 最佳实践

**不要为输入 DTO 定义未使用的属性**

仅定义用例**所需的属性**！否则，**客户端**使用应用服务方法**会造成混淆**。您当然可以定义**可选属性**，但是当客户端提供它们时，它们应该影响用例的工作方式。

这条规则乍一看似乎没有必要。谁会为方法定义未使用的参数（输入 DTO 属性）？但它会发生，尤其是当您尝试重用输入 DTOs 时。



**不要重复使用输入 DTOs**

为**每个用例定义一个专门的输入 DTO**（应用程序服务方法）。否则，在某些情况下不会使用某些属性，这违反了上面定义的规则：不要为输入 DTOs 定义未使用的属性。

有时，为两个用例重用同一个 DTO 类似乎很有吸引力，因为它们几乎相同。即使它们现在是一样的，到时候它们可能会变得不同，你会遇到同样的问题。**代码复制是比耦合用例更好的做法**。

另一种重用输入 DTOs 的方法是相互**继承** DTOs。虽然这在极少数情况下很有用，但大多数情况下它也会产生同样的问题。



**示例：用户应用服务**

```csharp
public interface IUserAppService : IApplicationService
{
    Task CreateAsync(UserDto input);
    Task UpdateAsync(UserDto input);
    Task ChangePasswordAsync(UserDto input);
}
```

`IUserAppService` 使用 `UserDto` 作为所有方法（用例）中的输入 DTO。 `UserDto` 定义如下：

```csharp
public class UserDto
{
    public Guid Id { get; set; }
    public string UserName { get; set; }
    public string Email { get; set; }
    public string Password { get; set; }
    public DateTime CreationTime { get; set; }
}
```

对于这个例子；

- `Id` 在 *Create* 方法中不被使用，因为它由服务器生成。
- `Password` 在 *Update* 方法中不使用，因为有修改密码有其他方法。
- `CreationTime` 从未被使用，因为我们不允许客户端发送创建时间。它应该在服务器中设置。

一个正确的实现可以是这样的：

```csharp
public interface IUserAppService : IApplicationService
{
    Task CreateAsync(UserCreationDto input);
    Task UpdateAsync(UserUpdateDto input);
    Task ChangePasswordAsync(UserChangePasswordDto input);
}
```

使用给定的输入 DTO 类：

```csharp
public class UserCreationDto
{
    public string UserName { get; set; }
    public string Email { get; set; }
    public string Password { get; set; }
}

public class UserUpdateDto
{
    public Guid Id { get; set; }
    public string UserName { get; set; }
    public string Email { get; set; }
}

public class UserChangePasswordDto
{
    public Guid Id { get; set; }
    public string Password { get; set; }
}
```

尽管编写了更多代码，但这是更易于维护的方法。

**例外情况：**此规则可能有一些例外：如果您总是希望**并行**开发两个方法，它们可能共享相同的输入 DTO（通过继承或直接重用）。例如，如果您有一个包含一些过滤器的报告页面，并且您有多个应用程序服务方法（如屏幕报告、excel 报告和 csv 报告方法）使用相同的过滤器但返回不同的结果，您可能希望重用相同的过滤器输入DTO 来**耦合这些用例**。因为，在本例中，无论何时更改过滤器，您都必须对所有方法进行必要的更改，以获得一致的报告系统。



**输入 DTO 验证逻辑**

- 仅在 DTO 内实施**简单验证**。使用数据注释验证属性或实现 `IValidatableObject` 进行简单验证。
- **不执行领域验证**。例如，不要尝试检查 DTOs 中的唯一用户名约束。



**示例：使用数据注释属性**

```csharp
using System.ComponentModel.DataAnnotations;

namespace IssueTracking.Users
{
    public class UserCreationDto
    {
        [Required]
        [StringLength(UserConsts.MaxUserNameLength)]
        public string UserName { get; set; }

        [Required]
        [EmailAddress]
        [StringLength(UserConsts.MaxEmailLength)]
        public string Email { get; set; }

        [Required]
        [StringLength(UserConsts.MaxEmailLength, MinimumLength = UserConsts.MinPasswordLength)]
        public string Password { get; set; }
    }
}
```

ABP 框架会自动验证输入 DTOs，抛出 `AbpValidationException` 并在输入无效的情况下向客户端返回 HTTP 状态 `400`。



::: tip
一些开发人员认为最好将验证规则和 DTOs 类分开。我们认为声明式（数据注释）方法实用且有用，不会导致任何设计问题。但是，如果您更喜欢其他方法，ABP 也支持 [FluentValidation](https://docs.abp.io/en/abp/latest/FluentValidation) 集成。

有关所有验证选项，请参阅[验证文档](https://docs.abp.io/en/abp/latest/Validation)。

:::



### 输出 DTO 最佳实践

- 保持输出 **DTO 计数最少**。尽可能重用（例外：不要重用输入 DTOs 作为输出 DTOs）。
- 输出 DTOs 可以包含比客户端代码中使用的**更多的属性**。
- 从 **Create** 和 **Update** 方法返回实体 DTO。

这些建议的主要目标是；

- 使客户端代码易于开发和扩展；
  - 在客户端处理**类似但不相同**的 DTOs 是有问题的。
  - UI/客户端会在将来需要其他属性是很常见的。返回实体的所有属性（通过考虑安全性和权限）使客户端代码易于改进，而无需接触后端代码。
  - 如果您向第 3 方客户开放 API，而您不知道每个客户的要求。
- 使服务端代码易于开发和扩展；
  - 您需要**了解和维护**的类较少。
  - 您可以重复使用 Entity->DTO **对象映射**代码。
  - 从不同的方法返回相同的类型使创建**新方法**变得容易和清晰。

**示例：从不同的方法返回不同的 DTOs**

```csharp
public interface IUserAppService : IApplicationService
{
    UserDto Get(Guid id);
    List<UserNameAndEmailDto> GetUserNameAndEmail(Guid id);
    List<string> GetRoles(Guid id);
    List<UserListDto> GetList();
    UserCreateResultDto Create(UserCreationDto input);
    UserUpdateResultDto Update(UserUpdateDto input);
}
```

*（我们没有使用异步方法来使示例更清晰，但是请在您的实际应用程序中使用异步方法！）*



上面的示例代码为每个方法返回不同的 DTOs 类型。您可以猜到，在查询数据、将实体映射到 DTOs 方面会有很多代码重复。

上面的 `IUserAppService` 服务可以简化：

```csharp
public interface IUserAppService : IApplicationService
{
    UserDto Get(Guid id);
    List<UserDto> GetList();
    UserDto Create(UserCreationDto input);
    UserDto Update(UserUpdateDto input);
}
```

使用同一个输出 DTO：

```csharp
public class UserDto
{
    public Guid Id { get; set; }
    public string UserName { get; set; }
    public string Email { get; set; }
    public DateTiem CreationTime { get; set; }
    public List<string> Roles { get; set; }
}
```

- 删除了 `GetUserNameAndEmail` 和 `GetRoles`，因为 `Get` 方法已经返回了必要的信息。
- `GetList` 现在返回与 `Get` 相同的结果。
- `Create` 和 `Update` 也返回相同的 `UserDto`。

如前所述，使用相同的 DTO 有很多优点。例如，考虑在 UI 上显示用户**数据列表**的场景。更新用户后，您可以获取返回值并**在 UI 上进行更新**。因此，您无需再次调用 `GetList`。这就是为什么我们建议将实体 DTO（此处为 `UserDto`）作为 `Create` 和 `Update` 操作的返回值返回的原因。



**讨论**

某些输出 DTO 建议可能并不适合所有场景。出于**性能**原因，可以忽略这些建议，尤其是在返回大**型数据集**或为自己的 UI 创建服务并且**并发请求过多**时。

在这些情况下，您可能希望**使用最少的信息创建专门的输出 DTOs**。上述建议特别适用于**维护代码库**比**可忽略的性能损失**更重要的应用程序。



**对象到对象映射**

当两个对象具有相同或相似的属性时，自动[对象到对象映射](https://docs.abp.io/en/abp/latest/Object-To-Object-Mapping)是一种将值从一个对象复制到另一个对象的有用方法。

DTO 和实体类通常具有相同/相似的属性，您通常需要从实体创建 DTO 对象。与手动映射相比，[ABP 的对象到对象映射系统](https://docs.abp.io/en/abp/latest/Object-To-Object-Mapping)与 [AutoMapper](http://automapper.org/) 集成使这些操作更加容易。

- 仅对**实体到输出DTO使用**自动对象映射。
- **输入DTO到实体**，**不适用**自动对象映射。

不应该使用输入 DTO 到实体自动映射有一些原因；

1. 实体类通常有一个**构造函数**，它接受参数并确保有效的对象创建。自动对象映射操作通常需要一个空的构造函数。
2. 大多数实体属性将具有**私有 setter**，您应该使用方法以受控方式更改这些属性。
3. 您通常需要**仔细验证和处理**用户/客户端输入，而不是盲目地映射到实体属性。

虽然其中一些问题可以通过映射配置解决（例如，AutoMapper 允许定义自定义映射规则），但它使您的业务代码**隐式/隐藏**并与基础设施**紧密耦合**。我们认为业务代码应该是明确的、清晰的、易于理解的。



有关本节中提出的建议的示例实现，请参阅下一章的*实体创建*部分。





::: warning
作者：Gerry Ge

出处：[实现领域驱动设计-实现：构建块](https://gerryge.com/blogs/2021/Implementing_Domain_Driven_Design/04_Implementation_The_Building_Blocks.html)

版权：本作品采用「[署名-非商业性使用-相同方式共享 4.0 国际](https://creativecommons.org/licenses/by-nc-sa/4.0/)」许可协议进行许可。

**转载请注明出处**
:::

