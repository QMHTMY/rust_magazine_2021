# 您想要编写一个 GUI 框架吗？

> 原文链接：https://www.cmyr.net/blog/gui-framework-ingredients.html

通过最近关于 Rust 中 GUI 编程的几次讨论，我留下了这样的印象：“GUI”这个词对不同的人来说意味着截然不同的东西。

我想尝试澄清这一点，首先通过描述人们称为 GUI 框架/工具包的一些不同事物，然后详细探讨其中一个经典桌面 GUI 框架的必要组件。

尽管这篇文章并不是特别针对 Rust，但它确实起源于 Rust：它很大程度上来自于我在 Druid 上工作的经验，Druid 是桌面版的 Rust GUI 工具包。

一旦我们对这个问题有了共同的理解，我们就可以更好地讨论这项工作在 Rust 中的状态，这将是后续帖子的主题 。


## 当我们谈论 GUI 时我们在谈论什么

GUI 框架可以有很多不同的东西，具有不同的用例和不同的部署目标。用于构建嵌入式应用程序的框架也不会在桌面上轻松运行；用于构建桌面应用程序的框架不会在网络上轻松运行。

无论具体情况如何，都需要认识到一个主要的分界线，即框架是否有望紧密集成到现有平台或环境中。

因此，在这条线的一侧是用于构建游戏、嵌入式应用程序和（在较小程度上）网络应用程序的工具。在这个世界中，您负责提供应用程序所需的几乎所有内容，并且您将与底层硬件密切交互：接受原始输入事件，并将您的 UI 输出到某种缓冲区或表面。（网络是不同的；这里浏览器供应商已经为您完成了集成工作。）

这条线的另一边是用于构建传统桌面应用程序的工具。在这个世界中，您必须与大量现有的平台 API、设计模式和约定紧密集成，而这种集成正是您设计复杂性的主要来源。

游戏和嵌入式 GUI
在我们开始深入研究桌面应用程序框架所期望的所有集成之前，让我们简要谈谈第一种情况。

嵌入式应用程序的游戏和 GUI（想想出租车后面的信息娱乐系统，或医疗设备上的界面）在许多方面与桌面 GUI 不同，其中大部分可以从系统集成的角度考虑：游戏和嵌入式应用程序不必做那么多。一般来说，游戏或嵌入式应用程序是一个自成一体的世界；只有一个“窗口”，应用程序负责绘制其中的所有内容。应用程序不需要担心菜单或子窗口；它不需要担心合成器，或与平台的 IME系统集成。尽管它们可能应该，但它们通常不支持复杂的脚本. 他们可以忽略富文本编辑。他们可能不需要支持font enumeration或fallback。他们经常忽略可访问性。

当然，他们自己也有额外的挑战。嵌入式应用程序必须更仔细地考虑资源限制，并且可能需要完全避免分配。当他们确实需要复杂脚本或文本输入等功能时，他们必须自己实现这些功能，而不能依赖系统提供的任何东西。

游戏是相似的，另外还有它们自己独特的性能问题和考虑因素，我没有资格谈论任何真正的细节。

游戏和嵌入式当然是有趣的领域。特别是嵌入式是我认为 Rust GUI 真的很有意义的地方，出于许多相同的原因，Rust 通常 对嵌入式使用具有强大的价值主张。

然而，一个旨在用于游戏或嵌入式开发的项目不太可能解决我们在桌面应用程序中期望的整个功能列表。


## “原生桌面应用程序”剖析

桌面应用程序的主要区别特征是它与平台的紧密集成。与游戏或嵌入式应用程序不同，桌面应用程序需要与主机操作系统以及其他软件密切交互。

我想尝试了解一些主要的必需集成点，以及一些可用于提供它们的可能方法。


### 窗口化

应用程序必须实例化和管理窗口。API 应该允许自定义窗口外观和行为，包括窗口是否可以调整大小，是否有标题栏等。 API 应该允许多个窗口，并且它还应该以某种方式支持模式和子窗口尊重平台约定。这意味着支持 应用程序模式窗口（例如从整个应用程序中窃取焦点直到处理的警报）以及窗口模式windows（从给定窗口窃取焦点直到处理的警报）。模态窗口用于实现大量常用功能，包括打开/保存对话框（可能是平台的特殊情况）警报、确认对话框以及标准 UI 元素，例如组合框和其他下拉菜单（想想一个文本字段的完成列表）。

API 必须允许相对于父窗口的位置精确定位子窗口。例如在组合框的情况下，当显示选项列表时，您可能希望在列表关闭时使用的相同基线位置绘制当前选定的项目。


### 标签

您还需要支持选项卡。您应该能够从选项卡组中拖出选项卡以创建新窗口，以及在窗口之间拖动选项卡。理想情况下，您希望使用平台的本机标签基础结构，但是……这很复杂。浏览器都推出了自己的实现，这可能是有充分理由的。你会想尊重周围的标签的用户的喜好（Mac系统，让我们的用户选择打开新窗口的标签，系统范围的），但是这将是一个额外的并发症。如果你跳过它，我会原谅你，但是如果你的框架看到很多用处，那么你每个月都会有人将它报告为错误，直到你死，而且他们没有错。

