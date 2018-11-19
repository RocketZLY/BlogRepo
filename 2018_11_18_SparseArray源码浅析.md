# SparseArray源码浅析

### 前言

最近有小伙伴去面试了，在群里分享的面试题有一道是关于`SparseArray`的，本来是不想看的o(╥﹏╥)o，没想到是个面试题，那没办法只能看看了。

### 概述

本文还是跟前面分析`HashMap`、`LruChache`的方式一样分别介绍构造方法、增、删、改、查方法。

这里先概括下`SparseArray`的实现有个初步的认识。

1. 作为存储键值对的容器跟`HashMap`是有很大的不同的，它是通过**两个大小相同的数组分别存储键和值，并且键只能是`int`类型的**。
2. 键和值插入的数组的位置是相同的，是**通过二分查找法得出的插入位置。所以键的数组也是有序的**。
3. 删除的时候并不是直接删除，而是给value添加一个标记，当要插入位置value为删除标记的时候可以重用，直到合适的时候才调用自己实现的`gc()`方法回收垃圾，压缩数组。

### 构造方法

```java
    private static final Object DELETED = new Object();//删除元素用到的删除标记
    private boolean mGarbage = false;//是否需要调用gc()方法压缩数组标记位

    private int[] mKeys;//存储key的数组
    private Object[] mValues;//存储值的数组
    private int mSize;//当前键值对个数
    public SparseArray() {
        this(10);//默认容量10
    }
    public SparseArray(int initialCapacity) {//初始化key和value数组
        if (initialCapacity == 0) {
            mKeys = EmptyArray.INT;
            mValues = EmptyArray.OBJECT;
        } else {
            mValues = ArrayUtils.newUnpaddedObjectArray(initialCapacity);
            mKeys = new int[mValues.length];
        }
        mSize = 0;
    }
```

构造方法比较简单就是初始化键和值的数组，默认容量为10。

### 删

这里先说删因为增的时候会用到删除的标记判断。

```java
    public void remove(int key) {
        delete(key);
    }
    public void delete(int key) {
        int i = ContainerHelpers.binarySearch(mKeys, mSize, key);//通过二分查找法找到key对应的index

        if (i >= 0) {//i>=0代表存在要删除的键值对
            if (mValues[i] != DELETED) {//如果值不为DELETED标记
                mValues[i] = DELETED;//将值置为DELETED标记
                mGarbage = true;//并将回收标记置为true 等待合适时机回收
            }
        }
    }
	//二分查找法
    static int binarySearch(int[] array, int size, int value) {
        int lo = 0;
        int hi = size - 1;

        while (lo <= hi) {
            final int mid = (lo + hi) >>> 1;
            final int midVal = array[mid];

            if (midVal < value) {
                lo = mid + 1;
            } else if (midVal > value) {
                hi = mid - 1;
            } else {
                return mid;  // value found
            }
        }
        return ~lo;  // value not present
    }
```

删除就是将key通过二分查找法找到插入的下标，然后将对应位置的值置为`DELETED`删除标记并且将`mGarbage`回收标记位置为true等待合适时间回收。

**这里重点需要注意的是这个二分查找法，如果在数组中找到key对应的位置则直接返回下标，否则返回`~lo`由于`lo`一定是正数取反则为负数所以如果返回值为负数则代表在数组中未找到key，并且`lo`是数组中大于key的第一个位置在增加新键值对的时候会作为插入位置使用。**

### 增

```java
    public void put(int key, E value) {
        int i = ContainerHelpers.binarySearch(mKeys, mSize, key);//通过二分查找法寻找key的下标

        if (i >= 0) {//有相同的key
            mValues[i] = value;//直接覆盖值
        } else {//没有找到相同的key
            i = ~i;//用前面删除方法说道的二分查找的lo作为要插入的位置

            if (i < mSize && mValues[i] == DELETED) {//如果要插入位置是删除标记则直接重用
                mKeys[i] = key;//覆盖key
                mValues[i] = value;//覆盖value
                return;
            }

            if (mGarbage && mSize >= mKeys.length) {//如果有垃圾需要回收并且元素数量>=数组长度则调用gc方法回收 第二个判断条件是为了不要频繁的调用gc()优化性能因为gc()方法会压缩数组涉及到数组的移动
                gc();

                // Search again because indices may have changed.
                i = ~ContainerHelpers.binarySearch(mKeys, mSize, key);//回收后重新计算下标
            }

            mKeys = GrowingArrayUtils.insert(mKeys, mSize, i, key);//插入key(可能扩容)
            mValues = GrowingArrayUtils.insert(mValues, mSize, i, value);//插入value(可能扩容)
            mSize++;//键值对数+1
        }
    }
```

通过二分查找法找到下标，如果存在相同key的键值对则直接覆盖值，如果不存在则看要插入位置值是否为`DELETED`如果是则直接覆盖key和value，如果不是则根据`mGarbage && mSize >= mKeys.length`条件判断是否要调用`gc()`回收，回收会可能会造成数组移动所以需要重新计算插入下标，然后插入新的键值对到键数组和值数组，并键值对数量+1。

