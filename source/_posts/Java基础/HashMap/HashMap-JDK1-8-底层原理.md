---
title: HashMap(JDK1.8)底层原理
tags:
  - HashMap
categories: [Java基础,HashMap]
date: 2020-10-13 16:25:34
---

# HashMap概述	

![Map图](https://i.loli.net/2020/10/19/24qXCrapKQlVB96.png)​									

> - HashMap是 Key-Value 对映射的抽象接口，该映射不包括重复的键，即一个键对应一个值。
> - 在HashMap中，其会根据hash算法来计算key-value的存储位置并进行快速存取。
> - HashMap允许key和value为null
> - 当Hash值计算元素位置时，可能会存在Hash冲突（Hash碰撞），HashMap是采用链表解决的。
> - HashMap的实现，在JDK7中采用的是数组+链表，JDK8中采用的是数组+链表+红黑树。
> - HashMap是非线程安全的，多线程同时put时候会导致数据不一致。HashTable和ConcurrentHashMap是线程安全的，在多线程情况下ConcurrentHashMap是更优的选择。
> - HashMap在1.7链表采用头插法避免遍历链表，多线程时扩容可能会发生死循环情况。1.8采用尾插法不会导致死循环，且引入红黑树加快查询效率。

<!--more-->	



# HashMap重要属性

```java
   /**
     * The default initial capacity - MUST be a power of two.
     * 初始map的容量大小为16（容量必须是2的幂次倍）
     */
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; //16

    /**
     * The maximum capacity, used if a higher value is implicitly specified
     * by either of the constructors with arguments.
     * MUST be a power of two <= 1<<30.
     *  Map的最大容量为2的30次方
     *  java中存放的是补码，1左移31位的为 16进制的0x80000000代表的是-2147483648–>所以最大只能是30
     */
    static final int MAXIMUM_CAPACITY = 1 << 30;

    /**
     * The load factor used when none specified in constructor.
     * Map 容量的记载因子，主要影响map的在什么时候扩容（此处为0.75，Map数组的容量大小的75%时开始扩容）
     */
    static final float DEFAULT_LOAD_FACTOR = 0.75f;

    /**
     * The bin count threshold for using a tree rather than list for a
     * bin.  Bins are converted to trees when adding an element to a
     * bin with at least this many nodes. The value must be greater
     * than 2 and should be at least 8 to mesh with assumptions in
     * tree removal about conversion back to plain bins upon
     * shrinkage.
     *  当桶(bucket)上的结点数大于这个值时会转成红黑树
     */
    static final int TREEIFY_THRESHOLD = 8;

    /**
     * The bin count threshold for untreeifying a (split) bin during a
     * resize operation. Should be less than TREEIFY_THRESHOLD, and at
     * most 6 to mesh with shrinkage detection under removal.
     *  当桶(bucket)上的结点数小于这个值时树转链表
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
     *  存储元素的table数组，容量总是2的幂次倍
     */
    transient Node<K,V>[] table;

    /**
     * Holds cached entrySet(). Note that AbstractMap fields are used
     * for keySet() and values().
     *  存储具体元素的集合
     */
    transient Set<Map.Entry<K,V>> entrySet;

    /**
     * The number of key-value mappings contained in this map.
     *  map的元素个数
     */
    transient int size;

    /**
     * The number of times this HashMap has been structurally modified
     * Structural modifications are those that change the number of mappings in
     * the HashMap or otherwise modify its internal structure (e.g.,
     * rehash).  This field is used to make iterators on Collection-views of
     * the HashMap fail-fast.  (See ConcurrentModificationException).
     * 每次扩容和更改map结构的计数器
     */
    transient int modCount;

    /**
     * The next size value at which to resize (capacity * load factor).
     * 临界值 当实际大小(容量*填充因子)超过临界值时，会进行扩容
     */
    int threshold;

    /**
     * The load factor for the hash table.
     * 加载因子
     */
    final float loadFactor;

```



# HashMap Node<k,v>节点

```java
 static class Node<K,V> implements Map.Entry<K,V> {
        // hash值
        final int hash;
        // key值
        final K key;
        // value 值
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
 		// 重写hashCode()方法
        public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }

        public final V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }
		// 重写equels方法
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




# HashMap四个构造方法

### HashMap() 无参构造方法

源码：

```java
	/**
     * 默认初始容量为16，加载因子为0.75
     */
    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }
