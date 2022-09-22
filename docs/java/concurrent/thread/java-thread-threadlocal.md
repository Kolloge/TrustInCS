### 什么是ThreadLocal？

ThreadLocal是线程变量共享的解决方案，其会给每个线程分配一个属于该线程的变量，各个线程拥有自己的变量副本，互相之间不影响，使得变量在线程之间隔离。

- 如对于Spring实现事务管理就是使用了ThreadLocal，在开启事务的时候单独获取到一个数据库连接放入到ThreadLocal中，后续整个被事务注解包裹住的方法的事务都会走该连接，同时与别的线程的事务进行隔离。同样的由于是使用ThradLocal的方式，所以自己新起线程执行的异步操作不会被包裹在当前的事务中，会导致事务失效。
- 对于SimpleDateFormat而言，大家都知道这不是线程安全的，因为底层对于calendar的add和clear操作未加锁，所以多线程情况下，会存在一个线程add的内容被另一个线程clear掉。一般的做法都是创建局部变量，不过这样的开销较大，方法每被调用一次，就会创建一个SimpleDateFormat对象。另一种解决方案就是使用ThreadLocal，为每一个线程初始化一个SimpleDateFormat。

### 如何使用ThreadLocal?

对于实际业务中如何使用ThreadLocal，简单的示例一下即可
```java
public class ThreadLocalUtil {

    private final static ThreadLocal<String> SERVICE_PARAM_RESOURCE = new ThreadLocal<>();


    public static void putMyStr(String str){
        SERVICE_PARAM_RESOURCE.set(str);
    }

    public static void getMyStr(){
        SERVICE_PARAM_RESOURCE.get();
    }

    public static void clean(){
        SERVICE_PARAM_RESOURCE.remove();
    }

}
```

### ThreadLocal原理分析

#### ThreadLocalMap类

```java
public class ThreadLocal<T> {
    //ThreadLocalMap是静态内部类
    static class ThreadLocalMap {
        /**
         * The entries in this hash map extend WeakReference, using
         * its main ref field as the key (which is always a
         * ThreadLocal object).  Note that null keys (i.e. entry.get()
         * == null) mean that the key is no longer referenced, so the
         * entry can be expunged from table.  Such entries are referred to
         * as "stale entries" in the code that follows.
         */
        //贴一下翻译：此哈希映射中的条目扩展了WeakReference，使用其主ref字段作为键（它始终是ThreadLocal对象）。注意，空键（即entry.get（）==null）意味着该键不再被引用，因此可以从表中删除该项。在下面的代码中，这些条目被称为“过时条目”。
        //其实这里主要是需要了解在内部的entry继承了弱引用，使得对key的引用为弱引用  
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;
        
            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }

        /**
         * 默认初始化容量的大小
         */
        private static final int INITIAL_CAPACITY = 16;

        /**
         * 可调整table大小，大小必须是2的n次幂
         */
        private Entry[] table;

        /**
         * table中的条目数
         */
        private int size = 0;

        /**
         * 触发resize的数量大小
         */
        private int threshold; // Default to 0
    }
}
```

#### ThreadLocal的set方法

```java
public class ThreadLocal<T> {
    public void set(T value) {
        //native方法，获取到当前的线程
        Thread t = Thread.currentThread();
        //利用getMap方法获取到当前线程的ThreadLocalMap
        ThreadLocalMap map = getMap(t);
        //常见的存在则放，不存在则创建好map之后创建
        if (map != null)
            //key是当前ThreadLocal
            map.set(this, value);
        else
            createMap(t, value);
    }

    //返回线程的ThreadLocalMap
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
}
```

```java
public class Thread implements Runnable {
    //而在Thread这个类里，成员变量threadLocals的定义如下
    /* 与此线程相关的ThreadLocal值。 此map通过ThreadLocal类维护 */
    ThreadLocal.ThreadLocalMap threadLocals = null;
}
```

- 不存在时进行创建操作

```java
public class ThreadLocal<T> {
    //当不存在Map的时候创建map方法调用ThreadLocalMap支持第一次放置值的构造器方法
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
}
```

```java
public class ThreadLocalMap {
    //带第一次值
    ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
        //和HashMap一样，初始化的大小都是16
        table = new Entry[INITIAL_CAPACITY];
        //思想和HashMap一样，对于2的n次幂而言，&操作等同于取余
        int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
        //根据计算好的位置放入key和value
        table[i] = new Entry(firstKey, firstValue);
        size = 1;
        //设置负载数，也就是触发resize的数量
        setThreshold(INITIAL_CAPACITY);
    }

    private void setThreshold(int len) {
        //HashMap默认使用的是0.75，而这里采用的是2/3
        threshold = len * 2 / 3;
    }
}
```

- 非创建，二次赋值的操作

