---
title: 将身份验证和标识迁移到 ASP.NET Core 2.0
author: scottaddie
description: 本文概述了迁移 ASP.NET Core 1.x 身份验证和标识为 ASP.NET Core 2.0 的最常见步骤。
ms.author: scaddie
ms.date: 06/21/2019
uid: migration/1x-to-2x/identity-2x
ms.openlocfilehash: f3817fa1808c331f7e167618e3bb00d68ad08571
ms.sourcegitcommit: 2cb857f0de774df421e35289662ba92cfe56ffd1
ms.translationtype: MT
ms.contentlocale: zh-CN
ms.lasthandoff: 12/25/2019
ms.locfileid: "75355183"
---
# <a name="migrate-authentication-and-identity-to-aspnet-core-20"></a>将身份验证和标识迁移到 ASP.NET Core 2.0

作者： [Scott Addie](https://github.com/scottaddie)和[Hao Kung](https://github.com/HaoK)

ASP.NET Core 2.0 具有新的身份验证和[标识](xref:security/authentication/identity)模型，可通过使用服务简化配置。 ASP.NET Core 1.x 应用程序使用身份验证或标识可以更新以使用新的模型，如下所述。

## <a name="update-namespaces"></a>更新命名空间

在1.x 中，在 `Microsoft.AspNetCore.Identity.EntityFrameworkCore` 命名空间中找到了类 `IdentityRole` 和 `IdentityUser`。

在2.0 中，<xref:Microsoft.AspNetCore.Identity> 命名空间成为几个此类类的新宿主。 对于默认标识代码，受影响的类包括 `ApplicationUser` 和 `Startup`。 调整 `using` 语句以解析受影响的引用。

<a name="auth-middleware"></a>

## <a name="authentication-middleware-and-services"></a>身份验证中间件和服务

在1.x 项目中，通过中间件配置身份验证。 为要支持的每个身份验证方案调用中间件方法。

下面的1.x 示例在*Startup.cs*中配置具有标识的 Facebook 身份验证：

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddIdentity<ApplicationUser, IdentityRole>()
            .AddEntityFrameworkStores<ApplicationDbContext>();
}

public void Configure(IApplicationBuilder app, ILoggerFactory loggerfactory)
{
    app.UseIdentity();
    app.UseFacebookAuthentication(new FacebookOptions {
        AppId = Configuration["auth:facebook:appid"],
        AppSecret = Configuration["auth:facebook:appsecret"]
    });
}
```

在2.0 项目中，通过服务配置身份验证。 在*Startup.cs*的 `ConfigureServices` 方法中注册每个身份验证方案。 `UseIdentity` 方法将替换为 `UseAuthentication`。

以下2.0 示例在*Startup.cs*中配置具有标识的 Facebook 身份验证：

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddIdentity<ApplicationUser, IdentityRole>()
            .AddEntityFrameworkStores<ApplicationDbContext>();

    // If you want to tweak Identity cookies, they're no longer part of IdentityOptions.
    services.ConfigureApplicationCookie(options => options.LoginPath = "/Account/LogIn");
    services.AddAuthentication()
            .AddFacebook(options =>
            {
                options.AppId = Configuration["auth:facebook:appid"];
                options.AppSecret = Configuration["auth:facebook:appsecret"];
            });
}

public void Configure(IApplicationBuilder app, ILoggerFactory loggerfactory) {
    app.UseAuthentication();
}
```

`UseAuthentication` 方法添加单个身份验证中间件组件，该组件负责自动身份验证和远程身份验证请求的处理。 它将所有单个中间件组件替换为一个公共中间件组件。

下面是每个主要身份验证方案的2.0 迁移说明。

### <a name="cookie-based-authentication"></a>基于 Cookie 的身份验证

选择以下两个选项之一，并在*Startup.cs*中进行必要的更改：

1. 使用带有标识的 cookie
    - 将 `UseIdentity` 替换为 `Configure` 方法中的 `UseAuthentication`：

        ```csharp
        app.UseAuthentication();
        ```

    - 在 `ConfigureServices` 方法中调用 `AddIdentity` 方法，以添加 cookie 身份验证服务。
    - （可选）调用 `ConfigureServices` 方法中的 `ConfigureApplicationCookie` 或 `ConfigureExternalCookie` 方法以调整标识 cookie 设置。

        ```csharp
        services.AddIdentity<ApplicationUser, IdentityRole>()
                .AddEntityFrameworkStores<ApplicationDbContext>()
                .AddDefaultTokenProviders();

        services.ConfigureApplicationCookie(options => options.LoginPath = "/Account/LogIn");
        ```

