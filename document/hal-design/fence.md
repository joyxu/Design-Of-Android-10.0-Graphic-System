# 同步机制
* * *

基于 Linux 的原子同步机制，数据结构和函数声明于 sync.h，实现于 sync.c。我们需要某种 OS 级别的原子同步机制来显式地管理内存缓冲区所有权的转移，从而验证内存“所有者”是否看到了内存的一致视图。

# 同步栅栏(Fence)

在 Android 图形系统里，是一个多进程多线程并发的模型。在 VSYNC 的驱动下，Buffer 与 HWC 的状态和数据流动十分复杂。为了使得图形界面输出正确，而且快速和平滑，系统需要同步机制。这其中，涉及到许多概念、假设和约定。

## 问题的由来

在文档[[microconference](https://lwn.net/Articles/569704/)]里介绍了 Android 系统中的关于同步的基础设施的一些背景知识，以及为什么它会被引入，为什么对于 Android 系统它十分重要的讨论。

同步是很重要的，因为它允许在图形和媒体管道里更好地利用并行性。通常你可以认为管道作为在一个系列上摆弄缓冲区的不同设备的集合。每个步骤都要彻底完成当前任务，在它输出缓冲区到下一步骤之前。然而，在许多情况下，在每个步骤中所需要的一些开销工作，不是严格需要持有要被处理的数据。所以，在缓冲区仍然在被填充的过程中，你可以尝试并行做这些开销的步骤，例如创建缓冲区给显示器。然而，这需要有一些内部锁机制，可以让一个步骤发出信号到下一个步骤，它实际上已经完成任务的时候。步骤之间的这项协议就是“同步契约”。 

## 显式同步 

现在的 Android 的“同步契约”是显式同步方式。也就是，如果用户跨进程/线程使用对象，不能假设该对象是进程两端自动同步的。用户要显式地调用同步框架接口，自己确保对象在两端的同步。

显式同步是必需的，它提供了一种以同步方式获取和释放 gralloc buffer 的机制。显式同步允许图形缓冲区的生产方和消耗方在完成对缓冲区的处理时发出信号。这允许 Android 异步地将要读取或写入的缓冲区加入队列，并且确定另一个消耗方或生产方当前不需要它们。有关详细信息，请参阅[同步框架](https://source.android.com/devices/graphics/index.html#synchronization_framework)一文。 

显式同步的优点包括不同设备上的行为差异较小、对调试的支持更好，并且测试指标更完善。例如，同步框架输出可以轻松指出问题区域和根本原因，而集中于 SurfaceFlinger 的演示时间戳可以显示系统的正常流程中发生事件的时间。 

该通信是通过使用同步栅栏(Fence)来完成的。在请求用于消耗或生产的缓冲区时，必须使用同步栅栏(Fence)。同步框架由三个主要构造块组成：sync_timeline、sync_pt 和 sync_fence。在文档[[implement-vsync](https://source.android.com/devices/graphics/implement-vsync)]和[[Android Sync](https://blog.linuxplumbersconf.org/2014/ocw/system/presentations/2355/original/03%20-%20sync%20&%20dma-fence.pdf)]中，对此有详细的说明。 

同步栅栏(Fence)是 Android 图形系统的关键部分。栅栏(Fence)允许 CPU 工作与并行的 GPU 工作相互独立进行，仅在存在真正的依赖关系时才会阻塞。 

例如，当应用提交在 GPU 上生成的缓冲区时，它还将提交一个栅栏(Fence)对象；该栅栏(Fence)仅在 GPU 完成写入缓冲区的操作时才会变为有信号量状态。由于真正需要 GPU 写入完成的唯一系统部分是显示设备硬件（由 HWC HAL 抽象的硬件），因此图形通道能够通过 SurfaceFlinger 将该栅栏(Fence)与缓冲区一起传递到 HWC 设备。只有在即将显示该缓冲区之前，设备才需要实际检查栅栏(Fence)是否已经变为有信号量状态。 

## 为什么需要显式的“同步契约”？ 

Android 早期的同步契约是隐式的，并且没有很好的定义。这造成了问题。因为驱动程序编写者往往误解或着误实现同步契约，导致难以调试的问题。此外，因为契约是隐式的，并且它的实现分散在很多驱动，有些是专有的，这会让它很难修改契约以提高性能。 

为了解决这些问题，Android 的显式同步机制被实现了。在 Android >= 4.2 版本以后, 使用的是一种被标准化的栅栏(Fence)机制, 这将在所有的驱动上有统一的工作方式。同步栅栏(Fence)是一个同步的框架，允许 SurfaceFlinger 建立一个时间线，并在时间线上设置同步点。其它线程和驱动程序可以阻塞在一个同步点上，并将等待直到时间线计数器已经跨越了这一点。可以有多个时间线，通过各种驱动程序管理；同步接口允许从不同的时间线合并同步点。这就是 SufaceFlinger 和 BufferQueue 流程如何管理跨多个驱动程序和进程的同步契约。

最终，显式的“同步契约”，意味着**伴随着的对象的所有权的移交**。

## 同步集成

如何将低层级同步框架与 Android 框架的不同部分(包括驱动程序)进行集成，以及彼此间如何通信？这方面有一些集成规范和参考约定。 

### 集成规范 

用于图形的 Android HAL 接口会遵循统一的规范，因此当文件描述符通过 HAL 接口传递时，始终会传输文件描述符的所有权。这意味着： 
* 如果你从同步框架收到栅栏文件描述符，就必须将其关闭。 
* 如果你将栅栏文件描述符返回到同步框架，框架将关闭它。 
* 要继续使用栅栏文件描述符，你必须复制该描述符。 

每当栅栏通过 BufferQueue（例如某个窗口将栅栏传递到 BufferQueue，指明其新内容何时准备就绪）时，该栅栏对象将被重命名。由于Android内核栅栏支持允许栅栏使用字符串作为名称，因此同步框架使用正在排队的窗口名称和缓冲区索引来命名栅栏（例如 SurfaceView:0）。这有助于进行调试来找出死锁的来源，因为名称会显示在 /d/sync 的输出和错误报告中。 

### ANativeWindow 集成 

ANativeWindow 接口(interface)针对栅栏机制进行了升级，在方法 dequeueBuffer、queueBuffer 和 cancelBuffer 都具有栅栏参数。  

### OpenGL ES 集成 

OpenGL ES 同步集成依赖于两个 EGL 扩展： 

* EGL_ANDROID_native_fence_sync。提供一种在 EGLSyncKHR 对象中包装或创建原生 Android 栅栏文件描述符的方法。 

* EGL_ANDROID_wait_sync。允许 GPU 端停止而不是在 CPU 中停止，使 GPU 等待 EGLSyncKHR。这与 EGL_KHR_wait_sync 扩展基本相同（有关详细信息，请参阅相关规范）。 

这些扩展可以独立使用，并由 libgui 中的编译标记控制。要使用它们，请首先实现 EGL_ANDROID_native_fence_sync 扩展以及关联的内核支持。接下来，为驱动程序添加对栅栏的 ANativeWindow 支持，然后在 libgui 中启用支持以使用 EGL_ANDROID_native_fence_sync 扩展。 

其次，在驱动程序中启用 EGL_ANDROID_wait_sync 扩展，并单独打开它。EGL_ANDROID_native_fence_sync 扩展包含完全不同的原生栅栏 EGLSync 对象类型，因此适用于现有 EGLSync 对象类型的扩展不一定适用于 EGL_ANDROID_native_fence 对象，以避免不必要的交互。 

### Hardware Composer 集成 

Hardware Composer 可处理三种类型的同步栅栏(Fence)： 

* 获取栅栏(Acquire fence)。每层(Layer)一个，在调用 HWC::set 之前设置。当 Hardware Composer 可以读取缓冲区时，该栅栏会变为有信号状态。 

* 释放栅栏(Release fence)。每层(Layer)一个，在 HWC::set 中由驱动程序填充。当 Hardware Composer 完成对缓冲区的读取时，该栅栏会变为有信号状态，以便框架可以再次开始将该缓冲区用于特定层。 

* 退出栅栏(Retire fence)。整个图形框架就一个，每次调用 HWC::set 时由驱动程序填充。HWC::set 操作会覆盖所有的层，并且当所有层的 HWC::set 操作完成时会变成有信号状态并通知框架。当在屏幕上进行下一设置操作时，退出栅栏将变为有信号状态。 

退出栅栏(Retire fence)可用于确定每个帧在屏幕上的显示时长。这有助于识别延迟的位置和来源，例如卡顿的动画。另外，退出栅栏(Retire fence)上的时间戳代表着实际发生 HW VSYNC 信号的时间点。用此时间戳可以校验 SW VSYNC 的偏差。

这就是 Android 文档里提到关于 VSYNC 信号的软件锁相回路 (PLL)算法：
* 绘图操作和 HWC 合成都是由 VSYNC 信号驱动。
* HW VSYNC 信号太耗电，所以不常开而需要 SW VSYNC。
* SW VSYNC 和实际硬件情况总是会有偏差，需要校准。
* 退出栅栏(Retire fence)上的时间戳代表着实际发生 HW VSYNC 信号的时间点。用此时间戳可以校验 SW VSYNC 的偏差。
* 当 Fence 时间戳校准的方法发现 SW VSYNC 的模型的误差超出阈值，将启动重新同步流程。

## HWC2 中的更改 

同步栅栏紧密集成到 HWC2 中，并且按以下类别进行划分： 
* 获取栅栏(Acquire fence)会与输入缓冲区一起传递到 setLayerBuffer 和 setClientTarget 调用。这些栅栏表示正在等待写入缓冲区，并且必须在 HWC 客户端或设备尝试从关联缓冲区读取数据以执行合成之前变为有信号量状态。  

* 释放栅栏(Release fence)在调用 presentDisplay 之后使用 getReleaseFences 调用进行检索，并与将在下一次合成期间被替换的缓冲区一起传回至应用。这些栅栏表示正在等待从缓冲区读取数据，并且必须在应用尝试将新内容写入缓冲区之前变为有信号量状态。 

* 退出栅栏(Retire fence)作为对 presentDisplay 的调用的一部分返回，每帧一个，说明该帧的合成何时完成，或者何时不再需要上一帧的合成结果。对于物理显示设备，这是当前帧显示在屏幕上之时，而且还可以解释为在其之后可以再次安全写入客户端目标缓冲区（如果适用）的时间。对于虚拟显示设备，这是可以安全地从输出缓冲区读取数据的时间。 

HWC 2.0 中同步栅栏(Fence)的含义相对于以前版本的 HAL 已有很大的改变。 

在 HWC v1.x 中，释放栅栏(Release fence)和退出栅栏(Retire fence)是推测性的。在帧 N 中检索到的缓冲区的释放栅栏或显示设备的退出栅栏不会先于在帧 N + 1 中检索到的栅栏变为有信号量状态。换句话说，该栅栏的含义是“不再需要你为帧 N 提供的缓冲区内容”。这是推测性的，因为在理论上，SurfaceFlinger 在帧 N 之后的一段不确定的时间内可能无法再次运行，这将使得这些栅栏在该时间段内不会变为有信号量状态。 

在 HWC 2.0 中，释放栅栏(Release fence)和退出栅栏(Retire fence)是非推测性的。在帧 N 中检索到的释放栅栏或退出栅栏，将在相关缓冲区的内容替换帧 N - 1 中缓冲区的内容后立即变为有信号量状态，或者换句话说，该栅栏的含义是“你为帧 N 提供的缓冲区内容现在已经替代以前的内容”。这是非推测性的，因为在硬件呈现此帧的内容之后，该栅栏应该在 presentDisplay 被调用后立即变为有信号量状态。 

# 同步栅栏(Fence)的使用

在上一章里详细介绍了 Fence 的概念和由来，本章对此做个简略小结。BufferQueue，既是使用 Buffer 的通道，也是使用同步栅栏(Fence)的通道。在 BufferQueue 两端周期性使用的4个方法：
* dequeueBuffer() 缓冲区出队。
* queueBuffer() 缓冲区入队。
* acquireBuffer() 帧出队。
* releaseBuffer() 帧入队。

4个都带有同步栅栏(Fence)参数。根据参数的进出方向不同，对应到[Hardware Composer 集成](#hardware-composer-%E9%9B%86%E6%88%90)一节里介绍的 HWC 使用的3个同步栅栏(Fence)中的2个：获取栅栏(Acquire fence)和释放栅栏(Release fence)。

根据参数的进出方向：
* dequeueBuffer() 缓冲区出队。空闲缓冲区带着的是消费者 HWC 传递过来的释放栅栏(Release fence)。生产者需要等待 Release fence 发信号，才代表着消费者获取内容完毕，该缓冲区可以交给生产者使用。
* queueBuffer() 缓冲区入队。生产者生成获取栅栏(Acquire fence)，代表着虽然缓冲区里的内容还在由 GPU 渲染，但生产端的 CPU 已经无事可做，所以提前提交给消费端 CPU 了。
* acquireBuffer() 帧出队。有内容的帧带着的是生产者传递过来的获取栅栏(Acquire fence)。消费者 HWC 需要等待 Acquire fence 发信号，才代表着生产者生成内容完毕，该帧可以交给消费者使用。
* releaseBuffer() 帧入队。消费者 HWC 生成释放栅栏(Release fence)，代表着虽然缓冲区里的内容还在等待读取，但消费端的 CPU 已经无事可做，所以提前交还给生产端 CPU 了。

# 同步栅栏(Fence)的处理节点

合成服务器(HWComposer)是内容汇总的地方，所以也是同步栅栏(Fence)集中处理的地方。
* 在读取每层的内容前，合成服务器或 HWC 做为消费者，都会等待生产者的获取栅栏(Acquire fence)信号。
* 为了并发性，合成服务器会尽快地给生产者释放 Buffer，并且附带一个释放栅栏(Release fence)。生产者必须等待该信号，表明合成服务器或 HWC 已经读取内容完毕，生产者才可以生成新的内容。
* 当合成操作真正执行完毕，底层 HWC 会发出退出栅栏(Retire fence)信号。该信号可以校验 SW VSYNC。

