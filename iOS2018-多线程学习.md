# iOS多线程学习

###1. 进程与线程

进程是一个正在执行的程序实例。进程是CPU调度的基本单位。在现代的操作系统中，线程才是 CPU 调度的基本单位。而进程作为线程的容器，是资源管理的单位。线程的执行也是串行的，采用与进程相同的调度算法使其并发执行

每个进程都拥有独立且受保护的内存空间，用来存放程序正文和数据以及其打开的文件、子进程、即将发生的报警、信号处理程序、账号信息等。线程只拥有程序计数器、寄存器、堆栈等少量资源，但与其他线程共享该进程的整个内存空间。因此线程切换速度比进程快 10 到 100 倍。

进程分为前台进程和后台进程。iOS 中的后台进程受到了极大的限制。后台进程只可以存在短暂的一段时间就会被系统置为 Suspended 状态。在这种状态下，进程将不能得到 CPU 时间片。当收到内存警告时，系统就会将处在 Suspended 状态后台进程从内存中移除。

<img src='http://o74he8slr.bkt.clouddn.com/thread3.png'>


###并发与并行

并发指能够让多个任务在逻辑上同时执行的程序设计，而并行则是指在物理上真正的同时执行。并行是并发的子集，属于并发的一种实现方式。通过时间片轮转实现的多任务同时执行是通过调度算法实现逻辑上的同步执行，属于并发，他们不是真正物理上的同时执行，不属于并行。当通过多核 CPU 实现并发时，多任务是真正物理上的同时执行，才属于并行。



###为什么使用多线程

但真正的程序中总会有 I/O 密集型线程。正在处理 I/O 的线程大部分时间都处在等待状态，它们不占用 CPU 资源。这时候，线程数量只有大于 CPU 数量时才能保证 CPU 高效运行。所以在遇到 I/O 任务时我们最好开启新的线程。这样做还有另一个好处。iOS 中只有在主线程才可以刷新 UI，如果这些 I/O 任务放在主线程，就可能会阻塞主线程后续的 UI 刷新任务，使界面产生卡顿。

所以使用多线程可以充分利用现在的多核 CPU、减少 CPU 的等待时间、防止主线程阻塞等。除了性能上的提升，对于批量任务，使用多线程也能使代码逻辑更加清晰。

###多线程应用实例

YYDispatchQueuePool 通过 CPU 核心数来限制总的线程数量（实际上只是将数量限制在合理的范围内），提高 CPU 利用率的同时又尽量减少线程切换的开销。

SDWebImage 在子线程批量处理从磁盘读取图片的任务。在 I/O 操作频繁的情况下，通过多线程充分利用等待时间，同时防止了主线程的阻塞。

###Pthreads介绍

Pthreads 定义了 C 语言的接口，拥有超过 100 个 API 用来创建和管理线程，这些 API 全都以 pthread_ 作为前缀。iOS 中 CFRunLoop 就是基于 Pthreads 来管理的。


###NSThread接口说明

苹果对 Pthreads 进行了面向对象的封装 NSTherad（也可以认为 NSTherad 只是用到了 Pthreads）。但在 iOS 开发中苹果为多线程提供了一套不用接触线程概念的 API —— GCD。Objective-C 进一步封装了这套 API，暴露给用户的是 NSOpertion 相关的对象。

#### 创建线程
NSThread 对象被创建时并不代表一个真正的线程也随之创建，只要当我们调用 NSThread 的 star 方法时才会创建真正的线程

```swift
//创建线程的两种方式
- (instancetype)initWithTarget:(id)target selector:(SEL)selector object:(nullable id)argument;
- (instancetype)initWithBlock:(void (^)(void))block;

//block

//创建线程的两种方式
var thread = Thread.init {
    print("Thread init - blockKKKKK");
}
thread.start() //真正创建了一个线程


let threadClass =  ThreadClass()
//selector
var seletorThread = Thread.init(target:threadClass, selector: #selector(ThreadClass.threadStart) , object: nil);
seletorThread.start()

```

