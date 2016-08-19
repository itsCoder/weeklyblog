---
title: Java-HashMap
date: 2016-08-18 11:37:48
tags: javacode
categories: About Java
---
java 集合框架源码分析系列之 HashMap

<!--more-->

### 引言

我们都知道 HashMap 输出是无序的。是因为存储时候 HashMap 会根据 key 值来决定 value 的存储位置。但是我们想过没有？为什么输出的时候顺序就会跟存储时候相反呢？JDK1.7 中的 HashMap 底层构造与 JDK1.8 中有哪些区别呢？

## 一 . JDK1.7 中的 HashMap

在 **JDK 1.7** 之前的：HashMap 的内部存储结构其实是**数组**和**链表**的结合。当实例化一个 HashMap 时，系统会创建一个长度为 Capacity 的 **Entry 数组**，这个长度在哈希表中被称为容量(Capacity)，在这个数组中可以存放元素的位置我们称之为“桶”(bucket)，每个 bucket 都有自己的索引，系统可以根据索引快速的查找 bucket 中的元素。 

每个 bucket 中存储一个元素，即一个 Entry 对象，但每一个 Entry 对象可以带一个**引用变量**，**用于指向下一个元素**，因此，在一个桶中，就有可能生成一个 **Entry 链**。

可如下图表示：

