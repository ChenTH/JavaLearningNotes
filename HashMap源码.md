# JDK1.8的HashMap部分源码解析
## 概述

HashMap主要来存放键值对。JDK1.8之前使用数组+链表的形式，JDK1.8之后进行了改变，使用了数组+链表或者红黑树的形式。

## 小概念普及
### 关系运算简介

| | 0 0| 0 1 | 1 1|
|---|---|---|---|
|与 &|0|0|1|
|或 \||0|1|1|
|异或 ^|0|1|0|

非～ ～1=0 ～0=1

## 成员变量

```java
 /**
     * The default initial capacity - MUST be a power of two.
     * 默认的初始容量，必须是2的次幂
     */
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

    /**
     * The maximum capacity, used if a higher value is implicitly specified
     * by either of the constructors with arguments.
     * MUST be a power of two <= 1<<30.
     * 最大容量
     */
    static final int MAXIMUM_CAPACITY = 1 << 30;

    /**
     * The load factor used when none specified in constructor.
     * 默认的负载因子
     */
    static final float DEFAULT_LOAD_FACTOR = 0.75f;

    /**
     * The bin count threshold for using a tree rather than list for a
     * bin.  Bins are converted to trees when adding an element to a
     * bin with at least this many nodes. The value must be greater
     * than 2 and should be at least 8 to mesh with assumptions in
     * tree removal about conversion back to plain bins upon
     * shrinkage.
     * 红黑树阈值，链表元素个数大于等于此值则转化为红黑树
     */
    static final int TREEIFY_THRESHOLD = 8;

    /**
     * The bin count threshold for untreeifying a (split) bin during a
     * resize operation. Should be less than TREEIFY_THRESHOLD, and at
     * most 6 to mesh with shrinkage detection under removal.
     * 普通链表阈值，红黑树元素个数小于等于此值则转化为普通链表
     */
    static final int UNTREEIFY_THRESHOLD = 6;

    /**
     * The smallest table capacity for which bins may be treeified.
     * (Otherwise the table is resized if too many nodes in a bin.)
     * Should be at least 4 * TREEIFY_THRESHOLD to avoid conflicts
     * between resizing and treeification thresholds.
     * 桶中结构转化为红黑树对应的table的最小大小
     */
    static final int MIN_TREEIFY_CAPACITY = 64;
    /**
     * The table, initialized on first use, and resized as
     * necessary. When allocated, length is always a power of two.
     * (We also tolerate length zero in some operations to allow
     * bootstrapping mechanics that are currently not needed.)
     * 存储数据的桶数组，数组大小总是2的次幂。
     */
    transient Node<K,V>[] table;

    /**
     * Holds cached entrySet(). Note that AbstractMap fields are used
     * for keySet() and values().
     * 存放具体元素的set
     */
    transient Set<Map.Entry<K,V>> entrySet;

    /**
     * The number of key-value mappings contained in this map.
     * map中的key-value个数
     */
    transient int size;

    /**
     * The number of times this HashMap has been structurally modified
     * Structural modifications are those that change the number of mappings in
     * the HashMap or otherwise modify its internal structure (e.g.,
     * rehash).  This field is used to make iterators on Collection-views of
     * the HashMap fail-fast.  (See ConcurrentModificationException).
     * HashMap的扩容和修改次数计数器 用于判断fail-fast
     */
    transient int modCount;

    /**
     * The next size value at which to resize (capacity * load factor).
     * 下一次扩容的阈值，元素个数到达此阈值即扩容
     * @serial
     */
    // (The javadoc description is true upon serialization.
    // Additionally, if the table array has not been allocated, this
    // field holds the initial array capacity, or zero signifying
    // DEFAULT_INITIAL_CAPACITY.)
    // 此外，如果table还没有被分配，则此值为初始容量或者0
    int threshold;

    /**
     * The load factor for the hash table.
     * 负载因子
     * @serial
     */
    final float loadFactor;

```

## 构造方法