也可以使用如下类方法来创建线程。这样创建完线程后还会自动调用 start 方法启动该线程。由于这两个方法没有返回值，我们无法在外部拿到线程对象，但我们也不用自己去管理线程对象的生命周期。

```swift
+ (void)detachNewThreadWithBlock:(void (^)(void))block;
+ (void)detachNewThreadSelector:(SEL)selector toTarget:(id)target withObject:(nullable id)argument;

//无返回值创建线程，并且自动start
Thread.detachNewThread {
    print("detachNewThread init");
}
//无返回值创建线程，并且自动start
Thread.detachNewThreadSelector(#selector(ThreadClass.threadAutoStart), toTarget: threadClass, with: nil);

```
除了创建自己的子线程外，我们也可以获取已存在的线程。NSThread 只提供了两个方法分别用来获取当前线程和主线程。
```objectivec
@property (class, readonly, strong) NSThread *currentThread;
@property (class, readonly, strong) NSThread *mainThread;
```
#### 启动线程
创建的线程对象后需要手动调用 start 方法才能启动线程。如果我们在创建线程时有制定任务，start 方法就会调用线程的入口方法 - mian。main 方法的默认实现会执行初始化时的 block 或 selector。main 方法只能通过子类重写，不能直接调用。子类化 NSThread 重写 main 方法时不比调用 super，我们可以按照自己的逻辑实现入口方法。
```objectivec
- (void)start;
- (void)main;
```
#### 派发任务
主线程在 App 运行期间始终存在，如果想向主线程的 Runloop 派发任务，可以使用下面的两个方法

如果想向子线程派发任务，则需要先手动启动子线程的 Runloop。然后使用performSelector:onThread:~两个方法向指定线程派发任务。当然如果你指定的线程是主线程，它们的效果就与前两个相同了。

最后我们也可以使用performSelectorInBackground:~向系统默认的后台线程派发任务。相比之下我们不需要自己管理子线程的生命周期，省去了需要不必要的麻烦。


```swift
Thread.performSelector(onMainThread: <#T##Selector#>, with: <#T##Any?#>, waitUntilDone: <#T##Bool#>)
Thread.performSelector(onMainThread: <#T##Selector#>, with: <#T##Any?#>, waitUntilDone: <#T##Bool#>, modes: <#T##[String]?#>)

// 在指定线程和指定 Runloop Mode 同步或异步执行任务
func perform(_ aSelector: Selector, on thr: Thread, with arg: Any?, waitUntilDone wait: Bool, modes array: [String]?)

func perform(_ aSelector: Selector, on thr: Thread, with arg: Any?, waitUntilDone wait: Bool)


// 在后台线程执行任务
Thread.performSelector(inBackground: <#T##Selector#>, with: <#T##Any?#>)

```

#### 优先级
线程优先级决定了任务开始执后系统资源分配的优先级。NSThread 的优先级通过浮点数变量 threadPriority 来控制，它的范围从 0 到 1，默认为 0.5。但这个属性在 iOS8 之后已经被废弃，取而代之的是枚举 NSQualityOfService（QoS）。

```objectivec
+ (double)threadPriority;
+ (BOOL)setThreadPriority:(double)p;
@property double threadPriority;
```

QoS 有五种优先级，默认为 NSQualityOfServiceDefault。它的出现统一了 Cocoa 中所有多线程技术的优先级。在此之前，NSOperation 和 NSThread 都通过 threadPriority 来指定优先级，而 GCD 则是根据 DISPATCH_QUEUE_PRIORITY_DEFAULT 等宏定义的整形数来指定优先级。正确的使用新的 QoS 来指定线程或任务优先级可以让 iOS 更加智能的分配硬件资源，以便于提高执行效率和控制电量。

