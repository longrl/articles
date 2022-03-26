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
通过上面的源码分析我们可以知道，每个线程都拥有一个ThreadLocalMap来存储，存储的实现是Entry数组散列的方式是int i = key.threadLocalHashCode & (len-1)；可以总结一下：声明一个ThreadLocal对象，其实这个对象相当于一个代理类，通过一些方法实现线程本地化存储，通过ThreadLocalMap来存储，另外可以发现大部分方法都是private修饰，每个Thread方法持有一个Map；还有一个比较细节的点就是它散列进数组的时候用的是threadLocalHashCode;
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

为何会内存泄漏，我们可以看到每个Thread持有一个ThreadLocalMap属于强引用，只要Thread被回收ThreadLocalMap也会紧接着被回收，不会存在内存泄漏问题；但是有一个特殊的使用场景是线程池的使用，为了提高效率Thead可能会一直活着，所以引用关系一直在，通过上面的源代码可以发现threadlocal与entry之间是弱引用，相当于threadlocal与ThreadLocalMap之间是弱引用，但是中间的值value确实强引用，一直存在就会可能产生内存溢出，其实threadlocal也做了处理，只要在使用的时候注意规范，具体可以看一下代码

```java
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
        return e;
    else
        return getEntryAfterMiss(key, i, e);
}

private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;

    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    size--;

    Entry e;
    int i;
    for (i = nextIndex(staleSlot, len);
        (e = tab[i]) != null;
        i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        if (k == null) {
            e.value = null;
            tab[i] = null;
            size--;
        } else {
            int h = k.threadLocalHashCode & (len - 1);
            if (h != i) {
                tab[i] = null;
                while (tab[h] != null)
                    h = nextIndex(h, len);
                tab[h] = e;
            }
        }
    }
    return i;
}
```
当发现 key == null 时 会调用expungeStaleEntry，也就是说当key在gc的时候被回收了之后，会调用expungeStaleEntry将value设为空，这样就能解决内存泄漏，但是只有在调用ThreadLocal的get、set、remove方法的时候才会触发expungeStaleEntry方法的执行，才会把被自动垃圾回收的ThreadLocal为null所对应的value和Entry才会设置为null。换句话说，正常的情况是不会出现内存泄露的，但是如果我们没有调用ThreadLocal对应的set、get、remove方法就不会把对应的value和Entry设置为null，这样就可能会出现内存泄露情况。对吧，那如何避免内存泄露的情况呢？那就是我们在不使用的时候就调用一下ThreadLoca的remove方法，来加快垃圾回收，避免内存泄露。

### 总结

思考：当我刚给ThreadLocal变量赋值，刚好经历一gc，那岂不是刚好被回收掉吗，导致取值取不到？

> 其实只是threadlocal与entry之间有一层弱引用，但是执行的时候线程里还有一个关于threadlocal的强引用