```

> 例子:
>
> ```
> Map<String,String> map = new HashMap<>();
> ```
>
> 初始容量`initialCapacity`为默认值 : 16
>
> 负载因子`loadFactor`为默认值: 0.75f 
>
> <key,value>对数量达到 16*0.75=12  将进行扩容，调用resize()方法



![](https://i.loli.net/2020/10/19/24qXCrapKQlVB96.png)

### HashMap(int initialCapacity, float loadFactor)

源码：

```java
	/**
     * 初始化map初始化的大小并制定加载因子的构造方法
     * @param  initialCapacity 初始容量大小
     * @param  loadFactor      加载因子
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

> 例子：
>
> ```java
> Map<String,String> map = new HashMap<8,0.5>();
> ```
>
> 初始化 `initCapacity`   8
>
> 负载因子`loadFactor`   0.5
>
> <key,value>对数量达到 8*0.5=4  将进行扩容，调用resize()方法



### HashMap(int initialCapacity) 

源码：

```java
	/**
     *
     * @param  initialCapacity 指定map的初始容量，默认加载因子为0.75
     * @throws IllegalArgumentException if the initial capacity is negative.
     */
    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }
```



### HashMap(Map<? extends K, ? extends V> m)

源码：

```java
     /**
     * 加载因子为0.75，指定一个map构建一个初始化的map
     */
    public HashMap(Map<? extends K, ? extends V> m) {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        putMapEntries(m, false);
    }
```



# 获取哈希值 hash() 方法

```java
      static final int hash(Object key) {
        int h;
        // key.hashCode()：返回散列值也就是hashcode
        // ^ ：按位异或
        // >>>:无符号右移，忽略符号位，空位都以0补齐
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }

```

其中 `h>>>16`  称作扰动函数

在JDK1.8的实现中，从速度、功效、质量来考虑

优化了高位运算的算法，通过hashCode()的高16位**异或**低16位实现

> 目的：使计算出来的hash值更加不确定 来**降低碰撞的概率**



`static final int hash(Object key)`得到的int类型的hash值

然后再通过`hash & (table.length -1)`来得到该对象在Node<k,v>数组中保存的位置

> 这就是为什么HashMap的容量必须是2的次幂倍数啦。
>
>  例如： 2^4 = 16, 16-1=15 = 0B(1111)。



​			![HashMap_hash](https://i.loli.net/2020/10/20/8yxgZaUKSVBNL6i.png)

# HashMap的Put方法

```java
 /**
     * 如果key存在，则value将替换原来的value值并返回被替换的那个值
     * @param key key值
     * @param value value值
     * @return  返回的值为，key下被替换的值
     */
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }

  /**
     * put key，value
     * @param hash key的hash值
     * @param key  key值
     * @param value value值
     * @param onlyIfAbsent 如果为true，key存在时value不予替换
     * @param evict if false, the table is in creation mode.
     * @return previous value, or null if none
     */
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        // 检验table是否为空或者length为0，如果是则调用resize方法进行初始化
        if ((tab = table) == null || (n = tab.length) == 0)
           // 数组扩容
            n = (tab = resize()).length;
            // 计算key对应的桶的位置是否存在存在元素，判断为null表示不存在
        if ((p = tab[i = (n - 1) & hash]) == null)
            // 在table[i]出创建一个新节点
            tab[i] = newNode(hash, key, value, null);
        else { // table[i] 处已经存在了元素（Hash碰撞）
            Node<K,V> e; K k;
            // 如果key值已经存在，key不为null的情况下，e记录下当前key的node节点
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
              // 如果桶中的引用类型为 TreeNode，则调用红黑树的插入方法
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else { // 链表模式
                // 循环当前链表
                for (int binCount = 0; ; ++binCount) {
                    //  p.next 为null，则将p.next指向先插入的node
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        // 如果链表长度大于设定阈值长度时（默认为8），将链表尝试转换成红黑树
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            //在里面还有一个条件 需要capacity==MIN_TREEIFY_CAPACITY=64才进行链表转换红黑树，否则知识进行resize()
                            treeifyBin(tab, hash);
                        // 跳出循环
                        break;
                    }
                    // key存在，直接跳出循环进行覆盖操作
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            if (e != null) { // 表示key已经存在
                V oldValue = e.value;
                // 如果允许替换，替换原值
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                  // 访问后回调
                afterNodeAccess(e);
                // 返回旧值
                return oldValue;
            }
        }
        // modCount加1
        ++modCount;
        // size是否大于阈值，大于则进行扩容
        if (++size > threshold)
            resize();
         // 插入后回调
        afterNodeInsertion(evict);
        return null;
    }

