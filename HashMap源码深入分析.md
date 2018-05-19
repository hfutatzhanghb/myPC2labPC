

## HashMap源码深入分析

**前言**：

​       是时候认真学习一波HashMap的源码（JDK1.8）了，首先这家伙几乎是面试必考的，还考的贼细，其次了解底层实现有助于我们提升自己的内力，写出更有效率的代码。鉴于网上讲HashMap的资料很多，我会在文末列出我认为比较好的资源。

​       本文不整虚嗑，上来就是凿源码，非战斗人员也不要撤离，毕竟鲁迅说过浮躁是学不好任何东西的。

​      three，two，one，Go！！！



### 一、构造函数

让我们先从构造函数说起，HashMap有四个构造方法，别慌

#### 1.1  HashMap()

```java
    // 1.无参构造方法、
	// 构造一个空的HashMap，初始容量为16，负载因子为0.75
	public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }
```

无参构造方法就没什么好说的了。

#### 1.2 HashMap(int initialCapacity)

```java
	// 2.构造一个初始容量为initialCapacity，负载因子为0.75的空的HashMap，
    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }
```

`HashMap(int initialCapacity)` 这个构造方法调用了**1.3**中的构造方法。

#### 1.3 HashMap(int initialCapacity, float loadFactor)

```java
    // 3.构造一个空的初始容量为initialCapacity，负载因子为loadFactor的HashMap
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

	//最大容量
	//static final int MAXIMUM_CAPACITY = 1 << 30;
```

​       当指定的初始容量< 0时抛出`IllegalArgumentException`异常，当指定的初始容量`> MAXIMUM_CAPACITY`时，就让初始容量 = ` MAXIMUM_CAPACITY `。当负载因子小于0或者不是数字时，抛出`IllegalArgumentException异常`。

设定`threshold`。 这个threshold = capacity * load factor 。当HashMap的size到了threshold时，就要进行resize，也就是扩容。



`tableSizeFor()`的主要功能是返回一个比给定整数大且最接近的2的幂次方整数，如给定10，返回2的4次方16.  

​	

我们进入`tableSizeFor(int cap)`的源码中看看：

```java
    //Returns a power of two size for the given target capacity.
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

**note：** HashMap要求容量必须是2的幂。



首先，`int n = cap -1 `是为了防止cap已经是2的幂时，执行完后面的几条无符号右移操作之后，返回的capacity是这个cap的2倍。 如果不懂可以往下看完几个无符号移位后再回来看。（建议自己在纸上画一下）

- 如果n这时为0了（经过了cap-1之后），则经过后面的几次无符号右移依然是0，最后返回的capacity是1（最后有个n+1的操作）。这里只讨论n不等于0的情况。

  

  以16位为例，假设开始时 n 为 0000 1xxx xxxx xxxx  （x代表不关心0还是1）

- 第一次右移 `n |=  n >>> 1;`

  由于n不等于0，则n的二进制表示中总会有一bit为1，这时考虑最高位的1。通过无符号右移1位，则将最高位的1右移了1位，再做或操作，使得n的二进制表示中与最高位的1紧邻的右边一位也为1，如0000 11xx xxxx xxxx 。  

- 第二次右移 `n |= n >>> 2;` 

  注意，这个n已经经过了`n |= n >>> 1;` 操作。此时n为0000 11xx xxxx xxxx ，则n无符号右移两位，会将最高位两个连续的1右移两位，然后再与原来的n做或操作，这样n的二进制表示的高位中会有4个连续的1。如0000 1111 xxxx xxxx 。 

- 第三次右移 `n |= n >>> 4;`

  这次把已经有的高位中的连续的4个1，右移4位，再做或操作，这样n的二进制表示的高位中会有8个连续的1。如0000 1111 1111 xxxx 。 

第。。。，你还忍心让我继续推么？相信聪明的你已经想出来了，容量最大也就是32位的正数，所以最后一次     `n |= n >>> 16; ` 可以保证最高位后面的全部置为1。当然如果是32个1的话，此时超出了`MAXIMUM_CAPACITY` ，所以取值到 `MAXIMUM_CAPACITY` 。 



从https://blog.csdn.net/huzhigenlaohu/article/details/51802457这篇博客中找了张示例图：

![tableSizeFor示例图](D:\typora笔记\J2SE\assets\tableSizeFor示例图.png)

注意，得到的这个capacity却被赋值给了threshold。 这里我和这篇博客的博主开始的想法一样，认为应该这么写：`this.threshold = tableSizeFor(initialCapacity) * this.loadFactor; `  因为这样子才符合threshold的定义：threshold = capacity * load factor  。但是，请注意，在构造方法中，并没有对table这个成员变量进行初始化，table的初始化被推迟到了put方法中，在put方法中会对threshold重新计算 。



我说一下我在理解这个tableSizeFor函数中间遇到的坑吧，我在想如果n=-1时的情况，因为初始容量可以传进来0。我将n= -1 和下面几条运算一起新写了个测试程序，发现输出都是 -1。 这是因为计算机中数字是由补码存储的，-1的补码是 0xffffffff。所以无符号右移之后再进行或运算之后还是 -1。 那我想如果就无符号右移呢？ 比如-1>>>10。听我娓娓道来，32个1无符号右移10位后，高10位为0，低22位为1，此时这个数变成了正数，由于正数的补码和原码相同，所以就变成了0x3FFFFF即10进制的4194303。涨姿势了。



好开森，这个构造方法我们算是拿下了。怎么样，我猜你现在一定很激动，Hey，old  Fe，这才刚开始。接下来看最后一个构造方法。



#### 1.4 HashMap(Map<? extends K, ? extends V> m)

```java
    // 4. 构造一个和指定Map有相同mappings的HashMap，初始容量能充足的容下指定的Map,负载因子为0.75
	public HashMap(Map<? extends K, ? extends V> m) {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        putMapEntries(m, false);
    }
