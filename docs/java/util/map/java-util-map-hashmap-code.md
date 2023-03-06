## HashMap源码解析
### HashMap关键变量及方法
```java
public class HashMap<K,V> extends AbstractMap<K,V> implements Map<K,V>, Cloneable, Serializable {
    /**
     * 默认的初始化容量大小16 - 必须是2的n次幂
     */
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
    
    /**
     * 限定的最大容量
     */
    static final int MAXIMUM_CAPACITY = 1 << 30;
    
    /**
     * 扩容个时常说到的负载因子的默认值
     */
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
    
    /**
     * 链表转化为红黑树的元素阈值，这个值必须大于2，实际应该至少大于8，在这里有人喜欢问为什么选择8，这个放到后面说
     */
    static final int TREEIFY_THRESHOLD = 8;

    /**
     * 红黑树退回到链表的元素阈值，这个值需要比TREEIFY_THRESHOLD小但是至少为6
     */
    static final int UNTREEIFY_THRESHOLD = 6;

    /**
     * 链表转化为红黑树不仅考量当前链表里有多少元素，同样的还需要考量整个map的容量，这里指定最低树化容量为64，这个放到后面扩容的时候细说
     */
    static final int MIN_TREEIFY_CAPACITY = 64;

    /**
     * node数组，node里记录了下个node，标准的hash拉链法结构
     */
    transient Node<K,V>[] table;

    /**
     * node内记录了hash，key，value以及下一个node的引用
     */
    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;

        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }

        public final K getKey()        { return key; }
        public final V getValue()      { return value; }
        public final String toString() { return key + "=" + value; }

        public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }

        public final V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }

        public final boolean equals(Object o) {
            if (o == this)
                return true;
            if (o instanceof Map.Entry) {
                Map.Entry<?,?> e = (Map.Entry<?,?>)o;
                if (Objects.equals(key, e.getKey()) &&
                        Objects.equals(value, e.getValue()))
                    return true;
            }
            return false;
        }
    }

    /**
     * 整个map中的key value对set，遍历时可以使用
     */
    transient Set<Map.Entry<K,V>> entrySet;

    /**
     * map中的kv对数量
     */
    transient int size;

    /**
     * 结构改变次数，和ArrayList里一样
     */
    transient int modCount;

    /**
     * 触发resize的阈值，等于 容量 乘以 负载因子
     */
    int threshold;

    /**
     * 负载因子，扩容条件限制参数之一
     */
    final float loadFactor;
}
```


### HashMap构造方法
HashMap一共四个常用的构造方法，通过上面的了解，我们知道创建map时只是初始化一些值，实际底层node数组的初始化会在首次使用的时候进行，因此除了以Map为参数的构造方法外，其它的构造方法只是改变一下负载因子等值。
```java
public class HashMap<K,V> extends AbstractMap<K,V> implements Map<K,V>, Cloneable, Serializable {

    /**
     * 指定负载因子及初始容量的构造方法
     *
     * @param  initialCapacity 初始容量
     * @param  loadFactor      负载因子
     * @throws IllegalArgumentException 初始容量为负数或者负载因子非正数或不是一个数值时抛出异常
     */
    public HashMap(int initialCapacity, float loadFactor) {
        //初始容量不得为负数
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                    initialCapacity);
        //不能超出最大容量值
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        //Float.isNaN用于判断是否是一个数值，也就是判断传递进来的NaN，比如传入0.0f/0.0f这种，Float.isNaN返回true
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                    loadFactor);
        this.loadFactor = loadFactor;
        //tableSizeFor方法也是重点之一，保证threshold是2的n次幂
        this.threshold = tableSizeFor(initialCapacity);
    }

    /**
     * 指定了默认负载因子
     */
    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }

    /**
     * 都走默认值
     */
    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }

    /**
     * 负载因子同样使用过默认，putMapEntries方法为核心
     */
    public HashMap(Map<? extends K, ? extends V> m) {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        putMapEntries(m, false);
    }

    final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
        //首先拿到放入map的size
        int s = m.size();
        if (s > 0) {
            //size大于0的时候，如果本身table为null也就是首次使用，还得先进行初始化操作
            if (table == null) { // pre-size
                //通过传入map的size来获取threshold
                float ft = ((float)s / loadFactor) + 1.0F;
                int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                        (int)ft : MAXIMUM_CAPACITY);
                if (t > threshold)
                    threshold = tableSizeFor(t);
            }
            else if (s > threshold)
                //传入map的size大于threshold需要扩容，resize具体放在后面说
                resize();
            //遍历放置
            for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
                K key = e.getKey();
                V value = e.getValue();
                //put调用的也是putVal，放到后面说
                putVal(hash(key), key, value, false, evict);
            }
        }
    }
}
```

