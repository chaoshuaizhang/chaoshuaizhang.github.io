---
title: 基于JDK8的LinkedHashMap分析
date: 2017-09-08 19:15:49
tags: map
categories: java
---

## 前言

>1. LikedHashMap是基于HashMap的，所以数据结构也是数组 + 双向链表/红黑树，而最近最少使用就是在 双向链表/红黑树上实现的，嗯，应该是这样的！
>2. 链表和红黑树之间的“关系”是：
>3. red 和 black代表的含义是：


####到底是双向循环链表 还是 ***双向链表***？
```java
// link at the end of list
private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
    LinkedHashMap.Entry<K,V> last = tail;
    tail = p;
    if (last == null)
	    //链表中无数据
        head = p;
    else {
	    //链表中有数据：则让插入的p.before->oldTail，oldTail.after->p
	    //所以也可以证明：并不是双向循环链表，只是双向链表
        p.before = last;
        last.after = p;
    }
}
```


#####Node 和 Entry关系
![Alt text](./1535089543109.png)

#####put流程图



注意：上述的TreeNode并不是TreeMap中的节点类型（但是类型属性差不多）
###先挑出几个方法
>accessOrder为true：会把访问过的元素放在链表后面，放置顺序是访问的顺序，最后访问的在后边
accessOrder为flase：则遍历时按插入顺序来遍历
**LruCache默认就是true**
```java
 //把p元素链接到尾部，put时会用到。
 //数据结构就是改变tail的指向
 // link at the end of list
 private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
     LinkedHashMap.Entry<K,V> last = tail;
     tail = p;
     if (last == null)
         head = p;
     else {
	     //改变指向
         p.before = last;
         last.after = p;
     }
 }

//src源数据，dst目标数据-----apply src's links to dst
//把src替换为dst
// apply src's links to dst
private void transferLinks(LinkedHashMap.Entry<K,V> src,
                           LinkedHashMap.Entry<K,V> dst) {
	//得到目标node的before
    LinkedHashMap.Entry<K,V> b = dst.before = src.before;
    //得到目标node的after
    LinkedHashMap.Entry<K,V> a = dst.after = src.after;
    //如果before为空-->说明src就是head头结点
    if (b == null)
        head = dst;
    else
	    //before的指向改变，由src变为dst
        b.after = dst;
    //after的设置同上
    if (a == null)
        tail = dst;
    else
        a.before = dst;
}


//新创建一个节点，并调用linkNodeLast添加到尾部
Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
    LinkedHashMap.Entry<K,V> p = new LinkedHashMap.Entry<K,V>(hash, key, value, e);
    //调用上述的linkNodeLast方法
    linkNodeLast(p);
    return p;
}


//这个方法在LinkedHashMap中做了实现，但是只在HashMap中调用，移除最早放入Map的对象
//evict：是否收回
void afterNodeInsertion(boolean evict) { // possibly remove eldest
    LinkedHashMap.Entry<K,V> first;
    //LinkedHashMap默认removeEldestEntry方法返回为false，也就是说，在map中放入新元素时，默认不删除最老的元素，但是我们可重写removeEldestEntry方法
    if (evict && (first = head) != null && removeEldestEntry(first)) {
        K key = first.key;
        //可以看到移除的是header
        removeNode(hash(key), key, null, false, true);
    }
}

/**
* 看它的注释就行，写的很清楚！
*/
protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
    return false;
}

```

####核心方法
```java
/**
* 核心方法，把最近使用的Node插入到尾部---算法核心就是通过一个temp进行"指针"指向变换
*/
void afterNodeAccess(Node<K,V> e) { // move node to last
	//定义temp，保存tail节点
    LinkedHashMap.Entry<K,V> last;
    //如果accessOrder=true并且队尾不是e（当然队尾是null这种情况是不可能的，否则就进不到这个方法），则执行下边的操作，把e从对中插到队尾
    if (accessOrder && (last = tail) != e) {
	    //定义temp，保存要插入的节点，同时保存e的before和after，便于下边指向转变
        LinkedHashMap.Entry<K,V> p =
            (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
        p.after = null;
        //说明e是头节点
        if (b == null)
	        //既然e是头结点，它被插入队尾后，a自然就成了头结点
            head = a;
        else
	        //e位于队中，则e移到队尾后，b和a自然就连接了
            b.after = a;
        if (a != null)
            a.before = b;
        else
	        //a为空，说明e本身就是尾节点，感觉这块直接就可以结束了，因为本身就在队尾
	        //则把b赋值给last
            last = b;
        //说明e.before = null，那么e就是头结点
        if (last == null)
	        //p（也就是上述的e）设为头结点
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
        tail = p;
        ++modCount;
    }
}
```

