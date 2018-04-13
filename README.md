# 线程编程指南 -- 关于线程编程

多年来，计算机的最高性能在很大程度上受限于计算机内核中单个微处理器的运算速度。然而，随着单个处理器的运算速度开始达到其实际限制，芯片制造商切换到多核设计来使计算机有机会同时执行多个任务。虽然OS X无论何时都在利用这些内核执行与系统相关的任务，但我们自己的应用程序也可以通过线程来利用这些内核。


## 什么是线程？

线程是在应用程序内部实现多个执行路径的相对轻量级的方式。在系统层面，程序并排运行，系统根据程序的需求和其他程序的需求为每个程序分配执行时间。但是在每个程序中，存在一个或多个可用于同时或以几乎同时的方式执行不同的任务的执行线程。系统本身实际上管理着这些执行线程，并调度它们到可用内核上运行。同时，还能根据需要提前中断它们以允许其他线程运行。

从技术角度讲，线程是管理代码执行所需的内核级和应用级数据结构的组合。内核级数据结构协调调度事件到线程和在一个可用内核中抢先调度线程。应用级数据结构包含用于存储函数调用的调用堆栈和应用程序需要用于管理和操作线程的属性和状态的结构。

在非并发的应用程序中，只有一个执行线程。该线程以应用程序的主例程开始和结束，并逐个分支到不同的方法或函数中，以实现应用程序的整体行为。相比之下，支持并发的应用程序从一个线程开始，并根据需要添加更多线程来创建额外的执行路径。每个新路径都有自己的独立于应用程序主例程中的代码运行的自定义启动例程。在应用程序中有多个线程提供了两个非常重要的潜在优势：
- 多个线程可以提高应用程序的感知响应能力。
- 多个线程可以提高应用程序在多核系统上的实时性能。

如果应用程序只有一个线程，那么该线程必须做所有的事情。其必须响应事件，更新应用程序的窗口，并执行实现应用程序行为所需的所有计算。只有一个线程的问题是它一次只能做一件事情。如果一个计算需要很长时间才能完成，那么当我们的代码忙于计算它所需的值时，应用程序会停止响应用户事件和更新其窗口。如果这种行为持续时间足够长，用户可能会认为我们的应用程序被挂起了并试图强行退出它。但是，如果将自定义计算移至单独的线程，则应用程序的主线程可以更及时地自由响应用户交互。

随着多核计算机的普及，线程提供了一种提高某些类型应用程序性能的方法。执行不同任务的线程可以在不同的处理器内核上同时执行，从而使应用程序可以在给定的时间内执行更多的工作。

当然，线程并不是解决应用程序性能问题的万能药物。线程提供的益处也会带来潜在的问题。在应用程序中执行多个路径可能会增加代码的复杂度。每个线程必须与其他线程协调行动，以防止它破坏应用程序的状态信息。由于单个应用程序中的线程共享相同的内存空间，所有它们可以访问所有相同的数据结构。如果两个线程试图同时操作相同的数据结构，则其中一个线程可能会以破坏数据结构的方式覆盖另一个线程的更改。即使有适当的保护措施，我们仍然需要对编译器优化保持注意，因为编译器优化会在我们的代码中引入细微的错误。

## 线程术语

在讨论线程及其支持技术之前，有必要定义一些基本术语。

如果你熟悉UNIX系统，则可能会发现本文档中的术语“任务”的使用有所不同。在UNIX系统中，有时使用术语“任务”来指代正在运行的进程。

本文档采用一下术语：
- 术语“线程”用于指代单独的代码执行路径。
- 术语“进程”用于指代正在运行的可执行文件，它可以包含多个线程。
- 术语“任务”用于指代需要执行的抽象工作概念。

## 线程的替代方案

自己创建线程的一个问题是它们会给代码添加不确定性。线程是一种相对较底层且复杂的支持应用程序并发的方式。如果不完全了解设计的含义，则可能会遇到同步或校时问题，其严重程度可能会从细微的行为变化到应用程序崩溃以及用户数据的损坏。

另一个要考虑的因素是是否需要线程或并发。线程解决了如何在同一进程中同时执行多个代码路径的具体问题。但是在有些情况下，并不能保证并发执行我们需要的工作。线程会在内存消耗和CPU时间方面为进程带来了巨大的开销。我们可能会发现这种开销对于预期的任务来说太大了，或者其他选项更容易实现。

下表列出了线程的一些替代方案。
| Technology | Description |
|---------------|--------------|
| Operation objects | 在OS X v10.5中引入的操作对象是通常在辅助线程上执行的任务的封装器。这个封装器隐藏了执行任务的线程管理方面，让我们可以自由地专注于任务本身。通常将操作对象与一个操作队列对象结合使用，操作队列对象实际上管理一个或多个线程上的操作对象的执行。 |
| Grand Central Dispatch (GCD) | 在OS X v10.6中引入的Grand Central Dispatch是线程的另一种替代方案，可以让我们专注于需要执行的任务而不是线程管理。使用GCD，我们可以定义要执行的任务并将其添加到工作队列中，该工作队列可以在适当的线程上处理我们的任务计划。工作队列会考虑可用内核的数量和当前负载，以便比使用线程更有效地执行任务。 |
| Idle-time notifications | 对于相对较短且优先级很低的任务，空闲时间通知让我们可以在应用程序不太忙时执行任务。Cocoa使用`NSNotificationQueue`对象为空闲时间通知提供支持。要请求空闲时间通知，请使用`NSPostWhenIdle`选项向默认`NSNotificationQueue`对象发布通知。队列会延迟通知对象的传递，直到run loop变为空闲状态。 |
| Asynchronous functions | 系统接口包含许多为我们提供自动并发性的异步功能。这些API可以使用系统守护进程和进程或者创建自定义线程来执行任务并将结果返回给我们。在设计应用程序时，寻找提供异步行为的函数，并考虑使用它们而不是在自定义线程上使用等效的同步函数。 |
| Timers | 可以在应用程序的主线程上使用定时器来执行相对于使用线程而言过于微不足道的定期任务，但是需要定期维护。 |
| Separate processes | 尽管比线程更加重量级，但在任务仅与应用程序切向相关的情况下，创建单独的进程可能很有用。如果任务需要大量内存或必须使用root权限执行，则可以使用进程。例如，我们可以使用64位服务器进程来计算大型数据集，而我们的32位应用程序会将结果显示给用户。 |

