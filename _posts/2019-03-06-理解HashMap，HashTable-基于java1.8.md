---
title: "理解HashMap-基于java1.8"

url: "https://wsk1103.github.io/"

tags:
  - Java
---

**java -version** ：jdk 1.8.0_191


# HashMap
## 构造
![image](https://raw.githubusercontent.com/wsk1103/images/master/hashmap/1.png)

## 类内参数，方法
![image](https://raw.githubusercontent.com/wsk1103/images/master/hashmap/3.png)

![image](https://raw.githubusercontent.com/wsk1103/images/master/hashmap/4.png)

![image](https://raw.githubusercontent.com/wsk1103/images/master/hashmap/5.png)



## 实现
![image](https://raw.githubusercontent.com/wsk1103/images/master/hashmap/2.png)
基于**数组 + 链表**实现。

当单条链表的个数达到**8**个时候，链表就会被转换成**红黑树**。

链表时间复杂度**O(n)**，红黑树时间复杂度**O(log n)**
## 静态参数


```
    /**
     * The default initial capacity - MUST be a power of two.
     */
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
    初始化时，默认数组的大小
    /**
     * The maximum capacity, used if a higher value is implicitly specified
     * by either of the constructors with arguments.
     * MUST be a power of two <= 1<<30.
     */
    static final int MAXIMUM_CAPACITY = 1 << 30;
    数组的最大值，2^30。
    /**
     * The load factor used when none specified in constructor.
     */
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
    默认负载因子，当新增的key在Map中超过 (Map中key的数量 * 负载因子) 的时候，就会扩容。
    扩容是比较耗内存的，所以设计的时候，很多时候都要考虑一下Map的大小，防止频繁扩容或者占用无畏的空间。
    /**
     * The bin count threshold for using a tree rather than list for a
     * bin.  Bins are converted to trees when adding an element to a
     * bin with at least this many nodes. The value must be greater
     * than 2 and should be at least 8 to mesh with assumptions in
     * tree removal about conversion back to plain bins upon
     * shrinkage.
     */
    static final int TREEIFY_THRESHOLD = 8;
    当单个链表里面的key个数超过8个时候，转换成红黑树。
    /**
     * The bin count threshold for untreeifying a (split) bin during a
     * resize operation. Should be less than TREEIFY_THRESHOLD, and at
     * most 6 to mesh with shrinkage detection under removal.
     */
    static final int UNTREEIFY_THRESHOLD = 6;

    /**
     * The smallest table capacity for which bins may be treeified.
     * (Otherwise the table is resized if too many nodes in a bin.)
     * Should be at least 4 * TREEIFY_THRESHOLD to avoid conflicts
     * between resizing and treeification thresholds.
     */
    static final int MIN_TREEIFY_CAPACITY = 64;
    
    //超过这个值则以翻倍形式扩容(resize)
    //threshold = capacity * load factor
    int threshold;
    map的阈值
```

## put 方法

```
    /**
     * Associates the specified value with the specified key in this map.
     * If the map previously contained a mapping for the key, the old
     * value is replaced.
     *
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
在存入到HashMap的时候，需要先计算一下hash值，然后根据hash值存放到对应的数组和链表中。
```
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```
函数作用：高16bit不变，低16bit和高16bit做了一个异或

假设 h = hashCode() = 1111 1111 1111 1111 1111 0101 1010 0011

步数 |操作 | 值 | 
|---|---|---|
1 | h = hashCode() | 1111 1111 1111 1111 1111 0101 1010 0011
2 | h | 1111 1111 1111 1111 1111 0101 1010 0011
3 | h >>> 16 | 0000 0000 0000 0000 1111 1111 1111 1111 
4 | h ^ (h >>> 16) | 1111 1111 1111 1111 0000 1010 0101 1100
5 | (n - 1) & hash | 0000 0000 0000 0000 0000 0000 0000 1111(15) <br> 1111 1111 1111 1111 0000 1010 0101 1100 
6 | 结果 | 1100(12)
则结果 (n - 1) & hash 值为12。

可以看出，HashMap是允许key为null的，当key为null的时候，hash值为0。

注意：当key存放是一个对象的时候，一般情况下我们需要重写这个对象的hashCode()方法，防止严重碰撞，或者链表/红黑树过长。
```
    /**
     * Implements Map.put and related methods
     *
     * @param hash hash for key
     * @param key the key
     * @param value the value to put
     * @param onlyIfAbsent if true, don't change existing value
     * @param evict if false, the table is in creation mode.
     * @return previous value, or null if none
     */
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            //重置HashMap
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null)
            //当数组tab中对应的位置为空，则直接存放到该tab[]中。
            //i = (n - 1) & hash 取模定位数组位置
            tab[i] = newNode(hash, key, value, null);
        else {
            //已经存在该key的hash值或者碰撞冲突。
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                // 已经存在，保存起来，用于返回oldValue
                e = p;
            else if (p instanceof TreeNode)
                //放入树节点中，也就是已经是红黑树的情况。
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                //循环列表
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        //放到列表尾部
                        p.next = newNode(hash, key, value, null);
                        //当列表个数超过8个的时候，转换成红黑树。
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    //已经存在该key
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            //如果已经存在，替换，然后返回被替换前的值。
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                //当参数onlyIfAbsent设置为false的时候，即使相同的key，也不插入。
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);//这个没有什么用，空方法
                return oldValue;
            }
        }
        //修改的次数，该值用transient修饰，无法被序列号.
        ++modCount;
        //当key的数量超过阈值时，翻倍并重新将key插入一个新的map中。
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);//这个没有什么用，空方法
        return null;
    }