###put方法
```java
//put方法调用的是HashMap的put
public V put(K key, V value) {
	//调putVal
    return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
    Node<K,V>[] tab; 
    Node<K,V> p; 
    int n, i;
    //当第一次往map中put时table是null，所以会走这个if，也就是会调用resize()
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    //当所想放的位置为null时，直接放(至于需不需要扩容，到下边再说)
    if ((p = tab[i = (n - 1) & hash]) == null)
	    //创建新节点，当然此时next是null
        tab[i] = newNode(hash, key, value, null);
    else {
	    //场景 1
	    //想放的位置不是空（已经存在一个链表或一棵树了）
        Node<K,V> e;
        K k;
        //在hash相等的情况下，(如果key==) 或者 (key !==但是equals方法相等)
        //两个对象不==，但是equals相等--->说明重写了equals，规定这种对象只有equals就认为是相同的两个key
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            //得到旧的值（这块儿发生在链表或树的第一个元素就和要插入的数据key一样的情况）
            e = p;
        //场景 2
        //是否是红黑树类型，并且放置新的节点
        else if (p instanceof TreeNode)
	        //若存在相同的key的话，返回旧的值
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        //场景 3
        else {
            for (int binCount = 0; ; ++binCount) {
	            //到达尾部，说明链表或树不存在目标key，则插入新元素
                if ((e = p.next) == null) {
	                //构建新节点（就是要插入的节点）
                    p.next = newNode(hash, key, value, null);
                    //The bin count threshold for using a tree rather than list for a bin.
                    //如果当前链表的数据个数大于等于8时，把链表转为红黑树--->JDK8做的优化
                    //解释下：-1 for 1st？？？
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
	                    //转为红黑树
                        treeifyBin(tab, hash);
                    break;
                }
                //走到这里说明e!=null，即没有到达尾部呢
                //如果两个key相同
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        // existing mapping for key，存在相同key
        //上边的场景 1、2 会走到这里
        if (e != null) { 
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            //不管是get还是put，只要hit到相同的key，都会把其放置尾部（前提是accessOrder=true）
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    //是否需要扩容
    if (++size > threshold)
        resize();
    //在插入后要做的操作（LinkedHashMap对其做了实现）
    afterNodeInsertion(evict);
    return null;
}
```

###treeifyBin方法
```java
final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    //tab长度小于MIN_TREEIFY_CAPACITY，很奇怪，为什么这个判断不在上边进入treeifyBin方法之前判断！！！
    //感觉进到treeifyBin里边就是为了链表转为树的，resize什么的为什么也要在这里！！！
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        TreeNode<K,V> hd = null, tl = null;
        //遍历链表，把普通节点转为树节点
        do {
	        //p：替换后的树节点
            TreeNode<K,V> p = replacementTreeNode(e, null);
            //第一次时tl肯定为null
            if (tl == null)
	            //第一次循环时赋值hd = p
                hd = p;
            else {
	            //接下来每次都走这里
                p.prev = tl;
                tl.next = p;
            }
            //每次都赋值 tl 为 新节点 p
            tl = p;
        } while ((e = e.next) != null);
        //e.next为null时跳出循环
        //会出现 hd == null 的情况吗？
        if ((tab[index] = hd) != null)
	        //以hd(头？)开始，把上述生成的各个树节点，链接成一棵树（可以发现上述并没有去设置树的一些特有属性：left、right），下边的treeify就是做这些操作的
            hd.treeify(tab);
    }
}
```

