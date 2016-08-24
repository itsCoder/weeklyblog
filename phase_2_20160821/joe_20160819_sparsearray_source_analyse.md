# SparseArray的使用及实现原理

> 文章来源：itsCoder 的 [WeeklyBolg](https://github.com/itsCoder/weeklyblog) 项目
>
> itsCoder主页：[http://itscoder.com/](http://itscoder.com/)
>
> 作者：[Joe](http://extremej.itscoder.com/about/)
>
> 审阅者：[allenwu](http://allenwu.itscoder.com/)

### 序言

相信大家都用过`HashMap`用来存放键值对，最近在项目中使用`HashMap`的时候发现，有时候 IDE 会提示我这里的`HashMap`可以用`SparseArray`或者`SparseIntArray`等等来代替。

细心的朋友可能也发现了这个提示，并且会发现并不是所有的`HashMap`都会提示替换。今天就来一探究竟，到底`SparseArray`跟`HaspMap`相比有什么优缺点，又是在什么场景下来使用的呢？

如果你对于`HashMap`的实现原理还不是很了解，推荐你阅读[allen](http://allenwu.itscoder.com/)的这篇[Java 集合框架源码分析系列之 HashMap]()。

### SparseArray 的使用

#### 创建

```java
SparseArray<Student> sparseArray = new SparseArray<>();
SparseArray<Student> sparseArray = new SparseArray<>(capacity);
```

首先来看看如何创建一个`SparseArray`，前文说了`SparseArray`是用来替换`HashMap`的，而`SparseArray`只需要指定一个泛型，似乎说明`key`的类型在`SparseArray`内部已经指定了呢？

`SparseArray`有两个构造方法，一个默认构造方法，一个传入容量。

#### put()

创建完`sparseArray`后，来看看怎么往里面存放数据吧。

```java
sparseArray.put(int key,Student value);
```

噢，原来`SparseArray`存放的键值对中的键是`int`型的数据，为什么呢？后面分析源码的时候再讲。

`put()`就跟`HashMap`的使用方法一样。

#### get()

```java
sparseArray.get(int key);
sparseArray.get(int key,Student valueIfNotFound);
```

#### remove()

```java
sparseArray.remove(int key);
```

#### index

前面几个都跟`HashMap`没有什么太大区别，而这个`index`就是`SparseArray`所特有的属性了，这里为了方便理解先提一嘴，`SparseArray`从名字上看就能猜到跟数组有关系，事实上他底层是两条数组，一组存放`key`，一组存放`value`，知道了这一点应该能猜到`index`的作用了。

`index` — `key`在数组中的位置。`SparseArray`提供了一些跟`index`相关的方法：

```java
sparseArray.indexOfKey(int key);
sparseArray.indexOfValue(T value);
sparseArray.keyAt(int index);
sparseArray.valueAt(int index);
sparseArray.setValueAt(int index);
sparseArray.removeAt(int index);
sparseArray.removeAt(int index,int size);
```

### SparseArray 实现原理

前面简单的介绍了 `SparseArray` 的使用，为了在实际工作中最合理的选用数据结构，深入的了解每种数据结构的实现原理是很有必要的，这样可以更好的理解和比较不同数据结构之间的优缺点，比死记概念要更好，甚至可以根据自己的具体需求去实现最适合需求的数据结构。

话不多说，打开源码来一探究竟。我看源码的习惯，是先看这个类文件的注释，一般能在整体上给个思路。

> SparseArrays map integers to Objects.  Unlike a normal array of Objects,there can be gaps in the indices.  It is intended to be more memory efficient than using a HashMap to map Integers to Objects, both because it avoids auto-boxing keys and its data structure doesn't rely on an extra entry object for each mapping.

这段注释基本解释了该类的作用：**使用`int[]`数组存放`key`，避免了`HashMap`中基本数据类型需要装箱的步骤，其次不使用额外的结构体（Entry)，单个元素的存储成本下降。**

如果你对装箱的概念还不清楚，可以看看小黑屋的这篇文章：[Java中的自动装箱与拆箱](http://droidyue.com/blog/2015/04/07/autoboxing-and-autounboxing-in-java/)。

#### 初始化

`SparseArray`没有继承任何其他的数据结构，实现了`Cloneable`接口。

```java
private int[] mKeys;
private Object[] mValues;
private int mSize;//当前实际存放的数量
public SparseArray() {this(10);}
public SparseArray(int initialCapacity) {
  		//如果容量为0，获取两个长度为0的数组
        if (initialCapacity == 0) {
            mKeys = EmptyArray.INT;
            mValues = EmptyArray.OBJECT;
        } else {
          //创建两个长度相同的数组，一个放key,一个放values
            mValues = ArrayUtils.newUnpaddedObjectArray(initialCapacity);
            mKeys = new int[mValues.length];
        }
        mSize = 0;
    }
```

初始化`SparseArray`只是简单的创建了两个数组。

#### put()

接下来就是往`SparseArray`中存放数据。

```java
public void put(int key, E value) {
  	// 首先通过二分查找去 key 数组中查找要插入的 key，返回索引
    int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

    if (i >= 0) {
      // 如果 i>=0 说明数组中已经有了该key，则直接覆盖原来的值
        mValues[i] = value;
    } else {
      // 取反，这里得到的i应该是最适合该key的插入位置，具体怎么得到的，后面会说
        i = ~i;
		// 如果索引小于当前已经存放的长度，并且这个位置上的值为DELETED(即被标记为删除的值)
        if (i < mSize && mValues[i] == DELETED) {
          // 直接赋值并返回，注意 size 不需要增加
            mKeys[i] = key;
            mValues[i] = value;
            return;
        }
		// 到这一步说明直接赋值失败，检查当前是否被标记待回收且当前存放的长度已经大于或等于了数组长度
        if (mGarbage && mSize >= mKeys.length) {
          	// 回收数组中应该被干掉的值
            gc();
			// 重新再获取一下索引，因为数组发生了变化
            // Search again because indices may have changed.
            i = ~ContainerHelpers.binarySearch(mKeys, mSize, key);
        }
		// 最终在 i 位置上插入键与值，并且size ＋1
        mKeys = GrowingArrayUtils.insert(mKeys, mSize, i, key);
        mValues = GrowingArrayUtils.insert(mValues, mSize, i, value);
        mSize++;
    }
}
```

上面这段代码是一次插入数据的操作，单看的话有些难懂，因为插入跟删除之间有一定的关系，所以要看懂这段代码，还必须搞懂删除的逻辑。在看删除之前，还是先大体梳理一下插入的几个特点：

- **存放`key`的数组是有序的（二分查找的前提条件）**
- **如果冲突，新值直接覆盖原值，并且不会返回原值（`HashMap`会返回原值）**
- **如果当前要插入的 key 的索引上的值为DELETE，直接覆盖**
- **前几步都失败了，检查是否需要`gc()`并且在该索引上插入数据**

插入的逻辑大体上是这四点，理解起来可能还是有些抽象，我们来几张图：

##### 冲突直接覆盖



![冲突直接覆盖原值](http://7xtakx.com1.z0.glb.clouddn.com/sparse_array_put_one.jpg)

上面这个图，插入一个`key=3`的元素，因为在`mKeys`中已经存在了这个值，则直接覆盖。

##### 插入索引上为DELETED

![索引上为DELETED](http://7xtakx.com1.z0.glb.clouddn.com/sparse_array_put_two.jpg)

注意`mKeys`中并没有 3 这个值，但是通过二分查找得出来，目前应该插入的索引位置为 2 ，即`key=4`所在的位置，而当前这个位置上对应的`value`标记为`DELETED`了，所以会直接将该位置上的`key`赋值为 3 ，并且将该位置上的`value`赋值为`put()`传入的对象。

##### 索引上有值，但是应该触发`gc()`

![触发gc()](http://7xtakx.com1.z0.glb.clouddn.com/sparse_array_put_three.jpg)

注意这个图跟前面的几个又一个区别，那就是数组已经满容量了，而且 3 应该插入的位置已经有 4 了，而 5 所指向的值为`DELETED`，**这种情况下，会先去回收`DELETED`,重新调整数组结构，图中的例子则会回收 5 ,然后再重新计算 3 应该插入的位置**

##### 满容且无法`gc()`

![满容且无法触发gc()](http://7xtakx.com1.z0.glb.clouddn.com/sparse_array_put_four.jpg)

这种情况下，就只能对数组进行扩容，然后插入数据。

结合这几个图，插入的流程应该很清晰了，但是`put()`还有几个值得我们探索的点，首先就是二分查找的算法，这是一个很普通的二分算法，注意最后一行代码，当找不到这个值的时候`return ~lo`，实际上到这一步的时候，理论上`lo==mid==hi`。所以这个位置是最适合插入数据的地方。但是为了让能让调用者既知道没有查到值，又知道索引位置，做了一个取反操作，返回一个负数。这样调用处可以首先通过正负来判断命中，之后又可以通过取反获取索引位置。

```java
// This is Arrays.binarySearch(), but doesn't do any argument validation.
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

第二个点就是，插入数据具体是怎么插入的。

```java
mKeys = GrowingArrayUtils.insert(mKeys, mSize, i, key);
mValues = GrowingArrayUtils.insert(mValues, mSize, i, value);
```

```java
public static int[] insert(int[] array, int currentSize, int index, int element) {
    assert currentSize <= array.length;//断言
	// 如果当前的长度加1还是小于数组长度
    if (currentSize + 1 <= array.length) {
      // 复制数组,没有进行扩容
        System.arraycopy(array, index, array, index + 1, currentSize - index);
        array[index] = element;
        return array;
    }
	// 需要扩容，分为两步，首先复制前半部分
    int[] newArray = ArrayUtils.newUnpaddedIntArray(growSize(currentSize));
    System.arraycopy(array, 0, newArray, 0, index);
    // 插入数据
  	newArray[index] = element;
  	// 复制后半部分
    System.arraycopy(array, index, newArray, index + 1, array.length - index);
    return newArray;
}
```

`put()`部分的代码就全部完毕了，接下来先来看看`remove()`是怎么处理的？

#### remove()

```java
public void remove(int key) {
    delete(key);
}

public void delete(int key) {
   	// 找到该 key 的索引
    int i = ContainerHelpers.binarySearch(mKeys, mSize, key);
	// 如果存在，将该索引上的 value 赋值为 DELETED
    if (i >= 0) {
        if (mValues[i] != DELETED) {
            mValues[i] = DELETED;
          	// 标记当前状态为待回收
            mGarbage = true;
        }
    }
}
private static final Object DELETED = new Object();
```

事实上，`SparseArray`在进行`remove()`操作的时候分为两个步骤：

- 删除`value` — 在`remove()`中处理
- 删除`key` —  在`gc()`中处理，注意这里不是系统的 GC，只是`SparseArray` 的一个方法

`remove()`中，将这个`key`指向了`DELETED`，这时候`value`失去了引用，如果没有其它的引用，会在下一次系统内存回收的时候被干掉。来看一张图：

![remove](http://7xtakx.com1.z0.glb.clouddn.com/sparse_array_remove.png)

但是可以看到`key`仍然保存在数组中，并没有马上删除，目的应该是为了保持索引结构，同时不会频繁压缩数组，保证索引查询不会错位，那么`key`什么时候被删除呢？当`SparseArray`的`gc()`被调用时。

> To help with performance, the container includes an optimization when removing keys: instead of compacting its array immediately, it leaves the removed entry marked as deleted. The entry can then be re-used for the same key, or compacted later in a single garbage collection step of all removed entries. This garbage collection will need to be performed at any time the array needs to be grown or the the map size or entry values are retrieved.

#### gc()

```java
private void gc() {
    // Log.e("SparseArray", "gc start with " + mSize);

    int n = mSize;
    int o = 0;
    int[] keys = mKeys;
    Object[] values = mValues;

    for (int i = 0; i < n; i++) {
        Object val = values[i];
		// 当前这个 value 不等于 DELETED
        if (val != DELETED) {
            if (i != o) {
              	// i != o
              	// 将索引 i 处的 key 赋值给 o 处的key
                keys[o] = keys[i];
              	// 同时将值也赋值给 o 处
                values[o] = val;
              	// 最后将 i 处的值置为空
                values[i] = null;
            }
			// o 向后移动一位
            o++;
        }
    }

    mGarbage = false;
    mSize = o;

    // Log.e("SparseArray", "gc end with " + mSize);
}
```

上面的这段代码，直接看可能理解起来也比较困难，主要是理解 `o` 只有在值等于`DELETED`的时候才不会向后移，也就是说，当`i`向后移动一位的时候，`o`还在值为`DELETED`的地方，而这时候因为`i != o`，就会触发第二个判断条件，将`i`位置的元素向前移动到`o`处。来看一张图：

![gc 原理图](http://7xtakx.com1.z0.glb.clouddn.com/sparse_array_gc.png)

如上图所示，在 3 之前，`i`与`o`都是相等的，而到 3 的时候，因为值为`DELETED`，所以只有`i++`，而`o`的值仍然等于 2，**重点来了，到 4 的时候，发现`i!=o`,则会将 4 向前移动到 3，这时候`o++`了，但是因为`o`始终小于`i`一位（这个例子里面），因此后面的元素均会向前移动一位。**

`gc()`的原理了解了，那么在什么情况下会触发`gc()`呢？上面已经知道在添加元素的时候可能会触发`gc()`，除了添加元素，前文提到过一系列跟`index`有关的方法，事实上在调用这些方法的时候，都会试图去触发`gc()`，这样可以返回给调用者一个精确的索引值。

#### get()

```java
public E get(int key, E valueIfKeyNotFound) {
    int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

    if (i < 0 || mValues[i] == DELETED) {
        return valueIfKeyNotFound;
    } else {
        return (E) mValues[i];
    }
}
```

`get()`中的代码就比较简单了，通过二分查找获取到`key`的索引，通过该索引来获取到`value`

### SparseArray 的系列

除了前面分析的`SparseArray`，其实还有其它的一些类似的数据结构，它们总结起来就是用于存放基本数据类型的键值对：

- `SparseIntArray` — int:int
- `SparseBooleanArray`— int:boolean
- `SparseLongArray`— int:long

就不一一列举了，有兴趣的可以一个一个去看看，实现原理都差不太多。

### 总结

了解了`SparseArray`的实现原理，就该来总结一下它与`HashMap`之间来比较的优缺点

优势：

- 避免了基本数据类型的装箱操作
- 不需要额外的结构体，单个元素的存储成本更低
- 数据量小的情况下，随机访问的效率更高

有优点就一定有缺点

- 插入操作需要复制数组，增删效率降低
- 数据量巨大时，复制数组成本巨大，`gc()`成本也巨大
- 数据量巨大时，查询效率也会明显下降

学习完了`SparseArray`，相信你对这个系列的数据结构有了更深的认识，什么时候选择什么样的数据结构，在一定程度上对于程序的运行效率会有那么一些帮助。