```java
    /**
     * Constructs an empty <tt>HashMap</tt> with the specified initial
     * capacity and load factor.
     *
     * @param  initialCapacity the initial capacity
     * @param  loadFactor      the load factor
     * @throws IllegalArgumentException if the initial capacity is negative
     *         or the load factor is nonpositive
     */
    public HashMap(int initialCapacity, float loadFactor) {
        //判断初始容量是否小于0，小于0则报错
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        //判断初始容量是否大于最大容量，大于则置为最大容量
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        //判断负载因子是否小于等于0 是否是一个数字
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        //为负载因子赋值                                       
        this.loadFactor = loadFactor;
        //为扩容阈值赋值（tableSizeFor函数用于找到大于initialCapacity的最近的2的次幂）
        this.threshold = tableSizeFor(initialCapacity);
    }

    /**
     * Constructs an empty <tt>HashMap</tt> with the specified initial
     * capacity and the default load factor (0.75).
     *
     * @param  initialCapacity the initial capacity.
     * @throws IllegalArgumentException if the initial capacity is negative.
     */
    public HashMap(int initialCapacity) {
        //参数只有初始容量，使用默认负载因子，并调用另一个构造方法
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }

    /**
     * Constructs an empty <tt>HashMap</tt> with the default initial capacity
     * (16) and the default load factor (0.75).
     */
    public HashMap() {
        //无参数 则只制定默认负载因子
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }

    /**
     * Constructs a new <tt>HashMap</tt> with the same mappings as the
     * specified <tt>Map</tt>.  The <tt>HashMap</tt> is created with
     * default load factor (0.75) and an initial capacity sufficient to
     * hold the mappings in the specified <tt>Map</tt>.
     *
     * @param   m the map whose mappings are to be placed in this map
     * @throws  NullPointerException if the specified map is null
     */
    public HashMap(Map<? extends K, ? extends V> m) {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        putMapEntries(m, false);
    }
    /**
     * Returns a power of two size for the given target capacity.
     * 根据传入的值返回一个2的次幂
     */
    static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
```

可以看出以上的所有的初始化过程都没有对talbe进行初始化。并且在传入initialCapacity的构造函数中对threshold进行了初始化，所以threshold除了记录扩容阈值之外，还在HashMap初始化时记录初始容量或直接置为0。

## node的数据结构
```java
    /**
     * Basic hash bin node, used for most entries.  (See below for
     * TreeNode subclass, and in LinkedHashMap for its Entry subclass.)
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
            //key与value的hashCode进行异或
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
```

## hash计算方法

```java
    /**
     * Computes key.hashCode() and spreads (XORs) higher bits of hash
     * to lower.  Because the table uses power-of-two masking, sets of
     * hashes that vary only in bits above the current mask will
     * always collide. (Among known examples are sets of Float keys
     * holding consecutive whole numbers in small tables.)  So we
     * apply a transform that spreads the impact of higher bits
     * downward. There is a tradeoff between speed, utility, and
     * quality of bit-spreading. Because many common sets of hashes
     * are already reasonably distributed (so don't benefit from
     * spreading), and because we use trees to handle large sets of
     * collisions in bins, we just XOR some shifted bits in the
     * cheapest possible way to reduce systematic lossage, as well as
     * to incorporate impact of the highest bits that would otherwise
     * never be used in index calculations because of table bounds.
     */
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```
## put

### 普通的put方法
    可见普通的put方法仅仅是接收了key value参数并调用了putVal方法
```java
/**
     * Associates the specified value with the specified key in this map.
     * If the map previously contained a mapping for the key, the old
     * value is replaced.
     * 在map中创建key与value的对应关系，如果map中之前已经存在key的对应关系，则之前的对应关系会被替换。
     * @param key key with which the specified value is to be associated
     * @param value value to be associated with the specified key
     * @return the previous value associated with <tt>key</tt>, or
     *         <tt>null</tt> if there was no mapping for <tt>key</tt>.
     *         (A <tt>null</tt> return can also indicate that the map
     *         previously associated <tt>null</tt> with <tt>key</tt>.)
     */
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
```
### putAll方法
    putAll是直接调用了putMapEntries方法
 ```java
    /**
     * Copies all of the mappings from the specified map to this map.
     * These mappings will replace any mappings that this map had for
     * any of the keys currently in the specified map.
     * 从传入的map中复制所有的对应关系到当前map，如果一个key值在传入的map和当前map中皆有对应关系，
       则可能会覆盖当前map中的对应关系会被覆盖。
     * @param m mappings to be stored in this map
     * @throws NullPointerException if the specified map is null
     */
    public void putAll(Map<? extends K, ? extends V> m) {
        putMapEntries(m, true);
    }
```
```java
    /**
     * Implements Map.putAll and Map constructor
     * 把传入的map加入本HashMap，用于Map.putAll或者构造map
     * @param m the map 传入的map
     * @param evict false when initially constructing this map, else
     * true (relayed to method afterNodeInsertion). 如果是初始化构造时
       使用为false，其余时候为true
     */
    final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
        //获取传入map的大小
        int s = m.size();
        //如果传入的map有元素则进入
        if (s > 0) {
            //如果本HashMap尚未初始化
            if (table == null) { // pre-size
                //
                float ft = ((float)s / loadFactor) + 1.0F;
                int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                         (int)ft : MAXIMUM_CAPACITY);
                if (t > threshold)
                    threshold = tableSizeFor(t);
            }
            //如果本HashMap已经初始化但是传入map的大小大于了当前的扩容阈值则调整map的大小
            //注意此处是用传入的map大小与当前map的threshold进行比较
            //理论上说应该用当前map的大小与传入map的大小的和与threshold进行比较
            //在jdk1.7版本中有如下一段注释来解释这个行为
            /*
            * Expand the map if the map if the number of mappings to be added
            * is greater than or equal to threshold.  This is conservative; the
            * obvious condition is (m.size() + size) >= threshold, but this
            * condition could result in a map with twice the appropriate capacity,
            * if the keys to be added overlap with the keys already in this map.
            * By using the conservative calculation, we subject ourself
            * to at most one extra resize.
            */
            /*当待加入的映射关系个数大于threshold时对map进行扩容。这是一个保守的方法，很显然判断条件应该是(m.size()+size)>=threshold，不过这个条件可能会让map的容量比实际需要容量大一倍,因为在传入的map中可能会有和当前map重复的key（重复的key会被覆盖，所以实际容量会比m.size()+size小）.所以使用保守的计算方法，最多进行一次额外的扩容。
            */
            else if (s > threshold)
                //调整map大小
                resize();
            //循环向添加当前map添加原map中数据
            for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
                K key = e.getKey();
                V value = e.getValue();
                putVal(hash(key), key, value, false, evict);
            }
        }
    }
```
### put方法的最终函数putVal