![](http://7xrl8j.com1.z0.glb.clouddn.com/1.8%20hashmap.jpg)


Crazy Java 上面说了，发生冲突的元素以**链表形式存储**，必须按顺序搜索。Hash 表包含如下属性：

* 容量(capacity):hash 表中桶的数量，也就是上面 Entry 数组的长度。
* 初始化容量(inital capacity)。
* 尺寸(size):当前 Hash 表中记录的数量。
* 负载因子(load factor):size/capacity。负载因子为 0 则表示为空的 Hash 表，0.5 表示半满的 Hash 表。**轻负载**的具有冲突少，适宜插入与查询的特点。

* 负载极限：当 Hash 表中的负载因子达到指定的极限时，Hash 表会自动的增加桶的数量，重新分配，这种行为被称为 **rehash。**

总之默认加载因子 (0.75) 在时间和空间成本上寻求一种折衷。加载因子过高虽然减少了空间开销，但同时也增加了查询成本（在大多数 HashMap 类的操作中，包括 get 和 put 操作，都反映了这一点）。在设置初始容量时应该考虑到映射中所需的条目数及其加载因子，以便最大限度地减少 rehash 操作次数。如果初始容量大于最大条目数除以加载因子，则不会发生 rehash 操作。 

* [处理Hash冲突的方法](http://carmen-hongpeng.iteye.com/blog/1706415)

## 二 . JDK1.7 中 HashMap

我们首先来分析一下 JDK1.7 中的 HashMap 相关定义和实现原理。下面是常量,属性以及构造函数的定义：

### 2.1 常量定义

``` java
// table 数组即 Entry 数组的初始值
static final int DEFAULT_INITIAL_CAPACITY = 16;
// 最大的容量，左移即 2 的 30 次方
static final int MAXIMUM_CAPACITY = 1 << 30;
// 负载因子
static final float DEFAULT_LOAD_FACTOR = 0.75f;
// 储存key-value键值对的数组，一个键值对对象映射一个Entry对象
transient Entry[] table;
// 键值对的数目
transient int size;
// 调整HashMap大小门槛，该变量包含了HashMap能容纳的key-value对的极限，它的值等于HashMap的容量乘以负载因子
int threshold;
// 加载因子
final float loadFactor;
// HashMap结构修改次数,防止在遍历时，有其他的线程在进行修改
transient volatile int modCount;
```

### 2.2 构造函数

``` java

public HashMap(int initialCapacity, float loadFactor) {
	// 初始大小为0 即抛出异常
	if (initialCapacity < 0)
			throw new IllegalArgumentException("Illegal initial capacity: "
					+ initialCapacity);
		// 初始大小超过指定的最大值，则容量为指定的最大值
		if (initialCapacity > MAXIMUM_CAPACITY)
			initialCapacity = MAXIMUM_CAPACITY;
		// 负载因子也不能为0
		if (loadFactor <= 0 || Float.isNaN(loadFactor))
			throw new IllegalArgumentException("Illegal load factor: "
					+ loadFactor);
		// Find a power of 2 >= initialCapacity
		int capacity = 1;
		// 使得capacity 的大小为2的幂
		while (capacity < initialCapacity)
			capacity <<= 1;
		this.loadFactor = loadFactor;
		// size 超过这个数值就会扩容,即其为阈值
		threshold = (int) (capacity * loadFactor);
		table = new Entry[capacity];
		init();
}
```

### 2.3 Get() 函数源码

``` java
public V get(Object key) {
  		// key 为 null 的情况
        if (key == null)
            return getForNullKey();
  		// key ！= null 的情况
        int hash = hash(key.hashCode());
  		// 遍历 table 
        for (Entry<K,V> e = table[indexFor(hash, table.length)];
             e != null;
             e = e.next) {
            Object k;
          	// 相等则返回 e.value
            if (e.hash == hash && ((k = e.key) == key || key.equals(k)))
                return e.value;
        }
        return null;
    }
```

当 key 为 null 时候：

``` java
private V getForNullKey() {
  		// 遍历 table[0] 的链表
        for (Entry<K,V> e = table[0]; e != null; e = e.next) {
            if (e.key == null)
                return e.value;
        }
        return null;
    }
```

 该方法是一个私有方法，只在 get 中被调用。该方法判断 table[0] 中的链表是否包含 key 为 null 的元素，包含则返回 value，不包含则返回 null。为什么是遍历 **table[0] 的链表**？因为 key 为 null 的时候获得的 hash 值都是 0。

### 2.4 Put() 函数源码

**注意下 putForNullKey( ) 这个函数**,说明 HashMap 是允许 key 为 Null 的。

```java
public V put(K key, V value) {
		// 即上图的 Entry 数组为空，则初始化
        if (table == EMPTY_TABLE) {
            inflateTable(threshold);
        }
		// put 进去的 key 为空的话,去 putForNullKey(value)这个方法存放
        if (key == null)
            return putForNullKey(value);
		// key 不为空，计算 hashkey
        int hash = hash(key);
		// 根据 hashkey 找到在 Entry 数组中的位置
        int i = indexFor(hash, table.length);
		// 每个位置都是一个链表，循环直到找到有位置，当前 Entry 数组位置 e 存在
		// 每个节点都有一个 next 索引 指向 链的下一个节点
  		// 这种情况是 Entry 数组上已经插入了元素。
        for (Entry<K,V> e = table[i]; e != null; e = e.next) {
            Object k;
            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);
				// 返回原来这个位置上的值
                return oldValue;
            }
        }
        // 表示更改了一次底层数据结构
        modCount++;
		// 当前 Entry 数组位置为空，即还没有插入元素。则直接插入
        addEntry(hash, key, value, i);
        return null;
}
// addEntry 方法中会检查当前 table 是否需要 resize
void addEntry(int hash, K key, V value, int bucketIndex) {
		// Entry 桶位不为空，就是已经插入了值，并且 size 大于 一定的量了
        if ((size >= threshold) && (null != table[bucketIndex])) {
		// 当前 map 中的 size 如果大于 threshole 的阈值，则 resize 将 table 的 length 扩大2倍。
            resize(2 * table.length); 
            hash = (null != key) ? hash(key) : 0;
            bucketIndex = indexFor(hash, table.length);
        }
		//增加数组的长度
        createEntry(hash, key, value, bucketIndex);
}
```


下面就是对**数组进行扩容了**，很自然的问一句什么时候进行扩容？通过上面我们知道了，HashMap 有一个冲突因子 一般折中取值为 0.75，而数组的默认长度为 16，所以当 HashMap **数组中的元素** 超过了 12 就会进行扩容  resize(2 * table.length) 由这一句可以看出，扩大一倍，即32，然后还要**重新计算数据元素的位置。**

``` java
	void resize(int newCapacity) {
		// 将旧数组元素输出
        Entry[] oldTable = table;
		// 将旧数组容量输出保存
        int oldCapacity = oldTable.length;
		// 如果说旧容量已经达到最大的容量了
        if (oldCapacity == MAXIMUM_CAPACITY) {
			// 这里表示整形能够表示的最大数值，在前面 ArrayList 中应该也有
            threshold = Integer.MAX_VALUE;
            return;
        }
        // 构造一个新的 table 数组
        Entry[] newTable = new Entry[newCapacity];
		// 转移 数据
        transfer(newTable, initHashSeedAsNeeded(newCapacity));
        table = newTable;
		// 计算新表的 负载极限
        threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
	    }
	// 将当前 table 的 Entry 转移到新的 table 中
    void transfer(Entry[] newTable, boolean rehash) {
		// 新 table 的容量
        int newCapacity = newTable.length;
		// 将原 table 中数据迭代输出
        for (Entry<K,V> e : table) {
            while(null != e) {
                Entry<K,V> next = e.next;
                if (rehash) {
                    e.hash = null == e.key ? 0 : hash(e.key);
                }
				// 重新计算索引了
                int i = indexFor(e.hash, newCapacity);
                e.next = newTable[i];
                newTable[i] = e;
                e = next;
            }
        }
   	 }
```

另外底下这个算法(公式)是用来计算 Hash 数值的，加入了高位计算，防止低位不变，高位变化时，降低造成的 Hash 冲突。**但是实际的原理不是很懂。**

``` java
static int hash(int h) {  
    h ^= (h >>> 20) ^ (h >>> 12);  
    return h ^ (h >>> 7) ^ (h >>> 4);  
}
```

同时也在前面一段代码也看到了 indexFor( )---顾名思义，他就是用来寻找在 Entry 数组（即 table 表），相应 Hash 数值所在的位置。这个要重点说下，一般对**[哈希表的散列](http://baike.baidu.com/view/329976.htm)**很自然地会想到用Hash 值对 length 取模（即除法散列法），Hashtable 中也是这样实现的，这种方法基本能保证元素在哈希表中散列的比较均匀，但取模会用到除法运算，效率很低，HashMap 中则通过 h&(length-1) 的方法来代替取模，同样实现了均匀的散列，但效率要高很多，这也是 HashMap 对 Hashtable 的一个改进。源码的精妙之处啊！

``` java
static int indexFor(int h, int length) {  
    return h & (length-1);  
}
```

另外一个 Fail-Fast 机制，这个的出现就是因为 HashMap 是 **线程不安全的**，也就是说如果同时几个线程对 HashMap 进行修改，将会抛出异常，而这个机制的实现就是因为 modCount 。我们每一次，对 HashMap 的操作这个数值都会增加，而且也是**全局的**(是否是来自父类 Map？)。

``` java
HashIterator() {
    expectedModCount = modCount;
    if (size > 0) { // advance to first entry
    Entry[] t = table;
    while (index < t.length && (next = t[index++]) == null)
        //...
   }
}
```

``` java
final Entry<K,V> nextEntry() {     
    if (modCount != expectedModCount)     
        throw new ConcurrentModificationException();
```

看见上面的 modCount != expectedModCount 吗？ 如果不相等，就表示其他线程已经修改了，这时候就会抛出异常。从而保证 HashMap 是线程安全的。
另外，我们自然想到 上面这段代码可以在别的线程中，那么modCount 的同步性就很重要，其他线程要时刻知道它的数值，这时候 voliate 关键字就可以做到了。如下可验证：

``` java
// HashMap结构修改次数,防止在遍历时，有其他的线程在进行修改  
transient volatile int modCount;
```

现在我们解决一下引言中的疑问:  HashMap **不保证元素输出时和添加时顺序一致**，为什么呢？

个人感觉就是因为扩容然后重新分配的原因，虽然在 **我们用户**(写代码的) 在 put() 元素进入 HashMap 的时候没感觉到，但是底层机制应该时刻都在检查**要不要扩容**，**要不要重新分配**，这样动态的过程很难保证元素的顺序不变。

这样，jdk 7 中就差不多了，这种方法最大的缺点就是：一旦所有的 key 都相同(极端情况)，即**冲突**，HashMap 就会退化成一个链表(即在 table 表中的同一个位置不停的放置元素)，get() 的复杂度就会变成 o(n)。

所以 jdk 8 为了 优化这一点，采用了**红黑树来解决这些超过一定量的键值对**。


## 三 . JDK1.8 中的 HashMap

![](http://7xrl8j.com1.z0.glb.clouddn.com/1.7%20hashcode.jpg)

上面就是 JDK 1.8 中数据的存储方式，很明显看见了一颗红黑树。

### 3.1 常量数值的定义

``` java
	// 最开始的容量，必须是2的次方，这里即2的4次方
	static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
    // 最大容量
	static final int MAXIMUM_CAPACITY = 1 << 30;
    // 负载
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
	// list to tree 的临界值
    static final int TREEIFY_THRESHOLD = 8;
    // 删除冲突节点后，hash相同的节点数目小于这个数，红黑树就恢复成链表
    static final int UNTREEIFY_THRESHOLD = 6;
    // 扩容的临界值
    static final int MIN_TREEIFY_CAPACITY = 64;
	// 存储元素的数组
	transient Node<k,v>[] table;
```

### 3.2 Node 节点定义

``` java
// 继承自 Map.Entry<K,V>
static class Node<K,V> implements Map.Entry<K,V> {
       final int hash;
       final K key;
       V value;
	   // 指向下一个节点
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
		// 返回 Hash 值
        public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }

        public final V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }
		// 重写 equals() 
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

### 3.3 构造函数

``` java
	public HashMap(int initialCapacity, float loadFactor) {
       // 同上 jdk 1.7
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

计算 key 的 Hash 采用的函数,但是这个不是**真正的索引**，索引在后面还要进行一次运算。

``` java
static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```

### 3.4 红黑树定义

``` java
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
        TreeNode<K,V> parent;  // 父
        TreeNode<K,V> left;    //左
        TreeNode<K,V> right;   //右
        TreeNode<K,V> prev;    // needed to unlink next upon deletion
        boolean red;           //判断颜色
        TreeNode(int hash, K key, V val, Node<K,V> next) {
            super(hash, key, val, next);
        }
        //返回根节点
        final TreeNode<K,V> root() {
            for (TreeNode<K,V> r = this, p;;) {
                if ((p = r.parent) == null)
                    return r;
                r = p;
       }
```

### 3.5 put() 函数

``` java
     public V put(K key, V value) {
       // 调用 putVal() 函数 
       return putVal(hash(key), key, value, false, true);
     }
	 final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
		// tab 没有分配初始的空间或者为空，就resize一次
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
		// 指定hash值节点为空则直接在tab表中插入，注意这里(n - 1) & hash 才是真正的hash值
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
		// 不为空
        else {
            Node<K,V> e; K k;
			// 计算当前节点的key.hash 与要插入的 key.hash
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
				// 相同则覆盖
                e = p; 
			// 若不同的话，并且当前节点已经在TreeNode上了
            else if (p instanceof TreeNode)
				// 创建新的树节点
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
			// key.hash不同并且也不再TreeNode上，在链表上找到p.next==null
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
						// 表示在表尾插入
                        p.next = newNode(hash, key, value, null);
						// 新增节点后如果节点个数到达阈值，则将链表转换为红黑树
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
							// 这里有可能还会跟64比较，判断是否需要对 table进行扩容
                            treeifyBin(tab, hash);
                        break;
                    }
					// 如果找到了同hash、key的节点，那么直接退出循环
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
					// 更新 p 指向下一节点
                    p = e;
                }
            }
			// 退出循环后的操作，hash 和 key 都相同，则覆盖更新
            if (e != null) { 
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```

### 3.6 getNode() 函数

get 函数最终还是要通过 getNode() 来进行操作：

``` java
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    // table已经初始化，长度大于0，根据hash寻找table中的项也不为空
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        // 桶中第一项(数组元素)相等
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        // 桶中不止一个结点
        if ((e = first.next) != null) {
            // 为红黑树结点
            if (first instanceof TreeNode)
                // 在红黑树中查找
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            // 否则，在链表中查找
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

### 3.7 resize() 函数 

扩容函数，指的是对 table 表进行的操作，resize() 函数分析如下：

``` java
		final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        // 对 newCap 的取值
        if (oldCap > 0) {
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
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
    	//...省略//
                        while ((e = next) != null);
						// 这里就是冲突的链表分成两个队列存储
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

### 3.8 treeifyBin() 函数

判断是**对 table 表进行扩容** 还是将**冲突节点调整成红黑树：**

``` java
	
	final void treeifyBin(Node<K,V>[] tab, int hash) {
        int n, index; Node<K,V> e;
		// table 为空或者 table 的桶位还没达到 64，进行 table 扩容，而不是红黑调整
        if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
            resize();
		// >=64,进行红黑树转化
        else if ((e = tab[index = (n - 1) & hash]) != null) {
            TreeNode<K,V> hd = null, tl = null;
            do {
                TreeNode<K,V> p = replacementTreeNode(e, null);
                if (tl == null)
                    hd = p;
                else {
                    p.prev = tl;
                    tl.next = p;
                }
                tl = p;
            } while ((e = e.next) != null);
            if ((tab[index] = hd) != null)
              // 调整成红黑树  
              hd.treeify(tab);
        }
    }
```



## 四 . 个人总结

上面说了很多，由于 HashMap 有三种数据结构，数组即 table 表，链表，红黑树。我们很自然考虑到，每种结构达到什么程度时，table 会进行扩容,链表会向红黑树转化。

总结就是当 table 数达到 64 后，冲突节点数为 8 时，则进行链表向树结构转换，也就是说当 table 表还未达到 64时，每个桶位冲突的 key 达到 8 个以及上时，我们会一边进行 table 表的扩容，一边会将**冲突的链表**分成两个队列，而此时并没有将链表向红黑树转换。这些操作都在 resize() 中。

## 五 .参考和扩展阅读

* [HashMap 在1.7 与 1.8 之间的区别](http://www.jianshu.com/p/77b87a964088)


* [Hash 算法深入了解](http://blog.csdn.net/qa962839575/article/details/44889553)


* [HashMap 在Android中应用](http://www.jianshu.com/p/e54047b2b563)
* [LinkedHashMap 的实现原理(推荐)](http://allenwu.itscoder.com/2016/05/24/Java-linkedhashmap/)

  ​