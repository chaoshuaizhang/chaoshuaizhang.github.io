---
title: 基于JDK7的HashMap分析
date: 2017-09-01 10:41:38
tags: map
categories: java
---

### put方法（put涉及的方法比较多）

#### 核心的属性值

```java
//默认的大小
static final int DEFAULT_INITIAL_CAPACITY = 16;
//最大容量（当超过这个最大容量时，最大容量将会设置为Integer.MAX_VALUE）
static final int MAXIMUM_CAPACITY = 1 << 30;
//默认的加载因子
static final float DEFAULT_LOAD_FACTOR = 0.75f;
```
#### put方法
>1. 插入元素
2. 是否有重复值
3. 命中后的操作（只有个别集合会实现命中后的操作：比如用于实现Lru的linkedlist）

```java
public V put (K key, V value){
    if (key == null)//key=null时
        return putForNullKey(value);
	//通过key的hashCode计算出
    int hash = hash(key.hashCode());
    int i = indexFor(hash, table.length);
    for (Entry<K, V> e = table[i]; e != null; e = e.next) {
        Object k;
        //当前链表中的e的hash == 当前元素的key的hash 并
        且 (key相等  或者  key equals k)，则认为当前元素与链表中的元素相等
        //因为hash值相等，并不代表元素相等
        //从下边也可以想到：get时，如果key equals 或者key hash相等 并且 key相等，则表明是所要get的key
        if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
            V oldValue = e.value;
            e.value = value;
            //hashMap中这个方法为空方法体，但在LinkedHashMap中有实现，具体看link的分析
			//这个方法的意思是：找到目标值
            e.recordAccess(this);
            return oldValue;
        }
    }
    modCount++;
	//当前链表不包含此元素，则添加
    addEntry(hash, key, value, i);
    return null;
}
```

##### hash的计算

>计算hash值---这个算法**如此复杂**是为了使节点更均匀，同时降低碰撞的机会。

```java
static int hash (int h){
    //通过key的hashCode重新计算出一个分布更均匀的值，减少插入时的碰撞
    h ^= (h >>> 20) ^ (h >>> 12);
    return h ^ (h >>> 7) ^ (h >>> 4);
}
```

##### 找到目标值所在的目标位置
>这个方法没有用取模运算来找目标位置，而使用了与运算

```java
/**
 * Returns index for hash code h.

 * 取模运算，得到在table中的索引位置
 */
static int indexFor (int h, int length){

    //其实就是取模运算，和h % length的结果是一样的，只不过下面这种位运算效率更高
    //能用位运算代替模运算的前提是：length必须=2^n，下边的例子可以分析出原因
    return h & (length - 1);
}
```
###### 实例

```java
假设length=16
1. 位运算
 00010011
 00001111 即 length-1=15
=00000011 结果为 3

2. 取模运算：被除数-商*除数=模
 00010011右移5位=1 即 商为 1
 00010011 - 1 * 16 = 3 即 余数为 3
=00000011
```

##### 添加元素
>添加元素时涉及到扩容

```java
void addEntry ( int hash, K key, V value,int bucketIndex){
    Entry<K, V> e = table[bucketIndex];//e 表示下一个值（当前值的next）
    table[bucketIndex] = new Entry<K, V>(hash, key, value, e);
    if (size++ >= threshold)//当数组table中不为空的数量 >= threshold时，进行扩容
        //扩容的实现---数组每次扩大两倍
        resize(2 * table.length);
	createEntry(hash, key, value, bucketIndex);
}
```
>如果size > Integer.MAX_VALUE了怎么办？
>>可以看到，size是int类型，当size = Integer.MAX_VALUE时，size+n都等于-Integer.MAX_VALUE，所以永远走不到扩容的流程中，而是直接添加（注意这里是可以添加成功的，我的理解是：当数据量很大时不需要为其增加扩容机制了，只要来数据就存，前期的扩容不过是为了避免链表太长，查找慢）。

#### 扩容

```java
void resize (int newCapacity){
    Entry[] oldTable = table;
    int oldCapacity = oldTable.length;
	//扩容前的数组大小如果已经达到最大(2^30)了
    if (oldCapacity == MAXIMUM_CAPACITY) {
		//修改threshold值为(2^31-1)，直接返回
        threshold = Integer.MAX_VALUE;
        return;
    }
    Entry[] newTable = new Entry[newCapacity];
	//扩容后，可能会在transfer方法中重新计算每个元素在数组中的位置（照应了那句话：此类不保证映射的顺序，特别是它不保证该顺序恒久不变。）
    transfer(newTable);
    table = newTable;
    threshold = (int) (newCapacity * loadFactor);
}

/**
 * Creates new entry.
 */
Entry( int h, K k, V v, Entry < K, V > n){
    value = v;
    next = n;
    key = k;
    hash = h;
}
```

### get方法

```java
public V get (Object key){
    if (key == null)
        return getForNullKey();
    int hash = hash(key.hashCode());//得到对应的hash
    for (Entry<K, V> e = table[indexFor(hash, table.length)]; e != null; e = e.next) {
        Object k;
        //和put时的逻辑一样。
        if (e.hash == hash && ((k = e.key) == key || key.equals(k)))
            return e.value;
    }
    return null;
}
```

#### equals和hashCode
* 关于hashCode方法，一致的约定是：
1. 重写了euqls方法的对象必须同时重写hashCode()方法。
2. 如果2个对象通过equals调用后返回是true，那么这个2个对象的hashCode方法也必须返回同样的int型散列码。
3. 如果2个对象通过equals返回false，他们的hashCode返回的值允许相同。(然而，程序员必须意识到，hashCode返回独一无二的散列码，会让存储这个对象的hashtables更好地工作。)
* 在上述约定成立的情况下：
1. hashCode相等，元素不一定相等，所以还需要再次调用equals方法。
2. hashCode不等，则元素一定不等()
3. 要同时复写equals方法和hashCode方法，而不要只写其中一个。
4. 两个不同对象的hashCode相同，这种现象称为冲突，冲突会导致操作哈希表的时间开销增大，hashCode()方法目的纯粹用于提高效率，所以尽量定义好的 hashCode()方法，能加快哈希表的操作。 

### 线程安全问题
>HasnMap在多线程情况下可能会造成死循环。还记得上边的transfer方法么

```java
void transfer(Entry[] newTable){
    Entry[] src = table;
    int newCapacity = newTable.length;
    for (int j = 0; j < src.length; j++) {
        Entry<K,V> e = src[j];
        if (e != null) {
            src[j] = null;
            do {
				//【1】
                Entry<K,V> next = e.next;
                //计算老的节点在新的Map中的位置
                int i = indexFor(e.hash, newCapacity);
                e.next = newTable[i];
                newTable[i] = e;
                e = next;
            } while (e != null);
        }
    }
}
```

>注意上边的【1】，当线程1执行到此处时挂起，交由线程2执行，线程2执行直到执行完毕，此时所有的元素都已经放入到新的map中了，并且A和B换了个顺序。
此时如果线程1继续执行则next此时是B，但是B.next却指向了A（这是继续遍历就永远是）。

* 线程 1 执行
![@线程 1 执行](./1564657580030.png)

* 线程 2 执行
![@线程 2 执行](./1564657591945.png)


