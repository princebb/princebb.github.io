---
layout:     post
title:      "CopyOnWriteArrayList踩坑记"
date:       2021-03-29
author:     "Allen"
header-img: "img/code-bg.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Android
---
#### 一、背景

前段时间开发了一款Flutter插件，用于对原生的[Raw Gnss](https://developer.android.com/guide/topics/sensors/gnss)数据进行采集，并且支持高频率的IMU数据写入。设计了一个缓存池，缓存3分钟采集的日志信息，采用了多线程添加数据，每隔一分钟会执行一个定时任务，清理过期数据。为了省事儿，我当时直接使用了CopyOnWriteArrayList缓存字符串，后续使用过程中，发现后台频繁爆出gc回收垃圾的日志，经过排查，定位到了这个并发类上，通过阅读源码，才知道，这个坑原来是自己理解不到位所致，同时对于为什么CopyOnWriteArrayList只适用于读多写少的场景，又有了深层次的理解。

#### 二、含义

从CopyOnWriteArrayList的字面意思可以看到，这是一个**写时复制**的ArrayList，当容器需要被修改的时候，不直接修改当前容器，而是先将当前容器进行 Copy，复制出一个新的容器，然后修改新的容器，完成修改之后，再将原容器的引用指向新的容器。这样就完成了整个修改过程。

因为容器每次修改都是创建新的副本，所以对于旧容器来说，是不可变的，也就是线程安全的，不需要加锁等同步操作。所以我们可以利用CopyOnWriteArrayList的**不变性**进行并发的读取操作。

CopyOnWriteArrayList 的所有修改操作（add，set等）都是通过创建底层数组的新副本来实现的，所以 CopyOnWrite 容器也是一种读写分离的思想体现，读和写使用不同的容器，所以写入时不会阻塞读取操作，读写可以同时进行，只有写写需要进行同步操作，所以如果读取的场景大过于写入的话，使用它就比较适合。

#### 三、源码分析

说了这么多，那我遇到的坑到底是从哪里来的呢?相信已经有同学从我刚才的解释中猜到了一二，但我们还是从源码的角度来直击问题吧。

> 以下源码分析基于Android 11

##### Add

我们先来看添加操作

```java
public void add(int index, E element) {
    synchronized (lock) {
        Object[] elements = getArray();
        int len = elements.length;
        if (index > len || index < 0)
            throw new IndexOutOfBoundsException(outOfBounds(index, len));
        Object[] newElements;
        int numMoved = len - index;
        if (numMoved == 0)
            newElements = Arrays.copyOf(elements, len + 1);
        else {
            newElements = new Object[len + 1];
            System.arraycopy(elements, 0, newElements, 0, index);
            System.arraycopy(elements, index, newElements, index + 1,
                             numMoved);
        }
        newElements[index] = element;
        setArray(newElements);
    }
}
```
首先，使用了sync关键字对代码块进行包裹，将添加元素的逻辑进行了加锁。这样做的好处是后来的add操作，只有等待上次添加完成，释放锁后，才能执行。

getArray()返回了一个用volatile修饰的数组，保证了在所有线程看到的数据的一致性，这个数组也是当前list里的实际元素。

接下来，对index进行了异常的判断，这点也是值得我们学习的地方。对于外部传递的参数，我们需要考虑异常情况的处理。

然后根据len-index来判断是在数组中间还是数组末尾插入元素。此处使用了arraycopy进行数据的拷贝，将插入的空间留出。之后对新数组的index进行值的设置。

最后将新数组的引用指向原来的数组。

##### Remove

我们再来看Remove操作

```java
public E remove(int index) {
    synchronized (lock) {
        Object[] elements = getArray();
        int len = elements.length;
        E oldValue = get(elements, index);
        int numMoved = len - index - 1;
        if (numMoved == 0)
            setArray(Arrays.copyOf(elements, len - 1));
        else {
            Object[] newElements = new Object[len - 1];
            System.arraycopy(elements, 0, newElements, 0, index);
            System.arraycopy(elements, index + 1, newElements, index,
                             numMoved);
            setArray(newElements);
        }
        return oldValue;
    }
}
```

和Add方法类似，唯一不同的地方在于它是对数组进行一个缩减操作。在此就不赘述了。只是简单提一下numMoved要减一的原因是在于数组以0为起始索引，此处len等于elements.length，所以需要减一。

##### 缺点

通过对上面2段代码的分析，可以很明显的看到，在复制、删除操作时，在内存中会同时存在多份内存对象，这样，在list元素较多时，除了会耗费大量的CPU资源外，对内存的开销也不小，这也是我为什么会在后台看到频繁GC的原因。

#### 四、解决方案

末尾说说我是怎么解决开头的问题吧。

因为我需要对队列的头结点进行remove操作，移除过期的数据，并对新来的数据，添加到队尾，所以将CopyOnWriteArrayList替换为了LinkedList，并对添加和删除操作使用了sync同步锁进行修饰。