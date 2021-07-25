---
title: 实现领域驱动设计-示例用例
date: 2021-07-18
tags:
 - ABP
 - DDD
categories:
 - ABP
 - DDD
sidebar: 'auto'
sticky: 5
prev: ./04_Implementation_The_Building_Blocks
next: ./06_Domain_Logic_Application_Logic
---

本系列文章，翻译至[Implementing Domain Driven Design](https://abp.io/books/implementing-domain-driven-design)
<!-- more -->

# 示例用例

本节将演示一些示例用例并讨论替代方案。



### 实体创建

从实体/聚合根类创建对象是该实体生命周期的第一步。 聚合 / 聚合根规则和最佳实践部分建议为实体类**创建一个主构造函数**，以保证**创建一个有效的实体**。因此，每当我们需要创建该实体的实例时，我们应该始终**使用该构造函数**。

请参阅下面的`Issue`聚合根类：

```csharp
public class Issue : AggregateRoot<Guid>
{
    public Guid RepositoryId { get; private set; }
    public string Title { get; private set; }
    public string Text { get; set; }
    public Guid? AssignedUserId { get; internal set; }

    public Issue(
        Guid id,
        Guid repositoryId,
        string title,
        string text = null
        ) : base(id)
    {
        RepositoryId = repositoryId;
        Title = Check.NotNullOrWhiteSpace(title, nameof(title));
        Text = text; //允许为空/null
    }
    private Issue() { } /* ORMs使用该空构造函数 */

    public void SetTitle(string title)
    {
        Title = Check.NotNullOrWhiteSpace(title, nameof(title));
    }

    //...
}
```

- 此类保证通过其构造函数创建有效实体。
- 如果您稍后需要更改 `Title`，则需要使用 `SetTitle` 方法，该方法继续保持 Title 处于有效状态。
- 如果要将此问题分配给用户，则需要使用 `IssueManager`（它在分配之前实现了一些业务规则 - 请参阅上面的*领域服务*部分）。
- `Text` 属性有一个公共的 setter，因为它也接受空值，并且在这个例子中没有任何验证规则。它在构造函数中也是可选的。

让我们看一个用于创建问题的应用程序服务方法：

```csharp
public class IssueAppService : ApplicationService.IIssueAppService
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

    public async Task<IssueDto> CreateAsync(IssueCreationDto input)
    {
        //创建有效实体
        var issue = new Issue(
            GuidGenerator.Create(),
            input.RepositoryId,
            input.Title,
            input.Text
        );

        //应用领域其他操作
        if (input.AssignedUserId.HasValue)
        {
            var user = await _userRepository.GetAsync(input.AssignedUserId.Value);
            await _issueManager.AssignToAsynce(issue, user);
        }

        //保存
        await _issueRepository.InsertAsync(issue);

        //返回代表新问题的DTO
        return ObjectMapper.Map<Issue, IssueDto>(issue);
    }
}
```

`CreateAsync` 方法；

- 使用`Issue`**构造函数**来创建一个有效的问题。它使用 [IGuidGenerator](https://docs.abp.io/en/abp/latest/Guid-Generation) 服务传递 Id。此处不使用自动对象映射。
- 如果客户端想要在创建对象时将此问题分配给用户，它会使用 `IssueManager` 来执行此操作，方法是允许 `IssueManager` 在此分配之前执行必要的检查。
- 将实体**保存**到数据库。
- 最后使用 `IObjectMapper` 返回一个 `IssueDto`，它是通过从新的 `Issue` 实体**映射**自动创建的。



### 在实体创建中应用领域规则

示例`Issue`实体没有关于实体创建的业务规则，除了构造函数中的一些简单验证。但是，可能存在实体创建应检查一些额外业务规则的情况。

例如，假设您**不希望**在已经存在具有**完全相同**`Title`的问题的情况下创建问题。在哪里执行这个规则？在**应用服务**中实现这个规则是**不合适**的，因为它是一个**核心业务（领域）规则**，应该总是被检查。

此规则应在**领域服务**中实现，在本例中为 `IssueManager`。因此，我们需要强制应用层始终使用 `IssueManager` 来创建新`Issue`。

首先，我们可以将 `Issue` 构造函数设为`internal`，而不是`public`：

```csharp
public class Issue : AggregateRoot<Guid>
{
    //...
    internal Issue(
        Guid id,
        Guid repositoryId,
        string title,
        string text = null
        ) : base(id)
    {
        RepositoryId = repositoryId;
        Title = Check.NotNullOrWhiteSpace(title, nameof(title));
        Text = text;//允许为空/null
    }
    //...
}
```

这会阻止应用程序服务直接使用构造函数，因此它们将使用 `IssueManager`。然后我们可以向 `IssueManager` 添加一个 `CreateAsync` 方法：

```csharp
using System;
using System.Threading.Tasks;
using Volo.Abp;
using Volo.Abp.Domain.Repositories;
using Volo.Abp.Domain.Services;

namespace IssueTracking.Issues
{
    public class IssueManager : DomainService
    {
        private readonly IRepository<Issue, Guid> _issueRepository;

        public IssueManager(IRepository<Issue, Guid> issueRepository)
        {
            _issueRepository = issueRepository;
        }

        public async Task<Issue> CreateAsync(
            Guid repositoryId,
            string title,
            string text = null)
        {
            if (await _issueRepository.AnyAsync(i => i.Title == title))
            {
                throw new BusinessException("IssueTracking:IssueWithSameTitleExists");
            }

            return new Issue(
                GuidGenerator.Create(),
                repositoryId,
                title,
                text
            );
        }
    }
}
```

- `CreateAsync` 方法检查是否已经存在具有相同标题的问题，并在这种情况下引发业务异常。
- 如果没有重复，它会创建并返回一个新`Issue`。

为了使用 `IssueManager` 的 `CreateAsync` 方法，将 `IssueAppService` 更改为如下所示：

```csharp
public class IssueAppService : ApplicationService, IIssueAppService
{
    private readonly IssueManager _issueManager;
    private readonly IRepository<Issue, Guid> _issueRepository;
    private readonly IRepository<AppUser, Guid> _userRepository;

    public IssueAppService(
        IssueManager issueManager,
        IRepository<Issue, Guid> issueRepository,
        IRepository<AppUser, Guid> userRepository)
    {
        _issueManager = issueManager;
        _issueRepository = issueRepository;
        _userRepository = userRepository;
    }

    public async Task<IssueDto> CreateAsync(IssueCreationDto input)
    {
        // 使用IssueManager创建一个有效实体
        var issue = await _issueManager.CreateAsync(
            input.RepositoryId,
            input.Title,
            input.Text
        );

        // 应用领域其他操作
        if (input.AssignedUserId.HasValue)
        {
            var user = await _userRepository.GetAsync(input.AssignedUserId.Value);
            await _issueManager.AssignToAsync(issue, user);
        }

        // 保存
        await _issueRepository.InsertAsync(issue);

        // 返回代表新问题的DTO
        return ObjectMapper.Map<Issue, IssueDto>(issue);
    }
}

// *** IssueCreationDto class ***
public class IssueCreationDto
{
    public Guid RepositoryId { get; set; }
    [Required]
    public string Title { get; set; }
    public Guid? AssignedUserId { get; set; }
    public string Text { get; set; }
}
```



### 讨论：为什么在`IssueManager`中`Issue`没有保存到数据库中？

您可能会问“**为什么 `IssueManager` 没有将`Issue`保存到数据库中？**”。我们认为这是应用服务的责任。

因为，应用服务在保存它之前可能需要对问题对象进行额外的更改/操作。如果领域服务保存它，那么*保存*操作将重复；

- 由于两次数据库交互，这会导致性能下降。
- 它需要涵盖这两种操作的显式数据库事务。
- 如果其他操作因业务规则而取消实体创建，则应在数据库中回滚事务。

当您检查 `IssueAppService` 时，您将看到`IssueManager.CreateAsync` 方法中不将 `Issue` 保存到的数据库的优点。否则，我们将需要执行一次*插入*（在 `IssueManager` 中）和一次*更新*（在分配之后）。



### 讨论：为什么应用服务中没有实现重复标题检查？

我们可以简单地说“因为它是**核心领域逻辑**，应该在领域层中实现”。但是，它带来了一个新问题“**您如何确定**它是核心领域逻辑，而不是应用程序逻辑？” （稍后我们将详细讨论差异）。

对于这个例子，一个简单的问题可以帮助我们做出决定：“如果我们有另一种方式（用例）来创建问题，我们是否仍应应用相同的规则？是否应*始终*实施该规则”。您可能会想“为什么我们有第二种方式来创建问题？”。然而，在现实生活中，你有；

- 应用程序的**最终用户**可能会在应用程序的标准 UI 中创建问题。
- 您可能有自己的员工使用的第二个**后台**应用程序，并且您可能希望提供一种创建问题的方法（在这种情况下可能具有不同的授权规则）。
- 您可能有一个对 3rd 方客户端开放的 HTTP API，并且它们会创建问题。
- 您可能有一个**后台工作服务**，它会执行某些操作并在检测到某些问题时创建问题。通过这种方式，它将在没有任何用户交互的情况下创建问题（并且可能没有任何标准的授权检查）。
- 您可能在 UI 上有一个按钮，可以将某些内容（例如，讨论）转换为问题。

我们可以举出更多的例子。

所有这些都应该通过**不同的应用服务方法**来实现（参见下面的多应用层部分），但它们**始终**遵循规则：新问题的标题不能与任何现有问题相同！这就是为什么这个逻辑是**核心领域逻辑**，应该位于领域层，而**不应该**在所有这些应用服务方法中**重复**。



### 更新/操作实体

创建实体后，用例会对其进行更新/操作，直到将其从系统中删除。可以有不同类型的用例直接或间接地改变一个实体。

在本节中，我们将讨论更改问题的多个属性的典型更新操作。

这一次，从*更新* DTO 开始：

```csharp
public class UpdateIssueDto
{
    [Required]
    public string Title {get;set;}
    public string Text{get;set;}
    public Guid? AssignedUserId{get;set;}
}
```

通过与 `IssueCreationDto` 进行比较，您看不到 `RepositoryId`。因为，我们的系统不允许跨存储库移动问题（假设为 GitHub 存储库）。只有 `Title` 是必需的，其他属性是可选的。

让我们看看 `IssueAppService` 中的*更新*实现：

```csharp
public class IssueAppService : ApplicationService.IIssueAppService
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

    public async Task<IssueDto> UpdateAsync(Guid id, UpdateIssueDto input)
    {
        //从数据库中获取实体
        var issue = await _issueRepository.GetAsync(id);

        //修改标题
        await _issueManager.ChangeTitleAsync(issue, input.Title);

        //修改所属用户
        if (input.AssignedUserId.HasValue)
        {
            var user = await _userRepository.GetAsync(input.AssignedUserId.Value);
            await _issueManager.AssignToAsync(issue, user);
        }

        //修改文本（没有业务规则，可接受所有值）
        issue.Text = input.Text;

        //更新数据库实体
        await _issueRepository.UpdateAsync(issue);

        //返回代表新问题的DTO
        return ObjectMapper.Map<Issue, IssueDto>(issue);
    }
}
```

- `UpdateAsync` 方法获取 `id` 作为单独的参数。它不包含在 `UpdateIssueDto` 中。这是一个设计决策，可帮助 ABP 在您将此服务[自动公开](https://docs.abp.io/en/abp/latest/API/Auto-API-Controllers)为 HTTP API 端点时正确定义 HTTP 路由。所以，这与 DDD 无关。
- 它首先从数据库中**获取**`Issue`实体。
- 使用 `IssueManager` 的 `ChangeTitleAsync` 而不是直接调用 `Issue.SetTitle`(...)。因为我们需要像在*实体创建*中那样实现**重复的标题检查**。这需要对 `Issue` 和 `IssueManager` 类进行一些更改（将在下面解释）。
- 如果此请求正在更改**分配的用户**，则使用 `IssueManager` 的 `AssignToAsync` 方法。
- 直接设置 `Issue.Text` 因为没有**业务规则**。如果我们以后需要，我们可以随时重构。
- **保存**对数据库的**更改**。同样，保存更改的实体是协调业务对象和事务的应用程序服务方法的责任。如果 `IssueManager` 已经在 `ChangeTitleAsync` 和 `AssignToAsync` 方法内部保存，则会出现两次数据库操作（请参阅上面的讨论：为什么在`IssueManager`中`Issue`没有保存到数据库中？上面）。
- 最后使用 `IObjectMapper` 返回一个 `IssueDto`，它是通过从更新的 `Issue` 实体**映射**自动创建的。

如前所述，我们需要对 `Issue` 和 `IssueManager` 类进行一些更改。

首先，在 `Issue` 类中将 `SetTitle` 方法设置为 `internal`

```csharp
internal void SetTitle(string title)
{
	Title=Check.NotNullOrWhiteSpace(title,nameof(title));
}
```

然后向 `IssueManager` 添加了一个新方法来更改标题：

```csharp
public async Task ChangeTitleAsync(Issue issue, string title)
{
    if (issue.Title == title)
    {
        return;
    }

    if (await _issueRepository.AnyAsync(i => i.Title == title))
    {
        throw new BusinessException("IssueTracking:IssueWithSameTitleExists");
    }

    issue.SetTitle(title);
}
```







::: warning
作者：Gerry Ge

出处：[https://gerryge.com/blogs/2021/Implementing_Domain_Driven_Design/05_Example_User_Cases.html](https://gerryge.com/blogs/2021/Implementing_Domain_Driven_Design/05_Example_User_Cases.html)

版权：本作品采用「[署名-非商业性使用-相同方式共享 4.0 国际](https://creativecommons.org/licenses/by-nc-sa/4.0/)」许可协议进行许可。

**转载请注明出处**
:::
