# 基本原理与设计思路
* * *

# 旧的设计的问题

在设计 X11 的年代，网络刚刚兴起，大型主机配上图形终端是用户使用计算机的普遍模式，所以采用 Client / Server 架构是很自然的事情。但是随着 PC 的兴起和大型主机的没落，在 PC 本地采用 C/S 架构设计图形系统，不但是多此一举的事情，而且还会影响性能。因此受到诸多人的指责。

原始的 X Server 的设计，可以简单的总结为客户端请求绘图/服务端执行绘图并显示这样的模式。见下面的示意图1。 
![server side drawing](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/original-design/01-server-side-drawing.svg)

在这种模式中，客户端软件是不知道自己绘图的效果的，除非它向图形服务器发出请求，要求服务器回传当前的图形缓冲区回来。因此，有些功能就很难实现。例如，如果应用软件想录制当前的操作画面，需要操作完成后再要服务器回传回来才知道效果，就会很影响效率。 

因为绘图算法库存在于图形服务器一端，还带来了几个问题： 
* X Server 会又大又复杂，很难优化，又容易造成系统崩溃。并且，X Server 是直接操作设备驱动的程序，这往往意味着它具有 root 特权。而大而复杂的 X Server 的安全问题，一直是大家的诟病之一。
* 用户对图形库一直不满意。因为每天都会有新的需求，而绘图算法本身也在进化。而服务器本身又要求稳定，所以新的绘图效果很难进入到 X Server 中。例如，反锯齿算法和新的字体解析库，就很难进入到 X Server 中。 
* 更进一步的，新的模块就更难加入到 X Server 中。例如，视频解码库，进入到 X Server 中是不可能的事情。所以，在早期的 X Window 系统里，要播放视频，在客户端，软件解码一帧视频，当成一张图像传送给 X Server 显示，接着再来。如此反复循环，效率极低。 

# 新的设计的想法

于是有人针对上述问题提出了客户端绘图方案。见下面的示意图2。 
![client side drawing](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/original-design/02-client-side-drawing.svg)

在新方案中，出现了两个大的变化。 

一是将绘图算法库移动到客户端，由应用软件选择和自行调用。这样的好处有： 
* Grahpic Server 可以变小而易于维护，而绘图算法库可以单独升级而不影响服务器。 
* 应用软件可以按照需求，选择合适的图形库，例如，选择 cairo / skia / hwui 等图形库。 