```

套路，直接刚 `putMapEntries(m,false)` 。源码如下：

```java
    
	/**
     * 将m的所有元素存入本HashMap实例中
     */
    final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
        //得到 m 中元素的个数
        int s = m.size();
        //当 m 中有元素时，则需将map中元素放入本HashMap实例。
        if (s > 0) {
            // 判断table是否已经初始化，如果未初始化，则先初始化一些变量。（table初始化是在put时）
            if (table == null) { // pre-size
                // 根据待插入的map 的 size 计算要创建的　HashMap 的容量。
                float ft = ((float)s / loadFactor) + 1.0F;
                int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                         (int)ft : MAXIMUM_CAPACITY);
                // 把要创建的　HashMap 的容量存在　threshold　中
                if (t > threshold)
                    threshold = tableSizeFor(t);
            }
            // 如果table初始化过
            // 判断待插入的　map 的 size,若　size 大于　threshold，则先进行　resize()，进行扩容
            else if (s > threshold)
                resize();
            //然后就开始遍历 带插入的 map ，将每一个 <Key ,Value> 插入到本HashMap实例。
            for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
                K key = e.getKey();
                V value = e.getValue();
                // put(K,V)也是调用　putVal　函数进行元素的插入
                putVal(hash(key), key, value, false, evict);
            }
        }
    }

```

这里面有个table变量，我们连同几个变量一起介绍下：

```java
    /**
     * The table, initialized on first use, and resized as
     * necessary. When allocated, length is always a power of two.
     * (We also tolerate length zero in some operations to allow
     * bootstrapping mechanics that are currently not needed.)
     */
    transient Node<K,V>[] table;

    /**
     * The number of key-value mappings contained in this map.
     */
    transient int size;

    /**
     * The number of times this HashMap has been structurally modified
     * Structural modifications are those that change the number of mappings in
     * the HashMap or otherwise modify its internal structure (e.g.,
     * rehash).  This field is used to make iterators on Collection-views of
     * the HashMap fail-fast.  (See ConcurrentModificationException).
     */
    transient int modCount;

    /**
     * The next size value at which to resize (capacity * load factor).
     *
     * @serial
     */
    // (The javadoc description is true upon serialization.
    // Additionally, if the table array has not been allocated, this
    // field holds the initial array capacity, or zero signifying
    // DEFAULT_INITIAL_CAPACITY.)
	//因为 tableSizeFor(int) 返回值给了threshold
    int threshold;

    /**
     * The load factor for the hash table.
     *
     * @serial
     */
    final float loadFactor;
```

其实就是哈希表。HashMap使用链表法避免哈希冲突（相同hash值），当链表长度大于TREEIFY_THRESHOLD（默认为8）时，将链表转换为红黑树，当然小于UNTREEIFY_THRESHOLD（默认为6）时，又会转回链表以达到性能均衡。 我们看一张HashMap的数据结构（数组+链表+红黑树 ）就更能理解table了：

![HashMap的数据结构](D:\typora笔记\J2SE\assets\HashMap的数据结构.png)



再回到putMapEntries函数中，如果table为null，那么这时就设置合适的threshold，如果不为空并且指定的map的size>threshold，那么就**resize()**。然后把指定的map的所有Key，Value，通过**putVal**添加到我们创建的新的map中。

**putVal**中有个**hash(key)**，那我们就先来看看**hash(key)**:

```java
/**
     * key 的 hash值的计算是通过hashCode()的高16位异或低16位实现的：(h = k.hashCode()) ^ (h >>> 16)
     * 主要是从速度、功效、质量来考虑的，这么做可以在数组table的length比较小的时候
     * 也能保证考虑到高低Bit都参与到Hash的计算中，同时不会有太大的开销
     */
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }

