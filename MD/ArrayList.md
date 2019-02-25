# ArrayList/Vector 的底层分析

## ArrayList

### 要点提炼

- 核心要点：自动扩容机制
  - 无参构造器默认大小 10
  - 添加元素前先进行扩容校验 --> 调用 `ensureCapaciyInternal(int minCapacity)`方法
  - 先尝试扩展为原来的 1.5 倍，如果不满足需求，直接扩展为需求值
  - 最后使用 `System.arrayCopy()` 复制到新数组
- 线程不安全
- 性能提示
  - 按数组下标访问元素的性能很高
  - 直接再数组末尾加入元素的性能也较高
  - 按下标插入到指定位置、删除元素，则需要使用 `System.arrayCopy` 来移动部分受影响的元素，性能就变差
- 值可以为空，因为元素被存入一个`Object[]`数组，并没有没有做校验

### 解析

`ArrayList` 实现于 `List`、`RandomAccess` 接口。可以插入空数据，也支持随机访问。

> 1. `ArrayList`使用无参数构造器时默认大小是 10
> 2. `ArrayList`线程不安全

`ArrayList `相当于动态数据，其中最重要的两个属性分别是：`elementData` 数组，以及 `size` 大小。
在调用 `add()` 方法的时候：

```java
    public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }
```

- 首先进行扩容校验。
- 将插入的值放到尾部，并将 size + 1 。

如果是调用 `add(index,e)` 在指定位置添加的话：
```java
    public void add(int index, E element) {
        rangeCheckForAdd(index);

        ensureCapacityInternal(size + 1);  // Increments modCount!!
        //复制，向后移动
        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);
        elementData[index] = element;
        size++;
    }
```


- 也是首先扩容校验。
- 接着对数据进行复制，目的是把 index 位置空出来放本次插入的数据，并将后面的数据向后移动一个位置。

其实扩容最终调用的代码:
```java
    private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
```

也是一个数组复制的过程。

由此可见 `ArrayList` 的主要消耗是数组扩容以及在指定位置添加数据，在日常使用时最好是指定大小，尽量减少扩容。更要减少在指定位置插入数据的操作。

#### 序列化

由于`ArrayList `是基于动态数组实现的，所以并不是所有的空间都被使用。因此使用了 `transient` 修饰，可以防止被自动序列化。

##### 为什么要使用序列化？

> 1. 交流：两台运行同样代码的主机如果需要交流，那么使用序列化进行传输虽然不是最好的方法，但是确实可以工作
> 2. 持久化：将对象转换为字节数组，然后保存到数据库中
> 3. 深拷贝：如果你想获得一个对象的深拷贝，但是又不想重写`clone()`方法，那么序列化再反序列化是一个好方法
> 4. 缓存：某些时候应用创建对象需要 10min 但是反序列化操作只需要 10s，那么将对象序列化后存入文件会比把这个巨大的对象保留在内存中好多了
> 5. 跨 JVM 同步：序列化可以在运行着不同 JVM 的不同机器上使用
>
> -- [What is the purpose of Serialization in Java?](https://stackoverflow.com/questions/2232759/what-is-the-purpose-of-serialization-in-java) 

> 在 Java 中，我们使用 `transient`来修饰不想被序列化的对象。同时遵循下面特点：
>
> - 一旦变量被`transient`修饰，变量将不再是对象持久化的一部分，该变量内容在序列化后无法获得访问
> - `transient`关键字只能修饰变量，而不能修饰方法和类。静态变量是不能被`transient`关键字修饰的。变量如果是用户自定义类变量，则该类需要实现`Serializable`接口
> - 被`transient`关键字修饰的变量不能被序列化，一个静态变量不管是否被其修饰，均不能被序列化
>
> -- [Java transient关键字使用小记](http://www.cnblogs.com/lanxuezaipiao/p/3369962.html)
>
> -- [Google 搜索：transient 修饰](https://www.google.com/search?q=transient%20修饰) 

```java
transient Object[] elementData;
```

因此 `ArrayList `自定义了序列化与反序列化：

```java
    private void writeObject(java.io.ObjectOutputStream s)
        throws java.io.IOException{
        // Write out element count, and any hidden stuff
        int expectedModCount = modCount;
        s.defaultWriteObject();

        // Write out size as capacity for behavioural compatibility with clone()
        s.writeInt(size);

        // Write out all elements in the proper order.
        //只序列化了被使用的数据
        for (int i=0; i<size; i++) {
            s.writeObject(elementData[i]);
        }

        if (modCount != expectedModCount) {
            throw new ConcurrentModificationException();
        }
    }

    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        elementData = EMPTY_ELEMENTDATA;

        // Read in size, and any hidden stuff
        s.defaultReadObject();

        // Read in capacity
        s.readInt(); // ignored

        if (size > 0) {
            // be like clone(), allocate array based upon size not capacity
            ensureCapacityInternal(size);

            Object[] a = elementData;
            // Read in all elements in the proper order.
            for (int i=0; i<size; i++) {
                a[i] = s.readObject();
            }
        }
    }
```

> 当对象中自定义了 writeObject 和 readObject 方法时，JVM 会调用这两个自定义方法来实现序列化与反序列化。


从实现中可以看出 ArrayList 只序列化了被使用的数据。

## Vector

`Voctor` 也是实现于 `List` 接口，底层数据结构和 `ArrayList` 类似,也是一个动态数组存放数据。不过是在 `add()` 方法的时候使用 `synchronize` 进行同步写数据，但是开销较大，所以 `Vector` 是一个同步容器并不是一个并发容器。

以下是 `add()` 方法：
```java
    public synchronized boolean add(E e) {
        modCount++;
        ensureCapacityHelper(elementCount + 1);
        elementData[elementCount++] = e;
        return true;
    }
```

以及指定位置插入数据:
```java
    public void add(int index, E element) {
        insertElementAt(element, index);
    }
    public synchronized void insertElementAt(E obj, int index) {
        modCount++;
        if (index > elementCount) {
            throw new ArrayIndexOutOfBoundsException(index
                                                     + " > " + elementCount);
        }
        ensureCapacityHelper(elementCount + 1);
        System.arraycopy(elementData, index, elementData, index + 1, elementCount - index);
        elementData[index] = obj;
        elementCount++;
    }
```

- `Vector`自带的方法都是线程安全的，因为使用了`synchronized`关键字修饰
- 但是如果是复合方法，则需要客户端进行加锁

```java
public static Object getLast(Vector list) {
    int lastIndex = list.size() - 1;
    return list.get(lastIndex);
}

public static void deleteLast(Vector list) {
    int lastIndex = list.size() - 1;
    list.remove(lastIndex);
}
```

线程 A 调用`getLast()`方法的同时，线程 B 调用`deleteLast()`方法，若是线程 B 先删除了最后一个元素，线程 A 就会发生数组越界异常。