```java
        /**
     * Implements Map.put and related methods
     *
     * @param hash hash for key
     * @param key the key
     * @param value the value to put
     * @param onlyIfAbsent if true, don't change existing value 
                            如果为true则不修改已经存在的值
     * @param evict if false, the table is in creation mode.
                        如果为false则进入创建模式（初始化）
     * @return previous value, or null if none
     */
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        //如果table没有初始化或者tab的长度为0则初始化table
        if ((tab = table) == null || (n = tab.length) == 0)
            //初始化table并取得table的长度
            n = (tab = resize()).length;
        //取得tab中放入的位置的值为p，如果p为null则hash没有冲突 直接放入
        //n为2的次幂 n-1为一个全1项 与hash与可得到一个小于n的比较均匀的分布值
        //例如n为32 hash为40则有如下运算 
        //n  :00000000000000000000000000100000 
        //n-1:00000000000000000000000000011111 
        //40 :00000000000000000000000000101000
        //(n-1)&40:00000000000000000000000000001000 = 8
        //故放入table[8]中
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        //如果放入的位置当前有值则进行链表或红黑树的插入
        else {
            Node<K,V> e; K k;
            //如果key与当前p中key相同则找到了赋值的位置，的把p值直接赋值给e以供最后统一赋值
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            //如果p为一个树节点 则进入树节点处理流程
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            //如果不为树节点 则进入链表处理流程
            else {
                //循环遍历链表
                for (int binCount = 0; ; ++binCount) {
                    //如果遍历到了链表的末尾
                    if ((e = p.next) == null) {
                        //新建节点并插入到表尾
                        p.next = newNode(hash, key, value, null);
                        //如果链表长度等于了树化阈值则进行树化
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    //判断当前节点的key是否与传入的key相同，相同则直接结束循环（找到了赋值的位置）
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            //判断e是否为null 即是否是找到了key相同的历史映射 如果在面直接插入了新映射此处e应为null
            if (e != null) { // existing mapping for key
                //取得旧值
                V oldValue = e.value;
                //如果onlyIfAbsent为false即修改已经存在的值 或者oldValue为null则重新赋值
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                //HashMap中无用
                afterNodeAccess(e);
                return oldValue;
            }
        }
        //修改技术增加
        ++modCount;
        //调整size的值并判断是否大于了扩容阈值 如果大于扩容阈值则进行扩容
        if (++size > threshold)
            resize();
        //HashMap中无用
        afterNodeInsertion(evict);
        return null;
    }
```