作者：命运I邂逅
链接：https://www.jianshu.com/p/3287cd3cec4b
來源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
```

异或运算：(h = key.hashCode()) ^ (h >>> 16)

原 来 的 hashCode : 1111 1111 1111 1111 0100 1100 0000 1010 
移位后的hashCode: 0000 0000 0000 0000 1111 1111 1111 1111 
进行异或运算 结果：1111 1111 1111 1111 1011 0011 1111 0101

这样做的好处是，可以将hashcode高位和低位的值进行混合做异或运算，而且混合后，低位的信息中加入了高位的信息，这样高位的信息被变相的保留了下来。掺杂的元素多了，那么生成的hash值的随机性会增大。



刚才我们漏掉了**resize()**和**putVal()** 两个函数，现在我们按顺序分析一波：

首先`resize()` ,先看一下哪些函数调用了`resize()`，从而在整体上有个概念：



![调用了resize的函数](D:\typora笔记\J2SE\assets\调用了resize的函数.png)

接下来上源码：

```java
	final Node<K,V>[] resize() {
		// 保存当前table
        Node<K,V>[] oldTab = table;
        // 保存当前table的容量
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        // 保存当前阈值
        int oldThr = threshold;
        // 初始化新的table容量和阈值 
        int newCap, newThr = 0;
        /*
        1. resize（）函数在size　> threshold时被调用。oldCap大于 0 代表原来的 table 表非空，
           oldCap 为原表的大小，oldThr（threshold） 为 oldCap × load_factor
        */
        if (oldCap > 0) {
            // 若旧table容量已超过最大容量，更新阈值为Integer.MAX_VALUE（最大整形值），这样以后就不会自动扩容了。
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
             // 容量翻倍，使用左移，效率更高
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                // 阈值翻倍
                newThr = oldThr << 1; // double threshold
        }
        /*
        2. resize（）函数在table为空被调用。oldCap 小于等于 0 且 oldThr 大于0，代表用户创建了一个 HashMap，但是使用的构造函数为      
           HashMap(int initialCapacity, float loadFactor) 或 HashMap(int initialCapacity)
           或 HashMap(Map<? extends K, ? extends V> m)，导致 oldTab 为 null，oldCap 为0， oldThr 为用户指定的 HashMap的初始容量。
    　　*/
        else if (oldThr > 0) // initial capacity was placed in threshold
            //当table没初始化时，threshold持有初始容量。还记得threshold = tableSizeFor(t)么;
            newCap = oldThr;
        /*
        3. resize（）函数在table为空被调用。oldCap 小于等于 0 且 oldThr 等于0，用户调用 HashMap()构造函数创建的　HashMap，所有值均采用默认值，oldTab（Table）表为空，oldCap为0，oldThr等于0，
        */
        else {               // zero initial threshold signifies using defaults
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        // 新阈值为0
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
        // 初始化table
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        if (oldTab != null) {
            // 把 oldTab 中的节点　reHash 到　newTab 中去
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    // 若节点是单个节点，直接在 newTab　中进行重定位
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    // 若节点是　TreeNode 节点，要进行 红黑树的 rehash　操作
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    // 若是链表，进行链表的 rehash　操作
                    else { // preserve order
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        // 将同一桶中的元素根据(e.hash & oldCap)是否为0进行分割，分成两个不同的链表，完成rehash
                        do {
                            next = e.next;
                            // 根据算法　e.hash & oldCap 判断节点位置rehash　后是否发生改变
                            //最高位==0，这是索引不变的链表。
                            if ((e.hash & oldCap) == 0) { 
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            //最高位==1 （这是索引发生改变的链表）
                            else {  
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {  // 原bucket位置的尾指针不为空(即还有node)  
                            loTail.next = null; // 链表最后得有个null
                            newTab[j] = loHead; // 链表头指针放在新桶的相同下标(j)处
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            // rehash　后节点新的位置一定为原来基础上加上　oldCap
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
}

```

**什么时候扩容：**通过HashMap源码可以看到是在put操作时，即向容器中添加元素时，判断当前容器中元素的个数是否达到阈值（当前数组长度乘以加载因子的值）的时候，就要自动扩容了。

 **扩容(resize)：**其实就是重新计算容量；而这个扩容是计算出所需容器的大小之后重新定义一个新的容器，将原来容器中的元素放入其中。

 

 

 

 

 