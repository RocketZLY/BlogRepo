# HashMap源码解析

### 前言

这段时间有空，专门填补了下基础，把常用的`ArrayList`、`LinkedList`、`HashMap`、`LinkedHashMap`、`LruCache`源码看了一遍，`List`相对比较简单就不单独介绍了，`Map`准备用两篇的篇幅，分别介绍`HashMap`和（`LruCache`+`LinkedHashMap`），因为`LruCache`是用`LinkedHashMap`实现的所以就和Lru一起介绍了。

### 概述

- `HashMap`是一个用来存储键值对的容器，并且key唯一value可以重复，线程不安全，遍历时无序。
- 底层是通过数组实现称之为**哈希桶**，数组里面装的是**单项链表**。
- 哈希桶的**容量是2的次方**，这样做的目的是为了计算插入位置的时候可以直接用位运算与替代取余操作提高效率 。
- **默认扩容方式为容量 * 2、阈值 * 2**，添加元素时当链表长度>=8时会转换为红黑树提高查找效率，扩容时当红黑树中元素<=6时会转回链表。扩容后元素的下标是根据hash与上旧的容量算出，如果==0则代表在低位下标不变，如果 != 0则代表在高位则为原下标+原容量。
- 从迭代器可以看出**迭代顺序是无序的**，按桶的下标从小到大，链表从前往后迭代。
- key的哈希值并不是仅仅通过`hashCode()`方法返回，还加上了扰动函数使hashcode的高位也能参与插入桶下标的计算减少哈希冲突，因为`hashCode()`方法返回的是Int型的值而Int取值范围是2的32次方与上（我们桶数-1）计算插入下标的方式，默认情况只有低位参与了运算，那么即使`hashCode()`方法返回的值是唯一的但是由于只有低位参与运算大大的增大了碰撞的可能性，所以需要扰动函数处理下让高位也参与进下标的计算来减少哈希碰撞的可能性。

### 正文

接下来将按构造方法、增、删、改、查、迭代的顺序一一讲解，看源码相对会比较枯燥，不过没事我会加上大量的注释帮助理解。接下来开始吧。

### 构造方法

```java
	static final int MAXIMUM_CAPACITY = 1 << 30;//容量最大值
	transient Node<K,V>[] table;//哈希桶
	final float loadFactor;//加载因子 threshold = 哈希桶.length * loadFactor
    int threshold;//阈值 当哈希桶中元素数量超过阈值的时候会触发resize()扩容
    static final float DEFAULT_LOAD_FACTOR = 0.75f;//默认加载因子
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; //默认容量16

	public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)//容量范围判断
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)//容量范围判断
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))//加载因子范围判断
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;//初始化加载因子
        this.threshold = tableSizeFor(initialCapacity);//返回通过tableSizeFor方法处理的容量，这里稍微有点歧义他把容量赋值给了threshold阈值，不过后面他会把这个阈值赋给容量然后重新计算阈值。
    }
	
	//获取新的容量，返回的值为最近接并且>=cap的2的n次方，方便后面用与运算代替取余
    static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }

    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);//调用第一个构造方法默认加载因子0.75
    }

    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }

    public HashMap(Map<? extends K, ? extends V> m) {//传入一个map存到我们新创建的map中
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        putMapEntries(m, false);
    }
```

可以发现上面的构造函数主要功能就是初始化加载因子`loadFactor`和容量，一般情况下加载因子我们使用默认的0.75，接下来看第四个构造方法中的`putMapEntries()`方法

```java
    final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
        int s = m.size();//拿到要添加的map的size
        if (s > 0) {//范围判断
            if (table == null) { // 哈希桶未初始化
                float ft = ((float)s / loadFactor) + 1.0F;//计算容量
                int t = ((ft < (float)MAXIMUM_CAPACITY) ?//容量边界判断
                         (int)ft : MAXIMUM_CAPACITY);
                if (t > threshold)
                    threshold = tableSizeFor(t);//获取最接近的并且>=t的2的n的值作为容量
            }
            else if (s > threshold)//如果size大于threshold扩容
                resize();//扩容
            for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {//for循环将值一一插入
                K key = e.getKey();
                V value = e.getValue();
                putVal(hash(key), key, value, false, evict);//put键值对到map
            }
        }
    }
```

