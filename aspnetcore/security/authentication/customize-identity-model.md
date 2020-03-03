---
title: 在 ASP.NET Core 中的标识模型自定义
author: ajcvickers
description: 本文介绍如何为 ASP.NET Core 标识自定义的基础的实体框架核心数据模型。
ms.author: avickers
ms.date: 07/01/2019
uid: security/authentication/customize_identity_model
ms.openlocfilehash: f549fdff4a416b5fadcb2b1078b051bbab8e402e
ms.sourcegitcommit: eb3e51d58dd713eefc242148f45bd9486be3a78a
ms.translationtype: MT
ms.contentlocale: zh-CN
ms.lasthandoff: 07/02/2019
ms.locfileid: "67500477"
---
# <a name="identity-model-customization-in-aspnet-core"></a>在 ASP.NET Core 中的标识模型自定义

通过[Arthur Vickers](https://github.com/ajcvickers)

ASP.NET Core 标识提供了一个框架，用于管理和存储在 ASP.NET Core 应用中的用户帐户。 标识添加到项目时**单个用户帐户**选择作为身份验证机制。 默认情况下，标识，可以使用的 Entity Framework (EF) Core 数据模型。 本文介绍如何自定义的身份标识模型。

## <a name="identity-and-ef-core-migrations"></a>标识和 EF Core 迁移

之前检查模型，它是有助于了解如何标识配合[EF Core 迁移](/ef/core/managing-schemas/migrations/)创建和更新数据库。 最高级别的过程是：

1. 定义或更新[代码中的数据模型](/ef/core/modeling/)。
1. 添加迁移，以将此模型转换为可以应用到数据库的更改。
1. 检查迁移正确表示你的意图。
1. 将应用迁移来更新要与模型同步的数据库。
1. 重复步骤 1 至 4 以进一步优化模型并使数据库保持同步。

使用以下方法之一来添加和应用迁移：

* **程序包管理器控制台**(PMC) 窗口，如果使用 Visual Studio。 有关详细信息，请参阅[EF Core PMC 工具](/ef/core/miscellaneous/cli/powershell)。
* 使用命令行在.NET Core CLI。 有关详细信息，请参阅[EF Core.NET 命令行工具](/ef/core/miscellaneous/cli/dotnet)。
* 单击**应用迁移**时运行该应用程序错误页上的按钮。

ASP.NET Core 具有开发时间错误页处理程序。 当应用运行时，该处理程序可以将应用迁移。 生产应用程序通常从迁移生成 SQL 脚本和部署数据库更改受控制的应用程序和数据库部署的一部分。

创建新的应用使用标识时，已经完成上述步骤 1 和 2。 也就是说，初始数据模型已存在，并已向项目添加初始迁移。 初始迁移仍需要将应用到数据库。 可以通过以下方法之一应用初始迁移：

* 运行`Update-Database`在 PMC 中。
* 运行`dotnet ef database update`命令行界面中。
* 单击**应用迁移**时运行该应用程序错误页上的按钮。

根据对模型进行更改，请重复前面的步骤。

## <a name="the-identity-model"></a>标识模型

### <a name="entity-types"></a>实体类型

标识模型包含以下实体类型。

|实体类型|描述                                                  |
|-----------|-------------------------------------------------------------|
|`User`     |表示的用户。                                         |
|`Role`     |表示一个角色。                                           |
|`UserClaim`|表示一个用户拥有的声明。                    |
|`UserToken`|表示用户的身份验证令牌。               |
|`UserLogin`|将用户与登录名相关联。                              |
|`RoleClaim`|表示一个角色内的所有用户授予的声明。|
|`UserRole` |联接实体相关联的用户和角色。               |

### <a name="entity-type-relationships"></a>实体的类型关系

[实体类型](#entity-types)彼此相关的以下方面：

* 每个`User`都可以具有许多`UserClaims`。
* 每个`User`都可以具有许多`UserLogins`。
* 每个`User`都可以具有许多`UserTokens`。
* 每个`Role`都可以具有许多关联`RoleClaims`。
* 每个`User`都可以具有许多关联`Roles`，和每个`Role`可与许多关联`Users`。 这是需要在数据库中的联接表的多对多关系。 联接表由`UserRole`实体。

### <a name="default-model-configuration"></a>默认模型配置

标识定义许多*上下文类*继承自[DbContext](/dotnet/api/microsoft.entityframeworkcore.dbcontext)若要配置和使用模型。 进行此配置使用[EF Core 代码 First Fluent API](/ef/core/modeling/)中[OnModelCreating](/dotnet/api/microsoft.entityframeworkcore.dbcontext.onmodelcreating)上下文类的方法。 默认配置是：

```csharp
builder.Entity<TUser>(b =>
{
    // Primary key
    b.HasKey(u => u.Id);

    // Indexes for "normalized" username and email, to allow efficient lookups
    b.HasIndex(u => u.NormalizedUserName).HasName("UserNameIndex").IsUnique();
    b.HasIndex(u => u.NormalizedEmail).HasName("EmailIndex");

    // Maps to the AspNetUsers table
    b.ToTable("AspNetUsers");

    // A concurrency token for use with the optimistic concurrency checking
    b.Property(u => u.ConcurrencyStamp).IsConcurrencyToken();

    // Limit the size of columns to use efficient database types
    b.Property(u => u.UserName).HasMaxLength(256);
    b.Property(u => u.NormalizedUserName).HasMaxLength(256);
    b.Property(u => u.Email).HasMaxLength(256);
    b.Property(u => u.NormalizedEmail).HasMaxLength(256);

    // The relationships between User and other entity types
    // Note that these relationships are configured with no navigation properties

    // Each User can have many UserClaims
    b.HasMany<TUserClaim>().WithOne().HasForeignKey(uc => uc.UserId).IsRequired();

    // Each User can have many UserLogins
    b.HasMany<TUserLogin>().WithOne().HasForeignKey(ul => ul.UserId).IsRequired();

    // Each User can have many UserTokens
    b.HasMany<TUserToken>().WithOne().HasForeignKey(ut => ut.UserId).IsRequired();

    // Each User can have many entries in the UserRole join table
    b.HasMany<TUserRole>().WithOne().HasForeignKey(ur => ur.UserId).IsRequired();
});

builder.Entity<TUserClaim>(b =>
{
    // Primary key
    b.HasKey(uc => uc.Id);

    // Maps to the AspNetUserClaims table
    b.ToTable("AspNetUserClaims");
});

builder.Entity<TUserLogin>(b =>
{
    // Composite primary key consisting of the LoginProvider and the key to use
    // with that provider
    b.HasKey(l => new { l.LoginProvider, l.ProviderKey });

    // Limit the size of the composite key columns due to common DB restrictions
    b.Property(l => l.LoginProvider).HasMaxLength(128);
    b.Property(l => l.ProviderKey).HasMaxLength(128);

    // Maps to the AspNetUserLogins table
    b.ToTable("AspNetUserLogins");
});

builder.Entity<TUserToken>(b =>
{
    // Composite primary key consisting of the UserId, LoginProvider and Name
    b.HasKey(t => new { t.UserId, t.LoginProvider, t.Name });

    // Limit the size of the composite key columns due to common DB restrictions
    b.Property(t => t.LoginProvider).HasMaxLength(maxKeyLength);
    b.Property(t => t.Name).HasMaxLength(maxKeyLength);

    // Maps to the AspNetUserTokens table
    b.ToTable("AspNetUserTokens");
});

builder.Entity<TRole>(b =>
{
    // Primary key
    b.HasKey(r => r.Id);

    // Index for "normalized" role name to allow efficient lookups
    b.HasIndex(r => r.NormalizedName).HasName("RoleNameIndex").IsUnique();

    // Maps to the AspNetRoles table
    b.ToTable("AspNetRoles");

    // A concurrency token for use with the optimistic concurrency checking
    b.Property(r => r.ConcurrencyStamp).IsConcurrencyToken();

    // Limit the size of columns to use efficient database types
    b.Property(u => u.Name).HasMaxLength(256);
    b.Property(u => u.NormalizedName).HasMaxLength(256);

    // The relationships between Role and other entity types
    // Note that these relationships are configured with no navigation properties

    // Each Role can have many entries in the UserRole join table
    b.HasMany<TUserRole>().WithOne().HasForeignKey(ur => ur.RoleId).IsRequired();

    // Each Role can have many associated RoleClaims
    b.HasMany<TRoleClaim>().WithOne().HasForeignKey(rc => rc.RoleId).IsRequired();
});

builder.Entity<TRoleClaim>(b =>
{
    // Primary key
    b.HasKey(rc => rc.Id);

    // Maps to the AspNetRoleClaims table
    b.ToTable("AspNetRoleClaims");
});

builder.Entity<TUserRole>(b =>
{
    // Primary key
    b.HasKey(r => new { r.UserId, r.RoleId });

    // Maps to the AspNetUserRoles table
    b.ToTable("AspNetUserRoles");
});
```

### <a name="model-generic-types"></a>模型的泛型类型

标识定义默认值[公共语言运行时](/dotnet/standard/glossary#clr)上面列出的每个实体类型 (CLR) 类型。 这些类型带有前缀*标识*:

* `IdentityUser`
* `IdentityRole`
* `IdentityUserClaim`
* `IdentityUserToken`
* `IdentityUserLogin`
* `IdentityRoleClaim`
* `IdentityUserRole`

而不是直接使用这些类型，类型可以用作基类，这些类对于应用自己的类型。 `DbContext`定义标识的类是通用的这样不同的 CLR 类型可以用于一个或多个模型中的实体类型。 这些泛型类型还允许`User`主键 (PK) 数据类型更改。

对于角色，支持使用标识时<xref:Microsoft.AspNetCore.Identity.EntityFrameworkCore.IdentityDbContext>应使用类。 例如：

```csharp
// Uses all the built-in Identity types
// Uses `string` as the key type
public class IdentityDbContext
    : IdentityDbContext<IdentityUser, IdentityRole, string>
{
}

// Uses the built-in Identity types except with a custom User type
// Uses `string` as the key type
public class IdentityDbContext<TUser>
    : IdentityDbContext<TUser, IdentityRole, string>
        where TUser : IdentityUser
{
}

// Uses the built-in Identity types except with custom User and Role types
// The key type is defined by TKey
public class IdentityDbContext<TUser, TRole, TKey> : IdentityDbContext<
    TUser, TRole, TKey, IdentityUserClaim<TKey>, IdentityUserRole<TKey>,
    IdentityUserLogin<TKey>, IdentityRoleClaim<TKey>, IdentityUserToken<TKey>>
        where TUser : IdentityUser<TKey>
        where TRole : IdentityRole<TKey>
        where TKey : IEquatable<TKey>
{
}

// No built-in Identity types are used; all are specified by generic arguments
// The key type is defined by TKey
public abstract class IdentityDbContext<
    TUser, TRole, TKey, TUserClaim, TUserRole, TUserLogin, TRoleClaim, TUserToken>
    : IdentityUserContext<TUser, TKey, TUserClaim, TUserLogin, TUserToken>
         where TUser : IdentityUser<TKey>
         where TRole : IdentityRole<TKey>
         where TKey : IEquatable<TKey>
         where TUserClaim : IdentityUserClaim<TKey>
         where TUserRole : IdentityUserRole<TKey>
         where TUserLogin : IdentityUserLogin<TKey>
         where TRoleClaim : IdentityRoleClaim<TKey>
         where TUserToken : IdentityUserToken<TKey>
```

还有可能在这种情况下使用而无需角色 （仅声明），标识<xref:Microsoft.AspNetCore.Identity.EntityFrameworkCore.IdentityUserContext%601>应使用类：

```csharp
// Uses the built-in non-role Identity types except with a custom User type
// Uses `string` as the key type
public class IdentityUserContext<TUser>
    : IdentityUserContext<TUser, string>
        where TUser : IdentityUser
{
}

// Uses the built-in non-role Identity types except with a custom User type
// The key type is defined by TKey
public class IdentityUserContext<TUser, TKey> : IdentityUserContext<
    TUser, TKey, IdentityUserClaim<TKey>, IdentityUserLogin<TKey>,
    IdentityUserToken<TKey>>
        where TUser : IdentityUser<TKey>
        where TKey : IEquatable<TKey>
{
}

// No built-in Identity types are used; all are specified by generic arguments, with no roles
// The key type is defined by TKey
public abstract class IdentityUserContext<
    TUser, TKey, TUserClaim, TUserLogin, TUserToken> : DbContext
        where TUser : IdentityUser<TKey>
        where TKey : IEquatable<TKey>
        where TUserClaim : IdentityUserClaim<TKey>
        where TUserLogin : IdentityUserLogin<TKey>
        where TUserToken : IdentityUserToken<TKey>
{
}
```

## <a name="customize-the-model"></a>自定义模型

模型自定义的起始点是派生自相应的上下文类型。 请参阅[泛型类型的模型](#model-generic-types)部分。 此上下文类型通常称为`ApplicationDbContext`和创建的 ASP.NET Core 模板。

使用的上下文来将模型配置为通过两种方式：

* 提供实体和键类型作为泛型类型参数。
* 重写`OnModelCreating`若要修改这些类型的映射。

重写时`OnModelCreating`，`base.OnModelCreating`应首先调用; 重写配置，应调用下一步。 EF Core 通常具有配置的最后一个 wins 策略。 例如，如果`ToTable`的实体类型的方法首先调用具有一个表名称，并用再说一遍更高版本使用不同的表名称，第二次调用中的表名。

### <a name="custom-user-data"></a>自定义用户数据

<!--
set projNam=WebApp1
dotnet new webapp -o %projNam%
cd %projNam%
dotnet add package Microsoft.VisualStudio.Web.CodeGeneration.Design 
dotnet aspnet-codegenerator identity  -dc ApplicationDbContext --useDefaultUI 
dotnet ef migrations add CreateIdentitySchema
dotnet ef database update
 -->

[自定义用户数据](xref:security/authentication/add-user-data)支持通过继承`IdentityUser`。 此类型命名惯例做法`ApplicationUser`:

```csharp
public class ApplicationUser : IdentityUser
{
    public string CustomTag { get; set; }
}
```

使用`ApplicationUser`作为上下文的泛型参数的类型：

```csharp
public class ApplicationDbContext : IdentityDbContext<ApplicationUser>
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
        : base(options)
    {
    }

    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder);
    }
}
```

无需重写`OnModelCreating`在`ApplicationDbContext`类。 EF Core 映射`CustomTag`按照约定的属性。 但是，需要更新，以创建新数据库`CustomTag`列。 若要创建列，请添加迁移时，并如中所述，然后更新数据库[标识和 EF Core 迁移](#identity-and-ef-core-migrations)。

更新*Pages/Shared/_LoginPartial.cshtml*和替换`IdentityUser`与`ApplicationUser`:

```cshtml
@using Microsoft.AspNetCore.Identity
@using WebApp1.Areas.Identity.Data
@inject SignInManager<ApplicationUser> SignInManager
@inject UserManager<ApplicationUser> UserManager
```

更新*Areas/Identity/IdentityHostingStartup.cs*或`Startup.ConfigureServices`和替换`IdentityUser`与`ApplicationUser`。

```csharp
services.AddDefaultIdentity<ApplicationUser>()
        .AddEntityFrameworkStores<ApplicationDbContext>()
        .AddDefaultUI();
```

在 ASP.NET Core 2.1 或更高版本，作为一个 Razor 类库提供标识。 有关详细信息，请参阅 <xref:security/authentication/scaffold-identity>。 因此，前面的代码需要调用<xref:Microsoft.AspNetCore.Identity.IdentityBuilderUIExtensions.AddDefaultUI*>。 如果标识基架用于标识文件添加到项目，删除对调用`AddDefaultUI`。 有关详细信息，请参见:

* [基架标识](xref:security/authentication/scaffold-identity)
* [添加、 下载和删除到标识的自定义用户数据](xref:security/authentication/add-user-data)

### <a name="change-the-primary-key-type"></a>更改主键类型

为 PK 列的数据类型创建数据库后更改是很多数据库系统有问题。 更改 PK 通常涉及删除并重新创建表。 因此，密钥类型时，应指定在初始迁移创建的数据库。

请按照下列步骤更改 PK 类型：

1. 如果在更改PK之前数据库已创建，运行`Drop-Database`(PMC) 或`dotnet ef database drop`(.NET Core CLI) 将其删除。
2. 确认删除数据库后，删除已有的初始迁移内容`Remove-Migration`(PMC) 或`dotnet ef migrations remove`(.NET Core CLI)。
3. 更新`ApplicationDbContext`类进行派生<xref:Microsoft.AspNetCore.Identity.EntityFrameworkCore.IdentityDbContext%603>。 指定的新类型`TKey`。 例如，若要使用`Guid`密钥类型：

    ```csharp
    public class ApplicationDbContext
        : IdentityDbContext<IdentityUser<Guid>, IdentityRole<Guid>, Guid>
    {
        public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
            : base(options)
        {
        }
    }
    ```

    ::: moniker range=">= aspnetcore-2.0"

    在前面的代码中，泛型类<xref:Microsoft.AspNetCore.Identity.IdentityUser%601>和<xref:Microsoft.AspNetCore.Identity.IdentityRole%601>必须指定要使用新的密钥类型。

    ::: moniker-end

    ::: moniker range="<= aspnetcore-1.1"

    在前面的代码中，泛型类<xref:Microsoft.AspNetCore.Identity.EntityFrameworkCore.IdentityUser%601>和<xref:Microsoft.AspNetCore.Identity.EntityFrameworkCore.IdentityRole%601>必须指定要使用新的密钥类型。

    ::: moniker-end

    `Startup.ConfigureServices` 必须更新为使用一般用户：

    ::: moniker range=">= aspnetcore-2.1"

    ```csharp
    services.AddDefaultIdentity<IdentityUser<Guid>>()
            .AddEntityFrameworkStores<ApplicationDbContext>()
            .AddDefaultTokenProviders();
    ```

    ::: moniker-end

    ::: moniker range="= aspnetcore-2.0"

    ```csharp
    services.AddIdentity<IdentityUser<Guid>, IdentityRole>()
            .AddEntityFrameworkStores<ApplicationDbContext>()
            .AddDefaultTokenProviders();
    ```

    ::: moniker-end

    ::: moniker range="<= aspnetcore-1.1"

    ```csharp
    services.AddIdentity<IdentityUser<Guid>, IdentityRole>()
            .AddEntityFrameworkStores<ApplicationDbContext, Guid>()
            .AddDefaultTokenProviders();
    ```

    ::: moniker-end

4. 如果正在使用自定义`ApplicationUser`，更新要继承的类`IdentityUser`。 例如：

    ::: moniker range="<= aspnetcore-1.1"

    [!code-csharp[](customize-identity-model/samples/1.1/MvcSampleApp/Models/ApplicationUser.cs?name=snippet_ApplicationUser&highlight=4)]

    ::: moniker-end

    ::: moniker range=">= aspnetcore-2.0"

    [!code-csharp[](customize-identity-model/samples/2.1/RazorPagesSampleApp/Data/ApplicationUser.cs?name=snippet_ApplicationUser&highlight=4)]

    ::: moniker-end

    更新`ApplicationDbContext`来引用自定义`ApplicationUser`类：

    ```csharp
    public class ApplicationDbContext
        : IdentityDbContext<ApplicationUser, IdentityRole<Guid>, Guid>
    {
        public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
            : base(options)
        {
        }
    }
    ```

    注册自定义数据库上下文类添加中的标识服务时`Startup.ConfigureServices`:

    ::: moniker range=">= aspnetcore-2.1"

    ```csharp
    services.AddDefaultIdentity<ApplicationUser>()
            .AddEntityFrameworkStores<ApplicationDbContext>()
            .AddDefaultUI()
            .AddDefaultTokenProviders();
    ```

    通过分析来推断主键的数据类型[DbContext](/dotnet/api/microsoft.entityframeworkcore.dbcontext)对象。

    在 ASP.NET Core 2.1 或更高版本，作为一个 Razor 类库提供标识。 有关详细信息，请参阅 <xref:security/authentication/scaffold-identity>。 因此，前面的代码需要调用<xref:Microsoft.AspNetCore.Identity.IdentityBuilderUIExtensions.AddDefaultUI*>。 如果标识基架用于标识文件添加到项目，删除对调用`AddDefaultUI`。

    ::: moniker-end

    ::: moniker range="= aspnetcore-2.0"

    ```csharp
    services.AddIdentity<ApplicationUser, IdentityRole>()
            .AddEntityFrameworkStores<ApplicationDbContext>()
            .AddDefaultTokenProviders();
    ```

    通过分析来推断主键的数据类型[DbContext](/dotnet/api/microsoft.entityframeworkcore.dbcontext)对象。

    ::: moniker-end

    ::: moniker range="<= aspnetcore-1.1"

    ```csharp
    services.AddIdentity<ApplicationUser, IdentityRole>()
            .AddEntityFrameworkStores<ApplicationDbContext, Guid>()
            .AddDefaultTokenProviders();
    ```

    <xref:Microsoft.Extensions.DependencyInjection.IdentityEntityFrameworkBuilderExtensions.AddEntityFrameworkStores*>方法接受`TKey`，该值指示主键的数据类型的类型。

    ::: moniker-end

5. 如果正在使用自定义`ApplicationRole`，更新要继承的类`IdentityRole<TKey>`。 例如：

    [!code-csharp[](customize-identity-model/samples/2.1/RazorPagesSampleApp/Data/ApplicationRole.cs?name=snippet_ApplicationRole&highlight=4)]

    更新`ApplicationDbContext`来引用自定义`ApplicationRole`类。 例如，下面的类引用自定义`ApplicationUser`和自定义`ApplicationRole`:

    ::: moniker range=">= aspnetcore-2.1"

    [!code-csharp[](customize-identity-model/samples/2.1/RazorPagesSampleApp/Data/ApplicationDbContext.cs?name=snippet_ApplicationDbContext&highlight=5-6)]

    注册自定义数据库上下文类添加中的标识服务时`Startup.ConfigureServices`:

    [!code-csharp[](customize-identity-model/samples/2.1/RazorPagesSampleApp/Startup.cs?name=snippet_ConfigureServices&highlight=13-16)]

    通过分析来推断主键的数据类型[DbContext](/dotnet/api/microsoft.entityframeworkcore.dbcontext)对象。

    在 ASP.NET Core 2.1 或更高版本，作为一个 Razor 类库提供标识。 有关详细信息，请参阅 <xref:security/authentication/scaffold-identity>。 因此，前面的代码需要调用<xref:Microsoft.AspNetCore.Identity.IdentityBuilderUIExtensions.AddDefaultUI*>。 如果标识基架用于标识文件添加到项目，删除对调用`AddDefaultUI`。

    ::: moniker-end

    ::: moniker range="= aspnetcore-2.0"

    [!code-csharp[](customize-identity-model/samples/2.0/RazorPagesSampleApp/Data/ApplicationDbContext.cs?name=snippet_ApplicationDbContext&highlight=5-6)]

    注册自定义数据库上下文类添加中的标识服务时`Startup.ConfigureServices`:

    [!code-csharp[](customize-identity-model/samples/2.0/RazorPagesSampleApp/Startup.cs?name=snippet_ConfigureServices&highlight=7-9)]

    通过分析来推断主键的数据类型[DbContext](/dotnet/api/microsoft.entityframeworkcore.dbcontext)对象。

    ::: moniker-end

    ::: moniker range="<= aspnetcore-1.1"

    [!code-csharp[](customize-identity-model/samples/1.1/MvcSampleApp/Data/ApplicationDbContext.cs?name=snippet_ApplicationDbContext&highlight=5-6)]

    注册自定义数据库上下文类添加中的标识服务时`Startup.ConfigureServices`:

    [!code-csharp[](customize-identity-model/samples/1.1/MvcSampleApp/Startup.cs?name=snippet_ConfigureServices&highlight=7-9)]

    <xref:Microsoft.Extensions.DependencyInjection.IdentityEntityFrameworkBuilderExtensions.AddEntityFrameworkStores*>方法接受`TKey`，该值指示主键的数据类型的类型。

    ::: moniker-end
6.   在 ASP.NET Core 2.1 或更高版本，如果使用工具将标识基架添加到项目中，你需要重新添加标识基架，并覆盖之前的模板内容。如果使用了自定义`ApplicationUser`，你还需要修改默认模板`_LoginPartial.cshtml`，将默认的`IdentityUser`替换成自定义类型。

### <a name="add-navigation-properties"></a>添加导航属性

更改关系的模型配置可能会更难于进行其他更改。 若要替换现有关系，而不是创建新的其他关系，必须格外小心。 具体而言，已更改的关系必须指定相同的外键 (FK) 属性作为现有的关系。 例如，之间的关系`Users`和`UserClaims`是，默认情况下，按如下所示指定：

```csharp
builder.Entity<TUser>(b =>
{
    // Each User can have many UserClaims
    b.HasMany<TUserClaim>()
     .WithOne()
     .HasForeignKey(uc => uc.UserId)
     .IsRequired();
});
```

此关系的 FK 指定为`UserClaim.UserId`属性。 `HasMany` 和`WithOne`调用不带参数，以创建不使用导航属性的关系。

导航属性添加到`ApplicationUser`，允许关联`UserClaims`若要从用户中引用：

```csharp
public class ApplicationUser : IdentityUser
{
    public virtual ICollection<IdentityUserClaim<string>> Claims { get; set; }
}
```

`TKey`为`IdentityUserClaim<TKey>`是用户的 PK 为指定的类型。 在这种情况下，`TKey`是`string`因为正在使用默认值。 它具有**不**的 PK 类型`UserClaim`实体类型。

现在，已存在的导航属性，它必须在配置`OnModelCreating`:

```csharp
public class ApplicationDbContext : IdentityDbContext<ApplicationUser>
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
        : base(options)
    {
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        modelBuilder.Entity<ApplicationUser>(b =>
        {
            // Each User can have many UserClaims
            b.HasMany(e => e.Claims)
                .WithOne()
                .HasForeignKey(uc => uc.UserId)
                .IsRequired();
        });
    }
}
```

请注意，完全按照以前，只能使用导航属性调用中指定配置关系`HasMany`。

在 EF 模型中，不是数据库仅存在导航属性。 因为关系 fk 后未发生更改，这种类型的模型更改不需要更新的数据库。 可以通过添加迁移后进行更改选中此项。 `Up`和`Down`方法为空。

### <a name="add-all-user-navigation-properties"></a>添加所有用户导航属性

下面的示例作为指南使用上的一节中，在用户配置单向导航属性的所有关系：

```csharp
public class ApplicationUser : IdentityUser
{
    public virtual ICollection<IdentityUserClaim<string>> Claims { get; set; }
    public virtual ICollection<IdentityUserLogin<string>> Logins { get; set; }
    public virtual ICollection<IdentityUserToken<string>> Tokens { get; set; }
    public virtual ICollection<IdentityUserRole<string>> UserRoles { get; set; }
}
```

```csharp
public class ApplicationDbContext : IdentityDbContext<ApplicationUser>
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
        : base(options)
    {
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        modelBuilder.Entity<ApplicationUser>(b =>
        {
            // Each User can have many UserClaims
            b.HasMany(e => e.Claims)
                .WithOne()
                .HasForeignKey(uc => uc.UserId)
                .IsRequired();

            // Each User can have many UserLogins
            b.HasMany(e => e.Logins)
                .WithOne()
                .HasForeignKey(ul => ul.UserId)
                .IsRequired();

            // Each User can have many UserTokens
            b.HasMany(e => e.Tokens)
                .WithOne()
                .HasForeignKey(ut => ut.UserId)
                .IsRequired();

            // Each User can have many entries in the UserRole join table
            b.HasMany(e => e.UserRoles)
                .WithOne()
                .HasForeignKey(ur => ur.UserId)
                .IsRequired();
        });
    }
}
```

### <a name="add-user-and-role-navigation-properties"></a>添加用户和角色导航属性

下面的示例作为指南使用上的一节中，用户和角色上配置所有关系的导航的属性：

```csharp
public class ApplicationUser : IdentityUser
{
    public virtual ICollection<IdentityUserClaim<string>> Claims { get; set; }
    public virtual ICollection<IdentityUserLogin<string>> Logins { get; set; }
    public virtual ICollection<IdentityUserToken<string>> Tokens { get; set; }
    public virtual ICollection<ApplicationUserRole> UserRoles { get; set; }
}

public class ApplicationRole : IdentityRole
{
    public virtual ICollection<ApplicationUserRole> UserRoles { get; set; }
}

public class ApplicationUserRole : IdentityUserRole<string>
{
    public virtual ApplicationUser User { get; set; }
    public virtual ApplicationRole Role { get; set; }
}
```

```csharp
public class ApplicationDbContext
    : IdentityDbContext<
        ApplicationUser, ApplicationRole, string,
        IdentityUserClaim<string>, ApplicationUserRole, IdentityUserLogin<string>,
        IdentityRoleClaim<string>, IdentityUserToken<string>>
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
        : base(options)
    {
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        modelBuilder.Entity<ApplicationUser>(b =>
        {
            // Each User can have many UserClaims
            b.HasMany(e => e.Claims)
                .WithOne()
                .HasForeignKey(uc => uc.UserId)
                .IsRequired();

            // Each User can have many UserLogins
            b.HasMany(e => e.Logins)
                .WithOne()
                .HasForeignKey(ul => ul.UserId)
                .IsRequired();

            // Each User can have many UserTokens
            b.HasMany(e => e.Tokens)
                .WithOne()
                .HasForeignKey(ut => ut.UserId)
                .IsRequired();

            // Each User can have many entries in the UserRole join table
            b.HasMany(e => e.UserRoles)
                .WithOne(e => e.User)
                .HasForeignKey(ur => ur.UserId)
                .IsRequired();
        });

        modelBuilder.Entity<ApplicationRole>(b =>
        {
            // Each Role can have many entries in the UserRole join table
            b.HasMany(e => e.UserRoles)
                .WithOne(e => e.Role)
                .HasForeignKey(ur => ur.RoleId)
                .IsRequired();
        });

    }
}
```

注意：

* 此示例还包括`UserRole`联接实体，这将需要导航从用户到角色的多对多关系。
* 请记得要更改类型的导航属性以反映该`ApplicationXxx`类型现在正在使用而不是`IdentityXxx`类型。
* 请记住使用`ApplicationXxx`中泛型`ApplicationContext`定义。

### <a name="add-all-navigation-properties"></a>添加所有导航属性

下面的示例作为指南使用上的一节中，所有实体类型上配置所有关系的导航的属性：

```csharp
public class ApplicationUser : IdentityUser
{
    public virtual ICollection<ApplicationUserClaim> Claims { get; set; }
    public virtual ICollection<ApplicationUserLogin> Logins { get; set; }
    public virtual ICollection<ApplicationUserToken> Tokens { get; set; }
    public virtual ICollection<ApplicationUserRole> UserRoles { get; set; }
}

public class ApplicationRole : IdentityRole
{
    public virtual ICollection<ApplicationUserRole> UserRoles { get; set; }
    public virtual ICollection<ApplicationRoleClaim> RoleClaims { get; set; }
}

public class ApplicationUserRole : IdentityUserRole<string>
{
    public virtual ApplicationUser User { get; set; }
    public virtual ApplicationRole Role { get; set; }
}

public class ApplicationUserClaim : IdentityUserClaim<string>
{
    public virtual ApplicationUser User { get; set; }
}

public class ApplicationUserLogin : IdentityUserLogin<string>
{
    public virtual ApplicationUser User { get; set; }
}

public class ApplicationRoleClaim : IdentityRoleClaim<string>
{
    public virtual ApplicationRole Role { get; set; }
}

public class ApplicationUserToken : IdentityUserToken<string>
{
    public virtual ApplicationUser User { get; set; }
}
```

```csharp
public class ApplicationDbContext
    : IdentityDbContext<
        ApplicationUser, ApplicationRole, string,
        ApplicationUserClaim, ApplicationUserRole, ApplicationUserLogin,
        ApplicationRoleClaim, ApplicationUserToken>
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
        : base(options)
    {
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        modelBuilder.Entity<ApplicationUser>(b =>
        {
            // Each User can have many UserClaims
            b.HasMany(e => e.Claims)
                .WithOne(e => e.User)
                .HasForeignKey(uc => uc.UserId)
                .IsRequired();

            // Each User can have many UserLogins
            b.HasMany(e => e.Logins)
                .WithOne(e => e.User)
                .HasForeignKey(ul => ul.UserId)
                .IsRequired();

            // Each User can have many UserTokens
            b.HasMany(e => e.Tokens)
                .WithOne(e => e.User)
                .HasForeignKey(ut => ut.UserId)
                .IsRequired();

            // Each User can have many entries in the UserRole join table
            b.HasMany(e => e.UserRoles)
                .WithOne(e => e.User)
                .HasForeignKey(ur => ur.UserId)
                .IsRequired();
        });

        modelBuilder.Entity<ApplicationRole>(b =>
        {
            // Each Role can have many entries in the UserRole join table
            b.HasMany(e => e.UserRoles)
                .WithOne(e => e.Role)
                .HasForeignKey(ur => ur.RoleId)
                .IsRequired();

            // Each Role can have many associated RoleClaims
            b.HasMany(e => e.RoleClaims)
                .WithOne(e => e.Role)
                .HasForeignKey(rc => rc.RoleId)
                .IsRequired();
        });
    }
}
```

### <a name="use-composite-keys"></a>使用复合键

前面几节所示更改的标识模型中使用的密钥类型。 更改要使用的复合键的标识密钥模型不支持或建议。 使用复合密钥与标识涉及更改标识管理器代码与模型交互的方式。 此自定义已超出本文的讨论范围。

### <a name="change-tablecolumn-names-and-facets"></a>更改表/列名称和分面

若要更改表和列的名称，请调用`base.OnModelCreating`。 然后，添加配置以重写任何默认值。 例如，若要更改所有标识表的名称：

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    base.OnModelCreating(modelBuilder);

    modelBuilder.Entity<IdentityUser>(b =>
    {
        b.ToTable("MyUsers");
    });

    modelBuilder.Entity<IdentityUserClaim<string>>(b =>
    {
        b.ToTable("MyUserClaims");
    });

    modelBuilder.Entity<IdentityUserLogin<string>>(b =>
    {
        b.ToTable("MyUserLogins");
    });

    modelBuilder.Entity<IdentityUserToken<string>>(b =>
    {
        b.ToTable("MyUserTokens");
    });

    modelBuilder.Entity<IdentityRole>(b =>
    {
        b.ToTable("MyRoles");
    });

    modelBuilder.Entity<IdentityRoleClaim<string>>(b =>
    {
        b.ToTable("MyRoleClaims");
    });

    modelBuilder.Entity<IdentityUserRole<string>>(b =>
    {
        b.ToTable("MyUserRoles");
    });
}
```

这些示例使用默认标识类型。 如果使用的应用类型，如`ApplicationUser`，配置该类型而不是默认类型。

下面的示例更改某些列名称：

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    base.OnModelCreating(modelBuilder);

    modelBuilder.Entity<IdentityUser>(b =>
    {
        b.Property(e => e.Email).HasColumnName("EMail");
    });

    modelBuilder.Entity<IdentityUserClaim<string>>(b =>
    {
        b.Property(e => e.ClaimType).HasColumnName("CType");
        b.Property(e => e.ClaimValue).HasColumnName("CValue");
    });
}
```

某些类型的数据库列可以配置某些*方面*(例如，最大值`string`允许的长度)。 下面的示例设置多个列的最大长度`string`模型中的属性：

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    base.OnModelCreating(modelBuilder);

    modelBuilder.Entity<IdentityUser>(b =>
    {
        b.Property(u => u.UserName).HasMaxLength(128);
        b.Property(u => u.NormalizedUserName).HasMaxLength(128);
        b.Property(u => u.Email).HasMaxLength(128);
        b.Property(u => u.NormalizedEmail).HasMaxLength(128);
    });

    modelBuilder.Entity<IdentityUserToken<string>>(b =>
    {
        b.Property(t => t.LoginProvider).HasMaxLength(128);
        b.Property(t => t.Name).HasMaxLength(128);
    });
}
```

### <a name="map-to-a-different-schema"></a>将映射到不同的架构

跨数据库提供程序，架构的行为可能有所不同。 对于 SQL Server，默认值是创建中的所有表*dbo*架构。 可以在不同的架构中创建的表。 例如：

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    base.OnModelCreating(modelBuilder);

    modelBuilder.HasDefaultSchema("notdbo");
}
```

::: moniker range=">= aspnetcore-2.1"

### <a name="lazy-loading"></a>延迟加载

在本部分中，添加了支持的身份标识模型中的延迟加载代理。 延迟加载很有用，因为它允许使用而无需首先确保它们正在加载导航属性。

实体类型可适用于采用多种方式的延迟加载中所述[EF Core 文档](/ef/core/querying/related-data#lazy-loading)。 为简单起见，使用该软件需要延迟加载代理：

* 安装[-Microsoft.EntityFrameworkCore.Proxies](https://www.nuget.org/packages/Microsoft.EntityFrameworkCore.Proxies/)包。
* 调用<xref:Microsoft.EntityFrameworkCore.ProxiesExtensions.UseLazyLoadingProxies*>内[AddDbContext\<TContext >](/dotnet/api/microsoft.extensions.dependencyinjection.entityframeworkservicecollectionextensions.adddbcontext)。
* 公共实体类型与`public virtual`导航属性。

下面的示例演示如何调用`UseLazyLoadingProxies`在`Startup.ConfigureServices`:

```csharp
services
    .AddDbContext<ApplicationDbContext>(
        b => b.UseSqlServer(connectionString)
              .UseLazyLoadingProxies())
    .AddDefaultIdentity<ApplicationUser>()
    .AddEntityFrameworkStores<ApplicationDbContext>();
```

请参阅前面的示例将导航属性添加到实体类型有关的指南。

## <a name="additional-resources"></a>其他资源

* <xref:security/authentication/scaffold-identity>

::: moniker-end