```java
public class ThreadLocalMap {
    //首次存放值的时候直接新建map没问题，但是二次放值的时候可能会遇到hash冲突，所以二次放值的方式需要进一步分析
    //非首次set时调用
    private void set(ThreadLocal<?> key, Object value) {

        // We don't use a fast path as with get() because it is at
        // least as common to use set() to create new entries as
        // it is to replace existing ones, in which case, a fast
        // path would fail more often than not.
        // 以上是原生注释，翻译一下：我们没有像get（）那样使用快速路径，因为使用set（）创建新条目和替换现有条目至少是一样常见的，在这种情况下，快速路径失败的频率更高
        Entry[] tab = table;
        int len = tab.length;
        //拿到table之后同样根据table的长度结合当前放入的ThreadLocal的hashCode来计算位置
        int i = key.threadLocalHashCode & (len - 1);
        //已经拿到了key在table中的位置为什么，为什么还要进行循环遍历呢？
        //这里涉及到的知识点就是解决hash冲突的方法，常见的HashMap使用的是拉链法也就是链地址法，当存在冲突时拉起一个链表进行存储
        //而ThreadLocalMap使用的是开发地址法，冲突后如果不是当前key，那么挨个往后寻找空位，直到找到空位在放值
        //nextIndex处理了开发地址法中处理首尾连接寻找的方式，贴在后边，会从头到尾循环找位置
        //可能会有小伙伴问，一直循环下去，假如每个位置都有值，那岂不是死循环了？其实不然，往下看
        for (Entry e = tab[i]; e != null; e = tab[i = nextIndex(i, len)]) {
            ThreadLocal<?> k = e.get();
            //如果直接是当前的key，直接更新
            if (k == key) {
                e.value = value;
                return;
            }
            //如果空，放值，同时处理一个问题处理一下过期数据的问题
            if (k == null) {
                //对应处理方法见下方
                replaceStaleEntry(key, value, i);
                return;
            }
        }
        //从上面走下来也就是tab[i]还等于null也就是没有放过值，那么就新建一个entry
        tab[i] = new Entry(key, value);
        int sz = ++size;
        //也就是超过负载进行扩容的机制避免死循环，因为不到满的时候就扩容了。
        //前面这个cleanSomeSlots是进行清除过期数据，如果清除了过期数据返回TURE，这样其实虽然现在set了值，但是我又清理了一个铁定不用触发扩容，若是没清那么再判断是否需要扩容。
        if (!cleanSomeSlots(i, sz) && sz >= threshold)
            //扩容
            rehash();
    }
}
```

- 在ThreadLocalMap中前后遍历的两个方法

```java
public class ThreadLocalMap {
    //承接遍历时获取下一个位置值的方法
    private static int nextIndex(int i, int len) {
        //超过长度时返回0，再次从0开始找
        return ((i + 1 < len) ? i + 1 : 0);
    }

    //减一操作，减到头之后从len开始减
    private static int prevIndex(int i, int len) {
        return ((i - 1 >= 0) ? i - 1 : len - 1);
    }
}
```

- 处理key为null情况过期数据的方法: replaceStaleEntry