> **注意**：使用`fork`函数启动单独的进程时，必须使用与调用`exec`函数或类似函数相同的方式调用`fork`函数。依赖于Core Foundation，Cocoa或者Core Data框架（显式或隐式）的应用程序必须对`exec`函数进行后续调用，否则这些框架的行为可能会不正确。

## 线程支持

OS X和iOS系统提供了多种技术来在我们的应用程序中创建线程，并且还为管理和同步需要在这些线程上完成的工作提供支持。以下各节介绍了在OS X和iOS中使用线程时需要注意的一些关键技术。

### 线程组件

尽管线程的底层实现机制是Mach线程，但很少（如果有的话）在Mach层面上使用线程。相反，我们通常使用更方便的POSIX API或其衍生工具之一。Mach实现确实提供了所有线程的基本特征，包括抢先执行模型和调度线程使它们彼此独立的能力。

下表列出了可以在应用程序中使用的线程技术。

| Technology | Description |
|--------------|---------------|
| Cocoa threads | Cocoa使用`NSThread`类实现线程。Cocoa也在`NSObject`类中提供了方法来生成新线程并在已经运行的线程上执行代码。 |
| POSIX threads | POSIX线程提供了基于C语言的接口来创建线程。如果我们不是在编写一个Cocoa应用程序，则这是创建线程的最佳选择。POSIX接口使用起来相对简单，并为配置线程提供了足够的灵活性。 |
| Multiprocessing<br>Services | Multiprocessing Services（多处理服务）是传统的基于C语言的接口，其被从旧版本Mac OS系统中过渡来的应用程序所使用。这项技术仅适用于OS X，应该避免在任何新的开发中使用它。相反，应该使用`NSThread`类或者POSIX线程。 |

启动线程后，线程将以三种主要状态中的一种来运行：运行中，准备就绪或者阻塞。如果一个线程当前没有运行，那么它可能处于阻塞状态并等待输入，或者它已准备好运行，但尚未安排执行。线程持续在这些状态之间来回切换，直到它最终退出并切换到终止状态。

当创建一个新的线程时，必须为该线程指定一个入口函数（或者Cocoa线程的入口方法）。这个入口函数构成了我们想要在线程上运行的代码。当函数返回时，或者当我们明确终止线程时，该线程会永久停止并被系统回收。由于线程的创建在内存和时间方面相当昂贵，所有建议在入口函数中执行大量工作或者设置run loop以允许执行重复性工作。

### Run Loop

run loop（运行循环）是用于管理事件异步到达线程的基础架构的一部分。run loop通过监听线程的一个或者多个事件源来工作。当事件到达时，系统会唤醒线程并调度事件到run loop，run loop再调度这些事件给我们指定的处理程序。如果没有事件存在，也没有事件准备好被处理，则run loop将线程置于休眠状态。

不需要在创建任何线程时都使用run loop，但使用run loop可以为用户提供更好的体验。run loop使得创建使用最少量资源的长期存活线程成为可能。因为在没有事件传入时，run loop会将线程置于休眠状态。所以它不需要执行浪费CPU周期的轮询，并能防止处理器本身进入休眠状态来节省功耗。

要配置run loop，只需要启动线程，获取对run loop对象的引用，然后安装事件处理程序并告知run loop开始运行。OS X提供的基础架构自动帮我们处理主线程run loop的配置。如果打算创建长期存活的辅助线程，则必须自行为这些线程配置run loop。

### 同步工具

线程编程的一个风险是多线程之间的资源争夺。如果多个线程同时试图使用或修改相同的资源，则可能会出现问题。缓解问题的一种方法是完全避免共享资源，并确保每个线程都操作自己独特的资源集合。但是当保持完全独立的资源不能满足需求时，可以使用锁，条件，原子操作和其他技术来同步对资源的访问。

锁为一次只能由一个线程执行的代码提供了蛮力形式的保护。最常见的锁是互斥锁。当一个线程试图获取另一个线程当前拥有的互斥锁时，该线程会被阻塞，直到另一个线程释放该互斥锁。一些系统框架为互斥锁提供了支持，尽管它们都基于相同的基础技术。另外，Cocoa提供了互斥锁的几种变体来支持不同类型的行为，例如递归。

除了锁之外，系统还为条件（condition）提供支持，以确保在应用程序中对任务进行正确排序。条件充当守门人，阻塞指定的线程，知道它所代表的条件变为`ture`。当这种情况发生时，条件释放线程并运行其继续运行。POSIX层和Foundation框架都为条件提供了直接支持。（如果使用操作对象，则可以配置操作对象之间的依赖关系来对任务的执行排序，这与条件提供的行为非常相似。）

虽然锁和条件在并发设计中非常常见，但原子操作是保护和同步数据访问的另一种方式。当对标量数据类型进行数学或逻辑运算时，原子操作提供了一种轻量级的替代锁的方案。原子操作使用特殊的硬件指令来确保在其他线程有机会访问变量之前完成对该变量的修改。

### 线程间通信

尽管一个好的设计可以最大限度地减少所需的通信次数，但是在某些时候，线程之间的通信是必要的。线程可能需要处理新的工作请求或者将工作进度报告给应用程序的主线程。在这些情况下，我们需要一种从一个线程向另一个线程获取信息的方法。幸运的是，线程共享相同进程空间的事实意味着我们有很多通信选项。

