---
title: ASP.NET Core Blazor 宿主模型
author: guardrex
description: 了解 Blazor WebAssembly 和 Blazor 服务器托管模型。
monikerRange: '>= aspnetcore-3.1'
ms.author: riande
ms.custom: mvc
ms.date: 02/12/2020
no-loc:
- Blazor
- SignalR
uid: blazor/hosting-models
ms.openlocfilehash: 54be0e032a60c69880f428e52f9d778032385dc5
ms.sourcegitcommit: 6645435fc8f5092fc7e923742e85592b56e37ada
ms.translationtype: MT
ms.contentlocale: zh-CN
ms.lasthandoff: 02/19/2020
ms.locfileid: "77447043"
---
# <a name="aspnet-core-opno-locblazor-hosting-models"></a>ASP.NET Core Blazor 宿主模型

作者： [Daniel Roth](https://github.com/danroth27)

[!INCLUDE[](~/includes/blazorwasm-preview-notice.md)]

Blazor 是一种 web 框架，旨在在基于[WebAssembly](https://webassembly.org/)的 .net 运行时（ *Blazor WebAssembly*）上的浏览器中运行客户端，或者在 ASP.NET Core （ *Blazor 服务器*）中运行服务器端。 无论采用何种托管模型，应用和组件模型*都是相同*的。

若要为本文中所述的托管模型创建项目，请参阅 <xref:blazor/get-started>。

有关高级配置，请参阅 <xref:blazor/hosting-model-configuration>。

## <a name="opno-locblazor-webassembly"></a>Blazor WebAssembly

Blazor 的主体宿主模型在 WebAssembly 上的浏览器中运行客户端。 将 Blazor 应用、其依赖项以及 .NET 运行时下载到浏览器。 应用将在浏览器线程中直接执行。 UI 更新和事件处理发生在同一进程中。 应用的资产将作为静态文件部署到 web 服务器或可为客户端提供静态内容的服务。

![[!基金.非 LOC （Blazor）] WebAssembly： [！基金.无 LOC （Blazor）] 应用在浏览器内的 UI 线程上运行。](hosting-models/_static/blazor-webassembly.png)

若要使用客户端托管模型创建 Blazor 应用，请使用 **Blazor WebAssembly 应用**模板（[dotnet new blazorwasm](/dotnet/core/tools/dotnet-new)）。

选择 **Blazor WebAssembly 应用程序**模板之后，你可以选择将应用配置为使用 ASP.NET Core 后端，方法是选中 " **ASP.NET Core 托管**" 复选框（"[dotnet new blazorwasm](/dotnet/core/tools/dotnet-new)"）。 ASP.NET Core 应用将 Blazor 应用程序提供给客户端。 Blazor WebAssembly 应用可通过网络使用 web API 调用或[SignalR](xref:signalr/introduction) （<xref:tutorials/signalr-blazor-webassembly>）与服务器交互。

模板包括处理的 `blazor.webassembly.js` 脚本：

* 下载 .NET 运行时、应用程序和应用程序的依赖项。
* 用于运行应用程序的运行时初始化。

Blazor WebAssembly 宿主模型具有以下几个优点：

* 没有 .NET 服务器端依赖项。 应用在下载到客户端之后完全正常运行。
* 完全利用客户端资源和功能。
* 工作从服务器卸载到客户端。
* 不需要 ASP.NET Core web 服务器来托管应用程序。 无服务器部署方案可能（例如，通过 CDN 提供应用）。

Blazor WebAssembly 托管有一些缺点：

* 应用程序限制为浏览器的功能。
* 需要支持的客户端硬件和软件（例如，WebAssembly 支持）。
* 下载大小较大，应用需要较长时间才能加载。
* .NET 运行时和工具支持不太成熟。 例如， [.NET Standard](/dotnet/standard/net-standard)支持和调试中存在限制。

## <a name="opno-locblazor-server"></a>Blazor 服务器

使用 Blazor 服务器托管模型，可在服务器上从 ASP.NET Core 应用中执行应用。 UI 更新、事件处理和 JavaScript 调用是通过 [SignalR](xref:signalr/introduction) 连接进行处理。

![浏览器与应用程序（托管在 ASP.NET Core 应用内的应用程序）通过 [！基金.无 LOC （SignalR）] 连接。](hosting-models/_static/blazor-server.png)

若要使用 Blazor 服务器托管模型创建 Blazor 应用，请使用 ASP.NET Core **Blazor 服务器应用**模板（[dotnet new blazorserver](/dotnet/core/tools/dotnet-new)）。 ASP.NET Core 应用承载 Blazor 服务器应用，并创建客户端连接到 SignalR 终结点。

ASP.NET Core 应用引用要添加的应用 `Startup` 类：

* 服务器端服务。
* 请求处理管道的应用。

`blazor.server.js` 脚本&dagger; 建立客户端连接。 应用负责根据需要保存和还原应用状态（例如，在网络连接丢失的情况下）。

Blazor Server 宿主模型具有以下几个优点：

* 下载大小明显小于 Blazor WebAssembly 应用，且应用加载速度快得多。
* 应用充分利用服务器功能，包括使用任何与 .NET Core 兼容的 Api。
* 服务器上的 .NET Core 用于运行应用程序，因此现有的 .NET 工具（如调试）可按预期方式工作。
* 支持瘦客户端。 例如，Blazor Server apps 适用于不支持 WebAssembly 的浏览器以及资源受限设备上的浏览器。
* 应用程序的 .NET/C#代码库（包括应用程序的组件代码）不会提供给客户端。

Blazor Server 宿主有一些缺点：

* 通常存在较高的延迟。 每个用户交互都涉及网络跃点。
* 无脱机支持。 如果客户端连接失败，应用将停止工作。
* 对于包含多个用户的应用而言，可伸缩性非常困难。 服务器必须管理多个客户端连接并处理客户端状态。
* 为应用提供服务需要 ASP.NET Core 服务器。 不可能的无服务器部署方案（例如，通过 CDN 为应用提供服务）。

&dagger;`blazor.server.js` 脚本是从 ASP.NET Core 共享框架中的嵌入资源提供的。

### <a name="comparison-to-server-rendered-ui"></a>与服务器呈现的 UI 的比较

了解 Blazor Server 应用程序的一种方法是，了解它与传统模型的不同之处在于，使用 Razor 视图或 Razor Pages 在 ASP.NET Core 应用程序中呈现 UI。 这两种模型都使用 Razor 语言来描述 HTML 内容，但在显示标记的方式上差别很大。

在显示 Razor 页面或视图时，每行 Razor 代码都会以文本形式发出 HTML。 在呈现后，服务器将释放页面或视图实例，包括生成的任何状态。 当对该页的另一请求出现时，例如，当服务器验证失败并显示验证摘要时：

* 整个页面将再次重新呈现到 HTML 文本。
* 该页将发送到客户端。

Blazor 应用由 UI 的可重用元素组成，这些元素称为*组件*。 组件包含C#代码、标记和其他组件。 呈现组件时，Blazor 生成包含的组件的图，类似于 HTML 或 XML 文档对象模型（DOM）。 此图包含属性和字段中保存的组件状态。 Blazor 计算组件图以生成标记的二进制表示形式。 二进制格式可以是：

* 转换为 HTML 文本（在预呈现&dagger;期间）。
* 用于在常规呈现期间有效地更新标记。

&dagger;预*呈现*&ndash; 请求的 Razor 组件在服务器上编译为静态 HTML 并发送到客户端，并在其中呈现给用户。 在客户端与服务器之间建立连接后，会将组件的静态预呈现元素替换为交互式元素。 预呈现使应用对用户的响应速度更快。

Blazor 中的 UI 更新由以下用户触发：

* 用户交互，如选择一个按钮。
* 应用触发器，如计时器。

关系图为重新呈现，并计算了 UI*差异*（差异）。 这种差异是更新客户端上 UI 所需的最小 DOM 编辑集。 将以二进制格式将差异发送到客户端，并由浏览器应用。

当用户在客户端上导航掉组件后，将释放该组件。 当用户与组件交互时，组件的状态（服务、资源）必须保存在服务器的内存中。 由于多个组件的状态可能同时由服务器维护，因此内存耗尽是必须解决的问题。 有关如何创作 Blazor Server 应用程序以确保最大程度地使用服务器内存的指导，请参阅 <xref:security/blazor/server>。

### <a name="circuits"></a>而言

Blazor Server 应用是在[ASP.NET Core SignalR](xref:signalr/introduction)的基础上构建的。 每个客户端通过一个或多个称为*线路*的 SignalR 连接与服务器通信。 线路是对可容忍暂时网络中断的 SignalR 连接 Blazor抽象。 当 Blazor 客户端发现 SignalR 连接已断开连接时，它会尝试使用新的 SignalR 连接重新连接到服务器。

连接到 Blazor Server 应用程序的每个浏览器屏幕（浏览器选项卡或 iframe）都使用 SignalR 连接。 与典型服务器呈现的应用相比，这一点还有一个重要的区别。 在服务器呈现的应用程序中，在多个浏览器屏幕中打开同一应用程序通常不会转换为服务器上的其他资源需求。 在 Blazor Server 应用程序中，每个浏览器屏幕都需要单独的线路和组件状态的单独实例，由服务器来管理。

Blazor 考虑关闭浏览器选项卡或导航到外部 URL，*正常*终止。 如果正常终止，则会立即释放线路和关联的资源。 例如，由于网络中断，客户端也可能断开连接。 Blazor 服务器会将断开连接的线路存储为可配置的间隔，以允许客户端重新连接。

### <a name="ui-latency"></a>UI 延迟

UI 延迟是指从启动的操作到 UI 更新的时间。 对于应用程序来说，更小的 UI 延迟值非常适合应用程序对用户的响应。 在 Blazor 服务器应用中，每个操作都将发送到服务器并进行处理，并将向后发送 UI 差异。 因此，UI 延迟是网络延迟和处理操作时服务器延迟的总和。

对于仅限于专用公司网络的业务线应用，对用户而言，由于网络延迟导致的延迟通常是让的。 对于通过 Internet 部署的应用，用户可能会对延迟造成明显的影响，尤其是用户广泛分散于各地。

内存使用率还会导致应用延迟。 增加的内存使用会导致频繁垃圾收集或将内存分页到磁盘，这两者都会降低应用程序性能，进而增加 UI 延迟。 有关详细信息，请参阅 <xref:security/blazor/server>。

Blazor Server apps 应进行优化，以通过减少网络延迟和内存使用率来最大限度地减少 UI 延迟。 有关测量网络延迟的方法，请参阅 <xref:host-and-deploy/blazor/server#measure-network-latency>。 有关 SignalR 和 Blazor的详细信息，请参阅：

* <xref:host-and-deploy/blazor/server>
* <xref:security/blazor/server>

### <a name="connection-to-the-server"></a>与服务器的连接

Blazor Server 应用需要与服务器建立活动的 SignalR 连接。 如果连接丢失，应用会尝试重新连接到服务器。 只要客户端的状态仍在内存中，客户端会话便会恢复而不会失去状态。

Blazor Server 应用 prerenders 为响应第一个客户端请求，该请求在服务器上设置 UI 状态。 当客户端尝试创建 SignalR 连接时，客户端必须重新连接到同一服务器。 使用多台后端服务器 Blazor 服务器应用应为 SignalR 连接实现*粘滞会话*。

我们建议将 [Azure SignalR 服务](/azure/azure-signalr)用于 Blazor Server 应用。 该服务允许将 Blazor Server 应用扩展到大量并发 SignalR 连接。 可以通过将服务的 `ServerStickyMode` 选项或配置值设置为 `Required`，为 Azure SignalR 服务启用粘滞会话。 有关详细信息，请参阅 <xref:host-and-deploy/blazor/server#signalr-configuration>。

使用 IIS 时，粘滞会话通过应用程序请求路由启用。 有关详细信息，请参阅[使用应用程序请求路由实现 HTTP 负载均衡](/iis/extensions/configuring-application-request-routing-arr/http-load-balancing-using-application-request-routing)。

## <a name="additional-resources"></a>其他资源

* <xref:blazor/get-started>
* <xref:signalr/introduction>
* <xref:blazor/hosting-model-configuration>
* <xref:tutorials/signalr-blazor-webassembly>