二是剪切域(clip-region)管理器变成了合成器(composer)。这意味着窗口间相互作用的算法有了增强： 
* 原本的剪切域(clip-region)算法，就是在窗口重叠遮盖的情况下，看看哪个窗口暴露出哪些矩形，然后如同剪报一样，把那些暴露出来的矩形剪出(cut)并粘贴(paste)到同一个背景里，最后显示出来。 
* 合成器(composer)处理的窗口，增加了一个alpha参数。窗口合成(compositing)算法增加了三大类： 
  * 窗口本身增加了缩放、旋转、剪切变形等矩阵变换算法。 
  * 窗口间合成增加了Porter-Duff的[合成算法](https://en.wikipedia.org/wiki/Alpha_compositing)。 
  * 合成结果，增加了颜色空间转换的算法。 

简单一句话，在新方案中，只由客户端生产内容，服务端只读取内容并不做修改，而是在显示路径上对内容进行修饰并显示出来。新方案的好处，除了增强了显示效果，提高了显示效率，还有一个好处，就是保证了显示的一致性。 

在原始的 X Window 方案中，假如把本文在屏幕上显示、送到打印机打印，或者制作成 PDF 文档这些操作，可能会有不同的显示效果，因为在不同的系统里，所自带的字体和字体引擎不一样。而在新方案里，会有一致的效果，因为只是对同一个渲染(render)后的原稿选择不同的显示后端(backend)而已。另外，这也容易实现在玩游戏中常用的边显示边录制游戏画面的功能，对应用来说，只是同时选择了两个显示后端，一个是硬件显示后端，一个是带视频编码和保存功能的虚拟显示后端。 

这种客户端绘图方案，最早是在 Plan9 OS 系统里的 8½ 的图形系统中得以实现。后来在 MS Windows 中也得到了实现。最后，过了不少年，终于在 Xorg 的 X Server 中，以[X Composite Extension](http://www.talisman.org/~erlkonig/misc/x11-composite-tutorial/)扩展的方式得以实现。但是，X Server 为了后向兼容，内部始终保留着原始的绘图算法代码。所以 X Server 的代码始终还是又大又复杂。而 Android 从一开始就选择了客户端绘图的方案，所以服务端的代码相对精简和易于理解。

# 新的设计需要注意的第一个问题

但是，在上图中有一个细节需要注意，就是在客户端和服务端的对应同一个窗口的内容是如何传输或同步的。 

我们先看在图形系统中一个窗口的抽象设计。见下图。

![窗口抽象示意图](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/original-design/03-window.svg)

一个窗口一般会分成两个部分：一个窗口本身的管理结构(struct)，一个是可以存放绘图内容的内存块，一般称为缓冲区(buffer)。在习惯上，一般在讨论这块内存本身的特性和功能时，会称之为缓冲区(buffer)；如果内存上面已经带有内容，则称之为帧(frame)。 

一个窗口的设计，一般会有三个基本参数：width / height / format，还有两个和硬件相关的参数：stride / usage，在后面进行讨论。从这三个基本参数，我们可以大致估计出一帧图形所需要的内存块大小。 

在这里我们可以估计一下，如果要在 RGBA8888 的颜色格式的窗口上以 25Hz 的速率播放 1080P 电影。则：
```
1920 * 1080 * 4 * 25 = 207360000 bytes
```

也就是一秒钟要生产 200MB 的内容。 

如果我们采用类似 X Window 的原始的 C/S 架构设计，见下图： 

![two stage copy](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/original-design/04-two-stage-copy.svg)

这样的设计简单，但是会造成对同一个内容占用两块内存，并且会有两次拷贝的问题。这既浪费内存又浪费时间。于是有人提出了零拷贝(zero-copy)方案。 

零拷贝(zero-copy)的设计思路相当简单：就是 Client 端和 Server 端共享同一块内存。Client 端在本地绘制完成后，用某种同步机制通知 Server 端，Server 端就把同一块内存的内容显示到屏幕上。要实现在进程间共享内存，在 UNIX/Linux 世界里就需要 OS 提供的文件句柄传输功能。 

# 内存共享机制的底层基础

文件句柄传输在 UNIX/Linux 进程间通信方式中是共享信息效率最高的方式。它的思路很简单，就是由一个进程打开某个文件，然后将文件句柄通过 OS 提供的机制传输给有需要的进程。虽然在各个进程里句柄号不一样，但是都是对应到 OS 里的同一个文件系统中的数据结构，操作的是同一个文件。这样，如果有一个进程写了文件，其它进程就可以读到修改内容。 

再进一步，在各个进程里，都可以使用"Memory Map(mmap)"系统调用将文件映射到内存里。这样，如果有一个进程修改了映射内存，其它进程也可以在其相应的映射内存里读到修改内容。再加上一些进程间的同步机制，进程间的大块数据就可以高效共享了。 

更进一步的，如果这个文件句柄并没有关联到某个具体的磁盘文件上，而只是内存中某段地址的映射，比如 Linux 中著名的"FrameBuffer(fbdev)"，打开文件就提供了显示内存的起始地址和大小，经过内存映射后，进程间就共享了同一块内存。 

现在 UNIX/Linux 在用户空间提供共享内存功能的软件，大多数都采用这种方法实现。在《UNIX环境高级编程》(APUE)一书的第17章"高级进程间通信"中的"17.4 传送文件描述符"节，对这种机制有详细的描述。网上有一个共享软件"libancillary"，对此也有经典的实现。如果对此感兴趣，也可以查看 glibc 里的"Shared Memory(shm)"模块，也是用类似的方法实现共享内存。 

简单一句话，零拷贝(zero-copy)方案就是用文件句柄传输功能来实现共享内存机制。在标准的 UNIX/Linux 系统里，进程间传输文件句柄，都是采用 socket 带外数据传输的方式。在 Android 系统里，也可以采用专有的 binder 系统调用传输文件句柄。这两者的效果都一样。 

另外，为了实现完整可用的共享内存机制，还需要 Linux 提供操作系统级别的原子同步机制，就是 Linux Fence 机制。后面会讨论这方面的内容。 

# 新的设计需要注意的第二个问题

因为现代 CPU 大多具有多核，因此多线程的程序在性能上更有优势。因为客户端绘图和服务端合成的代码分别运行在不同的进程/线程上，如果在一个窗口只包含一个缓冲区(Buffer)，当客户端渲染时，服务端只能等待，当服务端合成时，客户端也只能等待。这很容易就造成阻塞，从而影响显示效果，也浪费 CPU 的能力。 

所以，在窗口对象的设计中，采用双缓冲(double-buffer)是很自然的事情。在双缓冲(double-buffer)方案里： 
* 一个窗口的有两个缓冲区(Buffer)：front buffer & back buffer。 
* 客户端把刚刚渲染完的帧(Frame)标记为 front buffer，另一个 buffer 等服务端合成完以后则为 back buffer。客户端在 back buffer 进行新的渲染。 
* 服务端发现有新的 front buffer，则取过来进行合成并显示出去。 
* 这样，在生产者和消费者两端的速度一致的情况下，客户端和服务端就很少发生竞争资源的情况，因此就很少发生阻塞，显示就会平滑。这也是经典的以空间换时间的方案。 

既然能有双缓冲(double-buffer)方案，为什么没有三缓冲(triple-buffer)方案？更进一步的，既然能有三缓冲(triple-buffer)方案，各种软件的需求不同，为什么没有多缓冲(multi-buffer)方案？于是，Android 图形系统里著名的 BufferQueue 类就产生了。 

BufferQueue 类在 Android 图形系统里是核心类，管理着一个可变长队列的缓冲池(Buffer Pool)。如其名字所言，它本质上就是管理缓冲区队列(Buffer Queue)，按照需求对外提供缓冲区(Buffer)。BufferQueue 类在后面的章节里会有详细的分析和讨论。 

# 新的窗口方案设计 

因此，窗口对象新的设计示意图如下：

![zero-copy & BufferQueue](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/original-design/05-zero-copy.svg)

在新的设计里：
* Client Window 从 BufferQueue 得到的永远是 back buffer。客户端软件将在上面生成新的内容。
* Server Composer 从 BufferQueue 得到的永远 是 front buffer。服务端软件将从中得到客户已经生成的内容。
* Client / BufferQueue / Server 三方，分别处于不同的进程/线程中。三方之间的状态同步，使用 IPC(binder) 进行通信。同时，尽管跨越了进程，zero-copy 的技术保证了一份内容只有一个拷贝。这极大地减少了内存消耗也极大地提高了效率。
* Buffer & Pool本身的管理，包括 size / count / status 等，将由 BufferQueue 管理。这样减轻了两端的工作量。

# 小结

最终，Android 图形系统在设计上最基本的思路和功能就是： 
* 客户端完成绘图或渲染功能。特点是采用 GPU 绘图或渲染，为此在 EGL / OpenGLES 方面提供了基于 Android 的扩展。经典的实现是 skia 和 hwui 库。 
* 服务端实现完整的合成功能。标志是提供了完整的矩阵变换、Porter-Duff 合成算法、颜色空间转换功能。 
* C/S 架构之间采用 zero-copy 方式同步数据。就是用文件句柄传输功能、IPC(binder) 和 Fence 功能来实现共享内存机制。
* Client / BufferQueue / Server 三方协同工作，在设计模式中就是一个经典的 Producer-Consumer 模式。我们将在下一章简单回顾在 Android 图形系统中常用的设计模式。

