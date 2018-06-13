---
layout:     post
title:      "ConcurrentModificationException"
subtitle:   "ConcurrentModificationException出现的原因和解决方案"
date:       2018-06-11 20:30:00
author:     "chchaooo"
header-img: "img/main_banner.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Exception 容器
---

### ConcurrentModificationException

在对集合类进行操作时，有很多情况会出现ConcurrentModificationException。这篇文章我们来分不同的遍历方式和线程场景来讨论ConcurrentModificationException产生的原因和解决的方法。

下面我们先就使用迭代器出现异常来进行讨论，最后再讨论其他的遍历形式。

### Iterator

使用 Iterator 的好处在于可以使用相同方式去遍历集合中元素，而不用考虑集合类的内部实现（只要它实现了 java.lang.Iterable 接口）。比如，如果使用 Iterator 来遍历集合中元素，一旦不再使用 List 转而使用 Set 来组织数据，那遍历元素的代码不用做任何修改。

下面是个简单的迭代器的使用例子


``` java
/**
 * 如果在迭代器遍历的过程中，当前线程或者其他线程直接改变了遍历集合的内容
 * 此时会出现ConcurrentModificationException
 * */
public static void testIterator(){
    List<String> strs = new ArrayList<>();
    for(int i=0; i<10000; i++){
        strs.add(String.valueOf(i));
    }
    Iterator<String> it = strs.iterator();
    while (it.hasNext()){
        String item = it.next();
        System.out.println(item);
        strs.remove(item);
    }
}
```
So easy ？然而，执行的时候我们可以发现，上面的代码会报错：