```java
    private void gc() {
        // Log.e("SparseArray", "gc start with " + mSize);

        int n = mSize;//键值对数量
        int o = 0;//上一个值不是DELETED的下标
        int[] keys = mKeys;
        Object[] values = mValues;

        for (int i = 0; i < n; i++) {
            Object val = values[i];

            if (val != DELETED) {
                if (i != o) {//如果当前的i不等于o，则会将i后面所有元素往前移覆盖之前删除标记的数组 压缩数组
                    keys[o] = keys[i];//覆盖键
                    values[o] = val;//覆盖值
                    values[i] = null;//将i指向的值置为null
                }

                o++;
            }
        }

        mGarbage = false;//清理垃圾标记位置为false
        mSize = o;//更新键值对数

        // Log.e("SparseArray", "gc end with " + mSize);
    }
```

回收值数组中的`DELETED`标记的元素，具体实现是发现`DELETED`标记后将后面的元素整体往前移然后将最后的值置为null。

```java
    public static int[] insert(int[] array, int currentSize, int index, int element) {
        assert currentSize <= array.length;

        if (currentSize + 1 <= array.length) {//不需要扩容
            System.arraycopy(array, index, array, index + 1, currentSize - index);//将数组index和后面的元素往后移动一位
            array[index] = element;//在index位置插入element
            return array;
        }
        
        int[] newArray = new int[growSize(currentSize)];//需要扩容创建新数组
        System.arraycopy(array, 0, newArray, 0, index);//将index前的元素复制到新数组
        newArray[index] = element;//在index位置插入新元素
        System.arraycopy(array, index, newArray, index + 1, array.length - index);//将index和后面的元素依次复制到新数组
        return newArray;
    }

    public static int growSize(int currentSize) {//如果size小于4则扩容为8，否则当前容量*2
        return currentSize <= 4 ? 8 : currentSize * 2;
    }
```

插入的话跟`ArrayList`是差不多的，唯一的区别是扩容，如果当前size小于4则变为8，其他情况直接size*2。

### 改

```java
    public void setValueAt(int index, E value) {//根据传入index下标修改value
        if (mGarbage) {//是否需要回收，因为是根据index修改值所以需要排除DELETED标记元素的影响
            gc();
        }

        mValues[index] = value;
    }
```

比较简单没啥可说的，需要注意的是`SparseArray`中凡是根据index操作的方法都会判断是否需要`gc()`一下，以排除`DELETED`标记元素的干扰。

### 查

```java
    public E get(int key) {
        return get(key, null);
    }

    @SuppressWarnings("unchecked")
    public E get(int key, E valueIfKeyNotFound) {
        int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

        if (i < 0 || mValues[i] == DELETED) {//没找到
            return valueIfKeyNotFound;
        } else {//找到了
            return (E) mValues[i];
        }
    }
```

### 其他方法

这里我们看下与index相关的方法，验证下上面所说的

> 需要注意的是`SparseArray`中凡是根据index操作的方法都会判断是否需要`gc()`一下，以排除`DELETED`标记元素的干扰。

```java
    public int keyAt(int index) {
        if (mGarbage) {
            gc();
        }

        return mKeys[index];
    }
    @SuppressWarnings("unchecked")
    public E valueAt(int index) {
        if (mGarbage) {
            gc();
        }

        return (E) mValues[index];
    }
    public int indexOfKey(int key) {
        if (mGarbage) {
            gc();
        }

        return ContainerHelpers.binarySearch(mKeys, mSize, key);
    }
    public int indexOfValue(E value) {
        if (mGarbage) {
            gc();
        }

        for (int i = 0; i < mSize; i++) {
            if (mValues[i] == value) {
                return i;
            }
        }

        return -1;
    }
    public int indexOfValueByValue(E value) {
        if (mGarbage) {
            gc();
        }

        for (int i = 0; i < mSize; i++) {
            if (value == null) {
                if (mValues[i] == null) {
                    return i;
                }
            } else {
                if (value.equals(mValues[i])) {
                    return i;
                }
            }
        }
        return -1;
    }
```

可以看到无一例外都是判断了是否要进行垃圾回收然后在进行其他操作避免`DELETED`标记元素的干扰。

### 最后总结下

与`HashMap`相比

优点：

1. 键是`int[]`避免了装箱拆箱的消耗。
2. 不需要像`HashMap`每一个键值对创建一个`Node`对象存储，减少对象的创建。
3. 扩容时只需要数组扩容不需要重建哈希表。

缺点：

1. 插入的时候需要移动数组，删除后触发`gc()`也会移动数组进行压缩，效率低。
2. 增、删、查都是通过二分查找法找到键对应的下标在进行操作，时间效率低。

适用场景：数据量不大，空间比时间重要，key为int的情况。

对于我们客户端来说一般页面数据不会过千，那么`SparseArray`相对于`HashMap`在查询上不会有太大的区别，但是在内存上有很大的优势，所以综上所述一般情况下(数据量不过千)用`SparseArray`会好些。