```java
public class ThreadLocalMap {    
    /**
     * 二次放值的时候如果entry不为空，但是entry的key为空的情况，其实也就是被回收的情况
     * @param key 这次操作的threadLocal
     * @param value 需要放置的value
     * @param staleSlot 需要放置在table中的位置
     */
    private void replaceStaleEntry(ThreadLocal<?> key, Object value, int staleSlot) {
        Entry[] tab = table;
        int len = tab.length;
        Entry e;

        // Back up to check for prior stale entry in current run.
        // We clean out whole runs at a time to avoid continual
        // incremental rehashing due to garbage collector freeing
        // up refs in bunches (i.e., whenever the collector runs).
        //翻译一下：备份以检查当前运行中以前的过时条目。我们一次清除整个运行，以避免由于垃圾收集器成串地释放ref（即，每当收集器运行时）而导致的持续增量刷新。
        int slotToExpunge = staleSlot;
        //就是往前找的方式循环，其实就找找有没有也被回收掉key的entry
        for (int i = prevIndex(staleSlot, len); (e = tab[i]) != null; i = prevIndex(i, len))
            if (e.get() == null)
                slotToExpunge = i;

        // 因为当前位置的entry的key是空的，但是由于可能要放入的ThreadLocal引用并没有被回收，所以还需要往后找，看看能不能找到
        for (int i = nextIndex(staleSlot, len); (e = tab[i]) != null; i = nextIndex(i, len)) {
            ThreadLocal<?> k = e.get();

            // 发现原本计算好的位置可能因为hash冲突被别的key占用，但是占用的key被回收了，往后找的时候找到了当前key对应的entry
            if (k == key) {
                //直接更新值
                e.value = value;
                //这里其实就是把当前的entry移动到一开始算出来的位置，把key已失效的entry放到当前key之前所处在的位置，可以让下次使用该key的时候直接找到
                tab[i] = tab[staleSlot];
                tab[staleSlot] = e;

                // 到这里if被触发也就是往前找key为null的entry没找到，然后因为tab[i]和tab[staleSlot]调换了一下，所以目前key为null的entry被转到i这个位置去了，所以重新赋值一下
                if (slotToExpunge == staleSlot)
                    slotToExpunge = i;
                //做一做清理，对于slotToExpunge位的entry的清理其实是在expungeStaleEntry方法做的，逻辑放在下面讲
                cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
                return;
            }

            // 找一遍发现备份的slotToExpunge和传递进来的staleSlot相等，但是往后找的时候发现了新的entry的key为空，那么就更新slotToExpunge的值
            // 我们前面知道一开始是往前找key引用被回收的entry，然后给slotToExpunge赋值，这个判断成功也就是往前找没找到，往后找的时候找到了然后更新一下值。
            if (k == null && slotToExpunge == staleSlot)
                slotToExpunge = i;
        }

        //走到这里的就是key没找到，传入位置的key反正也被回收，直接新建一个entry放置在传入要替换的位置位置
        tab[staleSlot].value = null;
        tab[staleSlot] = new Entry(key, value);

        // slotToExpunge到这里，其实需要看情况，如果第一次往前找的时候找到到了，那就是第一次找到的值，如果第一次没找到，往后遍历的时候找到了那就是后边的值，相等的情况在上边已经被清理了就不用再处理了
        // 所以如果slotToExpunge和一开始传入的staleSlot不一致，那么就是找到了别的key被回收的entry
        if (slotToExpunge != staleSlot)
            //逻辑放在下面讲
            cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
    }
    

    //首先看看清理引用清除的entry的时候对入参slotToExpunge做了什么处理
    //这里的形参staleSlot首先我们知道，对于上面的操作来说，是一个key为null的entry在table中的位置
    private int expungeStaleEntry(int staleSlot) {
        Entry[] tab = table;
        int len = tab.length;

        // 直接将当前entry的value也清空了，用来避免key引用为空的情况下还保持着对value的引用而导致的内存泄漏
        tab[staleSlot].value = null;
        //直接把entry的引用去除
        tab[staleSlot] = null;
        size--;

        // Rehash until we encounter null
        Entry e;
        int i;
        //往后边再找找看看有没有别的key引用为null的过期entry
        for (i = nextIndex(staleSlot, len); (e = tab[i]) != null; i = nextIndex(i, len)) {
            ThreadLocal<?> k = e.get();
            if (k == null) {
                //找到了，直接去除引用
                e.value = null;
                tab[i] = null;
                size--;
            } else {
                //走到这里的表明entry还是正常的，重新计算一下key的hash
                int h = k.threadLocalHashCode & (len - 1);
                //如果计算出来的的值不等于当前的i值，其实也就是之前可能hash冲突了，然后被移动到这里的
                if (h != i) {
                    //直接把当前指向tab[i]的entry引用去掉
                    tab[i] = null;

                    // Unlike Knuth 6.4 Algorithm R, we must scan until
                    // null because multiple entries could have been stale.
                    //贴一下原注释的翻译：与Knuth 6.4算法R不同，我们必须扫描到空，因为多个条目可能已经过时。
                    //找到一个为null的空位置
                    while (tab[h] != null)
                        h = nextIndex(h, len);
                    //将当前的entry引到新的位置上
                    tab[h] = e;
                }
            }
        }
        //上面循环内部没有退出写法，所以只有for循环里的(e = tab[i]) != null才会使得循环结束，那么走到这里的时候就是取得的entry为null的情况
        return i;
    }

    //传入的i值对应的entry都是没问题的，从i往后找
    private boolean cleanSomeSlots(int i, int n) {
        boolean removed = false;
        Entry[] tab = table;
        int len = tab.length;
        //当传入的n右移一位之后还不到0时执行
        do {
            //找到下一个i值
            i = nextIndex(i, len);
            Entry e = tab[i];
            //如果获取的entry不为空，但是对应的key引用已经没了
            if (e != null && e.get() == null) {
                n = len;
                //是否过期了的被删除
                removed = true;
                //参考上面的方法
                i = expungeStaleEntry(i);
            }
        } while ( (n >>>= 1) != 0);
        return removed;
    }
}
```

#### ThreadLocal的get方法