这个方法中又出现了2个新的方法`resize()`扩容和`putVal()`增加，`putVal()`后面会讲，这里我们先看非常重要的扩容方法`resize()`

```java
    final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;//拿到旧的哈希桶
        int oldCap = (oldTab == null) ? 0 : oldTab.length;//旧的容量
        int oldThr = threshold;//旧的阈值
        int newCap, newThr = 0;//新的容量和阈值
        if (oldCap > 0) {//旧的哈希表存在
            if (oldCap >= MAXIMUM_CAPACITY) {//边界判断大于最大值
                threshold = Integer.MAX_VALUE;//阈值改为Integer.MAX_VALUE,容量不变
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)//新的容量为旧容量*2
                newThr = oldThr << 1; // 新的阈值为旧阈值*2
        }
        else if (oldThr > 0) //哈希表未初始化，但是有阈值
            newCap = oldThr;// 这个就是我们前面说过的他在构造方法的时候把容量赋给阈值的情况，这里他把前面计算得到的容量通过oldThr赋值给了新的newCap容量，后面他会重新计算阈值。
        else {//哈希桶未初始化 容量也未初始化
            newCap = DEFAULT_INITIAL_CAPACITY;//默认容量16
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);//默认阈值12
        }
        if (newThr == 0) {//如果前面判断走的else if即newThr为0重新计算阈值
            float ft = (float)newCap * loadFactor;//计算阈值
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);//边界判断
        }
        /**
        * 上面这一大段其实就是计算新的容量和阈值，容量的默认值为16阈值默认值为12，默认扩容方式是*2。
        * 下面的话则是新建一个桶然后把原来的数据装到新桶中
        */
        threshold = newThr;//初始化阈值
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];//创建新的桶
        table = newTab;//初始化桶
        if (oldTab != null) {//旧的桶不为空
            for (int j = 0; j < oldCap; ++j) {//遍历旧的桶
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {//如果桶中元素不为null赋值给e
                    oldTab[j] = null;//去除旧的桶中的引用
                    if (e.next == null)//如果链表中节点没有下一个元素则没发生碰撞
                        newTab[e.hash & (newCap - 1)] = e;//直接把节点的hash与上新的容量-1得出下标装入新桶中
                    else if (e instanceof TreeNode)//如果是树节点则代表此处是红黑树 由于红黑树不是本篇重点这里就略过了
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);//将红黑树中的节点添加到新的桶中
                    else { //该节点是个链表
                        Node<K,V> loHead = null, loTail = null;//低位的头和尾
                        Node<K,V> hiHead = null, hiTail = null;//高位的头和尾
                        Node<K,V> next;
                        do {
                            next = e.next;
                            if ((e.hash & oldCap) == 0) {//hash与上旧的容量如果==0则在低位，否则在高位
                                if (loTail == null)//如果尾部为null
                                    loHead = e;//添加到头部
                                else
                                    loTail.next = e;//尾部下一个为e
                                loTail = e;//尾部为e
                            }
                            else {//位置在高位 完成链表的组装
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);//如果下一个元素不为null
                        if (loTail != null) {//低位链表不为空
                            loTail.next = null;
                            newTab[j] = loHead;//添加到原始下标j
                        }
                        if (hiTail != null) {//高位链表不为空
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;//添加到原始下标j+旧的容量
                        }
                    }
                }
            }
        }
        return newTab;
    }
```

构造方法和扩容方法`resize()`就说完了，简单总结下。

1. 构造方法就是对加载因子`loadFactor`和容量做了初始化，虽然构造方法中容量一开始是`threshold`变量存储的有点奇怪不过后面，他会把`threshold`赋值给`newCap`并重新计算阈值所以没有问题。

2. 扩容方法`resize()`实现分为两步

   1. 计算新的容量和阈值，默认容量16阈值12，然后扩容的方式是*2
   2. 创建新的桶，将原有的元素放到新的桶中，需要注意的是插入新桶的下标是根据哈希值与上旧容量得出，低位的话下标不变，高位的话下标为原下标+原容量得出。



### 增、改

增和改都是同一个方法put这里就一起讲了

```java
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
```

先看下获取哈希值的`hash()`方法

```java
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```

可以看到它是key的哈希值异或了高位的值，这部分`^ (h >>> 16)`就是我们前面提到的扰动函数让高位也参与下标的运算减少哈希冲突的几率。