线程之间的通信方式有许多种，每种方式都有自己的优点和缺点。下表列出了可以在OS X中使用的最常用的通信机制（除了消息队列和Cocoa分布式对象，其他技术在iOS中也可用。），此表中的技术按照复杂性增加的顺序列出。

| 机制 | 描述 |
|-------|------|
| 直接传递消息 | Cocoa应用程序支持直接在其他线程上执行方法选择器的功能。这个能力意味着一个线程实质上可以在任何其他线程上执行一个方法。由于它们是在目标线程的上下文中执行的，所以以这种方式发送的消息会自动在该线程上序列化。 |
| 全局变量，共享内存和对象 | 在两个线程之间传递信息的另一种简单方法是使用全局变量，共享对象或共享内存块。虽然共享变量很快很简单，但它们比直接传递消息更脆弱。共享变量必须用锁或其他同步机制来小心保护，以确保代码的正确性。不这样做可能会导致竞争状况，数据损坏或者崩溃。 |
| 条件 | 条件是一个同步工具，可以使用它来控制线程何时执行代码的特定部分。可以将条件视为守门员，让线程只有在符合条件时才能运行。 |
| Run loop sources | 自定义run loop source是为了在线程上接收专用消息而设置的。因为它们是事件驱动的，所以当没有任何事件可以执行时，run loop source会将线程置于休眠状态，这可以提高线程的效率。 |
| Ports and sockets | 基于端口的通信是两个线程之间通信的更复杂的方式，但它是一种非常可靠的技术。更重要的是，端口和套接字可用于与外部实体（如其他进程和服务）进行通信。为了提高效率，端口是使用run loop source实现的，所以当没有数据在端口上等待时，线程会休眠。 |
| 消息队列 | 传统的多处理服务定义了用于管理传入和传出数据的先进先出（FIFO）的队列抽象概念。尽管消息队列简单方便，但并不像其他通信技术那样高效。 |
| Cocoa分布式对象 | 分布式对象是一种Cocoa技术，提供基于端口通信的高级实现。尽管有可能使用这种技术进行线程间通信，但是由于其产生的开销很大，所以并不鼓励这样做。分布式对象更适用于与其他进程进行通信，其中进程之间的开销已经很高。 |

## 设计技巧

### 避免明确地创建线程

手动编写线程创建代码非常繁琐而且可能容易出错，应该尽量避免这样做。OS X和iOS其他API为并发提供隐式支持。可以考虑使用异步API，GCD或操作对象来完成工作，而不是自己创建线程。这些技术在幕后做与线程相关的工作，并保证正确执行。另外，像GCD和操作对象这样的技术可以根据当前系统负载调整当前活跃线程的数量，从而比我们自己的代码更高效地管理线程。

### 合理地保持我们的线程处于忙碌状态

如果决定手动创建和管理线程，请记住线程会占用宝贵的系统资源。应该尽最大努力确保分配给线程的任何任务是长期存活的和能工作的。同时，不应该害怕终止那些大多数时间处于闲置状态的线程。线程会占用大量的内存，因此释放一个空闲线程不仅有助于减少应用程序的内存占用量，还可以释放更多物理内存供其他系统进程使用。

> **提示**：在开始终止空闲线程之前，应该始终记录应用程序当前性能的一组基础测量结果。在尝试更改之后，请进行其他测量以验证这些更改是否实际上改善了性能，而不是损害了性能。

### 避免共享数据结构

避免与线程相关的资源冲突的最简单和最容易的方法是为程序中的每个线程提供它所需的任何数据的副本。当我们最小化线程间的通信和资源竞争时，并行代码的工作效果最佳。

创建多线程应用程序非常困难。即使我们非常小心并且在代码中在所有正确的时刻锁定了共享的数据结构，我们的代码仍然可能在语义上是不安全的。例如，如果希望共享数据结构按照特定顺序修改，我们的代码可能会遇到问题。将代码更改为基于交易的模型以进行补偿随后可能让具有多个线程的性能优势消失。首先消除资源争夺会让设计更加简单并且性能优异。

### 线程和我们的用户界面

如果应用程序具有图形用户界面，则建议从应用程序的主线程接收与用户相关的事件并启动界面更新。这种途径有助于避免与处理用户事件和绘制窗口内容相关的同步问题。一些框架，例如Cocoa，通常需要这种行为，但即使对于那些不这样做的行为，在主线程上保持这种行为也有简化用于管理用户界面的逻辑的优点。

有一些值得注意的例外是从其他线程执行图形操作是有利的。例如，可以使用辅助线程来创建和处理图像并执行其他图像相关的计算。使用辅助线程进行这些操作可以大大提高性能。如果不确定特定的图形操作，请在主线程执行此操作。

### 在退出时知道线程行为

一个进程运行直到所有非分离线程退出。默认情况下，只有应用程序的主线程是非分离的，但是也可以创建其他的非分离线程。当用户退出应用程序时，通常被认为是适当的行为是立即终止所有分离线程，因为分离线程完成的工作被认为是可选的。然而，如果我们的应用程序使用后台线程将数据保存到磁盘或者执行其他关键工作，则可能需要创建非分离线程，以防止应用程序退出时丢失数据。

创建非分离（也称为可连接）线程需要额外的工作。由于大多数高级线程技术在默认情况下不会创建可连接线程，所以我们可能必须使用POSIX API来创建线程。另外，我们必须添加代码到应用程序的主线程，以便主线程最终退出时将其与非分离线程连接起来。

如果我们正在编写一个Cocoa应用程序，则也可以使用`applicationShouldTerminate:`代理方法来延迟应用程序的终止直到以后某个时间或者完全取消延迟。当延迟应用程序的终止时，应用程序需要等待直到任何临界线程完成其任务，然后调用`replyToApplicationShouldTerminate:`方法。

### 处理异常