```java
public class ThreadLocal<T> {
    public T get() {
        //一样的拿取到当前的线程，并获取到map
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            //找到当前threadLocal的entry，这里也做了一些处理，分析在下边
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                //返回值
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        //如果map为空，或者entry没有
        return setInitialValue();
    }

    private T setInitialValue() {
        //如果没有自己重写initialValue方法，就是返回个null
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            //这个是解决map在，entry没有的情况
            map.set(this, value);
        else
            //创建map
            createMap(t, value);
        return value;
    }
}
```
- 从ThreadLocalMap里取值时方法
```java
public class ThreadLocalMap {
    
    private Entry getEntry(ThreadLocal<?> key) {
        int i = key.threadLocalHashCode & (table.length - 1);
        Entry e = table[i];
        if (e != null && e.get() == key)
            return e;
        else
            //使用计算的索引位置找到的entry为空，或者不等于当前key时
            //这里主要有本来就没有
            //索引被回收
            //hash冲突被放在了后面
            return getEntryAfterMiss(key, i, e);
    }

    private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
        Entry[] tab = table;
        int len = tab.length;

        while (e != null) {
            ThreadLocal<?> k = e.get();
            //找到了对应的key位置，返回对应的entry即可
            if (k == key)
                return e;
            //这种情况是出现了过期数据，还是那个常见的清除方法
            if (k == null)
                expungeStaleEntry(i);
            else
                //这个就是往后迭代判断是不是因为hash冲突被存到后面去了
                i = nextIndex(i, len);
            e = tab[i];
        }
        //真找不到就返回null
        return null;
    }
}
```

#### ThreadLocal的remove方法

```java
public class ThreadLocal<T> {
    public void remove() {
        ThreadLocalMap m = getMap(Thread.currentThread());
        if (m != null)
            //ThreadLocalMap的remove
        m.remove(this);
    }
}
```

```java
public class ThreadLocalMap {
    private void remove(ThreadLocal<?> key) {
        Entry[] tab = table;
        int len = tab.length;
        int i = key.threadLocalHashCode & (len-1);
        //十分熟悉的操作，所以在remove里其实也会清除掉key为null的过期数据
        for (Entry e = tab[i]; e != null; e = tab[i = nextIndex(i, len)]) {
            if (e.get() == key) {
                e.clear();
                expungeStaleEntry(i);
                return;
            }
        }
    }
}
```


#### ThreadLocal的rehash方法
```java
public class ThreadLocalMap {
    private void rehash() {
        //这玩意会整个扫一遍清除过期数据
        expungeStaleEntries();

        // Use lower threshold for doubling to avoid hysteresis
        if (size >= threshold - threshold / 4)
            //这个没啥好讲的，就是扩容两倍，重算hash放到新的table里去
            resize();
    }

    //就这个
    private void expungeStaleEntries() {
        Entry[] tab = table;
            int len = tab.length;
            //挨个清理
            for (int j = 0; j < len; j++) {
                Entry e = tab[j];
                if (e != null && e.get() == null)
                    expungeStaleEntry(j);
            }
    }
}
```

### ThreadLocal中存在的内存泄露问题
- 首先我们需要明确内存泄漏的定义
  - 内存泄漏是指程序中分配的堆内存由于一些原因导致无法释放，造成系统内存的浪费。
- 一般针对ThreadLocal内存泄漏一股脑都是朝着弱引用进行分析，但是都是经典的COPY言论，看不出来什么东西。所以对于ThreadLocal里内存泄漏我们要分为以下几种情况
  - 针对非线程池情况下
    - 非线程池情况下，线程运行结束销毁时对应持有的ThreadLocalMap引用消失，也就被回收了。
  - 针对线程池情况下
    - 非静态ThreadLocal
      - 被GC回收导致entry的key为null，但是value还存在着，后续如果要分配，这块的entry无法被访问也没法分配，这就浪费了（这也是为什么ThreadLocal里做那么多清理操作的原因，所以这里正常操作其实不会泄漏）。
      - set了之后就不再get，set，remove，这种其实可以挽救
        - 虽然当前ThreadLocal没有再使用对应操作，只要别的ThreadLocal进行了get，set，remove等操作，由于一个线程下操作的都是同一个ThreadLocalMap，那么就有一定几率会被清理掉。否则只能靠触发扩容来整体清理了，这个十分困难。
        - 如果就你一个ThreadLocal，那甭管有没有GC给你回收，这块泄漏了就是泄漏了。
    - 静态的ThreadLocal
      - 一般推荐ThreadLocal使用static修饰，主要是避免TSO重复创建。
      - 出现内存泄漏就是我们使用方法有问题，set了之后就不再get，set，remove，而且这样的情况还不能通过别的ThreadLocal触发操作来进行清理，导致对应的entry一直保持在ThreadLocalMap，但是却用不上。
- 所以我们知道其实代码中已经对内存泄漏问题已经做了一系列的处理，正经使用的话不会存在问题的。
  - 牢记用完一定要remove，而且你不remove就算你不怕内存泄漏，你就不怕数据污染了？请求进来有条件的set导致如果这次没有进行set，导致我拿的还是上次set进去的内容。
- 可是非要不正经使用呢？
  - good luck