### HashMap的Put方法
和ArrayList的Add一样，Put的时候会涉及到扩容相关的操作（包含树化）。
```java
public class HashMap<K,V> extends AbstractMap<K,V> implements Map<K,V>, Cloneable, Serializable {

    /**
     * 划走
     */
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }

    /**
     * put的具体实现
     *
     * @param hash key的hash值
     * @param key the key
     * @param value the value to put
     * @param onlyIfAbsent 为true时，如果遇到相同key那么value不覆盖，除非原来key对应的value为null
     * @param evict 这参数和HashMap没啥关系，主要是给LinkedHashMap使用的
     * @return previous value, or null if none
     */
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        //tab-当前的table
        //p是当前key的hash对应存在的table具体元素
        //n为table数组的长度
        //i为当前key要放入的table索引位置
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        //局部变量tab指向当前的table并且进行了非空判断，不为空顺手赋值了n（长度不为0的时候虽然没有走到if内，但是n的赋值也完成了），这种写法在源码中很常见，平时如果大家的写法和这个不一致需要注意，平时自己写法是这样的看起来问题不大
        if ((tab = table) == null || (n = tab.length) == 0)
            //先完成初始化操作
            n = (tab = resize()).length;
        //拿到key对应索引位置的Node并赋值p
        if ((p = tab[i = (n - 1) & hash]) == null)
            //为空直接新建node并且赋值
            tab[i] = newNode(hash, key, value, null);
        else {
            //当前位置已有值也就是hash冲突了
            //e是要放入的node，可能是新建的也可能是旧的node
            //k是e对应的key值
            Node<K,V> e; K k;
            if (p.hash == hash &&
                    ((k = p.key) == key || (key != null && key.equals(k))))
                //直接在table数组元素首个（链表头节点或者树根节点）就是要修改的node
                e = p;
            else if (p instanceof TreeNode)
                //如果已经转化为了红黑树，就需要调用TreeNode的putTreeVal，内部还涉及树平衡等，放到后面说
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                //到这里就是链表了，并且链表头还不符合需求，对于链表，只能是逐个遍历
                for (int binCount = 0; ; ++binCount) {
                    //如果遍历到为空了还没有找到符合要求的，那么证明这个key确实之前没存过，新建个node添加到链表尾部
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        //证明链表长度超过树化阈值，需要进行树化操作
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            //树化操作，内部还有判断不一定真树化，放到后面说
                            treeifyBin(tab, hash);
                        break;
                    }
                    //找到了key之前存的node
                    if (e.hash == hash &&
                            ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            //修改值
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                //onlyIfAbsent配置为false或者旧值为null就可以修改值
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                //这个也不用管，还是给LinkedHashMap用的
                afterNodeAccess(e);
                return oldValue;
            }
        }
        //结构改变modCount+1
        ++modCount;
        //扩容操作
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }

    /**
     * 树版本的放值
     */
    final TreeNode<K,V> putTreeVal(HashMap<K,V> map, Node<K,V>[] tab,
                                   int h, K k, V v) {
        Class<?> kc = null;
        boolean searched = false;
        TreeNode<K,V> root = (parent != null) ? root() : this;
        for (TreeNode<K,V> p = root;;) {
            int dir, ph; K pk;
            if ((ph = p.hash) > h)
                dir = -1;
            else if (ph < h)
                dir = 1;
            else if ((pk = p.key) == k || (k != null && k.equals(pk)))
                return p;
            else if ((kc == null &&
                    (kc = comparableClassFor(k)) == null) ||
                    (dir = compareComparables(kc, k, pk)) == 0) {
                if (!searched) {
                    TreeNode<K,V> q, ch;
                    searched = true;
                    if (((ch = p.left) != null &&
                            (q = ch.find(h, k, kc)) != null) ||
                            ((ch = p.right) != null &&
                                    (q = ch.find(h, k, kc)) != null))
                        return q;
                }
                dir = tieBreakOrder(k, pk);
            }

            TreeNode<K,V> xp = p;
            if ((p = (dir <= 0) ? p.left : p.right) == null) {
                Node<K,V> xpn = xp.next;
                TreeNode<K,V> x = map.newTreeNode(h, k, v, xpn);
                if (dir <= 0)
                    xp.left = x;
                else
                    xp.right = x;
                xp.next = x;
                x.parent = x.prev = xp;
                if (xpn != null)
                    ((TreeNode<K,V>)xpn).prev = x;
                moveRootToFront(tab, balanceInsertion(root, x));
                return null;
            }
        }
    }
    
    
}
```


