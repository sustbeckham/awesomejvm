# JVM Anatomy Quark #9: JNI Critical and GC Locker

原文: https://shipilev.net/jvm/anatomy-quarks/9-jni-critical-gclocker/


![a](http://mailshark.nos-jd.163yun.com/document/static/6794A5F928D9B214EB9601DD7ED12010.png)

提问: JNI的Get\*Critical是如何和GC搭配的? 什么是GC Locker?

![b](http://mailshark.nos-jd.163yun.com/document/static/647B4495D42B63305A1E02E4C0FFC8CB.png)

如果您熟悉JNI，则知道有两种方法可以获取数组内容。 有Get\<PrimitiveType\>Array *系列方法，然后还有下边的Get/ReleasePrimitiiveArrayCritical：

	这两个函数的语义与现有的Get/Release*ArrayElements函数非常相似。如果可能，VM返回指
	向原始数组的指针；否则，将进行复制。但是，如何使用这些功能存在很大的限制。

这样做的好处是显而易见的:VM可以选择返回一个直接指针，而不是提供Java数组的副本，从而提高性能。这显然伴随着一些警告，这些警告在下面列出:

	调用GetPrimitiveArrayCritical之后，native代码在调用ReleasePrimitiveArrayCritical
	之前不应运行很长时间。 我们必须将这对函数(指这里的GetXXX和ReleaseXXX)中的代码视为在“临
	界区”中运行。 在临界区内，native代码不得调用其他JNI函数或任何可能导致当前线程阻塞并等待另
	一个Java线程的系统调用。 （例如，当前线程不得在另一个Java线程正在写入的流上调用read。）

	这些限制使native代码更有可能获得数组的未复制版本(怎么理解这个未复制版本?)，即使VM不支持固
	定(pinning)也是如此。 例如，当native代码持有指向通过GetPrimitiveArrayCritical获得的
	数组的指针时，VM可以暂时禁用垃圾收集。

![c](http://mailshark.nos-jd.163yun.com/document/static/DA9C4A117154B1B422C99285BF7E0AFF.png)

实际上，vm要维护的唯一强不变量是“关键”获取的对象不会移动。实施可以尝试不同的策略：

- 获取任何临界区对象时，请完全禁用GC。 到目前为止，这是最简单的应对策略，因为它不会影响其余的GC。 缺点是您必须无限期地阻止GC（基本上要让用户放心，要足够快地“释放”），这可能会引起问题。
- 固定对象，并在收集过程中对其进行处理。 如果收集器希望分配连续的空间，并且/或者希望收集器处理整个堆子空间，那么很难做到这一点。 例如，如果将对象固定在年轻代中，则现在无法“忽略”收集后剩下的年轻对象。 您也不能从那里移动对象，因为它破坏了您要强制执行的不变性。
- 将包含对象的子空间固定在堆中。 再说一次，如果GC对整个generation来说都是细粒度的，那将无济于事。 但是，如果您具有区域化的堆，则可以固定单个区域，并避免仅对该区域使用GC，从而使每个人都满意。

我们已经看到人们依靠JNI Critical临时禁用GC，但这仅适用于第一条，并且并非每个收集器都采用这种简单的行为。

![d](http://mailshark.nos-jd.163yun.com/document/static/411A9C75E64B19F1C6AB31135D25C076.png)

和往常一样，我们可以通过构造一个使用JNI Critical获取int []数组的实验来研究它，然后在完成处理后故意忽略释放数组的建议。 相反，它将在获取和释放之间分配和保留许多对象：

![e](http://mailshark.nos-jd.163yun.com/document/static/82EA4EDB8C57E54832BBE4797389BF8D.png)

这里这位老哥其实讲的不是很清楚。其实除了他提供的例子之外还要额外做好几件事情:

- 生产.h头文件, 通过javah命令
- 生成.o文件
	    gcc -I /Library/Java/JavaVirtualMachines/jdk1.8.0_161.jdk/Contents/Home/include/ -c CriticalGC.c
- 上一步需要注意的是jni.h在darwin目录下需要手工复制到上层目录
- 生成动态链接库(macos下后缀为dylib且名称要以lib开头), gcc -dynamiclib -o libCriticalGC.dylib CriticalGC.o

![f](http://mailshark.nos-jd.163yun.com/document/static/EA4BEBFC7D91288435A4D286F77B10FB.png)

 smoking gun在这里的意思是"确凿的证据"。

![g](http://mailshark.nos-jd.163yun.com/document/static/98AB642DAEFE7EA2A542CE9C05CFFE82.png)

如果尝试了GC，则JVM应该查看是否有人持有该锁。 如果有人这样做，那么至少对于Parallel，CMS和G1，我们无法继续使用GC( 这意味着其他的垃圾回收还是可以继续的么? 比如? )。 当最后一个关键的JNI操作以“ release”结束时，VM会检查是否有GCLocker阻止了待处理的GC(这里很重要。不是说最后一个release一定要GC, 而是看有没有hold的GC)，如果存在，则会触发GC。 这将产生一次GCLocker Initiated GC。

上图的triggers_GC指向了gcLocker的源码[http://hg.openjdk.java.net/jdk9/jdk9/hotspot/file/f36e864e66a7/src/share/vm/gc/shared/gcLocker.cpp#l138](http://hg.openjdk.java.net/jdk9/jdk9/hotspot/file/f36e864e66a7/src/share/vm/gc/shared/gcLocker.cpp#l138)。

![h](http://mailshark.nos-jd.163yun.com/document/static/33792304DC1F285285DCFD66663E3C6F.png)

在使用G1的情况下, 作者的例子中这里hang住了。我本地尝试也是hang住了, 输出了个Acquired就一直停在那里,然后jstack显示也是和上图一直在runnable状态。

![i](http://mailshark.nos-jd.163yun.com/document/static/760BC0C9593EA6A6313EF8117A16B525.png)

作者这里建议使用fastdebug的版本说会在断言处出现crash。我本地build后果然也会出现

    time /Users/mozilla/core/universe/jvm-openjdk9/build/macosx-x86_64-normal-server-slowdebug/images/jdk/bin/java -Djava.library.path=. -Xms4g -Xmx4g -verbose:gc -XX:+UseG1GC CriticalGC

	A fatal error has been detected by the Java Runtime Environment:

	Internal Error (/Users/mozilla/core/universe/jvm-openjdk9/hotspot/src/share/vm/gc/shared/gcLocker.cpp:96), pid=99012, tid=6659
	assert(!JavaThread::current()->in_critical()) failed: Would deadlock

和作者看到结果描述一致。

仔细观察这个堆栈跟踪，我们可以还原到底发生了什么：

- 我们尝试分配新对象(allocate_instance)，
- 没有TLAB可以满足分配要求
- 因此我们跳到慢路径分配以尝试获取新的TLAB(allocate_new_tlab)
- 然我们发现没有可用的TLAB，尝试分配，失败，并发现我们需要等待GCLocker initiate GC。 - 进入stall_until_clear等待…
- 但是由于我们是持有GCLocker的线程(在代码的acquire底层调用JNI_getPrimitiveArrayCritical)，因此在这里等待会导致死锁, Boom。

白话来讲就是我拿着GCLocker, 然后我特么不停的分配内存, 结果分配不下了要GC, 但是由于我自己是持有GCLocker的所以不允许GC, 死锁了。之前的CMS没有发生死锁是因为CMS发现内存不足的时候并不会去等GCLocker, 而G1却调用了。

![j](http://mailshark.nos-jd.163yun.com/document/static/5BD9C6C5DE57821FE5CB367DD4122E6A.png)

这是在规范之内的，因为测试已尝试在获取释放块中分配内容。 留下没有成对发布的JNI方法是一个使我们暴露于此的错误。 如果我们还没有离开，就无法在不调用JNI的情况下分配获取释放，这违反了“您不能调用JNI函数”的原则。
您可以调优收集器的测试以免失败，但是您会发现GCLocker延迟收集意味着当堆中的空间已经太少时我们可以启动GC，这将迫使我们进入Full GC。哎呀。

![k](http://mailshark.nos-jd.163yun.com/document/static/40FCA2B66E000A1D20530563380C4249.png)

UseShenandoahGC暂时在更高级别的hotspot中才会用到, 这里先略去。

![l](http://mailshark.nos-jd.163yun.com/document/static/F45DC8517170284D204734542D7DB1BE.png)

总结
处理JNI Critical需要VM的协助，以使用类似GCLocker的机制禁用GC，或者固定包含对象的子空间，或者单独固定对象。 不同的GC采用不同的策略来处理JNI Critical，并且在使用一个收集器运行时会看到明显的副作用（例如，延迟GC周期），在其他收集器上可能看不到。

请注意，规范说(这里的规范指的是JNI规范)：“在临界区内，native代码不得调用其他JNI函数”，这是最低要求。 上面的示例强调了这样一个事实，即在允许的规范范围内，实现的质量定义了破坏规范的严重程度。 一些GC可能会让更多的事情发生变化，而另一些GC可能会更具限制性。 如果想具备可移植性，请遵守规格要求，而不是实施细节。

或者，如果您依赖于实现细节（这是一个坏主意），并且使用JNI遇到了这些问题，请了解收集器在做什么，然后选择合适的GC。