![](https://cl.ly/3J2R45210l40/Image%202018-06-12%20at%204.52.31%20%E4%B8%8B%E5%8D%88.png)

从异常信息可以发现，异常出现在checkForComodification()方法中。

我们先不忙看checkForComodification()方法的具体实现，先根据程序的代码一步一步看ArrayList源码的实现：

首先看ArrayList的iterator()方法的具体实现，查看源码发现在ArrayList的源码中并没有iterator()这个方法，那么很显然这个方法应该是其父类或者实现的接口中的方法，我们在其父类AbstractList中找到了iterator()方法的具体实现，下面是其实现代码：

``` java
public Iterator<E> iterator() {
    return new Itr();
}
private class Itr implements Iterator<E> {
    /**
     * Index of element to be returned by subsequent call to next.
     */
    int cursor = 0;
    /**
     * Index of element returned by most recent call to next or
     * previous.  Reset to -1 if this element is deleted by a call
     * to remove.
     */
    int lastRet = -1;
    /**
     * The modCount value that the iterator believes that the backing
     * List should have.  If this expectation is violated, the iterator
     * has detected concurrent modification.
     */
    int expectedModCount = modCount;
    
    public boolean hasNext() {
        return cursor != size();
    }
    
    public E next() {
        checkForComodification();
        try {
            int i = cursor;
            E next = get(i);
            lastRet = i;
            cursor = i + 1;
            return next;
        } catch (IndexOutOfBoundsException e) {
            checkForComodification();
            throw new NoSuchElementException();
        }
    }
    
    public void remove() {
        if (lastRet < 0)
            throw new IllegalStateException();
        checkForComodification();
        try {
            AbstractList.this.remove(lastRet);
            if (lastRet < cursor)
                cursor--;
            lastRet = -1;
            expectedModCount = modCount;
        } catch (IndexOutOfBoundsException e) {
            throw new ConcurrentModificationException();
        }
    }
    
    final void checkForComodification() {
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
    }
}
```

首先我们看一下它的几个成员变量：

* cursor：表示下一个要访问的元素的索引
* lastRet：表示上一个访问的元素的索引
* expectedModCount：表示对ArrayList修改次数的期望值，它的初始值为modCount。
* modCount是AbstractList类中的一个成员变量

``` java
protected transient int modCount = 0;
```
该值表示对List的修改次数，查看ArrayList的add()和remove()方法就可以发现，每次调用add()方法或者remove()方法就会对modCount进行加1操作。

当调用list.iterator()返回一个Iterator之后，通过Iterator的hashNext()方法判断是否还有元素未被访问，我们看一下hasNext()方法，hashNext()方法的实现很简单：

```
public boolean hasNext() {
    return cursor != size();
}
```
只有当cursor到达队尾时，才等于size，其他时候，均还有可以访问的值（cursor从0开始递增）。

然后通过Iterator的next()方法获取到下标为0的元素，我们看一下next()方法的具体实现：

``` java
public E next() {
    checkForComodification();
    try {
        int i = cursor;
        E next = get(i);
        lastRet = i;
        cursor = i + 1;
        return next;
    } catch (IndexOutOfBoundsException e) {
        checkForComodification();
        throw new NoSuchElementException();
    }
}
```
这里是非常关键的地方：首先在next()方法中会调用checkForComodification()方法，然后根据cursor的值获取到元素，接着将cursor的值赋给lastRet，并对cursor的值进行加1操作。初始时，cursor为0，lastRet为-1，那么调用一次之后，cursor的值为1，lastRet的值为0。注意此时，modCount为0，expectedModCount也为0。

接着往下看，程序中调用list.remove()方法来删除该元素。

我们看一下在ArrayList中的remove()方法做了什么：

``` java
public boolean remove(Object o) {
    if (o == null) {
        for (int index = 0; index < size; index++)
            if (elementData[index] == null) {
                fastRemove(index);
                return true;
            }
    } else {
        for (int index = 0; index < size; index++)
            if (o.equals(elementData[index])) {
                fastRemove(index);
                return true;
            }
    }
    return false;
}

/*
 * Private remove method that skips bounds checking and does not
 * return the value removed.
 */
private void fastRemove(int index) {
    modCount++;
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,numMoved);
    elementData[--size] = null; // clear to let GC do its work
}
```
在ArrayList的remove操作中，最终对modCount进行加1操作（因为对集合修改了一次），那么注意此时各个变量的值：对于iterator，其expectedModCount为0，cursor的值为1，lastRet的值为0；对于list，其modCount为1，size为0。

执行完删除操作后，继续while循环，调用hasNext方法()判断，由于此时cursor为1，而size为10000，那么返回true，所以继续执行while循环，然后继续调用iterator的next()方法。注意，此时要注意next()方法中的第一句：checkForComodification()。

``` java
final void checkForComodification() {
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
}
```
很显然，此时modCount为1，而expectedModCount为0，因此程序就抛出了ConcurrentModificationException异常。到这里，想必大家应该明白为何上述代码会抛出ConcurrentModificationException异常了。关键点就在于：**调用list.remove()方法导致modCount和expectedModCount的值不一致**。

### 如何解决？

其实很简单，细心的朋友可能发现在Itr类中也给出了一个remove()方法：

``` java
public void remove() {
    if (lastRet < 0)
        throw new IllegalStateException();
    checkForComodification();
    try {
        AbstractList.this.remove(lastRet);
        if (lastRet < cursor)
            cursor--;
        lastRet = -1;
        expectedModCount = modCount;
    } catch (IndexOutOfBoundsException e) {
        throw new ConcurrentModificationException();
    }
}
```
在这个remove方法中，调用AbstractList的remove方法之后，同步也修改了expectedModCount值。

``` java
/**
 * 在迭代器遍历过程中，如果需要变更集合内容，
 * 必须要通过迭代器自身的it.remove()来进行
 * */
public static void testIteratorRemove(){
    List<String> strs = new ArrayList<>();
    for(int i=0; i<10000; i++){
        strs.add(String.valueOf(i));
    }
    Iterator<String> it = strs.iterator();
    while (it.hasNext()){
        String item = it.next();
        System.out.println(item);
        it.remove();
    }
}
```

### 多线程下如何？

``` java
/**
 * 单个线程使用迭代器遍历时来进行删除没有问题。
 * 
 * 多个线程中分别使用两个迭代器来进行遍历，
 * 当thread2中执行了remove操作之后，thread1会因为异常退出。
 * */
public static void testMultiThreadIteratorDelete(){
    List<String> strs = new ArrayList<>();
    for(int i=0; i<1000; i++){
        strs.add(String.valueOf(i));
    }
    Thread thread1 = new Thread(new Runnable() {
        @Override
        public void run() {
            Iterator<String> it = strs.iterator();
            while (it.hasNext()){
                System.out.println(it.next());
            }
        }
    });
    Thread thread2 = new Thread(new Runnable() {
        @Override
        public void run() {
            Iterator<String> it = strs.iterator();
             while (it.hasNext()){
                System.out.println("delete"+it.next());
                it.remove();
            }
        }
    });
    thread1.start();
    thread2.start();
}
```

多线程执行时，

![](https://cl.ly/3d1w433y0i31/Image%202018-06-12%20at%205.41.09%20%E4%B8%8B%E5%8D%88.png)

注意在strs.iterator()中，每次都new了一个新的Iterator（）后返回，所以这里一共有了3个modCount，ArrayList中的一个，两个迭代器各自有一个。当thread2删除操作会后，它只能同步到ArrayList和thread2中迭代器中的两个值。此时thread1中的值就和ArrayList中的不一致了，因此线程1异常退出。

要解决多线程下的这个问题，

* 使用synchronize来进行操作同步

### Array Index

``` java
/**
 * 直接使用index来进行删除，并没有涉及到迭代器
 * 逆序删除时，可以正常执行完成
 * 顺序删除时，则无法执行完成。循环执行到5000时，数组中只有5000个元素，最大下标4999，数组越界
 * */
public static void testIndex(){
    List<String> strs = new ArrayList<>();
    for(int i=0; i<10000; i++){
        strs.add(String.valueOf(i));
    }
    for(int i=10000-1; i>=0; i--){
        strs.remove(i);
        System.out.println(String.valueOf(i));
    }
//  for(int i=0; i<10000; i++){
//      strs.remove(i);
//      System.out.println(String.valueOf(i));
//  }
} 
```
    
通过index来遍历数组，这种情况在单线程和多线程情况下均不会出现ConcurrentModificationException,但是操作不当会出现IndexOutOfBoundsException.


### ForEach

ForEach是jdk5.0新增加的一个循环结构，可以用来处理集合中的每个元素而不用考虑集合定下标. ForEach中使用的集合必须是一个实现了Iterator接口的集合。

``` java
/**
 * 下面的测试，在删除第一个元素之后，就报错ConcurrentModificationException
 * */
public static void testForeach(){
    List<String> strs = new ArrayList<>();
    for(int i=0; i<10000; i++){
        strs.add(String.valueOf(i));
    }
    for (String item: strs) {
        strs.remove(item);
        System.out.println(String.valueOf(strs.size()));
    }
}
```

![](https://cl.ly/0i3P1S0z2r2Y/Image%202018-06-12%20at%204.13.34%20%E4%B8%8B%E5%8D%88.png)

**java.util.ArrayList$Itr.next(ArrayList.java:859)**
ArrayList中的迭代器中的next方法中抛出的异常。因此出现的原因和解决方案与迭代器相同

### 附录：

* 源码位置：[https://github.com/chchaooo/SyntaxDemo](https://github.com/chchaooo/SyntaxDemo)
* 博客原址：[https://chchaooo.github.io/2018/06/11/ConcurrentModificationException/](https://github.com/chchaooo/SyntaxDemo)