```objectivec
@property NSQualityOfService qualityOfService;

typedef NS_ENUM(NSInteger, NSQualityOfService) {
    NSQualityOfServiceUserInteractive = 0x21,
    NSQualityOfServiceUserInitiated = 0x19,
    NSQualityOfServiceUtility = 0x11,
    NSQualityOfServiceBackground = 0x09,
    NSQualityOfServiceDefault = -1
};
```

- NSQualityOfServiceUserInteractive：用来处理用户操作，例如界面刷新、动画等。优先级最高，即时执行。
- NSQualityOfServiceUserInitiated：处理初始化任务，为将来的用户操作作准备。例如加载文件或 Email 等。基本即时执行，最多几秒延迟。
- NSQualityOfServiceUtility：用户不需要立即结果的操作，一般伴随进度条。例如下载、数据导入、周期性的内容更新等。几秒到几分钟延迟。
- NSQualityOfServiceBackground：用于用户不可见的操作。例如简历索引、预加载、同步等。几分钟到数小时延迟。
- NSQualityOfServiceDefault：默认的 QoS 用来表示缺省值。当有可能通过其它途径推断出可能的 QoS 信息时，则使用推断出的 Qos。如果不能推断，则使用 UserInitiated 和 Utility 之间的 QoS。

Utility 及以下的优先级会受到 iOS9 中低电量模式的控制。另外，在没有用户操作时，90% 任务的优先级都应该在 Utility 之下

#### 休眠线程
在线程的运行期间我们可以调用如下方法使执行此行代码的线程进入休眠。调用的时候都需要指定休眠的时间，sleepUntilDate指定的是休眠到某个时间，sleepForTimeInterval指定的是休眠的秒数。线程休眠时 Runloop 不会被事件唤醒。

```objectivec
+ (void)sleepUntilDate:(NSDate *)date;
+ (void)sleepForTimeInterval:(NSTimeInterval)ti;
```

#### 线程信息

线程在运行期间有多种状态。可以用如下三个属性来判断线程的运行状态。
```swift
@available(iOS 2.0, *)
open var isExecuting: Bool { get }

@available(iOS 2.0, *)
open var isFinished: Bool { get }

@available(iOS 2.0, *)
open var isCancelled: Bool { get }
```
在线程初始化完成后，通过如下方法和属性，可以设置线程的名字，还可以在线程 start 之前设置线程堆栈的大小（单位 byte，最小 16KB 且必须为 4KB 的整数倍）。在线程运行期间可以动态的获取线程的调用堆栈以及调用堆栈的返回地址，还可以判断当前线程是否为主线程。
```objectivec
// 调用堆栈的返回地址
@property (class, readonly, copy) NSArray<NSNumber *> *callStackReturnAddresses;
// 调用堆栈的回溯
@property (class, readonly, copy) NSArray<NSString *> *callStackSymbols;
// 线程名字
@property (nullable, copy) NSString *name;
// 线程堆栈大小
@property NSUInteger stackSize;
```

如果想在线程的运行之前储存一些线程依赖的数据，可以使用 threadDictionary 属性。它被定义为只读的可变字典类型，防止直接的指针赋值等“误操作”替换掉系统储存的数据。NSThread 本身没有使用到这个属性，但是 Cocoa 中的其他类可能会使用它。例如，Foundation 用它来存储线程默认的 NSConnection 和 NSAssertionHandler 实例。所以我们在用它储存数据时要避免与系统的 Key 重名。
```OBJECTIVEC
@property (readonly, retain) NSMutableDictionary *threadDictionary;
```

使用如下方法可以判断当前线程是否为主线程。
```OBJECTIVEC
// 接受此消息的线程是否为主线程
@property (readonly) BOOL isMainThread;
// 执行此行代码的线程是否为主线程
@property (class, readonly) BOOL isMainThread;
```
#### 多线程状态

当有任意一个线程从主线程分离出去时，App 就被认为是多线程的。我们可以通过 isMultiThreaded 方法判断 App 当前是否处在多线程状态（Pthread 等非 Cocoa API 创建的线程不算）。只要某个子线程被创建后（这里的线程是正真的线程，不是 NSThread 对象），不需要正在运行，就认为是多线程状态。

