---
     layout:     post   				    
     title:      ArrayList扩容机制				
     subtitle:   grow()方法理解
     date:       2020-05-31	
     author:     Xin 						
     header-img: img/post-grey-bg.jpg 	
     catalog: true 						
     tags:								
         - Java
---

## ArrayList

ArrayList是一个数组，但却结合了List中列表元素可无线增长的特点。所以我认为它的名字起得非常好。首先，ArrayList的构造方法使用了泛型，也就是说，假如想实现一个int的ArrayList：
`ArrayList al = new ArrayList();`或`ArrayList al = new ArrayList<Integet>();`这些属于泛型的概念，不在此提及。

此外，ArrayList对于不设定初始数组长度的对象，会赋予一个默认的长度，`private static final int DEFAULT_CAPACITY = 10;`

## 自动扩容机制：何时扩容？

ArrayList什么时候扩容？读取源码可知，ArrayList在添加一个较长数量的元素时，会调用以下方法：

```java
public void ensureCapacity(int minCapacity) {
    if (minCapacity > elementData.length
        && !(elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA
             && minCapacity <= DEFAULT_CAPACITY)) {
        modCount++;
        grow(minCapacity);
    }
}
```

而在常规情况下，ArrayList会在每次`add()`方法调用时检查是否需要扩容

```java
private void add(E e, Object[] elementData, int s) {
    if (s == elementData.length)
        elementData = grow();
    elementData[s] = e;
    size = s + 1;
}

public boolean add(E e) {
    modCount++;
    add(e, elementData, size);
    return true;
}

public void add(int index, E element) {
    rangeCheckForAdd(index);
    modCount++;
    final int s;
    Object[] elementData;
    if ((s = size) == (elementData = this.elementData).length)
        elementData = grow();
    System.arraycopy(elementData, index,
                     elementData, index + 1,
                     s - index);
    elementData[index] = element;
    size = s + 1;
}

public boolean addAll(int index, Collection<? extends E> c) {
        rangeCheckForAdd(index);

        Object[] a = c.toArray();
        modCount++;
        int numNew = a.length;
        if (numNew == 0)
            return false;
        Object[] elementData;
        final int s;
        if (numNew > (elementData = this.elementData).length - (s = size))
            elementData = grow(s + numNew);

        int numMoved = s - index;
        if (numMoved > 0)
            System.arraycopy(elementData, index,
                             elementData, index + numNew,
                             numMoved);
        System.arraycopy(a, 0, elementData, index, numNew);
        size = s + numNew;
        return true;
    }
```

当代码调用add()后，ArrayList内部会再调用自己的`add(E e, Object[] elementData, int s)`方法，以此来确保当前的长度足够放下这个新的元素，反之则进行扩容。如果我们调用的是`add(int, E)`或者`addAll(Collection<? extends E> c)`，它在内部的实现过程中也还是会进行判断，是否需要扩容。

## 自动扩容机制：怎么扩容？扩容多大？

一般来说，我们认为ArrayList的扩容是1.5倍。但事实上，当原始数组长度不大于4的时候，扩容的大小是“1”个空间。原因来自于以下代码：

```java
    private Object[] grow(int minCapacity) {
        int oldCapacity = elementData.length;
        if (oldCapacity > 0 || elementData != DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            int newCapacity = ArraysSupport.newLength(oldCapacity,
                    minCapacity - oldCapacity, /* minimum growth */
                    oldCapacity >> 1           /* preferred growth */);
            return elementData = Arrays.copyOf(elementData, newCapacity);
        } else {
            return elementData = new Object[Math.max(DEFAULT_CAPACITY, minCapacity)];
        }
    }

private Object[] grow() {
        return grow(size + 1);
    }
```

可以看到，在大部分情况下（也就是数组长度>=4）的时候，`oldCapacity >> 1`的值会大于`minCapacity - oldCapacity = 1`，所以ArrayList选择的新的数组长度为`原始长度+原始长度/2`，换言之就是（约）1.5倍的扩容。

不是完全精准的1.5倍是因为，机器在进行位运算时，如果数值是一个奇数，运算后的结果并不精准。

`77>>1 -> 38`

所以这个位运算更像`Math.floorDiv(int 原始长度, 2)`的结果。

如果我们将数组的长度初始化一个很小的值（如0,1,2,3），在数组扩容1次后，增加的长度也只是1。通过以上的源码和简单的Java Math类的公式可以轻松的验证这一结果。

## 总结

1. ArrayList的初始默认长度为10
2. ArrayList会在每次添加元素时判断是否需要扩容
3. 在数组当前长度小于4的时候，每次扩容的大小为1个长度。
4. 在数组长度大于等于4的时候，每次扩容的大小为`Math.floorDiv(int 原始长度, 2)`，约为1.5倍。