## get
```java
 /**
     * Returns the value to which the specified key is mapped,
     * or {@code null} if this map contains no mapping for the key.
     *
     * <p>More formally, if this map contains a mapping from a key
     * {@code k} to a value {@code v} such that {@code (key==null ? k==null :
     * key.equals(k))}, then this method returns {@code v}; otherwise
     * it returns {@code null}.  (There can be at most one such mapping.)
     *
     * <p>A return value of {@code null} does not <i>necessarily</i>
     * indicate that the map contains no mapping for the key; it's also
     * possible that the map explicitly maps the key to {@code null}.
     * The {@link #containsKey containsKey} operation may be used to
     * distinguish these two cases.
     *
     * @see #put(Object, Object)
     */
    public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }

    /**
     * Implements Map.get and related methods
     *
     * @param hash hash for key
     * @param key the key
     * @return the node, or null if none
     */
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        //如果table被初始化并且长度大于0且key中有值则进入判断否则返回null
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            //如果table中的值的key 与传入的key相同则直接返回
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            //如果table中的值key与传入的key不同则进入后续的数据结构进行判断
            if ((e = first.next) != null) {
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }
```
## 扩容
```java
 /**
     * Initializes or doubles table size.  If null, allocates in
     * accord with initial capacity target held in field threshold.
     * Otherwise, because we are using power-of-two expansion, the
     * elements from each bin must either stay at same index, or move
     * with a power of two offset in the new table.
     * 初始化table大小或者对table大小进行翻倍。如果table为null，则按照threshold
       字段中保存的初始容量进行分配。如果table不为null，由于使用的是翻倍增加策略，则对
       table容量进行翻倍。并且之前在table中的元素应呆在原处或者移动到2倍位置处。
     * @return the table
     */
    final Node<K,V>[] resize() {
        //缓存当前table为oldTab
        Node<K,V>[] oldTab = table;
        //获取oldTab的大小，为oldCap
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        //获取当前的扩容阈值
        int oldThr = threshold;
        //初始化新的table大小与阈值为0
        int newCap, newThr = 0;
        //如果oldCap大于0，即当前map中的table存有值
        if (oldCap > 0) {
            //判断oldCap是否大于等于最大容量
            if (oldCap >= MAXIMUM_CAPACITY) {
                //如果oldCap大于等于了最大容量则把扩容阈值赋为int的最大值
                //给一个足够大的值，以后尽量不再触发扩容
                threshold = Integer.MAX_VALUE;
                //由于oldCap已经是最大值，故不再扩容 直接返回原table
                return oldTab;
            }
            //如果oldCap小于最大容量（即不满足上一个判断条件）
            //把oldCap的翻倍值赋给newCap并判断newCap的容量是否小于最大容量（此处newCap可能大于最大容量）
            //如果newCap小于最大容量并且oldCap大于等于初始容量则把阈值翻倍
            //todo 此处有个问题 为什么oldCap小于默认初始容量不行
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                //此处因为容量翻了一倍，阈值是容量*负载因子 所以阈值翻一倍即可（为了减少计算进行的性能优化，不然其实可以放到下面进行统一的阈值计算）
                newThr = oldThr << 1; // double threshold
        }
        //如果oldCap等于0，则证明table未初始化
        //此时如果oldThr大于0 则证明threshold中存储的是table的初始化长度（详情参见构造方法部分，带有initialCapacity的构造方法会把initialCapacity赋给threshold进行缓存）
        else if (oldThr > 0) // initial capacity was placed in threshold
            //把oldThr存的初始长度赋给newCap
            newCap = oldThr;
        //如果oldCap为0并且oldThr为0 则当前map使用不带参数的构造方法创建，且未进行过put和get操作（因为使用过则table会被初始化）
        else {               // zero initial threshold signifies using defaults
            //新容量为初始化容量
            newCap = DEFAULT_INITIAL_CAPACITY;
            //新阈值为初始化容量*负载因子
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }

        //如果新阈值为0 
        //表示在上面的第一个判断中newCap大于了最大容量或者oldCap小于了DEFAULT_INITIAL_CAPACITY
        //即不满足如下的条件：else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY && oldCap >= DEFAULT_INITIAL_CAPACITY)
        //或者进入了上面的第二个判断（ else if (oldThr > 0) ）
        if (newThr == 0) {
            //计算新的阈值
            float ft = (float)newCap * loadFactor;
            //如果新容量小于最大容量并且计算的阈值小于最大容量则使用计算后的负载因子
            //如果新容量大于等于了最大容量或者计算得到的阈值大于了最大容量则新阈值置为int的最大值（给一个足够大的值，以后尽量不再触发扩容）
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        //更新当前map的扩容阈值
        threshold = newThr;

        //以下是table扩容并重新赋值的逻辑
        @SuppressWarnings({"rawtypes","unchecked"})
        //首先根据newCap创建新的table
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        //把当前map的table置为新创建的table
        table = newTab;
        //如果之前的table被初始化过（table可能存有值）
        if (oldTab != null) {
            //遍历oldTab
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                //把当前位置的node赋给e，如果e不为null，表示当前node中有值
                if ((e = oldTab[j]) != null) {
                    //把旧table中值置为null（防止内存泄露）
                    oldTab[j] = null;
                    //如果e.next是null 表示当前桶中只有一个值，把当前值放入相应位置即可
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    //如果e为树节点则进入树节点重分配相关函数
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    //进入链表节点重分配逻辑
                    else { // preserve order
                        //当前节点的链表
                        Node<K,V> loHead = null, loTail = null;
                        //偏移后的链表
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        //
                        do {
                            next = e.next;
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
```