当抛出一个异常时，异常处理机制依赖于当前的调用堆栈来执行任何必要的清理。因为每个线程都有自己的调用堆栈，所以每个线程都负责捕获它自己的异常。当拥有的进程已经终止，在主线程和辅助线程中都是无法捕获到异常的。我们不能将一个未捕获的异常抛出到不同的线程进行处理。

如果需要通知另一个线程（例如主线程）当前线程中的异常情况，则应该捕获该异常并简单地向另一个线程发送消息表明发生了什么。取决于我们的模型以及我们试图执行的操作，捕获异常的线程可以继续处理（如果可能的话）、等待指令或者干脆退出。

> **注意**：在Cocoa中，`NSException`对象是一个自包含的对象，一旦它被捕获，它就可以从一个线程传递到另一个线程。

在某些情况下，可能会为我们自动创建异常处理程序。例如，Objective-C中的`@synchronized`指令包含一个隐式异常处理程序。

### 干净地终止我们的线程

让线程自然退出的最好方式是让其到达主入口点工作的末尾。虽然有函数能够立即终止线程，但这些函数只能作为最后的手段使用。在线程到达其自然终点之前终止它会阻止线程清理自身。如果线程已经分配内存、打开文件或者获取其他类型的资源，则我们的代码可能无法回收这些资源，从而导致内存泄露或者其他潜在问题。

### 库（Library）中的线程安全

虽然应用程序开发者可以控制应用程序是否使用多个线程执行，但库开发人员却不行。开发库时，我们必须假定调用库的应用程序是多线程的或者可以随时切换为多线程的。因此，我们应该始终为代码的临界区使用锁。

对于库开发人员来说，仅在应用程序变为多线程时才创建锁是不明智的。如果我们需要在某个时刻锁定我们的代码，请在使用库时尽早创建锁对象，最好在某个显示调用中初始化库。虽然也可以使用静态库初始化函数来创建此类锁，但只有在没有其他方式时才尝试这样做。初始化函数的执行会增加加载库所需的时间，并可能对性能产生负面影响。

> **注意**：始终记住锁定和解锁库中的互斥锁的调用要保持平衡，还应该记住要锁定库数据结构，而不是依赖调用代码来提供线程安全的环境。

如果我们正在开发一个Cocoa库并希望应用程序在变为多线程时能够收到通知，可以为`NSWillBecomeMultiThreadedNotification`通知注册一个观察者。但不应该依赖收到此通知，因为在我们的库代码被调用之前，可能已经发送了此通知。


# 线程编程指南 --  线程管理

OS X和iOS中的每个进程（应用程序）都由一个或多个线程组成，每个线程代表通过应用程序的代码执行的单个路径。每个应用程序都以单个线程开始，该线程运行应用程序的主要功能。应用程序可以创建额外的线程，这些线程执行特定功能的代码。

当一个应用程序创建一个新的线程时，该线程将成为应用程序进程空间内的一个独立的实体。每个线程都有其自己的执行堆栈，并由内核独立调度运行时间。一个线程可以与其他线程和其他进程通信，执行I/O操作和执行其他任何我们可能需要的操作。但是，由于它们在同一个进程空间内，所以单个应用程序中的所有线程共享相同的虚拟内存空间，并具有与进程本身相同的访问权限。

本章提供了OS X和iOS中可用线程技术的概述以及如何在应用程序中使用这些技术的示例。

## 线程开销

线程在内存使用和性能方面对应用程序（和系统）有实际的成本。每个线程会在内核内存空间和程序的内存空间中请求内存分配。管理线程和协调线程调度所需的核心结构使用wired memory存储在内核中。线程的堆栈空间和per-thread数据存储在应用程序的内存空间中。当我们首次创建线程时，这些结构的大多数才会被创建并初始化。由于必需的与内核的交互，进程可能相对更昂贵。

下表量化了与在应用程序中创建新的用户级别的线程相关的大概成本。其中一些成本是可配置的，例如为辅助线程分配的堆栈空间数量。创建线程的时间成本是一个粗略的近似值，应仅用于相互比较。创建线程的时间成本可能因处理器负载、 计算机的速度以及可用系统和程序内存的数量而有很大的差异。

| Item | Approximate | Notes |
|-------|----------------|--------|
| 内核数据结构 | 大约1 KB | 该内存用于存储线程数据结构和属性，其中大部分分配为wired memory，因此无法被分页到磁盘。 |
| 堆栈空间 | 512 KB（辅助线程）<br>8 MB（OS X 主线程）<br>1 MB（iOS 主线程） | 辅助线程允许的最小堆栈大小为16 KB，堆栈大小必须是4 KB的倍数。这个内存的空间在创建线程的时候被放置在进程空间中，但是与该内存相关联的实际页面只有在需要的时候才会被创建。 |
| 创建耗时 | 大约90微秒 | 该值反映了创建线程的初始调用到线程入口点开始执行的时间间隔。该数据是通过分析在基于Intel的使用2 GHz Core Duo处理器和运行OS X v10.5 的RAM为1 GB的iMac上创建线程时生成的平均值和中值而确定的。 |