当 App 将要变为多线程时我们还可以通过 NSWillBecomeMultiThreadedNotification 通知来监听此状态。另外 NSThread 的接口中还有 NSDidBecomeSingleThreadedNotification 这个通知，但这个通知苹果并没有实现，放在这里逗你玩而已。

```objectivec
+ (BOOL)isMultiThreaded;
FOUNDATION_EXPORT NSNotificationName const NSWillBecomeMultiThreadedNotification;
FOUNDATION_EXPORT NSNotificationName const NSDidBecomeSingleThreadedNotification;
```

#### 退出线程

退出线程有两个方法，cancel 和 exit。cancel 方法将线程置为 cancelled 状态以表明线程将要退出，线程会执行完正在执行的任务才退出。而 exit 方法会立即退出当前线程，正在执行中的任务分配的资源将没有机会得到释放，所以正常情况下最好调用 cancel 方法。在线程将要退出时可以通过 NSThreadWillExitNotification 通知来监听此事件。

```objectivec
- (void)cancel;
+ (void)exit;
FOUNDATION_EXPORT NSNotificationName const NSThreadWillExitNotification;
```

###GCD的概念

GCD（Grand Central Dispatch） 是苹果宣称用来替换线程的一套基于 C 语言的 API。这套 API 引入了编程范式的变化，使从线程和线程函数的角度思考，变为从任务和队列的角度思考。GCD 使用队列来派发任务（block），队列分为串行队列和并发队列，任务的派发方式分为同步派发和异步派发。GCD 自己维护了一个底层的线程库实现，以支持并发和异步的执行模型。使用 GCD 可以减轻开发者处理并发问题的负担，减少类似于死锁之类的潜在错误，而且能够自动地随着逻辑处理器的个数而扩展。

队列都是先进先出的，所以串行队列和并发队列都是按照顺序执行的。但是串行队列需要等前一个任务执行完成才可以执行下一个任务，所以串行队列只需要一个线程就可以完成任务派发。而并发队列可以允许多个任务同时执行，虽然他们开始执行的时间是按照顺序的，但是执行完成的时间并不确定。并发执行只有通过新建线程来实现。还有一种特殊的串行队列-主队列，主队列的任务只能在主线程执行，并且需要等待主线程 Runloop 空闲时才能派发。



###NSOperation接口说明

Objective-C 对 GCD 的 API 进行了面向对象的封装，GCD 中的任务对应 NSOpertion 对象，GCD 中的队列则对应 NSOpertionQueue 对象。NSOpertion 和 NSOpertionQueue 还提供判断执行状态、取消任务、控制线程数量等更多任务管理的 API。所以 AFNetworking 与 SDWebImage 等管理大量独立任务的第三方都主要使用 NSOperation 实现多线程。

NSOperation 对“任务”进行了抽象。作为抽象基类，它为子类提供了十分有用且线程安全的方式来建立状态、优先级、依赖等模型。系统提了 NSBlockOperation 和 NSInvocationOperation 两个分别以 Block 和 Invocation 储存任务的具体实现。你也可以自己继承 NSOperation 实现自己特有的储存/执行任务的方式。

NSOperation 中的任务只能执行一次。将 NSOperation 添加到 NSOperationQueue 中后就会在子线程自动执行（直接执行或使用 GCD 间接执行）。如果你不想使用 NSOperationQueue，也可以手动调用 NSOperation 的 start 方法来执行它。调用 start 方法默认会在当前线程执行，而且你必须保证在 NSOperation 的 ready 状态下调用，否则将会抛出异常。如果想让 NSOperation 在子线程执行，需要我们手动将其放在子线程，我们也可以子类化 NSOperation 并重写 main 方法实现这个过程。这显然给我们带了很多的麻烦，所以除非特别需要，最好都使用 NSOperationQueue 来执行 NSOperation。

