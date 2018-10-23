# LruCache源码解析

### 前言

本篇将介绍`LruCache`，而`LruCache`是用`LinkedHashMap`实现的，`LinkedHashMap`继承`HashMap`所以没看过`HashMap`的先看下我另外篇博文[HashMap源码解析(JDK8)](https://blog.csdn.net/zly921112/article/details/83015457)再来看本篇。

接下来是正菜`LruCache`不过吃之前我们先看下前菜`LinkedHashMap`，只要`LinkedHashMap`弄明白了`LruCache`也就小菜一碟了，本文的`LinkedHashMap`是基于JDK1.8的。

### 概述

`LinkedHashMap`继承至`HashMap`大部分方法都直接沿用，不同点如下。

1. 相对于`HashMap`的节点**增加了首尾指针形成了一个双向链表**，并在`put()`插入的时候按先后顺序**新添加的节点插入到链表的尾部**，`remove()`删除的时候移除该节点保证链表的顺序。
2. 新增`accessOrder`参数，除了按**插入顺序**维护链表外，还可以按**访问顺序**，将操作过的节点移动到尾部方便实现我们的近期最少使用算法。
3. 内部是**有序的**，迭代的时候按链表顺序从前往后迭代。



`LruCache`内部使用`LinkedHashMap`实现，在构造方法的时候会让我们传入能存储的最大容量`maxSize`，在新增元素的时候会调用`trimToSize()`方法，方法内会将`sizeOf()`方法计算得到的每个元素的大小的总和`size`与构造方法传入的`maxSize`比较，如果大于`maxSize`则删除最旧的元素，即`LinkedHashMap`的头部元素，直到`size`<=`maxSize`。

### 正文

先来`LinkedHashMap`源码，再来`LruCache`的。还是按照构造方法、增、删、改、查、迭代器的顺序介绍。

### LinkedHashMap

首先我们要知道的是`LinkedHashMap`是`HashMap`的子类，大部分方法都是直接沿用`HashMap`的。

 ```java
public class LinkedHashMap<K,V>
    extends HashMap<K,V>
    implements Map<K,V>
 ```

### 构造方法

```java
    final boolean accessOrder;//是否按照访问顺序维护链表顺序 默认是false按照插入顺序维护
    transient LinkedHashMapEntry<K,V> head;//双向链表的头结点
    transient LinkedHashMapEntry<K,V> tail;//双向链表的尾结点
	public LinkedHashMap(int initialCapacity, float loadFactor) {
        super(initialCapacity, loadFactor);
        accessOrder = false;
    }

    public LinkedHashMap(int initialCapacity) {
        super(initialCapacity);
        accessOrder = false;
    }

    public LinkedHashMap() {
        super();
        accessOrder = false;
    }

    public LinkedHashMap(Map<? extends K, ? extends V> m) {
        super();
        accessOrder = false;
        putMapEntries(m, false);
    }

    public LinkedHashMap(int initialCapacity,
                         float loadFactor,
                         boolean accessOrder) {
        super(initialCapacity, loadFactor);
        this.accessOrder = accessOrder;
    }

    static class LinkedHashMapEntry<K,V> extends HashMap.Node<K,V> {//继承HashMap.Node并给每个结点新增before, after指针
        LinkedHashMapEntry<K,V> before, after;
        LinkedHashMapEntry(int hash, K key, V value, Node<K,V> next) {
            super(hash, key, value, next);
        }
    }
```

总结

1. 构造方法和`HashMap`差不多，就多了一个我们可以自己设置`accessOrder`值的。
2. 新增`head`和`tail`字段，方便获取链表的头尾。
3. 继承`HashMap.Node`并给每个结点增加首尾指针，方便顺序的维护。

### 增、改

`LinkedHashMap`并没有自己实现`put()`方法还是沿用父类`HashMap`的，最终会调到`putVal()`方法，这个方法在[HashMap源码解析(JDK8)](https://blog.csdn.net/zly921112/article/details/83015457)已经讲过了，这里我们直接看`LinkedHashMap`中的不同处。

```java
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);//在插入新元素的时候会调用newNode()方法创建新元素
        else {
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);//在插入新元素的时候会调用newNode()方法创建新元素
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);//如果存在相同key的键值对并且替换了旧的值则调用afterNodeAccess()方法
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);//如果插入了新的元素则调用afterNodeInsertion()方法
        return null;
    }
```

接下来看插入新元素会调用的两个方法`newNode()`和`afterNodeInsertion()`。

```java
    Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {//创建新的节点
        LinkedHashMapEntry<K,V> p =
            new LinkedHashMapEntry<K,V>(hash, key, value, e);
        linkNodeLast(p);//将新的节点添加到链表尾部
        return p;
    }

    private void linkNodeLast(LinkedHashMapEntry<K,V> p) {//添加新的节点到链表尾部
        LinkedHashMapEntry<K,V> last = tail;
        tail = p;
        if (last == null)
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
    }

    void afterNodeInsertion(boolean evict) { //插入新节点通知
        LinkedHashMapEntry<K,V> first;
        if (evict && (first = head) != null && removeEldestEntry(first)) {//判断是否要删除旧的节点
            K key = first.key;
            removeNode(hash(key), key, null, false, true);//移除节点
        }
    }

    protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {//默认返回false不会删除旧的元素
        return false;
    }
```

从源码可以看出`newNode()`是创建新节点插入到链表的尾部，`afterNodeInsertion()`是插入新节点的通知方法，里面会通过`evict`标记和`removeEldestEntry()`方法判断是否要移除旧的节点，默认实现是不移除。



再来看覆盖value的通知`afterNodeAccess()`方法

```java
    void afterNodeAccess(Node<K,V> e) { //移动节点到尾部
        LinkedHashMapEntry<K,V> last;
        if (accessOrder && (last = tail) != e) {//如果accessOrder为true并且覆盖的元素不是最后一个
            LinkedHashMapEntry<K,V> p =
                (LinkedHashMapEntry<K,V>)e, b = p.before, a = p.after;
            p.after = null;//将元素e的尾指针置为null
            if (b == null)
                head = a;
            else
                b.after = a;
            if (a != null)
                a.before = b;
            else
                last = b;
            if (last == null)
                head = p;
            else {//将e添加到链表尾部
                p.before = last;
                last.after = p;
            }
            tail = p;//尾指针指向e
            ++modCount;
        }
    }
```

如果我们`accessOrder`为true并且覆盖的节点不是最后一个节点，则将他移动到链表的尾部。

总结下增和改的操作

1. 如果不存在key相同的节点则通过`newNode()`创建新节点插入到链表的尾部，然后调用`afterNodeInsertion()`方法通知有新的节点插入，里面会通过`evict`标记和`removeEldestEntry()`方法判断是否要移除旧的节点，默认实现是不移除。
2. 如果存在相同key的节点，则默认覆盖该节点的值，然后调用`afterNodeAccess()`方法通知有节点被覆盖，如果`accessOrder`为true并且覆盖的节点不是最后一个节点，则将他移动到链表的尾部。

### 删

`LinkedHashMap`删除还是使用的`HashMap`的方法，只不过在删除的时候调用了`afterNodeRemoval()`方法通知，那我们直接看该方法的实现。

```java
    void afterNodeRemoval(Node<K,V> e) {
        LinkedHashMapEntry<K,V> p =
            (LinkedHashMapEntry<K,V>)e, b = p.before, a = p.after;
        p.before = p.after = null;
        if (b == null)
            head = a;
        else
            b.after = a;
        if (a == null)
            tail = b;
        else
            a.before = b;
    }
```

可以看到就是将要删除的节点前后指针置为null，然后把前后元素的指针调整下。

### 查

这个方法是`LinkedHashMap`自己的，不过内部还是调用的`HashMap`的`getNode()`方法获取节点

```java
    public V get(Object key) {
        Node<K,V> e;
        if ((e = getNode(hash(key), key)) == null)//调用父类的getNode()获取节点
            return null;
        if (accessOrder)//如果accessOrder为true
            afterNodeAccess(e);//将访问的元素移动到尾部
        return e.value;
    }
```

可以发现如果`accessOrder`为true则调用`afterNodeAccess()`方法，这个方法在前面增的时候说过会将传入的节点放到链表的尾部。

### 迭代

```java
    public Set<Map.Entry<K,V>> entrySet() {//获取entrySet
        Set<Map.Entry<K,V>> es;
        return (es = entrySet) == null ? (entrySet = new LinkedEntrySet()) : es;
    }

    final class LinkedEntrySet extends AbstractSet<Map.Entry<K,V>> {//entrySet对象
       	...
        public final Iterator<Map.Entry<K,V>> iterator() {
            return new LinkedEntryIterator();
        }
        ...
    }

    final class LinkedEntryIterator extends LinkedHashIterator
        implements Iterator<Map.Entry<K,V>> {//迭代器对象
        public final Map.Entry<K,V> next() { return nextNode(); }
    }
    
    abstract class LinkedHashIterator {
        LinkedHashMapEntry<K,V> next;
        LinkedHashMapEntry<K,V> current;
        int expectedModCount;

        LinkedHashIterator() {
            next = head;//从头部开始迭代
            expectedModCount = modCount;
            current = null;
        }

        public final boolean hasNext() {
            return next != null;
        }

        final LinkedHashMapEntry<K,V> nextNode() {//迭代方法 按链表顺序从前往后迭代
            LinkedHashMapEntry<K,V> e = next;
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            if (e == null)
                throw new NoSuchElementException();
            current = e;
            next = e.after;
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

通过实现可以发现`LinkedHashMap`是有序的，迭代是按链表插入顺序从前往后迭代。



### 总结

这里对`LinkedHashMap`做一个总结

1. `LinkedHashMap`内部维护了一个双向链表，保证了`LinkedHashMap`是一个有序的哈希表
2. 在构造方法的时候可以将`accessOrder`置为true，使`LinkedHashMap`按访问顺序维护链表的顺序
3. 迭代效率较高，直接从链表的头部到尾部依次迭代。



### LruCache

看了`LinkedHashMap`再来看`LruCache`就非常简单了。

### 构造方法

```java
    private int size;//当前lrucache大小 
	private int maxSize;//最大容量
	public LruCache(int maxSize) {
        if (maxSize <= 0) {
            throw new IllegalArgumentException("maxSize <= 0");
        }
        this.maxSize = maxSize;
        this.map = new LinkedHashMap<K, V>(0, 0.75f, true);//使用LinkedHashMap存储元素，并且将accessOrder置为了true
    }
```

在构造方法的时候我们传入该`LruCache`最大的容量`maxSize`，然后初始化`LinkedHashMap`作为容器存储元素并且将accessOrder置为了true。再来看一个计算每个元素size的关键方法`sizeOf()`

```java
    protected int sizeOf(K key, V value) {
        return 1;
    }
```

这个方法需要我们自己实现，用来计算每个添加进`LruCache`的元素的size。并且这个size的单位和构造方法传入的`maxSize`单位要保持一致，因为后面会用这两个值进行比对判断是否要移除旧的元素。

### 增、改

```java
    public final V put(K key, V value) {//添加新的元素
        if (key == null || value == null) {//边界判断
            throw new NullPointerException("key == null || value == null");
        }

        V previous;
        synchronized (this) {
            putCount++;
            size += safeSizeOf(key, value);//计算添加元素大小，添加进size变量
            previous = map.put(key, value);//添加到LinkedHashMap
            if (previous != null) {//如果之前有添加过
                size -= safeSizeOf(key, previous);//减去之前元素的size
            }
        }

        if (previous != null) {
            entryRemoved(false, key, previous, value);//通知有元素移除
        }

        trimToSize(maxSize);//判断是否超出maxSize要移除旧的元素
        return previous;
    }

    private int safeSizeOf(K key, V value) {//计算每个元素的size
        int result = sizeOf(key, value);//实际调用sizeOf获取元素大小
        if (result < 0) {
            throw new IllegalStateException("Negative size: " + key + "=" + value);
        }
        return result;
    }

    public void trimToSize(int maxSize) {//如果超出容量则移除旧的元素
        while (true) {
            K key;
            V value;
            synchronized (this) {
                if (size < 0 || (map.isEmpty() && size != 0)) {
                    throw new IllegalStateException(getClass().getName()
                            + ".sizeOf() is reporting inconsistent results!");
                }

                if (size <= maxSize) {//判断size是否超过maxSize
                    break;
                }

                Map.Entry<K, V> toEvict = map.eldest();//拿到LinkedHashMap最旧的元素即头部
                if (toEvict == null) {
                    break;
                }

                key = toEvict.getKey();
                value = toEvict.getValue();
                map.remove(key);//移除
                size -= safeSizeOf(key, value);//减去移除元素的size
                evictionCount++;
            }

            entryRemoved(true, key, value, null);//通知有元素移除
        }
    }

    public Map.Entry<K, V> eldest() {//LinkedHashMap中最旧的元素head
        return head;
    }

```

增加和修改的元素先通过`sizeOf()`方法计算要加入元素大小添加到`size`字段，然后把元素添加进map，如果之前有key相同的元素则`size`减去之前元素的大小，然后调用`trimToSize()`方法判断`size`是否大于了`maxSize`如果大于了，则删除map中最旧的元素直到`size`小于等于`maxSize`。

### 删

```java
    public final V remove(K key) {
        if (key == null) {
            throw new NullPointerException("key == null");
        }

        V previous;
        synchronized (this) {
            previous = map.remove(key);//移除节点
            if (previous != null) {
                size -= safeSizeOf(key, previous);//减去移除节点的size
            }
        }

        if (previous != null) {
            entryRemoved(false, key, previous, null);//通知有元素移除
        }

        return previous;
    }
```

### 查

```java
    public final V get(K key) {
        if (key == null) {
            throw new NullPointerException("key == null");
        }

        V mapValue;
        synchronized (this) {
            mapValue = map.get(key);
            if (mapValue != null) {
                hitCount++;
                return mapValue;
            }
            missCount++;
        }

        V createdValue = create(key);//获取到的mapValue为null则调用create()方法，create()默认返回的null
        if (createdValue == null) {
            return null;
        }

        synchronized (this) {
            createCount++;
            mapValue = map.put(key, createdValue);//将create()方法生成的值插入map

            if (mapValue != null) {//插入位置原先有值
                map.put(key, mapValue);//那么不插入我们create()方法生成的值
            } else {//原先位置没值
                size += safeSizeOf(key, createdValue);//计算插入值的大小赋值给size
            }
        }

        if (mapValue != null) {//原先位置有值
            entryRemoved(false, key, createdValue, mapValue);//通知createdValue插入失败
            return mapValue;
        } else {//插入了新的值
            trimToSize(maxSize);//判断是否要删除旧的元素
            return createdValue;
        }
    }

    protected V create(K key) {
        return null;
    }
```

一般情况下就是直接通过`LinkedHashMap`的`get()`方法直接查询元素。特殊情况如果没找到key对应的元素并且实现了`create()`方法，判断key对应位置是否有元素，如果没有则插入`create()`方法创建的元素，将元素大小添加到`size`，然后调用`trimToSize(maxSize)`调整判断是否要删除旧的元素。



### 总结

这里对`LruCache`做一个总结

1. 内部通过`LinkedHashMap`实现，`maxSize`和`sizeOf()`方法返回值单位要一致。
2. 每次添加元素的时候通过`sizeOf()`方法计算每个元素的大小，当`size`大于`maxSize`的时候会删除`LinkedHashMap`中最旧的元素即头部，直到`size`<=`maxSize`。





