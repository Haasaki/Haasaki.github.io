---
layout: post
#标题配置
title:  Hashmap深入理解
#时间配置
date:   2018-12-10 08:59:00 +0800
#大类配置
categories: java
#小类配置
tag: hashmap
---

* content
{:toc}

 HashMap也是我们使用非常多的集合，它是基于哈希表的 Map 接口的实现，以key-value的形式存在。在HashMap中，key-value总是会当做一个整体来处理，系统会根据hash算法来来计算key-value的存储位置，学过的同学都知道key-value使用起来非常方便，但是对于他的底层原理不是特别清楚。下面就来分析HashMap的原理。

### 一、HashMap的数据结构

java编程语言中，最基本的数据结构就两种。一个是数组，另外一个是模拟指针（引用），所有的数据结构都可以用这两个基本结构来构造的，hashmap也不例外。Hashmap实际上是一个数组和链表的结合体。IDK1.8里面对HaspMap做了一个更新，采用了数组+链表+红黑树，为什么要做更新呢？因为数组加链表，即使哈希函数取得再好，也很难达到元素百分百均匀分布，当 HashMap 中有大量的元素都存放到同一个桶中时，这个桶下有一条长长的链表，这个时候 HashMap 就相当于一个单链表，假如单链表有 n 个元素，遍历的时间复杂度就是 O(n)，完全失去了它的优势。针对这种情况，JDK 1.8 中引入了红黑树（查找时间复杂度为 O(logn)）来优化这个问题,当桶中的元素元素大于八个时就会转化为红黑树。图中的红色小圆点即代表一个key-value。

![图片1.png](https://upload-images.jianshu.io/upload_images/12475647-2d4860f57894bc77.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


新增的红黑树代码如下
    static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {

    TreeNode<K,V> parent;  // red-black tree links

    TreeNode<K,V> left;

    TreeNode<K,V> right;

    TreeNode<K,V> prev;    // needed to unlink next upon deletion

    boolean red;

    }
#### 红黑树这块，JDK1.8更新了一个重要的操作----桶的树形化treeifyBin()

看下代码
    
    //将桶内所有的 链表节点 替换成 红黑树节点

    1finalvoidtreeifyBin(Node[] tab, inthash) {

    2intn, index; Node e;

    3//如果当前哈希表为空，或者哈希表中元素的个数小于 进行树形化的阈值(默认为 64)，就去新建/扩容

    4if(tab == null|| (n = tab.length) < MIN_TREEIFY_CAPACITY)

    5resize();

    6elseif((e = tab[index = (n - 1) & hash]) != null) {

    7//如果哈希表中的元素个数超过了 树形化阈值，进行树形化

    8// e 是哈希表中指定位置桶里的链表节点，从第一个开始

    9TreeNode hd = null, tl = null; //红黑树的头、尾节点

    10do{

    11//新建一个树形节点，内容和当前链表节点 e 一致

    12TreeNode p = replacementTreeNode(e, null);

    13if(tl == null) //确定树头节点

    14hd = p;
 
    15else{

     16p.prev = tl;

    17tl.next = p;

    18}

     19tl = p;

    20} while((e = e.next) != null); 

    21//让桶的第一个元素指向新建的红黑树头结点，以后这个桶里的元素就是红黑树而不是链表了

    22if((tab[index] = hd) != null)
 
    23hd.treeify(tab);

    24}

    25}

    26TreeNode replacementTreeNode(Node p, Node next) {

    27returnnewTreeNode<>(p.hash, p.key, p.value, next);

    28}




上述操作做了这些事:

![image](http://upload-images.jianshu.io/upload_images/12475647-45fec2c47abb6e6c?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 二、HashMap的put操作
get操作可以利用下图解释
![hashMap put方法执行流程图.png](https://upload-images.jianshu.io/upload_images/12475647-ecb52f365544e4d1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
看下源码

     public V put(K key, V value) {
     return putVal(hash(key), key, value, false, true);
           }

    /**
     * Implements Map.put and related methods
     *
     * @param hash hash for key
     * @param key the key
     * @param value the value to put
     * @param onlyIfAbsent if true, don't change existing value
     * @param evict if false, the table is in creation mode.
     * @return previous value, or null if none
     */
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)  //判断table是否为空
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
           //如果节点key存在的话则直接替换值
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
           //判断该链是否为红黑树
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
              //该链为链表的情况
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                          //链表长度大于8转换为红黑树进行处理
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                      // key已经存在直接覆盖value
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        //超过最大容量 就扩容
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
        }



### 三、HashMap的get操作
直接上源码

        public V get(Object key) {
       Node<K,V> e;
       return (e = getNode(hash(key), key)) == null ? null : e.value;
     }
     final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    // table不为空 && table长度大于0 && table索引位置(根据hash值计算出)不为空
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {    
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k)))) 
            return first;	// first的key等于传入的key则返回first对象
        if ((e = first.next) != null) { // 向下遍历
            if (first instanceof TreeNode)  // 判断是否为TreeNode
            	// 如果是红黑树节点，则调用红黑树的查找目标节点方法getTreeNode
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            // 走到这代表节点为链表节点
            do { // 向下 hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;    // 找不到符合的返回空
}

### 四、HashMap的扩容
JDK1.8中的扩容有点难，讲讲JDK1.7中的扩容。HaspMap的默认容量是16个桶。他有一个负载因子，值为0.75，当桶中的键值的个数超过了负载因子和桶个数的乘积(大于等于阈值)就会进行扩容，也就是超过16*0.75=12时就会扩容。那么是怎么进行扩容的呢？通过JDK源码发现，桶的大小变为了原大小的一倍。**扩容(resize)**就是重新计算容量，向HashMap对象里不停的添加元素，而HashMap对象内部的数组无法装载更多的元素时，对象就需要扩大数组的长度，以便能装入更多的元素。当然Java里的数组是无法自动扩容的，方法是使用一个新的数组代替已有的容量小的数组，就像我们用一个小桶装水，如果想装更多的水，就得换大水桶。
