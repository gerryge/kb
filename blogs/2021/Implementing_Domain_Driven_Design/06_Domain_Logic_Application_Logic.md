---
title: 实现领域驱动设计-领域逻辑与应用逻辑
date: 2021-07-18
tags:
 - ABP
 - DDD
categories:
 - ABP
 - DDD
sidebar: 'auto'
sticky: 6
prev: ./05_Example_User_Cases
next: ./07_Reference_Books
---

本系列文章，翻译至[Implementing Domain Driven Design](https://abp.io/books/implementing-domain-driven-design)
<!-- more -->

# 领域逻辑与应用逻辑

如前所述，领域驱动设计中的业务逻辑分为两部分（层）：领域逻辑和应用逻辑：

![image-20210724231414579](./06_Domain_Logic_Application_Logic.assets/image-20210724231414579.png)



领域逻辑由系统的*核心领域规则*组成，而应用程序逻辑实现特定的应用程序*用例*。

虽然定义很明确，但实施起来可能并不容易。您可能不确定哪些代码应该放在应用层，哪些代码应该放在领域层。本节试图解释这些差异。



### 多应用层

当您的系统很大时，DDD 有助于**处理复杂性**。特别是，如果在**单个领域**中开发**多个应用程序**，那么**领域逻辑与应用程序逻辑的分离**就变得更加重要。

假设您正在构建一个具有多个应用程序的系统；

- 使用 ASP.NET Core MVC 构建的**公共网站应用程序**，用于向用户展示您的产品。这样的网站不需要身份验证即可查看产品。用户登录网站，仅当他们执行某些操作（例如将产品添加到购物车）时。
- 使用 Angular UI（使用 REST API）构建的**后台应用程序**。公司上班族用来管理系统的应用程序（如编辑产品说明）。
- 与公共网站相比，具有更简单 UI 的**移动应用程序**。它可以通过 REST API 或其他技术（如 TCP sockets）与服务器通信。



![image-20210724232249421](./06_Domain_Logic_Application_Logic.assets/image-20210724232249421.png)



每个应用程序都会有不同的**需求**、不同的**用例**（应用程序服务方法）、不同的 **DTOs**、不同的**验证**和**授权**规则……等等。

将所有这些逻辑混合到一个应用程序层中会使您的服务包含太多`if`条件，这些具有**复杂业务逻辑**的条件会使您的代码**更难开发、维护和测试**并导致潜在的错误。

如果您在一个领域中有多个应用程序；

- 为每个应用程序/客户端类型创建**单独的应用程序层**，并在这些单独的层中实现应用程序特定的业务逻辑。
- 使用**单个领域层**来共享核心域逻辑。

这样的设计使得区分领域逻辑和应用逻辑变得更加重要。

为了更清楚地了解实现，您可以为每种应用程序类型创建不同的项目 (.csproj)。例如;

- `IssueTracker.Admin.Application` 和 `IssueTracker.Admin.Application.Contracts` 项目作为后台（管理）应用程序。
- `IssueTracker.Public.Application` 和
  `IssueTracker.Public.Application.Contracts` 项目作为公共网站应用程序。
- `IssueTracker.Mobile.Application` 和
  `IssueTracker.Mobile.Application.Contracts` 项目作为移动应用程序。



### 示例

本节包含一些应用程序服务和领域服务示例，以讨论如何决定将业务逻辑放置在这些服务中。



#### **示例：在领域服务中创建新组织**

```csharp
public class OrganizationManager : DomainService
{
    private readonly IRepository<Organization> _organizationRepository;
    private readonly ICurrentUser _currentUser;
    private readonly IAuthorizationService _authorizationService;
    private readonly IEmailSender _emailSender;

    public OrganizationManager(
        IRepository<Organization> organizationRepository,
        ICurrentUser currentUser,
        IAuthorizationService authorizationService,
        IEmailSender emailSender
    )
    {
        _organizationRepository = organizationRepository;
        _currentUser = currentUser;
        _authorizationService = authorizationService;
        _emailSender = emailSender;
    }

    public async Task<Organization> CreateAsync(string name)
    {
        if (await _organizationRepository.AnyAsync(x => x.Name == name))
        {
            throw new BusinessException("IssueTracking:DuplicateOrganizationName");
        }
        
        await _authorizationService.CheckAsync("OrganizationCreationPermission");
        
        Logger.LogDebug($"Creating organization {name} by {_currentUser.UserName}");
        
        var organization = new Organization();
        
        await _emailSender.SendAsync(
        "systemadmin@issuetracking.com",
        "New Organization",
        "A new organization created with name: " + name
        );
        
        return organization;
    }
}
```

让我们一步一步看`CreateAsync`方法，讨论每一步代码部分是否应该在领域服务中；

- **正确：** 它首先检查**重复的组织名称**，并在这种情况下抛出异常。这与核心领域规则有关，我们绝不允许重名。
- **错误：** 领域服务不应执行**授权**。[授权](https://docs.abp.io/en/abp/latest/Authorization)应该在应用层完成。
- **错误：** 它发送了一封关于这个新组织创建的电子邮件。我们认为这也是一个用例特定的业务逻辑。您可能希望在不同的用例中创建不同类型的电子邮件，或者在某些情况下不需要发送电子邮件。



#### **示例：在应用程序服务中创建新组织**

```csharp
public class OrganizationAppService : ApplicationService
{
    private readonly OrganizationManager _organizationManager;
    private readonly IPaymentService _paymentService;
    private readonly IEmailSender _emailSender;

    public OrganizaitonAppService(
        OrganizationManager organizationManager,
        IPaymentService paymentService,
        IEmailSender emailSender
    )
    {
        _organizationManager = organizationManager;
        _paymentService = paymentService;
        _emailSender = emailSender;
    }

    [UnitOfWork]
    [Authorize("OrganizationCreationPermission")]
    public async Task<Organization> CreateAsync(CreateOrganizationDto input)
    {
        await _paymentService.ChargeAsync(
            CurrentUser.Id,
            GetOrganizationPrice()
        );
        
        var organization = await _organizationManager.CreateAsync(input.Name);
        
        await _organizationManager.InsertAsync(organization);
        
        await _emailSender.SendAsync(
        "systemadmin@issuetracking.com",
        "New Organization",
        "A new organization created with name: " + input.Name
        );
        
        return organization; // !!!
    }

    private double GetOrganizationPrice()
    {
        return 42;//从某地方获取...
    }
}
```

让我们一步一步看`CreateAsync`方法，讨论每一步代码部分是否应该在应用服务中；

- **正确：** 应用服务方法应该是工作单元（事务性）。 ABP 的[工作单元](https://docs.abp.io/en/abp/latest/Unit-Of-Work)系统使这个自动化（甚至不需要为应用程序服务添加 `[UnitOfWork]` 属性）。
- **正确：** [授权](https://docs.abp.io/en/abp/latest/Authorization)应该在应用层完成。在这里，它是通过使用 `[Authorize]` 属性来完成的。
- **正确：** 付款（一项基础设施服务）被调用为此操作收费（创建组织是我们业务中的一项付费服务）。
- **正确：** 应用程序服务方法负责保存对数据库的更改。
- **正确：** 我们可以向系统管理员发送电子邮件作为通知。
- **错误：** 不要从应用程序服务返回实体。而是返回 DTO。



### 讨论：为什么不把支付逻辑移到领域服务中呢？

您可能想知道为什么付款代码不在 `OrganizationManager` 中。这是一件**重要的事情**，我们永远不想**错过付款**。

然而，**重要的是不足以**将代码视为核心业务逻辑。我们可能还有其他用例，在这些用例中，我们不收取创建新组织的费用。例如；

- 管理员用户可以使用后台应用程序创建一个新组织，无需支付任何费用。
- 一个后台工作的数据导入/集成/同步系统也可能需要创建没有任何支付操作的组织。

如您所见，**付款不是创建有效组织的必要操作**。它是一个特定于用例的应用程序逻辑。



#### **示例：CRUD 操作**

```csharp
public class IssueAppService
{
    private readonly IssueManager _issueManager;

    public IssueAppService(IssueManager issueManager)
    {
        _issueManager = issueManager;
    }

    public async Task<IssueDto> GetAsync(Guid id)
    {
        return await _issueManager.GetAsync(id);
    }

    public async Task CreateAsync(IssueCreationDto input)
    {
        await _issueManager.CreateAsync(input);
    }

    public async Task UpdateAsync(UpdateIssueDto input)
    {
        await _issueManager.UpdateAsync(input);
    }

    public async Task DeleteAsync(Guid id)
    {
        await _issueManager.DeleteAsync(id);
    }
}
```

此应用程序服务本身什么都不做，而是将所有工作委托给领域服务。它甚至将 DTOs 传递给 `IssueManager`。

- **不要**仅为**没有任何领域逻辑**的简单 **CRUD** 操作创建领域服务方法。
- **切勿**将 **DTOs** 传递给领域服务或从域服务返回 **DTOs**。

应用服务可以直接使用存储库来查询、创建、更新或删除数据，除非在这些操作过程中需要执行一些领域逻辑。在这种情况下，创建领域服务方法，但仅限于那些真正需要的方法。



::: tip
不要仅仅认为将来可能需要它们（[YAGNI](https://en.wikipedia.org/wiki/You_aren%27t_gonna_need_it)）就创建这样的 CRUD 领域服务方法！在需要时执行并重构现有代码。由于应用层优雅地抽象了领域层，重构过程不会影响 UI 层和其他客户端。
:::





::: warning
作者：Gerry Ge

出处：[https://gerryge.com/blogs/2021/Implementing_Domain_Driven_Design/06_Domain_Logic_Application_Logic.html](https://gerryge.com/blogs/2021/Implementing_Domain_Driven_Design/06_Domain_Logic_Application_Logic.html)

版权：本作品采用「[署名-非商业性使用-相同方式共享 4.0 国际](https://creativecommons.org/licenses/by-nc-sa/4.0/)」许可协议进行许可。

**转载请注明出处**
:::
