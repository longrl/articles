## ThreadLocal深度解析

### 设计思想

产生线程不安全的原因，对共享变量的操作会引发线程安全问题，在java中什么是共享变量，比较典型的就是指类的成员变量，多个线程操作单实例变量；

分析java内存布局，虚拟机栈的栈帧中保存的局部变量是线程独有的，不会产生线程安全问题，ThreadLocal的设计思路就是通过线程本地化的方式来避免；
通过一个代理类创建专属于每个线程内存区域，代理类中不保存任何状态相关的东西，通过Thread类持有防止内存泄漏；

### 源码分析
分析ThreadLocal中get()方法，看看是怎么么做到本地存储的，可以看到的是通过getMap方法传入当前线程获取的是t.threadLocals，即获取当前类的threadLocals成员变量；可以发现Thread中的threadLocals是一个ThreadLocalMap；ThreadLocalMap是ThreadLocal中的一个静态内部类，具体可以看一下下面的代码，了解了解它的结构；存储结构其实是一个Entry数组，看源码可以直到Entry继承了WeakReference<ThreadLocal<?>>
可以看到Entry的构造函数super(k)，通过createMap(t, value)函数，t.threadLocals = new ThreadLocalMap(this, firstValue)，可以知道就是threadlocal,就是说Entry与threadlocal之间属于弱引用；
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
//ThreadLocalMap存放元素
private void set(ThreadLocal<?> key, Object value) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);
    for (Entry e = tab[i];
        e != null;
        e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();

        if (k == key) {
            e.value = value;
                return;
        }

        if (k == null) {
            replaceStaleEntry(key, value, i);
                return;
            }
    }
    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}

```
通过上面的源码分析我们可以知道，每个线程都拥有一个ThreadLocalMap来存储，存储的实现是Entry数组散列的方式是int i = key.threadLocalHashCode & (len-1)；可以总结一下：声明一个ThreadLocal对象，其实这个对象相当于一个代理类，实现一些方法实现线程本地化，通过ThreadLocalMap来存储，另外可以发现大部分方法都是private修饰，每个Thread方法持有一个Map；还有一个比较细节的点就是它散列进数组的时候用的是threadLocalHashCode;
```java
private final int threadLocalHashCode = nextHashCode();

private static AtomicInteger nextHashCode = new AtomicInteger();

private static final int HASH_INCREMENT = 0x61c88647;

private static int nextHashCode() {
    return nextHashCode.getAndAdd(HASH_INCREMENT);
}
```
通过看源码可以发现，有一个HASH_INCREMENT，通过AtomicInteger保证原子性的增加，这个值很特殊，它是斐波那契数 也叫 黄金分割数。hash增量为 这个数字，带来的好处就是 hash 分布非常均匀。


### 内存泄漏解决方法

### 总结
