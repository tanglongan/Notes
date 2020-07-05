CopyOnWrite指的是在“写”的时候，不是直接“写”源数据，而是把数据拷贝一份进行修改，而通过悲观锁或乐观锁的方式写回去。

为什么不直接修改数据，而要拷贝一份修改呢？为了在“读”的时候不加锁。

## CopyOnWriteArrayList

和ArrayList一样，CopyOnWriteArrayList的核心数据结构也是一个数组

```java
public class CopyOnWriteArrayList<E> implements List<E>, RandomAccess, Cloneable, java.io.Serializable {
    private transient volatile Object[] array;
    final transient ReentrantLock lock = new ReentrantLock();
}
```

CopyOnWriteArrayList的几个“读操作”相关的方法

```java
final Object[] getArray() {
    return array;
}
public int size() {
    return getArray().length;
}
public boolean contains(Object o) {
    Object[] elements = getArray();
    return indexOf(o, elements, 0, elements.length) >= 0;
}

public int indexOf(Object o) {
    Object[] elements = getArray();
    return indexOf(o, elements, 0, elements.length);
}

private static int indexOf(Object o, Object[] elements,int index, int fence) {
    if (o == null) {
        for (int i = index; i < fence; i++){
            if (elements[i] == null) return i;
        }
	} else {
        for (int i = index; i < fence; i++){
            if (o.equals(elements[i])) return i;
        }
    }
    return -1;
}
```

这些读操作相关的方法都没有使用锁。CopyOnWriteArrayList保证的“线程安全”，就应该是在“写”操作的方法上

```java
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();	//写操作在加锁时进行
    try {
        Object[] elements = getArray();
        int len = elements.length;
        Object[] newElements = Arrays.copyOf(elements, len + 1);//写的时候，先复制创建一个备份数组
        newElements[len] = e;									//在备份数组上写数据
        setArray(newElements);									//把备份数组替换掉旧数组
        return true;
    } finally {
        lock.unlock();
    }
}
```

其他的remove等写操作方法和add类似。

## CopyOnWriteArraySet

CopyOnWriteArraySet就是用CopyOnWriteArrayList实现的一个Set。保证所有元素不重复。其内部就是封装一个CopyOnWriteArrayList

```java
public class CopyOnWriteArraySet<E> extends AbstractSet<E> implements java.io.Serializable {
    private final CopyOnWriteArrayList<E> al;
    
    public CopyOnWriteArraySet() {
        al = new CopyOnWriteArrayList<E>();
    }
    public CopyOnWriteArraySet(Collection<? extends E> c) {
        if (c.getClass() == CopyOnWriteArraySet.class) {
            CopyOnWriteArraySet<E> cc = (CopyOnWriteArraySet<E>)c;
            al = new CopyOnWriteArrayList<E>(cc.al);
        } else {
            al = new CopyOnWriteArrayList<E>();
            al.addAllAbsent(c);
        }
    }
    
    public boolean add(E e) {
        return al.addIfAbsent(e);//添加元素的时候，保证不重复即可
    }
}
```