> **注意**：由于底层内核的支持，操作对象通常可用更快地创建线程。它们不是每次都从头开始创建线程，而是使用已驻留在内核中的线程池来节省分配时间。有关如何使用操作对象的更多信息，请参看[iOS并发编程 -- Operation Queues](https://www.jianshu.com/p/65ab102cac60)。

编写线程代码时需要考虑的另一个成本是生产成本。设计线程应用程序有时可能需要对组织应用程序数据结构的方式进行根本性更改。为了避免同步的使用，进行这些更改可能是必要的。这些更改可能会对设计不当的应用程序带来巨大的性能损耗。设计这些数据结构和调试线程代码中的问题可能会增加开发线程应用程序所需的时间。但是，避免这些成本会在运行时产生更大的问题。

## 创建线程

创建低级线程相对简单。在任何情况下，都必须有一个函数或者方法来充当线程的主入口点，并且必须使用可用线程例程中的一个来启动线程。以下部分显示了更常用的线程技术的基本创建过程。使用这些技术创建的线程将继承默认的一组属性，这些属性由我们使用的技术决定。

### 使用NSThread

有两种使用`NSThread`类创建一个线程的方法：
- 使用`detachNewThreadSelector:toTarget:withObject:`类方法来生成新的线程。
- 创建一个新的`NSThread`对象并调用其`start`方法。（仅在iOS和OS X v10.5之后支持。）

这两种技术都会在应用程序中创建一个分离线程。分离线程意味着线程退出时线程的资源会被系统自动回收。

因为`detachNewThreadSelector:toTarget:withObject:`方法在所有版本的OS X中都受支持，所以在现有的使用线程的Cocoa应用程序中经常会见到它。要分离一个新线程，只需提供想要用作线程入口点的方法名称（指定为选择器）、 定义该方法的对象以及要在启动时传递给线程的任何数据。以下示例显示了此方法的基本调用，该方法使用当前对象的自定义方法生成线程。
```
[NSThread detachNewThreadSelector:@selector(myThreadMainMethod:) toTarget:self withObject:nil];
```
在OS X v10.5之前，主要使用`NSThread`类来生成线程。虽然我们可以得到一个`NSThread`对象并访问一些线程属性，但是只能在线程本身运行后才能这样做。在OS X v10.5中，添加了用于创建`NSThread`对象而不立即生成相应的新线程的支持。（此支持在iOS中也可用。）此支持使得在启动线程之前可以获取和设置各种线程属性成为可能，它还使得可以使用该线程对象稍后引用正在运行的线程成为可能。

在OS X v10.5及更高版本中初始化`NSThread`对象的简单方法是使用`initWithTarget:selector:object:`方法。此方法使用与` detachNewThreadSelector:toTarget:withObject:`方法完全相同的信息来初始化新的`NSThread`实例。但是，它不会立即启动线程。要启动线程，请明确调用线程对象的`start`方法，如下所示：
```
NSThread* myThread = [[NSThread alloc] initWithTarget:self selector:@selector(myThreadMainMethod:) object:nil];

[myThread start];  // Actually create the thread
```
> **注意**：一种使用`initWithTarget:selector:object:`方法的替代方案是对`NSThread`进行子类化并覆写其`main`方法。可以使用`main`方法的重写版本来实现线程的主入口点。更多信息，请参看[NSThread Class Reference](https://developer.apple.com/documentation/foundation/thread)。

如果我们有一个其当前线程正在运行的`NSThread`对象，则一种发送消息到该线程的方法是使用应用程序中几乎任何对象的`performSelector:onThread:withObject:waitUntilDone:`方法。在OS X v10.5中引入了对线程（主线程除外）执行选择器的支持，这是在线程之间进行通信的便捷方式。（此支持在iOS中也可用。）使用该技术发送的消息由其他线程直接执行，作为目标线程正常运行循环处理的一部分。（当然，这意味着目标线程必须在其run loop中运行。）当我们以这种方式进行通信时，可能仍然需要某种形式的同步，但它比在线程之间设置端口要简单。

> **注意**：虽然`performSelector:onThread:withObject:waitUntilDone:`方法适用于线程之间的偶尔通信，但不应该使用该方法来处理线程之间的时间紧张或频繁的通信。

### 使用 POSIX 线程

OS X和iOS为使用POSIX线程API来创建线程提供了基于C语言的支持。该技术实际上可以用于任何类型的应用程序（包括Cocoa和Cocoa Touch应用程序），如果我们正在为多个平台编写软件，该技术可能会更方便。

以下代码显示了两个使用POSIX调用的自定义函数。LaunchThread函数创建一个新的线程，其主例程在PosixThreadMainRoutine函数中实现。由于POSIX默认将线程创建为可连接，因此此示例更改了线程的属性来创建分离线程。将线程标记为分离，可以让系统在该线程退出时立即回收资源。
```
#include <assert.h>
#include <pthread.h>

void* PosixThreadMainRoutine(void* data)
{
    // Do some work here.

    return NULL;
}

void LaunchThread()
{
    // Create the thread using POSIX routines.
    pthread_attr_t  attr;
    pthread_t       posixThreadID;
    int             returnVal;

    returnVal = pthread_attr_init(&attr);
    assert(!returnVal);
    returnVal = pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);
    assert(!returnVal);

    int     threadError = pthread_create(&posixThreadID, &attr, &PosixThreadMainRoutine, NULL);

    returnVal = pthread_attr_destroy(&attr);
    assert(!returnVal);
    if (threadError != 0)
    {
        // Report an error.
    }
}
```
如果将以上代码添加到某个源文件并调用LaunchThread函数，这样会在应用程序中创建一个新的分离线程。当然，使用这段代码创建的新线程不会做任何有用的事情。线程讲启动并立即退出。为了使事情更有趣，我们需要将代码添加到PosixThreadMainRoutine函数中以完成一些实际工作。为了确保线程知道要做什么工作，可以在创建时传递一些数据的指针给它。将此指针作为`pthread_create`函数的最后一个参数传递。

为了将新创建的线程的信息传递回应用程序的主线程，需要在目标线程之间建立通信路径。对于基于C的应用程序，线程之间有多种通信方式，包括使用端口、 条件或共享内存。对于长寿命的线程，几乎总是应该建立某种线程间通信机制，以便让应用程序的主线程检查线程的状态或者在应用程序退出时干净地关闭线程。

### 使用NSObject来生成一个线程

在iOS和OS X v10.5及更高版本中，所有对象都能够生成一个新线程，并使用该线程来执行对象的方法中的一个。`performSelectorInBackground:withObject:`方法创建一个新的分离线程，并使用指定的方法作为该线程的入口点。例如，如果我们有一些对象（由变量myObj表示），并且这些对象有一个名为doSomething的方法，我们想要在后台线程中运行该方法，则可以使用以下代码执行此操作：
```
[myObj performSelectorInBackground:@selector(doSomething) withObject:nil];
```
调用此方法的效果与将当前对象、选择器和参数对象作为参数调用`detachNewThreadSelector:toTarget:withObject:`方法的效果相同。会立即使用默认配置生成新线程，并启动运行新线程。在选择器内部，必须像任何线程一样配置线程。例如，需要设置一个自动释放池并且配置线程的run loop（如果打算使用它的话）。

### 在Cocoa应用程序中使用POSIX线程

虽然`NSThread`类是在Cocoa应用程序中创建线程的主要接口，但是我们也可以自由使用POSIX线程（如果这样做更加方便的话）。如果打算在Cocoa应用程序中使用POSIX线程，仍然应该了解Cocoa和线程之间的交互，并遵循以下部分的指导原则。

#### 保护Cocoa框架

对于多线程的应用程序，Cocoa框架使用锁和其他形式的内部同步来确保它们的行为正确。但是，为了防止这些锁在单线程的情况下降低性能，Cocoa不会在应用程序使用`NSThread`类生成其第一个新线程之前创建它们。如果我们仅使用POSIX线程例程生成新线程，则Cocoa不会收到告知它我们的应用程序现在是多线程的通知。当发生这种情况时，涉及Cocoa框架的操作可能会破坏应用程序的稳定性或者崩溃。

为了让Cocoa知道我们打算使用多线程，需要使用`NSThread`类生成一个单线程，并让该线程立即退出，线程入口点不需要做任何事情。使用`NSThread`生成线程的行为足以确保创建Cocoa框架所需的锁。

如果不确定Cocoa是否认为我们应用程序是多线程的，则可以使用`NSThread`的`isMultiThreaded`方法进行检查。

#### 混合使用POSIX和Cocoa锁

在同一个应用程序中混合使用POSIX和Cocoa锁是安全的。Cocoa锁和条件对象本质上只是POSIX互斥锁和条件的包装器。但是，对于给定的锁，必须使用相同的接口来创建和操作该锁。换句话说，不能使用Cocoa NSLock对象来操作使用`pthread_mutex_init`函数创建的互斥锁，反之亦然。

## 配置线程属性

在创建线程之后（有时在之前），可能需要配置线程环境的不同部分。以下各节介绍了可以进行的一些更改以及何时可以进行更改。

### 配置线程的堆栈大小

对于我们创建的每个新线程，系统都会在我们的进程空间中分配特定数量的内存来充当该线程的堆栈。堆栈管理栈帧，也是声明线程的任何局部变量的地方。分配给线程的内存数量在[线程开销](turn)已列出。

如果想要更改给定线程的堆栈大小，则必须在创建线程之前执行此操作。虽然使用`NSThread`设置堆栈大小仅适用于iOS和OS X v10.5及更高版本，但是所有线程技术都提供了一些设置堆栈大小的方法。下表列出了每种技术的不同选项。

| Technology | Option |
|---------------|---------|
| Cocoa | 在iOS和OS X v10.5及更高版本中，分配并初始化一个`NSThread`对象（不要使用`detachNewThreadSelector:toTarget:withObject:`方法）。在调用线程对象的`start`方法之前，请使用`setStackSize:`方法来指定新的堆栈大小。 |
| POSIX | 创建一个新的`pthread_attr_t`结构体并使用`pthread_attr_setstacksize`函数更改默认堆栈大小。创建线程时，将属性传递给`pthread_create`函数。 |
| Multiprocessing Services | 在创建线程时将相应的堆栈大小值传递给`MPCreateTask`函数。 |

### 配置线程局部存储

每个线程维护着一个可以从任何位置访问的键-值对的字典。可以使用此字典来存储希望在整个线程执行期间都存在的信息。例如，我们可以使用它通过线程的run loop的多次迭代来保存状态信息。

Cocoa和POSIX以不同的方式存储线程字典，所以不能混合和匹配这两种技术。但是，只要在线程代码中坚持使用一种技术，最终结果应该是相似的。在Cocoa中，使用`NSThread`对象的`threadDictionary`方法来检索一个`NSMutableDictionary`对象，可以向其中添加线程所需的任何key。在POSIX中，使用`pthread_setspecific`和`pthread_getspecific`函数来设置和获取线程的key和value。

### 设置线程的分离状态

大多数高级线程技术默认创建分离的线程。在大多数情况下，分离线程是首选，因为它们运行系统在线程完成其工作后立即释放线程的数据的数据结构。分离线程也不需要与应用程序进行明确地交互，这意味着是否从线程中检索结果由我们自行决定。相比之下，系统不会回收可连接线程的资源，直到另一个线程显式地与该线程和可能会阻塞执行该连接的线程的进程连接。

可以考虑将可连接线程看作类似于子线程。虽然它们仍然作为独立线程运行，但可连接线程必须由另一个线程在其资源可能被系统回收之前连接。可连接线程还提供了一种方式将数据从一个正在退出的线程传递到另一个线程。在线程退出之前，可连接线程可以将数据指针或其他返回值传递给`pthread_exit`函数。然后另一个线程可以通过调用`pthread_join`函数来获取这些数据。

> **重要提示**：在应用程序退出时，分离线程会被立即终止，但是可连接线程不会被立即终止。每个可连接线程必须在允许退出进程之前连接。所以，在线程正在执行不应中断的关键工作（如将数据保存到磁盘）的情况下，可连接线程可能更可取。

如果想要创建可连接的线程，唯一的方法是使用POSIX线程。POSIX默认将线程创建为可连接。要将线程标记为分离或可连接，请在创建线程之前使用`pthread_attr_setdetachstate`函数修改线程属性。线程启动后，可以通过调用`pthread_detach`函数来将可连接线程更改为分离线程。

### 设置线程优先级

创建的任何新线程都具有与其关联的默认优先级。内核的调度算法在确定要运行哪些线程时会考虑线程优先级，优先级较高的线程比较低优先级的线程更可能运行。较高的优先级并不能保证线程的具体执行时间，只是与较低优先级的线程相比，调度程序更有可能选择它。

> **重要提示**：将线程的优先级保留为默认值通常是一个好主意。增加一些线程的优先级也增加了在较低优先级的线程中出现饥饿状况的可能性。如果应用程序包含必须彼此交互的高优先级和低优先级线程，则较低优先级线程的饥饿可能会阻塞其他线程并导致性能瓶颈。

如果确实想修改线程优先级，Cocoa和POSIX都可以这样做。对于Cocoa线程，可以使用`NSThread`的`setThreadPriority:`类方法来设置当前正在运行的线程的优先级。对于POSIX线程，可以使用`pthread_setschedparam`函数。

## 编写我们的线程入口例程

大多数情况下，OS X中的线程入口点例程的结构与其他平台上的相同。初始化数据结构，做一些工作或者可选地配置一个run loop，并在线程代码完成时清理。根据我们的设计，在编写入门例程时可能需要执行一些额外的步骤。

### 创建自动释放池

链接了Objective-C框架的应用程序通常必须在其每个线程中至少创建一个自动释放池。如果应用程序使用托管模型 -- 应用程序处理保留和释放对象的位置 -- 自动释放池将捕获该线程中自动释放的所有对象。

如果应用程序使用垃圾回收而不是托管内存模型，则不需要创建自动释放池。自动释放池的存在并不会对垃圾回收应用程序造成危害，大多数情况下都会被忽略。在允许代码模块必须同时支持垃圾回收和托管内存模型的情况下，自动释放池必须存在以便支持托管内存模型代码，并且如果应用程序在启用垃圾回收的情况下运行，则会被忽略。

如果应用程序使用托管内存模型，则创建自动释放池是在线程入口例程中首先执行的操作。同样，销毁这个自动释放池应该是在线程中做的最后一件事。该池确保自动释放的对象被捕获，在线程本身退出之前它不会释放它们。以下代码显示了使用自动释放池的基本线程入口例程的结构。
```
- (void)myThreadMainRoutine
{
    NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init]; // Top-level pool

    // Do thread work here.

    [pool release];  // Release the objects in the pool.
}
```
**由于顶级自动释放池在线程退出之前不会释放其对象，因此长期线程应创建更多的自动释放池来更频繁地释放对象。例如，使用run loop的线程可能会在每次运行循环时创建和释放自动释放池。更频繁地释放对象可防止应用程序的内存占用过大，从而导致性能问题。与任何与性能相关的行为一样，应该测量代码的实际性能，并适当调整自动释放池的使用。**

有关内存管理和自动释放池的更多信息，请参看[Advanced Memory Management Programming Guide](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/MemoryMgmt.html#//apple_ref/doc/uid/10000011i)。

### 设置异常处理程序

如果应用程序捕获并处理异常，则应准备好线程代码以便捕获可能发生的任何异常。尽管在发生异常的地方处理异常是最好的，但如果未能在线程中捕获抛出的异常，则会导致应用程序退出。在线程入口例程中安装最终的**try/catch**可以让我们捕获任何未知的异常并提供适当的响应。

在Xcode中构建项目时，可以使用C++或Objective-C异常处理样式。有关设置如何在Objective-C中引发和捕获异常的信息，请参看[Exception Programming Topics](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Exceptions/Exceptions.html#//apple_ref/doc/uid/10000012i)。

### 设置Run Loop

当编写想要在单独的线程上运行的代码时，我们有两种选择。第一种选择是将线程的代码编写为一个很少或者根本不中断的长期任务，并在该任务完成时退出线程。第二种选择是把线程放到一个循环中，当有请求到达时，动态处理请求。第一种选择不需要为代码进行特殊配置，只需要开启执行想要执行的工作。但是，第二种选择涉及到设置线程的run loop。

OS X和iOS为在每个线程中实现run loop提供了内置支持。应用程序框架自动启动应用程序主线程的run loop。如果为创建的任何辅助线程配置了run loop，则需要手动启动该run loop。

## 终止线程

建议退出线程的方式是让其正常退出入口点例程，虽然Cocoa，POSIX和Multiprocessing Services提供了直接杀死线程的例程，但是强烈建议不要使用这样的例程。杀死一个线程阻止了该线程清理自身的行为。由该线程分配的内存可能会泄漏，并且线程当前正在使用的任何其他资源可能无法被正确清理，之后可能会造成潜在问题。

如果预计需要在操作过程中终止线程，则应该从一开始就设计线程来响应取消或者退出消息。对于长时间运行的操作，这可能意味着要定期停止工作并检查是否收到了这样的消息。如果收到消息要求线程退出，线程将有机会执行任何需要的清理和正常退出。否则，它可能会重新开始工作并处理下一个数据块。

响应取消消息的一种方式是使用run loop输入源来接收此类消息。以下示例显示了这个代码在线程的主入口例程中的外观结构。（该示例仅显示主循环部分，不包括设置自动释放池或配置实际工作的步骤。）该示例在run loop中安装了一个自定义输入源，该输入源可以从另一个线程向该线程发送消息。在执行完总的工作量的一部分后，线程会简要地运行run loop来查看有没有消息到达输入源。如果没有，run loop会立即退出，并循环继续下一个工作。由于处理程序不能直接访问`exitNow`局部变量，所以退出条件通过线程字典中的键值对传递。
```
- (void)threadMainRoutine
{
    BOOL moreWorkToDo = YES;
    BOOL exitNow = NO;
    NSRunLoop* runLoop = [NSRunLoop currentRunLoop];

    // Add the exitNow BOOL to the thread dictionary.
    NSMutableDictionary* threadDict = [[NSThread currentThread] threadDictionary];
    
    [threadDict setValue:[NSNumber numberWithBool:exitNow] forKey:@"ThreadShouldExitNow"];

    // Install an input source.
    [self myInstallCustomInputSource];

    while (moreWorkToDo && !exitNow)
    {
        // Do one chunk of a larger body of work here.
        // Change the value of the moreWorkToDo Boolean when done.

        // Run the run loop but timeout immediately if the input source isn't waiting to fire.
        [runLoop runUntilDate:[NSDate date]];

        // Check to see if an input source handler changed the exitNow value.
        exitNow = [[threadDict valueForKey:@"ThreadShouldExitNow"] boolValue];
    }
}
```

# 线程编程指南 -- Run Loop

Run loop（运行循环）是与线程相关的基础架构的一部分。run loop是一个用于调度工作和协调到达事件的接收的事件处理循环。run loop的目的是在有工作做时让线程忙碌，并在没有工作可做时让线程进入休眠状态。

Run loop管理不是完全自动的，必须设计线程代码以便在适当的时间启动run loop并响应传入的事件。Cocoa和Core Foundation都提供了run loop对象来帮助我们配置和管理线程的run loop。应用程序不需要明确创建run loop对象，每个线程（包括应用程序的主线程）都有一个关联的run loop对象。但是，只有辅助线程需要显式运行其run loop。作为应用程序启动过程的一部分，应用程序框架自动设置并在主线程上运行run loop。

以下内容提供了有关run loop的更多信息以及如何为应用程序配置run loop。有关run loop对象的更多信息，请参看[NSRunLoop Class Reference](https://developer.apple.com/documentation/foundation/nsrunloop)和[CFRunLoop Reference](https://developer.apple.com/documentation/corefoundation/cfrunloop)。

## Run Loop详解

Run loop是一个线程进入循环，使用它来运行事件处理程序以便响应传入的事件。我们的代码提供了用于实现run loop的实际循环部分的控制语句——换句话说， 我们的代码提供了驱动run loop的while或者for循环。在循环中，使用run loop对象来“运行”用来接收事件和调用已安装的处理程序的事件处理代码。

Run loop从两种不同类型的源中接收事件。输入源传递异步事件，通常是来自另一个线程或不同应用程序的消息。定时器源传递同步事件，发生在特定的时间或间隔重复。这两种类型的源都使用应用程序特定的处理程序来处理到达的事件。

下图显示了run loop和各种源的概念上的结构。输入源传递异步事件给对应的处理程序，并导致`runUntilDate:`方法（在线程关联的`NSRunloop`对象上调用）退出。定时器源传递事件到其处理程序例程，但是不会导致run loop退出。

![Structure of a run loop and its sources.png](https://upload-images.jianshu.io/upload_images/4906302-383c2c603bbf18b8.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

除了处理输入的源之外，run loop还会生成有关run loop行为的通知。已注册的运行循环观察者能够接收通知，并使用它们对线程执行附加处理。使用Core Foundation来在线程上安装运行循环观察者。

以下部分提供了与run loop的组件和它们的运转模式相关的更多信息，还描述了处理事件期间在不同时间生成的通知。

### Run Loop模式

run loop模式是要监听的输入源和定时器的集合和要通知的运行循环观察者的集合。每次运行run loop时，都要明确或隐式地指定要运行的特定“模式”。在运行循环过程中，只监听与该模式有关的源并允许其传递事件。（同样，只有与该模式相关的观察者才会收到run loop的进度的通知。）与其他模式关联的源会保留任何新事件，直到随后已适当的模式通过循环为止。

在代码中，可以通过名称来识别模式。Cocoa和Core Foundation都定义了一个默认模式和几个常用模式，以及用于在代码中指定这些模式的字符串。可以通过简单地为模式名称指定一个自定义字符串来自定义模式。虽然分配给自定义模式的名称是任意的，但这些模式的内容不是。必须确保将一个或多个输入源、 定时器或运行循环观察者添加到为它们创建的任何模式中，使它们生效。

可以使用模式在通过run loop的特定关口期间过滤掉不需要的源中的事件。大多数情况下，需要在系统定义的“默认”模式下运行run loop。但是，modal panel可能会以“模态”模式运行run loop。在此模式下，只有与modal panel相关的源才会将事件传递给线程。对于辅助线程，可以使用自定义模式来防止低优先级的源在时间紧张的运作期间传递事件。

> **注意**：模式基于事件的源来进行区分，而不是事件的类型。例如，不会使用模式来仅仅匹配鼠标按下事件或者仅匹配键盘事件。可以使用模式来监听不同的端口集、 暂时暂停定时器或者更改当前正在监听的源和运行循环观察者。

下表列出了Cocoa和Core Foundation定义的标准模式以及何时使用该模式的说明。名称列列出了用于在代码中指定模式的实际常量。

| Mode | Name | Description |
|--------|--------|---------------|
| Default | NSDefaultRunLoopMode(Cocoa)<br>kCFRunLoopDefaultMode(Core Foundation) | 默认模式是用于大多数操作的模式。大多数情况下，应该使用此模式启动run loop和配置输入源。 |
| Connection | NSConnectionReplyMode (Cocoa) | Cocoa将此模式与`NSConnection`对象一起使用来监听应答。很少需要自己使用这种模式。 |
| Modal | NSModalPanelRunLoopMode (Cocoa) | Cocoa使用这种模式来识别用于modal panel的事件。 |
| Event tracking | NSEventTrackingRunLoopMode (Cocoa) | Cocoa使用这种模式来限定在鼠标拖拽循环和其他类型的用户界面跟踪循环期间传入的事件。 |
| Common modes | NSRunLoopCommonModes (Cocoa)<br>kCFRunLoopCommonModes (Core Foundation) | 这是一个常用模式的可配置组。将输入源与此模式相关联也会将其与组中的每个模式相关联。对于Cocoa应用程序，默认情况下，此集合包含默认、 模态和事件跟踪模式。Core Foundation最初只包含默认模式。可以使用`CFRunLoopAddCommonMode`函数将自定义模式添加到该集合中。 |