### HashMap的树化方法
树化方法treeifyBin也是比较重要的方法，内部含有自身的判断，并非调用了treeifyBin方法就一定会树化。
```java
public class HashMap<K,V> extends AbstractMap<K,V> implements Map<K,V>, Cloneable, Serializable {
    final void treeifyBin(Node<K,V>[] tab, int hash) {
        int n, index; Node<K,V> e;
        //tab为空或者tab当前的length小于MIN_TREEIFY_CAPACITY（64） 首先回去触发扩容
        if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
            resize();
        else if ((e = tab[index = (n - 1) & hash]) != null) {
            //将hash值对应位置的链表换成树
            TreeNode<K,V> hd = null, tl = null;
            do {
                //这里就是一个链表节点变成了树节点
                TreeNode<K,V> p = replacementTreeNode(e, null);
                if (tl == null)
                    hd = p;
                else {
                    //TreeNode继承自LinkedHashMap.Entry，LinkedHashMap.Entry又继承自HashMap.Node，所以也会有prev和next
                    p.prev = tl;
                    tl.next = p;
                }
                tl = p;
            } while ((e = e.next) != null);
            if ((tab[index] = hd) != null)
                //上面我们知道，hd这个树还不算是个树，此时还是链表形式组织，所以在调用treeify之后才算是个正经树
                hd.treeify(tab);
        }
    }

    /**
     * Forms tree of the nodes linked from this node.
     * @return root of tree
     */
    final void treeify(Node<K,V>[] tab) {
        TreeNode<K,V> root = null;
        //一个循环遍历所有的节点
        for (TreeNode<K,V> x = this, next; x != null; x = next) {
            next = (TreeNode<K,V>)x.next;
            x.left = x.right = null;
            //先搞定根节点
            if (root == null) {
                x.parent = null;
                x.red = false;
                root = x;
            }
            else {
                
                K k = x.key;
                int h = x.hash;
                Class<?> kc = null;
                for (TreeNode<K,V> p = root;;) {
                    int dir, ph;
                    K pk = p.key;
                    if ((ph = p.hash) > h)
                        dir = -1;
                    else if (ph < h)
                        dir = 1;
                    else if ((kc == null &&
                            (kc = comparableClassFor(k)) == null) ||
                            (dir = compareComparables(kc, k, pk)) == 0)
                        dir = tieBreakOrder(k, pk);

                    TreeNode<K,V> xp = p;
                    if ((p = (dir <= 0) ? p.left : p.right) == null) {
                        x.parent = xp;
                        if (dir <= 0)
                            xp.left = x;
                        else
                            xp.right = x;
                        root = balanceInsertion(root, x);
                        break;
                    }
                }
            }
        }
        moveRootToFront(tab, root);
    }
}
```