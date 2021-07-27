
### 如何解决OOM
1. 要解决 OOM 异常或 heap space 的异常，一般的手段是首先通过内存映像分析工具（如 Eclipse Memory Analyzer）对 dump （HeadDumpOnOutOfMemoryError）出来的堆
转储快照进行分析，重点是确认内存中的对象是否是必要的，也就是要先分清楚到底是出现了内存泄漏（Memory Leak）还是内存溢出（Memory Overflow）

2. 如果是内存泄漏，可进一步通过工具查看泄漏对象到 GC Roots 的引用链。于是就能找到泄漏对象是通过怎样的路径与 GC Roots 相关联并
导致垃圾收集器无法自动回收它们的。掌握了泄漏对象的类型信息，以及 GC Roots 引用链的信息，就可以比较准确地定位出泄漏代码的位置。

3. 如果不存在内存泄漏，换句话说就是内存中的对象确实都还必须存活着，那就应当检查虚拟机的堆参数（-Xmx与-Xms），与机器物理内存对比
看是否还可以调大，从代码上检查是否存在某些对象生命周期过长、持有状态时间过长的情况，尝试减少程序运行期的内存消耗。


### OOM分类
- 堆溢出
- 栈溢出
- 方法区和运行时常量池溢出
- 直接内存溢出

#### 堆溢出
> java.lang.OutOfMemoryError:Javaheap space

1. 原因：
    - 创建一个超大对象，例如数组
    - 超出预期的访问量/数据量
    - 内存泄露，大量引用没有释放
    - 过度使用finailize导致对象没有被回收
    
2. 解决方案
    - 修改-Xmx参数调高JVM堆内存空间
    - 检查超大对象合理性，如数据库查询全部数据
    - 流量问题可以进行限流或者添加资源
    - 内存泄露问题，修改代码及时释放对象引用

#### 栈溢出
>  java.lang.StackOverflowError
1. 原因
    - 方法调用太深导致创建的栈帧太多，如递归调用或死循环
    - 栈帧的大小过大，导致栈空间耗尽。如方法的局部变量很多
2. 解决方案
    - 调整栈内存大小，-Xss设置一个线程的栈空间大小
    - 检查调用链
    - 检查过大栈帧方法
    
#### 方法区和溢出
> java.lang.OutOfMemoryError: Metaspace
> 通过生成代理类产生大量类填满方法区

1. 原因
    - 加载class数据太多或体积太大

2. 解决方案
    - XX:MaxMetaspaceSize，调整方法区大小
    - 可能是动态创建了大量class，但是JVM默认不会卸载class，可以设置参数允许卸载class
    

#### 运行时常量池溢出
> 字符串常量池在堆内存中，所以溢出是java.lang.OutOfMemoryError: Java heap space
> 通过String::intern()将字符串存入到字符串常量池

#### 直接内存溢出
> java.lang.OutOfMemoryError: Direct buffer memory
> ByteBuffer.allocateDirect();