```java
	static final int TREEIFY_THRESHOLD = 8;//链表转为红黑树的界限
	final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i//声明表变量tab，要插入的位置上的原始元素p，容量n，插入下标i
        if ((tab = table) == null || (n = tab.length) == 0)//如果表为空或者容量为0
            n = (tab = resize()).length;//初始化表
        if ((p = tab[i = (n - 1) & hash]) == null)//要插入位置上没有元素即没发生碰撞
            tab[i] = newNode(hash, key, value, null);//直接插入该位置
        else {//发生了碰撞
            Node<K,V> e; K k;//声明节点变量e代表找到了与要插入元素key一样节点
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))//如果要插入元素key的hash值与该位置上元素相同，并且key相等。
                e = p;//将要插入位置上的原始元素p赋值给e
            else if (p instanceof TreeNode)//如果要插入位置上的原始元素是树节点
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);//找到了key的hash值相同，key也相等的元素赋值给e
            else {//要插入位置上是一个链表
                for (int binCount = 0; ; ++binCount) {//遍历链表
                    if ((e = p.next) == null) {//如果下个元素为null
                        p.next = newNode(hash, key, value, null);//直接插入链表尾部
                        if (binCount >= TREEIFY_THRESHOLD - 1) //如果链表长度大于等于8
                            treeifyBin(tab, hash);//链表转为红黑树
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))//找到了key哈希值相同并且相等的元素就停止遍历
                        break;
                    p = e;
                }
            }
            if (e != null) { //存在key相同的
                V oldValue = e.value;//拿到旧的值
                if (!onlyIfAbsent || oldValue == null)//判断是否允许覆盖已有的键值对，默认可以覆盖
                    e.value = value;//替换value的值
                afterNodeAccess(e);
                return oldValue;//返回旧的值
            }
        }
        ++modCount;//修改数++
        if (++size > threshold)//判断size是否超过阈值
            resize();//扩容
        afterNodeInsertion(evict);
        return null;
    }
```

简单总结下

1. key的哈希值除了通过`hashCode()`方法获取，`^ (h >>> 16)`还异或了高位减少哈希冲突。
2. put元素的时候先判断该位置是否有元素，没有直接插入，有的话即哈希冲突了，那么比较key的哈希值是否相同并且key是否相等，如果相同默认情况会替换value，如果不相同插入链表尾部或者红黑树，如果链表长度大于等于8的话会转为红黑树，添加完成后再判断size是否大于`threshold`阈值，如果大于则扩容。



### 删

```java
    public V remove(Object key) {
        Node<K,V> e;
        return (e = removeNode(hash(key), key, null, false, true)) == null ?
            null : e.value;
    }

    final Node<K,V> removeNode(int hash, Object key, Object value,
                               boolean matchValue, boolean movable) {
        Node<K,V>[] tab; Node<K,V> p; int n, index;//声明变量tab为哈希表，p为要删除下标的元素，n为桶的长度，index为要插入的下标
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (p = tab[index = (n - 1) & hash]) != null) {//如果表不为空，要删除下标位置元素不为空
            Node<K,V> node = null, e; K k; V v;//node为要删除元素
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))//如果哈希相同值也相同
                node = p;
            else if ((e = p.next) != null) {//下一个元素不为空
                if (p instanceof TreeNode)//如果为红黑树
                    node = ((TreeNode<K,V>)p).getTreeNode(hash, key);//找到红黑树中key哈希相同值相同的元素
                else {//为链表
                    do {//遍历链表
                        if (e.hash == hash &&
                            ((k = e.key) == key ||
                             (key != null && key.equals(k)))) {//找到链表中key哈希相同值相同的元素
                            node = e;
                            break;
                        }
                        p = e;
                    } while ((e = e.next) != null);
                }
            }
            if (node != null && (!matchValue || (v = node.value) == value ||
                                 (value != null && value.equals(v)))) {//如果要删除节点不为空默认情况下不需要匹配值
                if (node instanceof TreeNode)//如果是红黑树
                    ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);//移除该节点
                else if (node == p)//如果第一个元素就是要删除的元素
                    tab[index] = node.next;//移除该元素
                else//如果是链表
                    p.next = node.next;//切断指针
                ++modCount;//修改修改数
                --size;//减少size
                afterNodeRemoval(node);
                return node;//返回删除的值
            }
        }
        return null;
    }
```

