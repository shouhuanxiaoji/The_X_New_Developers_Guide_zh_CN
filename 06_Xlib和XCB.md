Xlib 和 XCB
==========

*Alan Coopersmith*

本文英文原版，按章节链接
1. [Example: Converting xwininfo from Xlib to XCB](https://www.x.org/wiki/guide/xlib-and-xcb/#index1h2)
2. [Mixing Xlib & XCB calls](https://www.x.org/wiki/guide/xlib-and-xcb/#index2h2)
3. [Example: Converting xdpyinfo extension queries to XCB](https://www.x.org/wiki/guide/xlib-and-xcb/#index3h2)
4. [Extension libraries](https://www.x.org/wiki/guide/xlib-and-xcb/#index4h2)
5. [API documentation](https://www.x.org/wiki/guide/xlib-and-xcb/#index5h2)

关于Xlib和XCB，两个最常见的问题可能是“它们是什么？”以及“有什么区别？”

大多数编程语言使得 X 应用程序很难在网络中使用原始的 X 协议，并将返回的协议分解。因此，如果某些库能够处理这些任务，并提供适合于连接到 X 服务器的编程语言和环境的 API，那么编写 X 工具包和应用程序就会容易得多。

在 X 客户机库堆栈的底层是 Xlib 和 XCB，这两个帮助器库(实际上是一组库)提供了与 X 服务器对话的 API。Xlib 和 XCB 有不同的设计目标，它们是在 X Window 系统演变的不同时期开发的。

大多数应用程序开发人员应该谨慎地调用 Xlib 和 XCB。更高级别的工具包提供了更高效的编程模型，并支持现代应用程序所期望的特性，包括对复杂的国际化输入和输出、可访问性以及与桌面环境的集成的支持。然而，有时候应用程序会发现自己需要调用原始的底层 X11库来执行工具包不支持的操作。应用程序可能需要调用工具箱当前版本的编程模型中没有涉及的 X 扩展 API。绘图不被工具箱 API 包装的情况也很常见。什么事？—— po8]

最初的 C 语言 X11API 是 libX11，通常称为“ Xlib”。它被设计成类似于传统的库 API，隐藏了调用导致对服务器的协议请求的事实。不需要来自 X 服务器的响应的调用在缓冲区中排队，作为请求批量发送到服务器。那些需要响应的请求刷新所有缓冲请求，然后阻塞，直到接收到响应。

Xlib 的同步和异步行为的混合导致了一些问题。Xlib 的行为经常让新程序员感到困惑。调用有时候看起来可以工作，而其他时候则不能，因为哪个调用隐式地刷新缓冲区并不明显。许多调用的异步特性使得调试问题变得困难。当报告错误时，堆栈跟踪显示接收和处理错误时正在进行的调用，通常是在导致错误的调用之后进行的多次调用。最后，Xlib 的同步调用会产生可避免的往返延迟。这种延迟对应用程序性能有显著影响; 特别是，启动时间通常会大大增加。

经过多年使用 Xlib 的经验，并从它和其他协议接口库中学习，我们再次尝试为 x11定义一个 C 语言绑定: “ X11 c 绑定”层 XCB。XCB 在其设计中明确了协议的客户机-服务器性质。客户机负责决定何时刷新请求缓冲区、何时读取结果以及何时等待服务器响应。

例如，要查找窗口属性，Xlib 代码是一个函数调用:
```
XGetWindowProperty(dpy, win, atom, 0, 0, False, AnyPropertyType, &type_ret, &format_ret, &num_ret, &bytes_after, &prop_ret);
```

Xlib 向 X 服务器生成请求以检索属性，并将其追加到请求的缓冲区中。因为这是一个需要响应的请求，所以 Xlib 随后刷新缓冲区，将内容发送到 X 服务器。接下来，Xlib 等待 X 服务器处理属性检索请求之前的所有请求，并发送属性检索回复。然后 Xlib 将响应返回给客户端。

Xlib 还提供了包装属性请求的方便函数。这些方便的函数检索特定的属性，了解每个属性的详细信息以及如何请求和解码它。示例包括 XGetWMName 和 XGetWMHint。其中一些函数可以在 Xlib 之外编写，但是许多函数以非常重要的方式使用 Xlib 内部函数，因此是不可分割的。什么事？—— po8]

另一方面，XCB 以一种“明显的”机械方式提供直接从协议描述生成的函数。XCB 函数直接映射到协议，使用单独的函数将请求放入传出缓冲区，并在稍后从 X 服务器异步读取结果。上述代码的 XCB 版本是:
```
prop_cookie = xcb_get_property (dpy, False, win, atom, XCB_GET_PROPERTY_TYPE_ANY, 0, 0);
prop_reply = xcb_get_property_reply (dpy, prop_cookie, NULL);
```

XCB 的强大之处在于允许这两个步骤之间拥有尽可能多的代码。程序员决定何时等待数据，而不是在发出请求时被迫等待请求返回的数据。

示例: 将 xwininfo 从 Xlib 转换为 XCB
----------------------------------

Xwininfo 程序是一个命令行实用程序，用于打印 X 服务器上的窗口信息。通过命令行选项，它可以预先知道从服务器请求每个窗口的信息所需的大部分数据。因此，xwininfo 可以同时发出所有请求，然后等待结果的到来。当使用-tree 选项遍历窗口树时，xwininfo 可以一次为当前窗口的所有子窗口请求数据，进一步批处理。在单 CPU 服务器上的本地连接上，这意味着 X 客户机和服务器之间的上下文切换减少。在多核/CPU 服务器上，X 服务器可以在一个核心上处理请求，而客户机则在另一个核心上处理可用的响应，从而提高性能。在远程连接中，可以将请求分组成更接近连接 MTU 大小的数据包，而不是在发出需要响应的请求时只发送缓冲区中的任何请求。

Xwininfo 的1.1版本由艾伦•库珀史密斯(alancoopersmith)从 Xlib 转换成了 XCB。它通过一个 GNOME 桌面会话和几个客户端进行了测试。Xwininfo 以“ xwininfo-root-all”的方式运行: 它从窗口层次结构的根开始 xwininfo，并要求它遍历树，请求沿途每个窗口的所有可用信息。在这个示例会话中，它找到了114个窗口。(在 X 中，窗口只是一个用于绘制输出和接收事件的容器。X 窗口通常是“用户窗口”的区域或边框)。当在四核 Intel Nehalem CPU 上本地运行时，两个版本的运行速度都非常快(0.05秒或更少) ，以至于时间差太小而无法精确测量。为了测量远程性能，“ ssh-X”被用于从加利福尼亚到中国的一台计算机的 X11连接隧道，并从那里回到加利福尼亚的工作站，引入了大量的延迟。在这种情况下，两者之间的差异是显著的:
```
Xlib:0.03u 0.02s 8:19.12 0.0%
 xcb:0.00u 0.00s 0:45.26 0.0%
```
当然，xwininfo 在以下几个方面是一个不同寻常的 X 应用程序:

Xwininfo 以最快的速度运行请求，然后退出，而不是等待用户输入(除非您使用单击窗口选择它的模式，然后正常运行)。一旦启动并运行，大多数 X 应用程序将花费大部分时间等待用户输入，因此通过减少与 X 服务器通信所花费的时间，整个运行时不会减少太多。然而，应用程序启动通常受往返时间的支配，因此正确使用 XCB 可以减少运行在高延迟(甚至中等延迟)连接上的 X 应用程序的巨大启动时间。

* Xwininfo 只使用核心协议和形状扩展。它不使用大多数现代应用程序所使用的更复杂的扩展，如 Render 或 Xinput。XXB 和 GLX 尤其成问题，因为它们还没有在 XCB 版本中得到完全支持，尽管通过一些 Google Summer of Code 项目得到了支持。

* Xwininfo 足够小，可以一次性完全重做使用 XCB。大多数应用程序都比这大得多。XCB 主要针对新的代码和工具包: 它专门设计用于与现有的 Xlib 应用程序进行互操作。对 Xlib 和 XCB 的调用可以混合使用，因此如果需要，可以对 Xlib 应用程序进行部分或增量转换。

* Xwininfo 只使用了原始的 Xlib，没有使用任何工具箱。因此，它不必担心工具箱使用的是哪个 X 库。

* Xwininfo 只使用了一些 Xlib helper 函数。这使得它更直接地映射到 XCB。例如，依赖于 Xlib 的输入法框架、组合键处理或字符集转换的应用程序将更难移植。幸运的是，现代工具包无论如何都在工具包层中处理大部分这种功能。

Xwininfo 的确依赖于 Xlib helper 函数来从其他字符集中转换窗口名属性——-XCB 版本目前仅适用于 UTF-8和拉丁文 -1窗口名。由于大多数现代工具包都使用 UTF-8，因此可能没有人会注意到。具有本地化窗口名称的较老的应用程序将会失败，但是正在使用的应用程序很少。

Xlib 和 XCB 混合调用
-------------------
如上所述，XCB 提供了一种从 Xlib 到 XCB 的递增转换方法。可以使用 libX11打开显示，并将其返回的 Display 指针传递给现有代码、工具箱和库。要调用 XCB 函数，可以将 Display 指针转换为相同连接的 XCB _ connect _ t 指针。这样就可以从同一个应用程序调用 Xlib 和 XCB。

Xlib 和 XCB 的兼容性是通过将 libX11重新构建为 libxcb 之上的一个层来实现的。Xlib 和 XCB 共享相同的 X 服务器连接并来回传递对它的控制。这个选项是在 libX111.2中引入的，自从2010年发布 libX111.4以来，它一直存在(不再是可选的)。

示例: 将 xdpyinfo 扩展查询转换为 XCB
---------------------------------

xdpyinfo 是标准 X 窗口系统工具集中的另一个命令行工具。和 xwininfo 一样，xdpyinfo 打印很多关于 X 服务器的信息。 xdpyinfo 调用许多扩展，并且它的调用很少阻塞等待来自服务器的响应。但是，如果添加“-queryExt”选项，对于每个扩展，xdpyinfo 都会调用 XQueryExtension 来打印当前运行的服务器中分配给该扩展的请求、事件和错误 ID。这些 id 是动态分配的，并且取决于在给定服务器构建/配置中启用的扩展集。因此，扩展 id 列表是调试引用它们的 X 错误报告时的关键信息。当报告来自自定义错误处理程序（如 gtk 工具包中的错误处理程序）的 X 错误消息时，尤其需要使用“xdpyinfo -queryExt”：此类错误处理程序通常会省略默认 Xlib 错误处理程序中的扩展信息，因此阅读的人错误报告将无法识别遇到错误的扩展名。

Xlib 调用 XQueryExtension 一次获取一个扩展名，向 X 服务器发送一个请求以获取该扩展的 id 代码，然后等待响应以便将这些 id 返回给调用者。在用作此转换测试系统的 Xorg 1.7 服务器上，有 30 个活动的 X 扩展，因此有 30 个小数据包发送到 X 服务器，是 xdpyinfo 客户端在 poll() 中阻塞等待响应的 30 倍，以及 30 X 服务器在返回到自己的 select() 循环中再次阻塞之前经过客户端处理和请求调度代码的次数。

注意：XListExtensions 可用于获取可用 XQueryExtension 调用的扩展列表。

xdpyinfo 的一个简单补丁仅用两个循环替换了对 XQueryExtension 的调用循环。第一个循环为每个扩展调用 xcb_query_extension。发出整批查询后，第二个循环调用 xcb_query_extension_reply 开始收集批处理回复。使用“truss -c”收集系统调用计数显示了 xdpyinfo 客户端进行的系统调用数量的预期减少：

System call	Xlib	xcb
writev	40	11
poll	80	22
recv	117	29
total	237	62
Over a TCP connection, the switch to XCB for this transaction reduced both the number of packets and (due to tcp packet header overhead) the overall amount of data:

Xlib xcb
TCP 数据包 93 35
TCP 字节 11554 7726
对于大多数应用程序而言，这种更改远比批量转换为 XCB 更可行。找到应用程序正在等待来自服务器的数据的热点并转换它们。当应用程序正在收集有关 X 服务器和会话的信息时，应用程序启动代码中几乎总是有机会。仅将这些调用转换为更有效的 XCB 调用集可以带来重大的性能优势。 X 开发人员的早期工作通过将重复调用转换为 XInternAtom 来减少许多应用程序的延迟，只需一次调用即可通过 XInternAtoms 一次获取多个原子。 XCB 允许对这一原则进行概括。

扩展库
-----
X11 协议的每个新扩展都添加了客户端可以向 X 服务器发出的请求。为了允许客户端软件利用这些请求，大多数扩展都提供构建在 Xlib 或 XCB 之上的 API。这些 API 使用库的连接编组将它们的请求包含在发送到 X 服务器的流中。

在早期的 X11 版本中，许多较小且更常见的扩展被分组到一个公共库 libXext 中。今天你会发现其中有几个仍在使用，例如 MIT-SHM 共享内存扩展、非矩形窗口的 SHAPE 扩展和事件同步的 SYNC 扩展。但是，libXext 还包含一些 API 用于当前 Xorg 服务器版本中不再存在的扩展，例如 App-Group 和 Low-Bandwidth X (LBX)，以及许多应用从未使用的扩展，例如用于显示电源管理的 DPMS。由于无法在不破坏任何可能正在使用它们的现有应用程序的情况下从 libXext 中删除这些扩展 API，因此代码被卡在那里。

因此，新的 Xlib 扩展 API 不再添加到 libXext。而是为每个扩展创建一个使用 libX11 的新库。每个扩展都有一个库可以更轻松地为该扩展开发 API，弃用过时的扩展并仅将其链接到实际需要它的客户端。几乎所有现代扩展都有自己的 Xlib API 库——用于 RENDER 扩展的 libXrender，用于 COMPOSITE 扩展的 libXcomposite，等等。少数扩展是协议交互的核心，libX11 本身直接支持它们，例如 BigRequests、XC-MISC 和 XKB。

当 XCB 将其 API 样式添加到组合中时，它遵循较新的样式并为每个扩展创建了一个以“libxcb”为前缀的库——libxcb-composite、libxcb-render 等。由于 XCB 可以为扩展生成 API 代码自动从扩展协议的 XML 描述中，通过简单地将扩展描述添加到 xcb-proto 包并重建来创建新的扩展 API。不幸的是，一些较旧的扩展具有复杂的协议，在 XML 语法中不容易描述。正在进行扩展语法和代码生成器以处理这些问题的工作。 XKB 和 GLX 协议是当前的挑战。

API 文档
-------
要使用 Xlib 或 XCB 编写代码，您需要了解库 API 的详细信息。 Xlib 包含大多数功能的手册页，提供了良好的 API 参考。 libXext 包含一些它包含的扩展 API 的手册页，但不是全部。在单个基于 Xlib 的扩展库中，手册页的覆盖率更加参差不齐。

在位于 http://www.x.org/releases/current/doc/ 的 X.Org 在线文档集中还有更完整的 Xlib API 指南和许多扩展 API 的规范。

对于没有 Xlib 风格 API 文档的扩展，调用通常是对上述链接文档集中提供的协议规范的简单映射。

对于 XCB，文档更加依赖协议规范。生成的 API 是对 X 协议的精确映射；它尽可能直接地将 C 调用数据转换为 X 协议编码的数据包。 XCB API 中的连接管理和其他功能记录在 http://xcb.freedesktop.org/XcbApi/。为了方便开发人员，我们正在努力添加对 XCB 的支持，以便从 XML 协议描述中生成 Unix 样式参考手册页。

还有一个 XCB 教程，“使用 XCB 库进行基本图形编程”，位于 http://www.x.org/releases/current/doc/libxcb/tutorial/index.html。

帮助我们改进任何一个库堆栈的 API 文档总是值得赞赏的。有关详细信息，请参阅本指南后面的文档章节。