###treeify方法
**先不分析了，大致的作用就是把原链表的节点构造成一个平衡二叉树！平衡、均匀**
```java
/**
 * Forms tree of the nodes linked from this node.
 * @return root of tree
 */
final void treeify(Node<K,V>[] tab) {
    TreeNode<K,V> root = null;
    for (TreeNode<K,V> x = this, next; x != null; x = next) {
        next = (TreeNode<K,V>)x.next;
        x.left = x.right = null;
        if (root == null) {
            x.parent = null;
            x.red = false;
            root = x;
        }
        else {
            K k = x.key;
            int h = x.hash;
            Class<?> kc = null;
            for (TreeNode<K,V> p = root;;) {
                int dir, ph;
                K pk = p.key;
                if ((ph = p.hash) > h)
                    dir = -1;
                else if (ph < h)
                    dir = 1;
                else if ((kc == null &&
                          (kc = comparableClassFor(k)) == null) ||
                         (dir = compareComparables(kc, k, pk)) == 0)
                    dir = tieBreakOrder(k, pk);

                TreeNode<K,V> xp = p;
                if ((p = (dir <= 0) ? p.left : p.right) == null) {
                    x.parent = xp;
                    if (dir <= 0)
                        xp.left = x;
                    else
                        xp.right = x;
                    root = balanceInsertion(root, x);
                    break;
                }
            }
        }
    }
    moveRootToFront(tab, root);
}

```

###resize方法
```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    //老的表的长度，不是元素个数
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
	    //如果达到最大值了
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        //未达到最大值：oldCap<<1 扩大2倍后小于MAXIMUM_CAPACITY 并且 
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            //扩容机制默认是数组长度扩大两倍，至于threshold，扩大后还需要乘以加载因子
            newThr = oldThr << 1; // double threshold
    }
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    else {               
	    //首次put时就走到了这里，设置下newCap, newThr
	    // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        //threshold = 加载因子 * 数组长度
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    //设置新的threshold，数组"使用率"达到threshold时就会扩容
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            //e 保存oldTab[j]
            if ((e = oldTab[j]) != null) {
	            //
                oldTab[j] = null;
                if (e.next == null)
	                //e.next为空，说明oldTab[j]位置的链表的数据只有一个，只有一个的话，就直接把它放到newTab的hash位置即可
	                //把e放入新的Tab中
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
	                //假如是已经是红黑树节点
	                //上边已经设置过新的tab了，那么各个元素所处的位置应该也要变了，这里的操作是把原来树上的节点重新根据hash定位到新的tab中
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order(保持顺序)
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

###split()方法
```java
/**
 * Splits nodes in a tree bin into lower and upper tree bins,
 * or untreeifies if now too small. Called only from resize;
 * see above discussion about split bits and indices.
 *
 * @param map the map
 * @param tab the table for recording bin heads
 * @param index the index of the table being split
 * @param bit the bit of hash to split on
 */
//把一个大树拆分成多个小数(bit是oldTab的容量oldCap)
final void split(MyHashMap<K,V> map, MyHashMap.Node<K,V>[] tab, int index, int bit) {
    //保存this(当前节点)给b
    MyHashMap.TreeNode<K,V> b = this;
    // Relink into lo and hi lists, preserving order
    //小树？
    MyHashMap.TreeNode<K,V> loHead = null, loTail = null;
    //大树？
    MyHashMap.TreeNode<K,V> hiHead = null, hiTail = null;
    int lc = 0, hc = 0;
    for (MyHashMap.TreeNode<K,V> e = b, next; e != null; e = next) {
        //保存“当前位置”的链表的next
        next = (MyHashMap.TreeNode<K,V>)e.next;
        e.next = null;
        if ((e.hash & bit) == 0) {
            if ((e.prev = loTail) == null)
                loHead = e;
            else
                loTail.next = e;
            loTail = e;
            ++lc;
        }
        else {
            if ((e.prev = hiTail) == null)
                hiHead = e;
            else
                hiTail.next = e;
            hiTail = e;
            ++hc;
        }
    }

    if (loHead != null) {
        if (lc <= UNTREEIFY_THRESHOLD)
            //把树转为链表
            tab[index] = loHead.untreeify(map);
        else {
            tab[index] = loHead;
            if (hiHead != null) // (else is already treeified)
	            //把链表转为树
                loHead.treeify(tab);
        }
    }
    if (hiHead != null) {
        if (hc <= UNTREEIFY_THRESHOLD)
            tab[index + bit] = hiHead.untreeify(map);
        else {
            tab[index + bit] = hiHead;
            if (loHead != null)
                hiHead.treeify(tab);
        }
    }
}
```


###get方法
```java
public V get(Object key) {
    Node<K,V> e;
    if ((e = getNode(hash(key), key)) == null)
        return null;
    if (accessOrder)
	    
        afterNodeAccess(e);
    return e.value;
}
```
###JDK和SDK中实现不同
>后期再分析吧