删相对比较简单就是找到key对应下标的元素，如果存在并且key的哈希值相同key值也相等则移除，返回删除的value。

### 查

```java
    public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {//如果表不为空，下标对应位置元素不为空
            if (first.hash == hash && 
                ((k = first.key) == key || (key != null && key.equals(k))))//第一个就是要找的元素
                return first;//直接返回
            if ((e = first.next) != null) {//节点下一个元素不为空
                if (first instanceof TreeNode)//如果是红黑树
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);//返回找到的节点
                do {//遍历链表返回找到元素
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }
```

### 遍历

遍历是通过`entrySet()`方法获取了键值对的set集合来遍历

```java
    public Set<Map.Entry<K,V>> entrySet() {
        Set<Map.Entry<K,V>> es;
        return (es = entrySet) == null ? (entrySet = new EntrySet()) : es;
    }
```

然后我们直接看到他的迭代器

```java
    final class EntrySet extends AbstractSet<Map.Entry<K,V>> {
        ...
        public final Iterator<Map.Entry<K,V>> iterator() {
            return new EntryIterator();
        }
        ...
    }

    final class EntryIterator extends HashIterator
        implements Iterator<Map.Entry<K,V>> {
        public final Map.Entry<K,V> next() { return nextNode(); }//可以看到next方法就是调用迭代器的nextNode()方法
    }
    
    abstract class HashIterator {//迭代器对象
        Node<K,V> next;        // next entry to return
        Node<K,V> current;     // current entry
        int expectedModCount;  // for fast-fail
        int index;             // current slot

        HashIterator() {
            expectedModCount = modCount;
            Node<K,V>[] t = table;
            current = next = null;
            index = 0;
            if (t != null && size > 0) { //如果桶不为空
                do {} while (index < t.length && (next = t[index++]) == null);//按顺序从小到大查找出桶中第一个不为null的元素赋值给next
            }
        }

        public final boolean hasNext() {//如果next不为空则继续迭代
            return next != null;
        }

        final Node<K,V> nextNode() {
            Node<K,V>[] t;
            Node<K,V> e = next;
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            if (e == null)
                throw new NoSuchElementException();
            if ((next = (current = e).next) == null && (t = table) != null) {//如果表不为空并且next为空，则接着找到下一个不为null的节点
                do {} while (index < t.length && (next = t[index++]) == null);//按顺序从小到大查找出桶中不为null的元素赋值给next
            }
            return e;
        }

        public final void remove() {
            Node<K,V> p = current;
            if (p == null)
                throw new IllegalStateException();
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            current = null;
            K key = p.key;
            removeNode(hash(key), key, null, false, false);
            expectedModCount = modCount;
        }
    }
```

可以发现迭代是从小到大遍历桶中元素，如果节点是个链表则按照从前往后迭代，并且遍历是无序的。

### 总结

- `HashMap`是一个用来存储键值对的容器，并且key唯一value可以重复，线程不安全，遍历时无序。
- 底层是通过数组实现称之为**哈希桶**，数组里面装的是**单项链表**。
- 哈希桶的**容量是2的次方**，这样做的目的是为了计算插入位置的时候可以直接用位运算与替代取余操作提高效率 。
- **默认扩容方式为容量 * 2、阈值 * 2**，添加元素时当链表长度>=8时会转换为红黑树提高查找效率，扩容时当红黑树中元素<=6时会转回链表。扩容后元素的下标是根据hash与上旧的容量算出，如果==0则代表在低位下标不变，如果 != 0则代表在高位则为原下标+原容量。
- 从迭代器可以看出**迭代顺序是无序的**，按桶的下标从小到大，链表从前往后迭代。
- key的哈希值并不是仅仅通过`hashCode()`方法返回，还加上了扰动函数使hashcode的高位也能参与插入桶下标的计算减少哈希冲突，因为`hashCode()`方法返回的是Int型的值而Int取值范围是2的32次方与上（我们桶数-1）计算插入下标的方式，默认情况只有低位参与了运算，那么即使`hashCode()`方法返回的值是唯一的但是由于只有低位参与运算大大的增大了碰撞的可能性，所以需要扰动函数处理下让高位也参与进下标的计算来减少哈希碰撞的可能性。

细心的同学可能会发现这总结就是前面概述的copy，没错我就这么大胆的承认了，不过看过源码后再来看这个总结相信会有更多体会。