```



![HashMap_put()](https://i.loli.net/2020/10/19/BjTQaGYDCiLhbdU.png)

## 总结put()方法

1. **table数组是否为null**或**数组长度==0,**是则`resize()`扩容

2. 执行`hash(Object key)`得到一个`int`类型的hash值。

3. 判断table是否为空，为空表明这是第一个元素插入，则使用resize()进行扩容，初始大小默认16;

   直接插入Node节点，然后跳到第5步判断**是否扩容**。

4. 如果table不为空,则判断table[i]是否为空，为空则没有发生hash碰撞，直接插入node节点

   然后跳到第5步,否则进行下一步。

5. 如果table[i]不为空,则发生了hash碰撞，继续进行下面三种判断:
   1.桶中第一个元素(数组中的结点)的**hash值相等**，key相等，赋值给node节点e;
   2.hash值不相等,说明key不相等,若为**红黑树**，放入树中，赋值给node节点e;
   3.hash值不相等,说明key不相等,**链表**，遍历链表，尾插法插入到最后，并返回node节点e。

   (链表遍历里面还有长度的判断,当**链表长度>=8**并且**数组长度>=64**之后需要将**链表转化为红黑树**进行存储;

6. 判断新插入这个值是否导致`size`已经超过了`threshold`阈值，是则进行扩容。


# HashMap的get()方法

```java
 	// key获取元素
 	public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }

    /**
     * Implements Map.get and related methods.
     *
     * @param key的hash值
     * @param key值
     * @return the node, or null if none
     */
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
          //   1、HashMap中存储数据的数组table不为null；
         //   2、数组table长度大于0
         //   3、table已经创建，且通过hash值计算出的节点存放位置有节点存在；
        // 若上面三个条件都满足，才表示HashMap中可能有我们需要获取的元素
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
             // 定位到元素在数组中的位置后，我们开始沿着这个位置的链表或者树开始遍历寻找
            // 注：JDK1.8之前，HashMap的实现是数组+链表，到1.8开始变成数组+链表+红黑树
       	    // 首先判断这个位置的第一个节点的key值是否与参数的key值相等，
            // 若相等，则这个节点就是我们要找的节点，将其返回
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
           // 遍历链表数据
            if ((e = first.next) != null) {
                // 如果节点为红黑树结构，遍历数结构查询
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                 // 遍历链表查询
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        // 未查询到，返回null
        return null;
    }

```



# HashMap扩容机制 resize()

```java
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
       // 记录原table数组
        Node<K,V>[] oldTab = table;
        // 计算oldCap的大小
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        // 扩容临界值（cap * loadFactor）
        int oldThr = threshold;
        int newCap, newThr = 0;
       	// oldCap>0 表述map的buket数组已经初始化
        if (oldCap > 0) {
             // 如果已经达到容量最大值，不进行扩容（Hash碰撞）
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            // 扩容为原来的容量的2倍
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                // 左移一位（容量*2）
                newThr = oldThr << 1; // double threshold
        }
        // 若不满足上面的oldCap > 0，表示数组还未初始化
        // 若当前阈值不为0，则设置为当前阈值
        // 这是因为HashMap有两个带参构造器，可以指定初始容量，
        else if (oldThr > 0) 
            newCap = oldThr;
        else {       
            // 初始化为默认的容量值      
            newCap = DEFAULT_INITIAL_CAPACITY;   
            // 容量界线 16* 0.75
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY); 
        }
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
        // 扩容后的数组大小
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        // 将老数组的数据重新复制与新数组中
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                // j位置存在数据
                if ((e = oldTab[j]) != null) {
                    // 清空数据
                    oldTab[j] = null;
                    // 若table数组的j位置只有一个节点，则直接将这个节点放入新数组，
                    // 使用 & 替代 % 计算出余数，即下标
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode) 
                   		 // 红黑树节点-执行红黑树算法（hash冲突解决）
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // 链表数据小于限制数据-执行链表结构
                    	// 创建两个头尾节点，表示两条链表，旧数据的一条链表可能被查分到两条链表上
                    	// // 一条下标不变的链表，一条下标+oldCap
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                             // e.hash & oldCap==0 原链表的索引值
                            if ((e.hash & oldCap) == 0) {
                             	// 如果最后一个元素为null，说明为头节点
                                if (loTail == null)
                                	// 设置头结点为e
                                    loHead = e;
                                else // 头节点已经设置了元素，直接将该元素设置在尾节点的next
                                    loTail.next = e;
                                // 设置当前尾节点为e
                                loTail = e;
                            }
                            // 若与原容量做与运算，结果为1，表示将这个节点放入到新数组中，下标将改变
                            else { 
                                if (hiTail == null) // 尾节点为空，说明该链表还没有值
                                    // 设置头结点为e
                                    hiHead = e;
                                else
                                    // 头节点已经设置了元素，直接将该元素设置在尾节点的next
                                    hiTail.next = e;
                                // 设置当前尾节点为e
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        // 所有节点遍历完后，判断下标不变的链表是否有节点在其中
                        if (loTail != null) {
                            // 将这条链表的最后一个节点的next指向null
                            loTail.next = null;
                            // 将链表loHead放入新数组的相同位置
                            newTab[j] = loHead;
                        }
                        // 将变化后的元素生成的新链表放在新的计算位置
                        if (hiTail != null) {
                            hiTail.next = null;
                            // 这条链表放入的位置要在原来的基础上加上oldCap
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        // 将新创建的map Tab 返回
        return newTab;
    }

```

> **注意:** 这有一个算法公式`num % 2^n == num & (2^n - 1)`，上述resize方法执行的时候将求余运算转化为位&运算。