2. 使用没有标识的 cookie
    - 用 `UseAuthentication`替换 `Configure` 方法中的 `UseCookieAuthentication` 方法调用：

        ```csharp
        app.UseAuthentication();
        ```

    - 在 `ConfigureServices` 方法中调用 `AddAuthentication` 和 `AddCookie` 方法：

        ```csharp
        // If you don't want the cookie to be automatically authenticated and assigned to HttpContext.User,
        // remove the CookieAuthenticationDefaults.AuthenticationScheme parameter passed to AddAuthentication.
        services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
                .AddCookie(options =>
                {
                    options.LoginPath = "/Account/LogIn";
                    options.LogoutPath = "/Account/LogOff";
                });
        ```

### <a name="jwt-bearer-authentication"></a>JWT 持有者身份验证

在*Startup.cs*中进行以下更改：
- 用 `UseAuthentication`替换 `Configure` 方法中的 `UseJwtBearerAuthentication` 方法调用：

    ```csharp
    app.UseAuthentication();
    ```

- 在 `ConfigureServices` 方法中调用 `AddJwtBearer` 方法：

    ```csharp
    services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
            .AddJwtBearer(options =>
            {
                options.Audience = "http://localhost:5001/";
                options.Authority = "http://localhost:5000/";
            });
    ```

    此代码片段不使用标识，因此应通过将 `JwtBearerDefaults.AuthenticationScheme` 传递到 `AddAuthentication` 方法来设置默认方案。

### <a name="openid-connect-oidc-authentication"></a>OpenID Connect （OIDC）身份验证

在*Startup.cs*中进行以下更改：

- 用 `UseAuthentication`替换 `Configure` 方法中的 `UseOpenIdConnectAuthentication` 方法调用：

    ```csharp
    app.UseAuthentication();
    ```

- 在 `ConfigureServices` 方法中调用 `AddOpenIdConnect` 方法：

    ```csharp
    services.AddAuthentication(options =>
    {
        options.DefaultScheme = CookieAuthenticationDefaults.AuthenticationScheme;
        options.DefaultChallengeScheme = OpenIdConnectDefaults.AuthenticationScheme;
    })
    .AddCookie()
    .AddOpenIdConnect(options =>
    {
        options.Authority = Configuration["auth:oidc:authority"];
        options.ClientId = Configuration["auth:oidc:clientid"];
    });
    ```

- 将 `OpenIdConnectOptions` 操作中的 `PostLogoutRedirectUri` 属性替换为 `SignedOutRedirectUri`：

    ```csharp
    .AddOpenIdConnect(options =>
    {
        options.SignedOutRedirectUri = "https://contoso.com";
    });
    ```
    
### <a name="facebook-authentication"></a>Facebook 身份验证

在*Startup.cs*中进行以下更改：
- 用 `UseAuthentication`替换 `Configure` 方法中的 `UseFacebookAuthentication` 方法调用：

    ```csharp
    app.UseAuthentication();
    ```

- 在 `ConfigureServices` 方法中调用 `AddFacebook` 方法：

    ```csharp
    services.AddAuthentication()
            .AddFacebook(options =>
            {
                options.AppId = Configuration["auth:facebook:appid"];
                options.AppSecret = Configuration["auth:facebook:appsecret"];
            });
    ```

### <a name="google-authentication"></a>Google 身份验证

在*Startup.cs*中进行以下更改：
- 用 `UseAuthentication`替换 `Configure` 方法中的 `UseGoogleAuthentication` 方法调用：

    ```csharp
    app.UseAuthentication();
    ```

- 在 `ConfigureServices` 方法中调用 `AddGoogle` 方法：

    ```csharp
    services.AddAuthentication()
            .AddGoogle(options =>
            {
                options.ClientId = Configuration["auth:google:clientid"];
                options.ClientSecret = Configuration["auth:google:clientsecret"];
            });
    ```

### <a name="microsoft-account-authentication"></a>Microsoft 帐户身份验证

