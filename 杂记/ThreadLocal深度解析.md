## ThreadLocal深度解析

### 设计思想

产生线程不安全的原因，对共享变量的操作会引发线程安全问题，在java中什么是共享变量，比较典型的就是指类的成员变量，多个线程操作单实例变量；

分析java内存布局，虚拟机栈的栈帧中保存的局部变量是线程独有的，不会产生线程安全问题，ThreadLocal的设计思路就是通过线程本地化的方式来避免；
通过一个代理类创建专属于每个线程内存区域，代理类中不保存任何状态相关的东西，通过Thread类持有防止内存泄漏；

### 源码分析
分析ThreadLocal中get()方法，看看是怎么么做到本地存储的，可以看到的是通过getMap方法传入当前线程获取的是t.threadLocals，即获取当前类的threadLocals成员变量；可以发现Thread中的threadLocals是一个ThreadLocalMap；ThreadLocalMap是ThreadLocal中的一个静态内部类，具体可以看一下下面的代码，了解了解它的结构；
```java
public T get() {
    Thread t = Thread.currentThread();
    //获取map
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
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}

static class ThreadLocalMap {
    static class Entry extends WeakReference<ThreadLocal<?>> {
        Object value;
        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }
    private Entry[] table;

    private Entry getEntry(ThreadLocal<?> key) {
        int i = key.threadLocalHashCode & (table.length - 1);
        Entry e = table[i];
        if (e != null && e.get() == key)
            return e;
        else
            return getEntryAfterMiss(key, i, e);
    }
}

```


### 内存泄漏解决方法

### 总结