### 菜单

与窗口密切相关的是菜单；桌面应用程序应该尊重围绕窗口和应用程序菜单的平台约定。在 Windows（操作系统系列）上，菜单是窗口的一个组件。在 macOS 上，菜单是应用程序的一个属性，它会更新以反映可用于活动窗口的命令。在 linux 上，事情不太清楚。如果您使用 GTK，则有窗口和应用程序菜单，尽管后者已弃用。如果您直接针对 x11 或 Wayland，则需要自己实现菜单，理论上您可以随心所欲，尽管简单的方法是 Windows 风格的窗口菜单。

通常，对于应该提供哪些菜单以及应该在其中显示哪些命令，有明确的 约定；一个行为良好的桌面应用程序应该遵守这些约定。

### 绘图

要绘制应用程序的内容，您（至少）需要一个基本的 2D 图形 API。这应该提供填充和描边路径（使用颜色，包括透明度，以及径向和线性渐变）、布置文本、绘制图像、定义剪辑区域和应用转换的能力。理想情况下，您的 API 还提供了一些更高级的功能，例如混合模式和模糊，用于阴影等。

这些 API 以略有不同的形式存在于各种平台上。在 macOS 上，有CoreGraphics，在 Windows Direct2D 上，在 linux 上有Cairo。那么，一种方法是在这些平台 API 之上呈现一个通用的 API 抽象，在粗糙的边缘上涂油并填补空白。（这是我们目前采用的方法，使用piet库。）

这确实有其缺点。这些 API 足够不同（特别是在棘手的领域，例如 text），设计一个好的抽象可能具有挑战性，并且需要一些跳跃。微妙的不同平台行为会导致渲染不规则。

在任何地方都使用相同的渲染器会更简单。一种选择可能是像Skia这样的东西，它是 Chrome 和 Firefox 中使用的渲染引擎。这具有可移植性和一致性的优点，但代价是二进制大小和编译时间成本；使用skia-safe crate的Rust 二进制文件的发布版本的基线大小约为17M（我的方法不是很好，但我认为这是一个合理的基线。）

Skia 仍然是一个相当传统的软件渲染器，尽管它现在确实有重要的 GPU 支持。不过，归根结底，最令人兴奋的前景是那些将更多渲染任务转移到 GPU 的前景。

这里的一个初始挑战是用于 GPU 编程的 API 的多样性，即使对于相同的硬件也是如此。相同的物理 GPU 可以通过 Apple 平台上的Metal、Windows上的DirectX和许多其他平台上的Vulkan进行连接。使代码在这些平台上可移植需要重复实现、某种形式的交叉编译或 抽象层。后一种情况的问题在于，很难编写一个抽象来提供对高级 GPU 功能（例如计算能力）的充分控制，这些功能跨细微不同的低级 API。

一旦你弄清楚你想如何与硬件对话，你就需要弄清楚如何在 GPU 上有效和正确地光栅化 2D 场景。这也可能比您最初怀疑的更复杂。由于 GPU 擅长绘制 3D 场景，而且 3D 场景似乎比 2D 场景“更复杂”，因此 GPU 应该轻松处理 2D 似乎是一个自然的结论。他们不。3D 中使用的光栅化技术不太适合 2D 任务，例如裁剪到矢量路径或抗锯齿，而那些产生最佳结果的技术性能最差。更糟糕的是，一旦涉及到大量混合组或剪辑区域，这些传统技术在 2D 中的表现就会变得非常糟糕，因为每个区域都需要自己的临时缓冲区和绘制调用。

有一些有前途的新工作（例如piet-gpu）使用计算着色器，并且可以在 2D 成像模型中以平滑一致的性能绘制场景。这是一个活跃的研究领域。一个潜在的限制是计算着色器是一项相对较新的功能，并且仅在过去五年左右制造的 GPU 中可用。其他渲染器，包括Firefox 使用的WebRender，使用更传统的技术并具有更广泛的兼容性。

### 动画

哦，还有：无论您选择哪种方法，您还需要提供符合人体工程学的高性能动画 API。值得尽早考虑这一点；稍后尝试添加它会很烦人。

### 文本

不管你如何绘画，你都需要渲染 text。GUI 框架至少应该支持富文本、复杂脚本、文本布局（包括诸如换行、对齐和对齐之类的东西，理想情况下诸如在任意路径中换行之类的东西）。您需要支持表情符号。您还需要支持文本编辑，包括支持从右到左和BiDi。可以说这是一项非常庞大的事业。实际上，您有两种选择：捆绑 HarfBuzz或使用平台文本 API：macOS上的CoreText、Windows上的 DirectWrite以及可能的Pango+ Linux 上的 HarfBuzz。还有其他一些替代方案，包括一些有前途的 Rust 项目（例如 Allsorts、rustybuzz和swash），但这些项目都还不够完整，无法完全取代 HarfBuzz 或平台文本 API。

### 合成器