有关 Microsoft 帐户身份验证的详细信息，请参阅[此 GitHub 问题](https://github.com/aspnet/AspNetCore.Docs/issues/14455)。

在*Startup.cs*中进行以下更改：
- 用 `UseAuthentication`替换 `Configure` 方法中的 `UseMicrosoftAccountAuthentication` 方法调用：

    ```csharp
    app.UseAuthentication();
    ```

- 在 `ConfigureServices` 方法中调用 `AddMicrosoftAccount` 方法：

    ```csharp
    services.AddAuthentication()
            .AddMicrosoftAccount(options =>
            {
                options.ClientId = Configuration["auth:microsoft:clientid"];
                options.ClientSecret = Configuration["auth:microsoft:clientsecret"];
            });
    ```

### <a name="twitter-authentication"></a>Twitter 身份验证

在*Startup.cs*中进行以下更改：
- 用 `UseAuthentication`替换 `Configure` 方法中的 `UseTwitterAuthentication` 方法调用：

    ```csharp
    app.UseAuthentication();
    ```

- 在 `ConfigureServices` 方法中调用 `AddTwitter` 方法：

    ```csharp
    services.AddAuthentication()
            .AddTwitter(options =>
            {
                options.ConsumerKey = Configuration["auth:twitter:consumerkey"];
                options.ConsumerSecret = Configuration["auth:twitter:consumersecret"];
            });
    ```

### <a name="setting-default-authentication-schemes"></a>设置默认身份验证方案

在1.x 中， [AuthenticationOptions](/dotnet/api/Microsoft.AspNetCore.Builder.AuthenticationOptions?view=aspnetcore-1.1)基类的 `AutomaticAuthenticate` 和 `AutomaticChallenge` 属性应在单个身份验证方案上进行设置。 没有正确的方法来强制执行此操作。

在2.0 中，已将这两个属性作为单独 `AuthenticationOptions` 实例的属性删除。 可以在*Startup.cs*的 `ConfigureServices` 方法中的 `AddAuthentication` 方法调用中配置它们：

```csharp
services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme);
```

在上面的代码段中，默认方案设置为 `CookieAuthenticationDefaults.AuthenticationScheme` （"Cookie"）。

或者，使用 `AddAuthentication` 方法的重载版本设置多个属性。 在下面的重载方法示例中，将默认方案设置为 `CookieAuthenticationDefaults.AuthenticationScheme`。 可以在单独的 `[Authorize]` 属性或授权策略中指定身份验证方案。

```csharp
services.AddAuthentication(options =>
{
    options.DefaultScheme = CookieAuthenticationDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = OpenIdConnectDefaults.AuthenticationScheme;
});
```

如果满足以下条件之一，则定义2.0 中的默认方案：
- 希望用户自动登录
- 在不指定方案的情况下使用 `[Authorize]` 属性或授权策略

此规则的例外情况是 `AddIdentity` 方法。 此方法为你添加 cookie，并将默认身份验证和质询方案设置为应用程序 cookie `IdentityConstants.ApplicationScheme`。 此外，它还会将默认登录方案设置为外部 cookie `IdentityConstants.ExternalScheme`。

<a name="obsolete-interface"></a>

## <a name="use-httpcontext-authentication-extensions"></a>使用 HttpContext 身份验证扩展

`IAuthenticationManager` 接口是到1.x 身份验证系统的主要入口点。 它已替换为 `Microsoft.AspNetCore.Authentication` 命名空间中的一组新的 `HttpContext` 扩展方法。

例如，1.x 项目引用 `Authentication` 属性：

[!code-csharp[](../1x-to-2x/samples/AspNetCoreDotNetCore1App/AspNetCoreDotNetCore1App/Controllers/AccountController.cs?name=snippet_AuthenticationProperty)]

在2.0 项目中，导入 `Microsoft.AspNetCore.Authentication` 命名空间，并删除 `Authentication` 属性引用：

[!code-csharp[](../1x-to-2x/samples/AspNetCoreDotNetCore2App/AspNetCoreDotNetCore2App/Controllers/AccountController.cs?name=snippet_AuthenticationProperty)]

<a name="windows-auth-changes"></a>

## <a name="windows-authentication-httpsys--iisintegration"></a>Windows 身份验证（http.sys/IISIntegration）

Windows 身份验证有两种变体：

* 该主机仅允许经过身份验证的用户。 此变体不受2.0 更改的影响。
* 宿主允许匿名用户和经过身份验证的用户。 此变体受2.0 更改的影响。 例如，应用程序应允许[IIS](xref:host-and-deploy/iis/index)或[http.sys](xref:fundamentals/servers/httpsys)层上的匿名用户，但在控制器级别对用户进行授权。 在此方案中，在 `Startup.ConfigureServices` 方法中设置默认方案。

  对于[AspNetCore](https://www.nuget.org/packages/Microsoft.AspNetCore.Server.IISIntegration/)，请将默认方案设置为 `IISDefaults.AuthenticationScheme`：

  ```csharp
  using Microsoft.AspNetCore.Server.IISIntegration;

  services.AddAuthentication(IISDefaults.AuthenticationScheme);
  ```

  对于[AspNetCore](https://www.nuget.org/packages/Microsoft.AspNetCore.Server.HttpSys/)，请将默认方案设置为 `HttpSysDefaults.AuthenticationScheme`：

  ```csharp
  using Microsoft.AspNetCore.Server.HttpSys;

  services.AddAuthentication(HttpSysDefaults.AuthenticationScheme);
  ```

  未能设置默认方案会阻止授权（质询）请求使用以下例外：

  > `System.InvalidOperationException`：未指定任何 authenticationScheme，并且找不到 DefaultChallengeScheme。

有关更多信息，请参见<xref:security/authentication/windowsauth>。

<a name="identity-cookie-options"></a>

## <a name="identitycookieoptions-instances"></a>IdentityCookieOptions 实例

2\.0 更改的副作用是切换到使用命名选项而不是 cookie 选项实例。 将删除自定义标识 cookie 方案名称的功能。

例如，1.x 项目使用[构造函数注入](xref:mvc/controllers/dependency-injection#constructor-injection)将 `IdentityCookieOptions` 参数传递到*AccountController.cs*和*ManageController.cs*中。 可从提供的实例中访问外部 cookie 身份验证方案：

[!code-csharp[](../1x-to-2x/samples/AspNetCoreDotNetCore1App/AspNetCoreDotNetCore1App/Controllers/AccountController.cs?name=snippet_AccountControllerConstructor&highlight=4,11)]

上述构造函数注入在2.0 项目中变得不必要，可以删除 `_externalCookieScheme` 字段：

[!code-csharp[](../1x-to-2x/samples/AspNetCoreDotNetCore2App/AspNetCoreDotNetCore2App/Controllers/AccountController.cs?name=snippet_AccountControllerConstructor)]

1.x 项目使用 `_externalCookieScheme` 字段，如下所示：

[!code-csharp[](../1x-to-2x/samples/AspNetCoreDotNetCore1App/AspNetCoreDotNetCore1App/Controllers/AccountController.cs?name=snippet_AuthenticationProperty)]

在2.0 项目中，将前面的代码替换为以下代码。 可以直接使用 `IdentityConstants.ExternalScheme` 常量。

[!code-csharp[](../1x-to-2x/samples/AspNetCoreDotNetCore2App/AspNetCoreDotNetCore2App/Controllers/AccountController.cs?name=snippet_AuthenticationProperty)]

导入以下命名空间，解析新添加的 `SignOutAsync` 调用：

[!code-csharp[](../1x-to-2x/samples/AspNetCoreDotNetCore2App/AspNetCoreDotNetCore2App/Controllers/AccountController.cs?name=snippet_AuthenticationImport)]

<a name="navigation-properties"></a>

## <a name="add-identityuser-poco-navigation-properties"></a>添加 IdentityUser POCO 导航属性

已删除基 `IdentityUser` POCO （普通旧 CLR 对象）的实体框架（EF）核心导航属性。 如果你的1.x 项目使用这些属性，请将其手动添加回2.0 项目：

```csharp
/// <summary>
/// Navigation property for the roles this user belongs to.
/// </summary>
public virtual ICollection<IdentityUserRole<int>> Roles { get; } = new List<IdentityUserRole<int>>();

/// <summary>
/// Navigation property for the claims this user possesses.
/// </summary>
public virtual ICollection<IdentityUserClaim<int>> Claims { get; } = new List<IdentityUserClaim<int>>();

/// <summary>
/// Navigation property for this users login accounts.
/// </summary>
public virtual ICollection<IdentityUserLogin<int>> Logins { get; } = new List<IdentityUserLogin<int>>();
```

若要防止重复的外键，运行 EF Core 迁移时，将以下代码添加到你`IdentityDbContext`类的`OnModelCreating`方法 (后`base.OnModelCreating();`调用):

```csharp
protected override void OnModelCreating(ModelBuilder builder)
{
    base.OnModelCreating(builder);
    // Customize the ASP.NET Core Identity model and override the defaults if needed.
    // For example, you can rename the ASP.NET Core Identity table names and more.
    // Add your customizations after calling base.OnModelCreating(builder);

    builder.Entity<ApplicationUser>()
        .HasMany(e => e.Claims)
        .WithOne()
        .HasForeignKey(e => e.UserId)
        .IsRequired()
        .OnDelete(DeleteBehavior.Cascade);

    builder.Entity<ApplicationUser>()
        .HasMany(e => e.Logins)
        .WithOne()
        .HasForeignKey(e => e.UserId)
        .IsRequired()
        .OnDelete(DeleteBehavior.Cascade);

    builder.Entity<ApplicationUser>()
        .HasMany(e => e.Roles)
        .WithOne()
        .HasForeignKey(e => e.UserId)
        .IsRequired()
        .OnDelete(DeleteBehavior.Cascade);
}
```

<a name="synchronous-method-removal"></a>

## <a name="replace-getexternalauthenticationschemes"></a>替换 GetExternalAuthenticationSchemes

已删除同步方法 `GetExternalAuthenticationSchemes` 以支持异步版本。 1.x 项目的*控制器/ManageController*中包含以下代码：

[!code-csharp[](../1x-to-2x/samples/AspNetCoreDotNetCore1App/AspNetCoreDotNetCore1App/Controllers/ManageController.cs?name=snippet_GetExternalAuthenticationSchemes)]

此方法也出现在*Views/Account/Login 中。 cshtml* ：

[!code-cshtml[](../1x-to-2x/samples/AspNetCoreDotNetCore1App/AspNetCoreDotNetCore1App/Views/Account/Login.cshtml?name=snippet_GetExtAuthNSchemes&highlight=2)]

在2.0 项目中，使用 <xref:Microsoft.AspNetCore.Identity.SignInManager`1.GetExternalAuthenticationSchemesAsync*> 方法。 *ManageController.cs*中的更改类似于以下代码：

[!code-csharp[](../1x-to-2x/samples/AspNetCoreDotNetCore2App/AspNetCoreDotNetCore2App/Controllers/ManageController.cs?name=snippet_GetExternalAuthenticationSchemesAsync)]

在*Login*中，`foreach` 循环中访问的 `AuthenticationScheme` 属性更改为 `Name`：

[!code-cshtml[](../1x-to-2x/samples/AspNetCoreDotNetCore2App/AspNetCoreDotNetCore2App/Views/Account/Login.cshtml?name=snippet_GetExtAuthNSchemesAsync&highlight=2,19)]

<a name="property-change"></a>

## <a name="manageloginsviewmodel-property-change"></a>ManageLoginsViewModel 属性更改

`ManageLoginsViewModel` 对象用于*ManageController.cs*的 `ManageLogins` 操作。 在1.x 项目中，对象的 `OtherLogins` 属性返回类型为 `IList<AuthenticationDescription>`。 此返回类型需要 `Microsoft.AspNetCore.Http.Authentication`的导入：

[!code-csharp[](../1x-to-2x/samples/AspNetCoreDotNetCore1App/AspNetCoreDotNetCore1App/Models/ManageViewModels/ManageLoginsViewModel.cs?name=snippet_ManageLoginsViewModel&highlight=2,11)]

在2.0 项目中，返回类型更改为 `IList<AuthenticationScheme>`。 这一新的返回类型需要使用 `Microsoft.AspNetCore.Authentication` 导入替换 `Microsoft.AspNetCore.Http.Authentication` 导入。

[!code-csharp[](../1x-to-2x/samples/AspNetCoreDotNetCore2App/AspNetCoreDotNetCore2App/Models/ManageViewModels/ManageLoginsViewModel.cs?name=snippet_ManageLoginsViewModel&highlight=2,11)]

<a name="additional-resources"></a>

## <a name="additional-resources"></a>其他资源

有关详细信息，请参阅 GitHub 上[的有关 Auth 2.0](https://github.com/aspnet/Security/issues/1338)问题的讨论。