```

## get方法

```
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
```
get方法也是用hash值，快速定位到具体的数组位置。


```
    /**
     * Implements Map.get and related methods
     *
     * @param hash hash for key
     * @param key the key
     * @return the node, or null if none
     */
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                // 找到hash值相等 和 key也相等
                return first;
            if ((e = first.next) != null) {
                if (first instanceof TreeNode)
                    //已经转换成红黑树，红黑树搜索
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                do {
                    //循环遍历链表查找
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        //找不到
        return null;
    }
```
## resize方法
扩容

每次扩容都需要重新分配数组，并把老的值再分配到新的数组，耗能比较高，所以应该尽量避免扩容。例如在初始化的时候，判断该Map的key的数量可以会达到最大的数值，**Map map = new HashMap(keyNum);**
```
    /**
     * Initializes or doubles table size.  If null, allocates in
     * accord with initial capacity target held in field threshold.
     * Otherwise, because we are using power-of-two expansion, the
     * elements from each bin must either stay at same index, or move
     * with a power of two offset in the new table.
     *
     * @return the table
     */
    final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            if (oldCap >= MAXIMUM_CAPACITY) {
                //当数组tab的个数超过2^32的时候，已经不需要做什么了。
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                    //位运算，翻倍。
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        else {               // zero initial threshold signifies using defaults
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        //开始将旧的值存到新的map中
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
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

## 初始化构造器
HashMap提供4中初始化的构造器
#### 1. 参数 int initialCapacity, float loadFactor

initialCapacity：初始map的数组大小  
loadFactor：负载因子
```
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
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity);
    }
```
#### 2. 参数int initialCapacity
initialCapacity：初始map的数组大小  
```
    /**
     * Constructs an empty <tt>HashMap</tt> with the specified initial
     * capacity and the default load factor (0.75).
     *
     * @param  initialCapacity the initial capacity.
     * @throws IllegalArgumentException if the initial capacity is negative.
     */
    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }
```
#### 3. 无参

```
    /**
     * Constructs an empty <tt>HashMap</tt> with the default initial capacity
     * (16) and the default load factor (0.75).
     */
    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }
```
#### 4. 参数 Map<? extends K, ? extends V> m

```
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
```

## 循环遍历Map


```
map.forEach((key,value)->{
    System.out.println("key=" + key + " value=" + value);
});
```


## 其他说明

#### HashMap 和 HashTable 区别
- HashMap是unsynchronized，使用的时候要注意线程安全。
- HashMap允许key为null，而HashTable不允许key为null。
- HashTable线程安全的原理是在操作Map的方法上都加入synchronized。


#### HashMap 和 ConcurrentHashMap 区别
- HashMap是unsynchronized，使用的时候要注意线程安全。
- ConcurrentHashMap是synchronized，多线程的情况下，直接使用这个。
- ConcurrentHashMap 的原理是在操作每个数组对应的位置加锁，而不是整个Map加锁，所以性能比HashTabe好。

#### Fail-Fast机制
在多个线程同时操作同一个Map的时候，每次修改map，则modCount++，当检测到modCount发生改变的时候，则抛出异常 **throw new ConcurrentModificationException();**

#### 取模

如果数组tab的长度为n，如果要对hash进行取模，一般的做法是

```
hash % n
```
但是HashMap的取模是

```
hash & (n - 1)
```
因为n在初始化的时候，我们会把n定义为2的次幂，所以n-1 的二进制是01111111...

hash & n-1 实际上就是取保留低位值，结果是在n的范围内，类似取模运算。

假设 n = 16，hash = 1111 1111 1111 1111 1111 0101 1010 0011

|步数 | 计算 | 值
|----|----|-----|
1 | n -1 | 0000 0000 0000 0000 0000 0000 0000 0111(15)
2 | hash | 1111 1111 1111 1111 1111 0101 1010 0011
3 | hash & n-1 | 0000 0000 0000 0000 0000 0000 0000 0011(3)

当时当在初始化数组的时候，没有设置为2的次幂，那么就和一般的取模运算一样，没有什么性能改进。

#### 使HashMap线程安全
如果不使用ConcurrentHashMap，那么可以使用java.util包。

```
Map m = Collections.synchronizedMap(new HashMap(...));
```
同时Set,List也一样。

# HashTable
实现的接口和继承的类和 HashMap 一致，里面的方法，变量的定义也是基本一致的。
只是在操作数组和链表的时候，在所有的方法上都添加 **synchronized** 关键字。

例如 **add方法**

```
    /**
     * Maps the specified <code>key</code> to the specified
     * <code>value</code> in this hashtable. Neither the key nor the
     * value can be <code>null</code>. <p>
     *
     * The value can be retrieved by calling the <code>get</code> method
     * with a key that is equal to the original key.
     *
     * @param      key     the hashtable key
     * @param      value   the value
     * @return     the previous value of the specified key in this hashtable,
     *             or <code>null</code> if it did not have one
     * @exception  NullPointerException  if the key or value is
     *               <code>null</code>
     * @see     Object#equals(Object)
     * @see     #get(Object)
     */
    public synchronized V put(K key, V value) {
        // Make sure the value is not null
        if (value == null) {
            throw new NullPointerException();
        }

        // Makes sure the key is not already in the hashtable.
        Entry<?,?> tab[] = table;
        int hash = key.hashCode();
        int index = (hash & 0x7FFFFFFF) % tab.length;
        @SuppressWarnings("unchecked")
        Entry<K,V> entry = (Entry<K,V>)tab[index];
        for(; entry != null ; entry = entry.next) {
            if ((entry.hash == hash) && entry.key.equals(key)) {
                V old = entry.value;
                entry.value = value;
                return old;
            }
        }

        addEntry(hash, key, value, index);
        return null;
    }
```
从这里就可以看出， HashTable 是不允许存储的对象为 **null**

并且 HashTable 中的链表是不会转为红黑树。

有趣的是， HashTable 的初始容量是 **11**，而 **扩容操作** 是 **int newCapacity = (oldCapacity << 1) + 1** ，即*2+1 ，  
计算hash也比较简单

```
int hash = key.hashCode();
int index = (hash & 0x7FFFFFFF) % tab.length;
```




