## 多进程架构

本文介绍chromium的体系机构

## 问题

在某些方面，2006年左右的Web浏览器状态类似于过去的单用户，协作式多任务操作系统。由于在这样的操作系统中行为不端的应用程序可能会导致整个系统崩溃，因此Web浏览器中的行为不当网页也会崩溃。所需要的只是一个浏览器或插件错误，以关闭整个浏览器和所有当前运行的选项卡。

现代操作系统更加强大，因为它们将应用程序放入彼此隔离的单独进程中。一个应用程序中的崩溃通常不会损害其他应用程序或操作系统的完整性，并且每个用户对其他用户数据的访问受到限制。

## 架构概述

我们对浏览器选项卡使用单独的进程来保护整个应用程序免受渲染引擎中的错误和故障的影响。我们还限制从每个渲染引擎进程访问其他进程以及系统的其余部分。在某些方面，这为Web浏览带来了内存保护和访问控制为操作系统带来的好处。

我们指的是运行UI的主流程，并将选项卡和插件流程作为“浏览器流程”或“浏览器”进行管理。同样，特定于选项卡的进程称为“渲染进程”或“渲染器”。渲染器使用[Blink](https://www.chromium.org/blink)开源布局引擎来解释和布局HTML。

![架构](http://dev.chromium.org/_/rsrc/1220197832277/developers/design-documents/multi-process-architecture/arch.png)

### 管理渲染过程

每个渲染过程都有一个全局`RenderProcess`对象，用于管理与父浏览器进程的通信并维护全局状态。浏览器`RenderProcessHost` 为每个渲染过程维护一个对应的渲染过程，该过程管理渲染器的浏览器状态和通信。浏览器和渲染器使用[Chromium的IPC系统进行](http://dev.chromium.org/developers/design-documents/inter-process-communication)通信。

### 管理视图

每个渲染过程都有一个或多个`RenderView`对象，由对象管理`RenderProcess`，对应于内容选项卡。对应的`RenderProcessHost`维护`RenderViewHost` 对应于渲染器中的每个视图。每个视图都有一个视图ID，用于区分同一渲染器中的多个视图。这些ID在一个渲染器中是唯一的，但不在浏览器中，因此识别视图需要a `RenderProcessHost`和视图ID。从浏览器的内容的特定标签的通信通过这些完成的`RenderViewHost`对象，它知道如何通过邮件发送自己`RenderProcessHost`的`RenderProcess`，并到`RenderView`。

## 组件和接口

在渲染过程中：

- 在`RenderProcess`与相应的手柄IPC `RenderProcessHost`在浏览器中。`RenderProcess`每个渲染过程只有一个对象。这就是所有浏览器↔渲染器通信的发生方式。
- 该`RenderView`对象与`RenderViewHost`浏览器进程（通过RenderProcess）和我们的WebKit嵌入层进行通信。此对象表示选项卡或弹出窗口中一个网页的内容

在浏览器过程中：

- 该`Browser`对象表示顶级浏览器窗口。
- 该`RenderProcessHost`对象表示单个浏览器↔渲染器IPC连接的浏览器端。`RenderProcessHost`浏览器进程中有一个用于每个渲染过程。
- 该`RenderViewHost`对象封装的通信与远程`RenderView`和RenderWidgetHost处理用于输入和绘画RenderWidget在浏览器中。

有关此嵌入如何工作的更多详细信息，请参阅[Chromium如何显示网页](http://dev.chromium.org/developers/design-documents/displaying-a-web-page-in-chrome)  设计文档。



### 检测坠毁或行为不端的渲染器

每个与浏览器进程的IPC连接都会监视进程句柄。如果发出这些句柄的信号，则渲染过程已崩溃，并且会向选项卡通知崩溃。现在，我们会显示一个“悲伤标签”屏幕，通知用户渲染器已崩溃。按重新加载按钮或开始新的导航可以重新加载页面。发生这种情况时，我们注意到没有进程并创建一个新进程。

### 沙箱渲染器

鉴于渲染器在单独的进程中运行，我们有机会通过[沙盒](https://www.chromium.org/developers/design-documents/sandbox)限制其对系统资源的访问。例如，我们可以确保渲染器只能通过其父浏览器进程访问网络。同样，我们可以使用主机操作系统的内置权限来限制其对文件系统的访问。除了限制渲染器对文件系统和网络的访问外，我们还可以限制其对用户显示和相关对象的访问。我们在单独的Windows“ [桌面](https://msdn.microsoft.com/en-us/library/windows/desktop/ms682573(v=vs.85).aspx) ” 上运行每个渲染过程，这对用户是不可见的。这可以防止受损的渲染器打开新窗口或捕获击键。

### 回馈记忆

如果渲染器在单独的进程中运行，则将隐藏选项卡视为较低优先级变得简单明了。通常，Windows上最小化的进程会将其内存自动放入“可用内存”池中。在内存不足的情况下，Windows会在交换优先级较高的内存之前将此内存交换到磁盘，从而有助于保持用户可见程序的响应速度。我们可以将相同的原则应用于隐藏的标签。当渲染过程没有顶级选项卡时，我们可以释放该进程的“工作集”大小作为系统的提示，以便在必要时首先将该内存交换到磁盘。因为我们发现当用户在两个标签之间切换时减小工作集大小也会降低标签切换性能，我们会逐渐释放这个内存。这意味着如果用户切换回最近使用的选项卡，则该选项卡的内存比最近使用的选项卡更容易被分页。拥有足够内存来运行所有程序的用户根本不会注意到这个过程：Windows只会在需要时才会回收这些数据，因此在内存充足时不会有性能损失。这有助于我们在低内存情况下获得更佳的内存占用。与很少使用的背景标签相关联的内存可以完全换出，而前景标签的数据可以完全加载到内存中。相比之下，单进程浏览器将所有选项卡的数据随机分布在其内存中，并且不可能如此干净地分离已使用和未使用的数据，从而浪费内存和性能。插件和扩展Firefox风格的NPAPI插件在自己的进程中运行，与渲染器分开。这在[Plugin Architecture](http://dev.chromium.org/developers/design-documents/plugin-architecture)中有详细描述。 该[网站隔离](https://www.chromium.org/developers/design-documents/site-isolation)项目的目的是在渲染器之间提供更多的隔离，对于这个项目的早期交付包括在孤立进程中运行Chrome的HTML / JavaScript内容扩展。