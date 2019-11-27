# <a name="visual-studiotabvisual-studio"></a>[<span data-ttu-id="ae7bb-101">Visual Studio</span><span class="sxs-lookup"><span data-stu-id="ae7bb-101">Visual Studio</span></span>](#tab/visual-studio)

* <span data-ttu-id="ae7bb-102">按 Ctrl+F5 以在不使用调试程序的情况下运行。</span><span class="sxs-lookup"><span data-stu-id="ae7bb-102">Press Ctrl+F5 to run without the debugger.</span></span>

  [!INCLUDE[](~/includes/trustCertVS.md)]

  <span data-ttu-id="ae7bb-103">Visual Studio 启动 [IIS Express](/iis/extensions/introduction-to-iis-express/iis-express-overview) 并运行应用。</span><span class="sxs-lookup"><span data-stu-id="ae7bb-103">Visual Studio starts [IIS Express](/iis/extensions/introduction-to-iis-express/iis-express-overview) and runs the app.</span></span> <span data-ttu-id="ae7bb-104">地址栏显示 `localhost:port#`，而不是显示 `example.com`。</span><span class="sxs-lookup"><span data-stu-id="ae7bb-104">The address bar shows `localhost:port#` and not something like `example.com`.</span></span> <span data-ttu-id="ae7bb-105">这是因为 `localhost` 是本地计算机的标准主机名。</span><span class="sxs-lookup"><span data-stu-id="ae7bb-105">That's because `localhost` is the standard hostname for the local computer.</span></span> <span data-ttu-id="ae7bb-106">Localhost 仅为来自本地计算机的 Web 请求提供服务。</span><span class="sxs-lookup"><span data-stu-id="ae7bb-106">Localhost only serves web requests from the local computer.</span></span> <span data-ttu-id="ae7bb-107">Visual Studio 创建 Web 项目时，Web 服务器使用的是随机端口。</span><span class="sxs-lookup"><span data-stu-id="ae7bb-107">When Visual Studio creates a web project, a random port is used for the web server.</span></span>
 
# <a name="visual-studio-codetabvisual-studio-code"></a>[<span data-ttu-id="ae7bb-108">Visual Studio Code</span><span class="sxs-lookup"><span data-stu-id="ae7bb-108">Visual Studio Code</span></span>](#tab/visual-studio-code)

  [!INCLUDE[](~/includes/trustCertVSC.md)]

* <span data-ttu-id="ae7bb-109">按 **Ctrl-F5** 以在不使用调试程序的情况下运行。</span><span class="sxs-lookup"><span data-stu-id="ae7bb-109">Press **Ctrl-F5** to run without the debugger.</span></span>

  <span data-ttu-id="ae7bb-110">Visual Studio Code 启动 [Kestrel](xref:fundamentals/servers/kestrel)，启动浏览器并导航到 `http://localhost:5001`。</span><span class="sxs-lookup"><span data-stu-id="ae7bb-110">Visual Studio Code starts [Kestrel](xref:fundamentals/servers/kestrel), launches a browser, and navigates to `http://localhost:5001`.</span></span> <span data-ttu-id="ae7bb-111">地址栏显示 `localhost:port#`，而不是显示 `example.com`。</span><span class="sxs-lookup"><span data-stu-id="ae7bb-111">The address bar shows `localhost:port#` and not something like `example.com`.</span></span> <span data-ttu-id="ae7bb-112">这是因为 `localhost` 是本地计算机的标准主机名。</span><span class="sxs-lookup"><span data-stu-id="ae7bb-112">That's because `localhost` is the standard hostname for  local computer.</span></span> <span data-ttu-id="ae7bb-113">Localhost 仅为来自本地计算机的 Web 请求提供服务。</span><span class="sxs-lookup"><span data-stu-id="ae7bb-113">Localhost only serves web requests from the local computer.</span></span>

  
# <a name="visual-studio-for-mactabvisual-studio-mac"></a>[<span data-ttu-id="ae7bb-114">Visual Studio for Mac</span><span class="sxs-lookup"><span data-stu-id="ae7bb-114">Visual Studio for Mac</span></span>](#tab/visual-studio-mac)

  [!INCLUDE[](~/includes/trustCertMac.md)]

* <span data-ttu-id="ae7bb-115">在 Visual Studio 中，按 Alt-Cmd-Enter 可在不使用调试器的情况下运行  。</span><span class="sxs-lookup"><span data-stu-id="ae7bb-115">From Visual Studio, press **Alt-Cmd-Enter** to run without the debugger.</span></span> <span data-ttu-id="ae7bb-116">或者，导航到菜单栏，转到“运行”>“在不调试的情况下启动”  。</span><span class="sxs-lookup"><span data-stu-id="ae7bb-116">Alternatively, navigate to the menu bar and go to **Run>Start Without Debugging**.</span></span>

  <span data-ttu-id="ae7bb-117">Visual Studio 启动 [Kestrel](xref:fundamentals/servers/kestrel)，启动浏览器并导航到 `http://localhost:5001`。</span><span class="sxs-lookup"><span data-stu-id="ae7bb-117">Visual Studio starts [Kestrel](xref:fundamentals/servers/kestrel), launches a browser, and navigates to `http://localhost:5001`.</span></span>

<!-- End of VS tabs -->

---