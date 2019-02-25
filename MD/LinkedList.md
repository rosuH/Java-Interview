# LinkedList 底层分析

![](https://ws4.sinaimg.cn/large/006tKfTcly1fqzb66c00gj30p7056q38.jpg)

如图所示 `LinkedList` 底层是基于双向链表实现的，也是实现了 `List` 接口，所以也拥有 List 的一些特点(JDK1.7/8 之后取消了循环，修改为双向链表)。

## 要点提炼

- 双向链表实现
- 按下标访问元素需要遍历，速度慢
  - 由下表位置来判断是首遍历还是尾遍历
  - 随机访问性能差
- 插入、删除元素仅需要修改前后结点，比数组快
  - 依然要遍历到该结点位置
- 尽量使用`removeLast()`, `addFirst()`或是`Iterator()`上的`remove()`方法，能节省指针移动

## 新增方法

```java
    public boolean add(E e) {
        linkLast(e);
        return true;
    }
     /**
     * Links e as last element.
     */
    void linkLast(E e) {
        final Node<E> l = last;
        final Node<E> newNode = new Node<>(l, e, null);
        last = newNode;
        if (l == null)
            first = newNode;
        else
            l.next = newNode;
        size++;
        modCount++;
    }
```

可见每次插入都是移动指针，和 ArrayList 的拷贝数组来说效率要高上不少。

## 查询方法

```java
    public E get(int index) {
        checkElementIndex(index);
        return node(index).item;
    }
    
    Node<E> node(int index) {
        // assert isElementIndex(index);

        if (index < (size >> 1)) {
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }
```

通过`index` 离 size 中间距离来决定是用头遍历还是尾遍历。

- `node`函数会以`O(n/2)`的性能去获取一个结点
  - 也就是如果索引过半，那么从尾结点开始遍历

这样的效率是非常低的，特别是当 index 距离 size 的中间位置越近时。

总结：

- LinkedList 插入，删除都是移动指针效率很高。
- 查找需要进行遍历查询，效率较低。