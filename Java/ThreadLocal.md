# ThreadLocal

#### 使用
```
ThreadLocal threadLocal = new ThreadLocal(); 
threadLocal.set("A");
threadLocal.get();
```

#### 原理

> Thread类里有一个ThreadLocal变量， 该变量引用了一个Map，ThreadLocalMap，就是用于存放数据的map。
> 该map使用的一个Entry数组保存数据，Entry的key-value具体类型是weakReference<threadlocal>和object。
> 即实际的key是指向ThreadLocal类型变量的弱引用

![avatar](https://img-blog.csdn.net/20180523190740878?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3B1cHB5bHBn/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
```
ThreadLocal.set

public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}

ThreadLocal.getMap

ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}


Thread类的实例变量
/* ThreadLocal values pertaining to this thread. This map is maintained
 * by the ThreadLocal class. */
ThreadLocal.ThreadLocalMap threadLocals = null;

ThreadLocalMap 底层为Entry数组
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}

public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```

#### 问题
内存泄露

ThreadLocal有两个引用，一个是虚拟机栈中实例变量的引用，一个是ThreadLocalMap中Entry的弱引用。
当实例变量的引用被回收后，ThreadLocalMap中Entry的弱引用也会在下一次gc时被回收。而ThreadLocalMap
中Entry中的key被回收了，对用的value并没有被回收，value还被ThreadLocalMap保持强引用，但不可达。
所以只要当前线程未死亡，其ThreadLocalMap引用的value对象就不会被回收导致内存泄露。