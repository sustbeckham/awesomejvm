## 前言

每个程序员都或多或少遇到过相当多的疑难杂症问题排查的时刻。我自己也是工作中遇到许多稀奇古怪的问题。最开始我们排查问题使用的是jprofiler。特别是使用jprofiler来排查调用链路的耗时问题。如下图所示:

![333](http://mailshark.nos-jd.163yun.com/document/static/054E67A8EFE7DDB2F5DA000B3D07E9FD.jpg)

但是jprofiler只能用于排查一些本地的问题。对于一些生产环境的由于网络隔离在加上权限受限, jprofiler就不是那么好使了。这时候萌生了自己做个小工具的想法。同时参考了一些工具和apm的实现, 简单实现了所需的功能。 

我们现在思考下, 假设要开发一个java程序的监控工具，比如包含以下功能, 都需要怎么实现?

    1. 实时或周期性的获取java进程运行数据, 包括但不限于内存,线程,操作系统,GC等。
    2. 如何在运行时知道一个class是被哪个classloader加载的?
    3. 如何动态的知道一个方法的执行时间?(对于基础的排查性能问题很有用)
    4. 如何动态知道一个方法被调用时候的完整调用栈?
    5. 如何动态的知道一次调用下一个方法的入参,返回值?
    ...
 
## 基础部分

在这个【基础部分】里, 我们可以很轻松的解决上边的问题1。这要感谢JDK5后提供的两大神器：Instrument和management。前置提供了应用程序之外的代理（agent）程序来监测和协助运行在JVM上的应用程序, 后者可以实时获取应用程序的实时运行数据。

### Agent

我们遇到的第一个问题, 是如何将自己的监控程序和目标进程关联起来。
比如我1个monitor.jar,里边包含了我们的监控程序, 如何和生产环境正在运行的tomcat进程进行关联?

答案是JDK提供的agent机制。简单来说只需要做以下事情:

    1. 监控代码的jar中包含Agent-Class属性。该值的名字是自定义的agent类。
    2. 该类必须实现如下的方法:
       public static void agentmain(String agentArgs, Instrumentation inst);
    3. 使用VirtualMachine vm = VirtualMachine.attach(targetPid)关联到目标进程
    
其中Instrumentation非常重要, 后续还会说明。关于Agent-Class属性可以通过maven-assembly-plugin插件来设置:

    <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-assembly-plugin</artifactId>
        <executions>
            <execution>
                <goals>
                    <goal>single</goal>
                </goals>
                <phase>package</phase>
                <configuration>
                    <descriptorRefs>
                        <descriptorRef>jar-with-dependencies</descriptorRef>
                    </descriptorRefs>
                    <archive>
                        <manifestEntries>
                            <Premain-Class>xxx</Premain-Class>
                            <Agent-Class>xxx</Agent-Class>
                            <Can-Redefine-Classes>true</Can-Redefine-Classes>
                            <Can-Retransform-Classes>true</Can-Retransform-Classes>
                        </manifestEntries>
                    </archive>
                </configuration>
            </execution>
        </executions>
    </plugin>

更多内容可参考下文

    https://docs.oracle.com/javase/8/docs/api/java/lang/instrument/package-summary.html

![agent](https://cdn.app.compendium.com/uploads/user/e7c690e8-6ff9-102a-ac6d-e4aebca50425/f4a5b21d-66fa-4885-92bf-c4e81c06d916/Image/b179b390a311e8e80c849d2bd45972b5/java_agent_overview_min.png)    

### Instrumentation

核心内容都在java.lang.instrumentation包下, 主要的类有:

    public interface Instrumentation {
        ....
        java.lang.instrument.Instrumentation
        java.lang.instrument.ClassFileTransformer  
        ....
    }
    
ClassFileTransformer提供了类的转换功能, 可以对字节码进行修改。 而Instrumentation除了可以管理ClassFileTransformer之外, 还有一些其他功能。比如:

    getAllLoadedClasses()
    该方法可以获取当前虚拟机加载的所有Class对象。记住是所有。那在这里问题2就很好解决了。
    我只需要遍历这个结果集和给定的名字是否匹配即可。再通过getClassLoader()可以获取这个类到底被谁加载了。
    同时, getProtectionDomain().getCodeSource().getLocation().getFile() 还可以获取到当前类
    的具体路径, 对于排查问题会更有帮助。 

### 线程使用情况 

    java.lang.management.ThreadMXBean
    
    以下方法比较有用:
    getThreadCount()                //线程数
    getDaemonThreadCount()          //daemon线程数
    getPeakThreadCount()            //峰值
    getTotalStartedThreadCount()    //启动过的线程数
       
### 操作系统

    java.lang.management.OperatingSystemMXBean
    
    以下方法比较有用:
    getName()                       // 操作系统名称. 本机为 Mac OS X
    getArch()                       // 系统架构. 本机为 x86_64
    getAvailableProcessors()        // 处理器个数. 本机为 8
    getSystemLoadAverage()          // 过去1分钟的load
    getVersion()                    // 操作系统版本
     
### 内存

    java.lang.management.MemoryMXBean
    java.lang.management.MemoryManagerMXBean

### 垃圾回收

    java.lang.management.GarbageCollectorMXBean
    
    以下方法比较有用:
    getName()                       // 收集器英文名称
    getCollectionCount()            // 该收集器收集总次数
    getCollectionTime()             // 该收集器收集总时间(ms)
    
    注意:会有多个GarbageCollectorMXBean。
     
### 编译器

    java.lang.management.CompilationMXBean
    
    以下方法比较有用:
    getName()                       //返回JIT编译器名称
    getTotalCompilationTime()       //返回在编译上花费的累积耗费时间的近似值（以毫秒为单位）
    
    注意:需要调用isCompilationTimeMonitoringSupported方法来确定是否支持编译期的监控。

### 类加载

    java.lang.management.ClassLoadingMXBean

    以下方法比较有用:
    getLoadedClassCount()           //返回当前加载到 Java 虚拟机中的类的数量。
    getTotalLoadedClassCount()      //返回自 Java 虚拟机开始执行到目前已经加载的类的总数。
    getUnloadedClassCount()         //返回自 Java 虚拟机开始执行到目前已经卸载的类的总数。
    isVerbose()                     //测试是否已为类加载系统启用了 verbose 输出。
 
### 运行时数据

    java.lang.management.RuntimeMXBean
    
    以下方法比较有用:
    getName()                       //返回表示正在运行的 Java 虚拟机的名称
    getStartTime()                  //返回 Java 虚拟机的启动时间
    getManagementSpecVersion()      //返回正在运行的 Java 虚拟机实现的管理接口的规范版本。
    getSpecName()                   //返回 Java 虚拟机规范名称。
    getSpecVendor()                 //返回 Java 虚拟机规范供应商。
    getSpecVersion()                //返回 Java 虚拟机规范版本。
    getVmName()                     //返回 Java 虚拟机实现名称。本机为Java HotSpot(TM) 64-Bit Server VM
    getVmVendor()                   //返回 Java 虚拟机实现供应商
    getVmVersion()                  //返回 Java 虚拟机实现版本
    getInputArguments()             //返回传入的JVM启动参数
    getClassPath()                  //返回类路径
    getBootClassPath()              //返回bootstrap的path 
    
### 系统级别采集

之前整理了很多数据获取的方式，但是都是基于java进程本身的。这里介绍常用的基于linux系统本身的采集方式:

    cpu:        /proc/stat
    memory:     /proc/meminfo
    load:       /proc/loadavg
    网卡:       /proc/net/dev
    TCP&UDP:    /proc/net/snmp
    io:         /proc/diskstats

需要注意以下细节:

    1. proc文件系统是提供系统运行状态的利器。很多开源的linux监控系统都是基于proc来进行分析和开发。
    2. mac os并不支持proc。
    3. 不同的linux发行版在proc的输出展示上有微小的不同。比如redhat和fedora在展示TCP数据时列名稍稍有差异。                  

## 进阶

### class文件格式

对于理解JVM和深入理解Java语言， 学习并了解class文件的格式都是必须要掌握的功课。 原因很简单， JVM不会理解我们写的Java源文件， 我们必须把Java源文件编译成class文件， 才能被JVM识别， 对于JVM而言， class文件相当于一个接口， 理解了这个接口， 能帮助我们更好的理解JVM的行为；另一方面， class文件以另一种方式重新描述了我们在源文件中要表达的意思， 理解class文件如何重新描述我们编写的源文件， 对于深入理解Java语言和语法都是很有帮助的。 

    https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html
     
### 网络通信

思考2个问题:
    
    1. 本机应用部署了monitor.jar, 我们如果想通过命令行交互, 怎么做?
    2. 网络打通的远程服务器部署了monitor.jar, 命令行交互又该怎么做?

很容易的, 我们可以想到socket。摆在我们面前的有如下方案:

    netty
    mina
    nio

这里我建议直接上nio。原因如下:

    1. 我们要给monitor.jar减负。如果依赖了外部库, 会加重监控工具的臃肿程度;
    2. nio已经足够用了。这个交互并不需要多么高的性能。杀鸡焉用牛刀;
    3. mina的社区太不活跃了。虽然是个成熟的软件, 但是万一哪里踩坑都不好处理;    

### 会话保持

会话保持有2个重要的参数。

- 会话保持时间, 常量。一般定义为10分钟就够了。
- 触摸时间, 每次发起请求会更新触摸时间

可以后台起daemon线程, 周期性检查触摸时间和当前时间的差值是否超过了会话保持时间。如果超过需要关闭连接。

### 通信协议

既然确定了网络通信使用nio, 那我们务必要制定一套简单的通信协议, 能简明的告知服务端和客户端请求响应信息。这里我们拿dubbo来举例:

![323](https://upload-images.jianshu.io/upload_images/6687902-7c7d460007a0bf16.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/727/format/webp)

可以看到dubbo有明确的byte位来指明哪些byte存放什么内容。这样做的好处有2, 1是结构清晰, 2是可以构造出尽可能小的报文数据。在高并发的请求下是及其有用的(节省资源), 虽然我们这个project这里并不是很重要, 但是保持一个好习惯总是没错的。 

### 表达式语言 

通信的时候需要考虑是否有更灵活的表达方式。如果可以ognl是一个不错的选择。

## 核心

在很多场景下, 我们核心要处理的并不是直接获取数据, 而是通过改造目标类, 从而达到各种灵活的功能。 所以如何完成目标类的改造是核心问题。

### 选型

大部分情况下你可以有以下4种选择:

    javassist: 对于字节码以及指令良好的封装
    bytebuddy: 更优良的设计和封装, 并获得 Duke's Choice Awards 2015。也算是个明星项目了, 目前社区还很活跃, 国人开发的skywalking便是基于它
    asm:       成熟度高, 但相对更底层一些。需要开发者对字节码以及指令有一定了解 
    自研:      省省力气吧-_-

这里我的选择是直接上asm。理由是完成这个项目必不可少的需要了解字节码和方法指令, 那就干脆深入的去了解。其实ASM本身也提供了很多工具。

比如org.objectweb.asm.util.ASMifier就是一个带有Main方法的类。只需传入指定类的全名, 则会输出该类用asm描述的文本形式。

### 如何完成目标类的字节码更新

这里还是需要用到instrument工具。见:
 
    void addTransformer(ClassFileTransformer transformer, boolean canRetransform);
    void retransformClasses(Class<?>... classes) throws UnmodifiableClassException;

1. 首先通过addTransformer添加一个转换器。请务必canRetransform为true才可以完成运行时转换。
2. 然后调用retransformClasses来执行该转换器。此时你的transformer可以做任何你想做的猥琐事情。 

### 如何完成目标方法的改造
 
我们再回头看下问题3,4,5.

    3. 如何动态的知道一个方法的执行时间?(对于基础的排查性能问题很有用)
    4. 如何动态知道一个方法被调用时候的完整调用栈?
    5. 如何动态的知道一次调用下一个方法的入参,返回值?
 
可以看到可以规划为1个问题，那就是如何在目标方法调用前和返回前拦截?
ASM早就考虑到这种需求了, 所以给我们提供了完美的支持:
 
    org.objectweb.asm.commons.AdviceAdapter extends org.objectweb.asm.MethodVisitor
     
        protected void onMethodEnter() { }
     
        protected void onMethodExit(int opcode) { }
 
此时我们只需在对应方法的实现下完成自己的拦截代码即可。注意:onMethodExit方法是包含了正常返回以及异常返回的(抛出异常)。在这里问题3已经很好解决了。不考虑其他特殊情况下可以ThreadLocal存储一份时间来在onMethodEnter和onMethodExit之间比较即可。

问题4的解决方案也很简单。在前置方法使用 Thread.currentThread().getStackTrace()即可拿到完整的栈列表。只不过要注意考虑去掉一些无意义的调用行, 比如java.lang.Method.invokexxxx这种。
 
涉及到方法指令集操作的可以通过继承 org.objectweb.asm.commons.GeneratorAdapter 来完成。GeneratorAdapter提供了相当多的工具方法,比如我们常用的：

    loadArgArray() //获取参数列表
    box()          //装箱
    loadThis()     //实例方法加载index=0的slot
    ...

问题5在这里也很好解决, 通过GeneratorAdapter.loadArgArray()即可获取到。这里涉及到一点操作数栈的知识,下边会说。
    
###  如何取消目标方法的改造

当我们执行完成我们的监控后或连接关闭时, 需要取消类和方法的改造。否则改造后的类一直存在, 直到应用重启。 
如何取消类的改造也很简单, 参考ClassFileTransform的官方文档:

    If the implementing method determines that no transformations are needed,
    it should return <code>null</code>.

所以我们只需要在transform列表的末尾添加一个返回为空的transform即可。
   
### 类加载隔离

这里的自定义类加载器是必须的。我们很难在一个单独的jar包中完成所有的事情, 部分功能很有可能交给第三方去做, 如下:

    网络通信: Netty, Mina
    字节码指令: asm
    工具类: common-lang, guava

其中网络通信我们可以使用原生nio, 工具类可以自己来写, 但是asm的使用显得不可避免。这时会不可避免的和应用代码产生冲突。所以自定义ClassLoader很有必要。 

### jvm指令集

不管我们最终选择asm还是bytebuddy。对于jvm的指令集还是有必要深入了解一番的。相关的指令集可以大概分为以下几类:

#### 栈操作

    dup(包括dup2,dup2_x1等)
    swap
    pop

#### 常量

    xCONST(x代表了各种类型, 包括ACONST_NULL)
    BIPUSH
    SIPUSH
    LDC
    LDC2

#### 逻辑运算

    xADD
    xSUB
    xMUL
    xDIV
    xREM

#### 转型

    I2F,F2D,L2D ...

#### 对象&字段

    NEW
    PUTFIELD
    GETFIELD

#### 方法

    INVOKEVIRTUAL 声明过的方法
    INVOKESPECIAL private&构造器
    INVOKESTATIC
    INVOKEDYNAMIC 动态方法

#### 数组

    xALOAD
    xASTORE

#### 跳转&返回

    IFEQ,IFGE,IFNE...
    xRETURN,RETURN
    
如上列举出的指令相对会简单一些且会经常用到。这里有2个地方可以方便的查询这些指令:

    1. 当然是官网的jvm规范
    https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html
    2. 这里的解释相对官网会更通俗好懂一些
    https://cs.au.dk/~mis/dOvs/jvmspec/ref-Java.html 
    
### 操作数栈&本地变量表   

先上一张图,这张图比较老了。

![23232](http://incdn1.b0.upaiyun.com/2017/10/e074a3c18e1380822e110e92154977b6.png)

在JDK8之后, permanent区域已经被移除, 被新的MetaSpace所取代, 配合此文更佳: 

    http://ifeve.com/java-permgen-removed/
  
## 包装

写到这里, 核心的内容应差不多了。剩下的就是一些边角料。但是可以使你的project更加优雅和健全。

### 命令行解析

解析来自于命令行的参数并不是一件特别容易的事情。但是好在有优秀的工具:

    jcommander (http://jcommander.org/)
    jopts (https://github.com/jopt-simple/jopt-simple)

当然这里同样要考虑自己项目的复杂度, 我们的目标是尽可能做一个精简的监控程序。

### awk&shell 

如果你打算把你的project分享到gayhub上, awk&shell 务必要熟悉. 将监控程序做到自动化(下载,安装)  

    