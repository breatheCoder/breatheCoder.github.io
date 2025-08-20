# 典型回答


**在ArrayList的内部实现中，用于存储元素的数组（**`**transient Object[] elementData;**`**）被声明为transient，这意味着默认的序列化机制不会序列化这个数组。**



那么，ArrayList是如何实现序列化的呢？



**ArrayList自定义了序列化过程，通过重写writeObject和readObject方法。 **



之所以这么做，ArrayList的数组通常会有一些空的位置（因为容量会预留一些空间），为了节省空间和提高效率，ArrayList没有使用默认的序列化机制，而是自定义了序列化，只序列化实际存储的元素，而不是整个数组。



ArrayList重写了writeObject和readObject方法，如下所示：

```java
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
private void writeObject(java.io.ObjectOutputStream s)
    throws java.io.IOException{
    // Write out element count, and any hidden stuff
    int expectedModCount = modCount;
    s.defaultWriteObject();

    // Write out size as capacity for behavioural compatibility with clone()
    s.writeInt(size);

    // Write out all elements in the proper order.
    for (int i=0; i<size; i++) {
        s.writeObject(elementData[i]);
    }

    if (modCount != expectedModCount) {
        throw new ConcurrentModificationException();
    }
}
```



# 知识扩展
## 为什么底层数组要使用transient
ArrayList实际上是动态数组，每次在放满以后自动增长设定的长度值，如果数组自动增长长度设为100，而实际只放了一个元素，那就会序列化99个null元素。为了保证在序列化的时候不会将这么多null同时进行序列化，ArrayList把元素数组设置为transient。



所以，为了避免Java自带的序列化机制造成的空间浪费，把数组定义为transient，然后重写writeObject和readObject来实现序列化操作。