2D 图形是可能由桌面应用程序完成的绘图的主要部分，但它们不是唯一的部分。还有另外两种常见情况值得一提：视频和 3D 图形。在这两种情况下，我们都希望能够利用可用的硬件：对于视频，硬件 H.264 解码器，对于 3D，GPU。这归结为指示操作系统在我们窗口的某个区域嵌入视频或 3D 视图，这意味着与合成器进行交互。合成器是操作系统的组件，负责从各种来源（不同程序的不同窗口、视频播放、GPU 输出）获取显示数据，并将其组装成桌面的连贯图片。

也许思考为什么这对我们很重要的最好方法是考虑与滚动的交互。如果您有一个可滚动视图，并且该视图包含一个视频，您希望在滚动视图时视频与视图的其他内容同步移动。这比听起来要难。您不能只定义窗口的一个区域并在其中嵌入视频；您需要以某种方式告诉操作系统与滚动同步移动视频。

### 网页浏览器

让我们不要忘记这些：迟早会有人想要在他们的应用程序中显示一些 HTML（或一个实际的网站！）。我们真的不想捆绑整个浏览器引擎来实现这一点，但使用平台 webview也会牵涉到合成器，并且总体上使我们的生活显着复杂化。也许您的用户根本不需要那个网络视图？无论如何，有些事情要考虑。

## 输入处理

一旦您弄清楚如何管理窗口以及如何绘制内容，您就需要处理用户输入。我们可以粗略地将输入分为 指针、键盘和其他，其中其他是操纵杆、游戏手柄和其他HID 设备之类的东西。我们将忽略最后一个类别，只是说这很好，但不需要成为优先事项。最后，还有源自系统可访问性功能的输入事件；当我们谈论可访问性时，我们将处理这些。

对于指针和键盘事件，有一个相对简单的方法，然后有一个原则性的、正确的方法，但要正确得多。

### 指针输入

对于指针事件，简单的方法是呈现一个发送鼠标事件的 API，然后以一种使它们看起来像鼠标事件的方式发送触控板事件：忽略多次触摸、压力或其他不具有明显特征的触摸手势的特征类似于鼠标。该硬的方法是实现网络的一些等价PointerEvent API，你都能够充分代表了多点触摸的信息（从触控板都还有一个触控式显示器），其中和手写笔的输入事件。

以简单的方式执行指针事件是……好吧，假设您还可以为常见的触控板手势（例如捏缩放和两指滚动）提供事件，否则您的框架将立即使许多用户感到沮丧。虽然需要或想要进行高级手势识别或期望处理手写笔输入的应用程序数量相当少，但它们确实存在，并且不支持这些情况的桌面应用程序框架从根本上是有限的。

### 键盘输入

键盘输入的情况更糟，有两个方面：在这里，困难的情况既难以做到，而“简单的方法”则从根本上受到限制；走简单的路线意味着您的框架对于世界上的大部分人口来说基本上是无用的。

对于键盘输入来说，简单的方法非常简单：键盘的键通常与一个字符或字符串相关联，当用户按下一个键时，您可以取出该字符串并将其插入活动中的光标位置文本域。这对于单语英语文本相当有效，对于一般的拉丁语 1语言以及行为类似于拉丁语的脚本，例如希腊语或西里尔语或土耳其语，效果稍差但至少是一种。不幸的是（但并非巧合），大量程序员大多只键入 ASCII，但世界上的大部分地区都没有。服务于这些 用户需要与平台文本输入和 IME 系统集成，这是一个不幸的问题，它既是必不可少的，又是极其繁琐的。

IME 代表Input Method Editor，是将键盘事件转换为文本的平台特定机制的统称。对于大多数欧洲语言和文字，这个过程相当简单，您最多可能需要插入一个带重音的元音，但对于东亚语言（中文、日文和韩文，或统称为 CJK）来说，这个过程要复杂得多，因为以及其他各种复杂的脚本。

这在很多方面都很复杂。首先，它意味着给定文本字段和 IME 之间的交互是双向的：IME 需要能够修改文本框的内容，但还需要能够查询文本框的当前内容，在以便有适当的上下文来解释事件。同样，需要通知光标位置或选择状态的变化；相同的按键可能会根据周围的文本产生不同的输出。其次，我们还需要在屏幕上文本框的位置上使 IME 保持最新，因为 IME 经常为键盘事件的活动序列提供可能输入的“候选”窗口。最后（实际上并不像最后，我已经写了三千字，但还没有完成）由于底层平台 API 的差异，以跨平台方式实现 IME 非常复杂；macOS 需要可编辑的文本字段来 实现协议，然后让文本字段处理接受和应用来自 IME 的更改，而Windows API使用 锁定和释放机制；设计对这两种方法的抽象是一个额外的复杂层。

还有一个与文本输入相关的额外复杂性：在 macOS 上，您需要支持Cocoa Text System，它允许用户指定可以发出各种文本编辑和导航命令的系统范围的键绑定。

总而言之：正确处理输入需要大量工作，如果您不这样做，您的框架基本上就是一个玩具。
