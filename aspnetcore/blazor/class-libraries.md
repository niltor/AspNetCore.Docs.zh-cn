---
title: ASP.NET Core Razor 组件类库
author: guardrex
description: 了解如何在来自外部组件库 Blazor 应用中包含组件。
monikerRange: '>= aspnetcore-3.1'
ms.author: riande
ms.custom: mvc
ms.date: 01/23/2020
no-loc:
- Blazor
- SignalR
uid: blazor/class-libraries
ms.openlocfilehash: 32088b43f91174596f6b9251d36782e806f966b9
ms.sourcegitcommit: d2ba66023884f0dca115ff010bd98d5ed6459283
ms.translationtype: MT
ms.contentlocale: zh-CN
ms.lasthandoff: 02/14/2020
ms.locfileid: "77213244"
---
# <a name="aspnet-core-razor-components-class-libraries"></a>ASP.NET Core Razor 组件类库

作者： [Simon Timms](https://github.com/stimms)

可在[Razor 类库（RCL）](xref:razor-pages/ui-class)中跨项目共享组件。 *Razor 组件类库*可以包含在其中：

* 解决方案中的另一个项目。
* NuGet 包。
* 引用的 .NET 库。

正如组件是常规 .NET 类型一样，RCL 提供的组件是普通的 .NET 程序集。

## <a name="create-an-rcl"></a>创建 RCL

按照 <xref:blazor/get-started> 一文中的指导配置 Blazor 的环境。

# <a name="visual-studiotabvisual-studio"></a>[Visual Studio](#tab/visual-studio)

1. 创建新项目。
1. 选择 " **Razor 类库**"。 选择 **“下一步”** 。
1. 在 "**创建新的 Razor 类库**" 对话框中，选择 "**创建**"。
1. 在“项目名称”字段提供项目名称，或接受默认项目名称。 本主题中的示例使用项目名称 `MyComponentLib1`。 选择 **“创建”** 。
1. 将 RCL 添加到解决方案：
   1. 右键单击解决方案。 选择 "**添加** > **现有项目**"。
   1. 导航到 RCL 的项目文件。
   1. 选择 RCL 的项目文件（ *.csproj*）。
1. 在应用中添加对 RCL 的引用：
   1. 右键单击应用程序项目。 选择 "**添加** > **引用**"。
   1. 选择 RCL 项目。 选择“确定”。

> [!NOTE]
> 如果在从模板生成 RCL 时选中 "**支持页和视图**" 复选框，则还会将 *_Imports*文件添加到生成的项目的根目录中，并提供以下内容以启用 razor 组件创作：
>
> ```razor
> @using Microsoft.AspNetCore.Components.Web
> ```
>
> 手动将该文件添加到生成的项目的根目录中。

# <a name="net-core-clitabnetcore-cli"></a>[.NET Core CLI](#tab/netcore-cli)

1. 在命令行界面中，将**Razor 类库**模板（`razorclasslib`）与[dotnet new](/dotnet/core/tools/dotnet-new)命令一起使用。 在下面的示例中，创建了一个名为 `MyComponentLib1`的 RCL。 执行命令时，将自动创建包含 `MyComponentLib1` 的文件夹：

   ```dotnetcli
   dotnet new razorclasslib -o MyComponentLib1
   ```

   > [!NOTE]
   > 如果从模板生成 RCL 时使用 `-s|--support-pages-and-views` 开关，则还应将 *_Imports*文件添加到生成的项目的根目录中，并提供以下内容以启用 razor 组件创作：
   >
   > ```razor
   > @using Microsoft.AspNetCore.Components.Web
   > ```
   >
   > 手动将该文件添加到生成的项目的根目录中。

1. 若要将库添加到现有项目，请在命令行界面中使用[dotnet add reference](/dotnet/core/tools/dotnet-add-reference)命令。 在下面的示例中，将 RCL 添加到应用中。 在应用程序的项目文件夹中执行以下命令，并在其中包含库的路径：

   ```dotnetcli
   dotnet add reference {PATH TO LIBRARY}
   ```

---

## <a name="consume-a-library-component"></a>使用库组件

若要使用另一个项目的库中定义的组件，请使用以下方法之一：

* 使用带有命名空间的完整类型名称。
* 使用 Razor [\@](xref:mvc/views/razor#using)的指令。 可以按名称添加单个组件。

在下面的示例中，`MyComponentLib1` 是包含 `SalesReport` 组件的组件库。

可以使用命名空间的完整类型名称引用 `SalesReport` 组件：

```razor
<h1>Hello, world!</h1>

Welcome to your new app.

<MyComponentLib1.SalesReport />
```

如果使用 `@using` 指令将库引入到范围中，也可以引用此组件：

```razor
@using MyComponentLib1

<h1>Hello, world!</h1>

Welcome to your new app.

<SalesReport />
```

将 `@using MyComponentLib1` 指令包含在顶级 *_Import*文件中，使库的组件可用于整个项目。 将指令添加到任何级别的 *_Import razor*文件，以将该命名空间应用于文件夹中的单个页面或一组页面。

## <a name="build-pack-and-ship-to-nuget"></a>生成、打包和传送到 NuGet

由于组件库是标准的 .NET 库，因此将其打包并传送到 NuGet 与将任何库打包到 NuGet 没有什么不同。 在命令行界面中使用[dotnet pack](/dotnet/core/tools/dotnet-pack)命令执行打包：

```dotnetcli
dotnet pack
```

在命令行界面中使用[dotnet nuget push](/dotnet/core/tools/dotnet-nuget-push)命令将包上传到 nuget。

## <a name="create-a-razor-components-class-library-with-static-assets"></a>使用静态资产创建 Razor 组件类库

RCL 可以包括静态资产。 静态资产可用于使用库的任何应用。 有关详细信息，请参阅 <xref:razor-pages/ui-class#create-an-rcl-with-static-assets>。

## <a name="additional-resources"></a>其他资源

* <xref:razor-pages/ui-class>
