title: JUC集合 ConcurrentHashMap详解
author: yunlongn
tags:
  - Java
categories:
  - Java
date: 2022-03-04 20:14:00
---
# JUC集合: ConcurrentHashMap详解

> JDK1.7之前的ConcurrentHashMap使用分段锁机制实现，JDK1.8则使用数组+链表+红黑树数据结构和CAS原子操作实现ConcurrentHashMap；本文将分别介绍这两种方式的实现方案及其区别。@pdai

- JUC集合: ConcurrentHashMap详解
  - [带着BAT大厂的面试问题去理解](#带着bat大厂的面试问题去理解)
  - [为什么HashTable慢](#为什么hashtable慢)
  - ConcurrentHashMap - JDK 1.7
    - [数据结构](#数据结构)
    - [初始化](#初始化)
    - [put 过程分析](#put-过程分析)
    - [初始化槽: ensureSegment](#初始化槽-ensuresegment)
    - [获取写入锁: scanAndLockForPut](#获取写入锁-scanandlockforput)
    - [扩容: rehash](#扩容-rehash)
    - [get 过程分析](#get-过程分析)
    - [并发问题分析](#并发问题分析)
  - ConcurrentHashMap - JDK 1.8
    - [数据结构](#数据结构-1)
    - [初始化](#初始化-1)
    - [put 过程分析](#put-过程分析-1)
    - [初始化数组: initTable](#初始化数组-inittable)
    - [计数 addCount()](#addCount():计数)
    - [链表转红黑树: treeifyBin](#链表转红黑树-treeifybin)
    - [扩容: tryPresize](#扩容-trypresize)
    - [数据迁移: transfer](#数据迁移-transfer)
    - [get 过程分析](#get-过程分析-1)
  - [对比总结](#对比总结)
  - [参考文章](#参考文章)



<!--more-->

## [¶](#带着bat大厂的面试问题去理解) 带着BAT大厂的面试问题去理解

TIP

请带着这些问题继续后文，会很大程度上帮助你更好的理解相关知识点。@pdai

- 为什么HashTable慢? 它的并发度是什么? 那么ConcurrentHashMap并发度是什么?
- ConcurrentHashMap在JDK1.7和JDK1.8中实现有什么差别? JDK1.8解決了JDK1.7中什么问题
- ConcurrentHashMap JDK1.7实现的原理是什么? 分段锁机制
- ConcurrentHashMap JDK1.8实现的原理是什么? 数组+链表+红黑树，CAS
- ConcurrentHashMap JDK1.7中Segment数(concurrencyLevel)默认值是多少? 为何一旦初始化就不可再扩容?
- ConcurrentHashMap JDK1.7说说其put的机制?
- ConcurrentHashMap JDK1.7是如何扩容的? rehash(注：segment 数组不能扩容，扩容是 segment 数组某个位置内部的数组 HashEntry<K,V>[] 进行扩容)
- ConcurrentHashMap JDK1.8是如何扩容的? tryPresize
- ConcurrentHashMap JDK1.8链表转红黑树的时机是什么? 临界值为什么是8?
- ConcurrentHashMap JDK1.8是如何进行数据迁移的? transfer

## [¶](#为什么hashtable慢) 为什么HashTable慢

Hashtable之所以效率低下主要是因为其实现使用了synchronized关键字对put等操作进行加锁，而synchronized关键字加锁是对整个对象进行加锁，也就是说在进行put等修改Hash表的操作时，锁住了整个Hash表，从而使得其表现的效率低下。

## [¶](#concurrenthashmap---jdk-17) ConcurrentHashMap - JDK 1.7

在JDK1.5~1.7版本，Java使用了分段锁机制实现ConcurrentHashMap.

简而言之，ConcurrentHashMap在对象中保存了一个Segment数组，即将整个Hash表划分为多个分段；而每个Segment元素，即每个分段则类似于一个Hashtable；这样，在执行put操作时首先根据hash算法定位到元素属于哪个Segment，然后对该Segment加锁即可。因此，ConcurrentHashMap在多线程并发编程中可是实现多线程put操作。接下来分析JDK1.7版本中ConcurrentHashMap的实现原理。

### [¶](#数据结构) 数据结构

整个 ConcurrentHashMap 由一个个 Segment 组成，Segment 代表”部分“或”一段“的意思，所以很多地方都会将其描述为分段锁。注意，行文中，我很多地方用了“槽”来代表一个 segment。

简单理解就是，ConcurrentHashMap 是一个 Segment 数组，Segment 通过继承 ReentrantLock 来进行加锁，所以每次需要加锁的操作锁住的是一个 segment，这样只要保证每个 Segment 是线程安全的，也就实现了全局的线程安全。

![](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/20220304080609.png)

`concurrencyLevel`: 并行级别、并发数、Segment 数，怎么翻译不重要，理解它。默认是 16，也就是说 ConcurrentHashMap 有 16 个 Segments，所以理论上，这个时候，最多可以同时支持 16 个线程并发写，只要它们的操作分别分布在不同的 Segment 上。这个值可以在初始化的时候设置为其他值，但是一旦初始化以后，它是不可以扩容的。

再具体到每个 Segment 内部，其实每个 Segment 很像之前介绍的 HashMap，不过它要保证线程安全，所以处理起来要麻烦些。

### [¶](#初始化) 初始化

- `initialCapacity`: 初始容量，这个值指的是整个 ConcurrentHashMap 的初始容量，实际操作的时候需要平均分给每个 Segment。
- `loadFactor`: 负载因子，之前我们说了，Segment 数组不可以扩容，所以这个负载因子是给每个 Segment 内部使用的。

```java
public ConcurrentHashMap(int initialCapacity,
                         float loadFactor, int concurrencyLevel) {
    if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    if (concurrencyLevel > MAX_SEGMENTS)
        concurrencyLevel = MAX_SEGMENTS;
    // Find power-of-two sizes best matching arguments
    int sshift = 0;
    int ssize = 1;
    // 计算并行级别 ssize，因为要保持并行级别是 2 的 n 次方
    while (ssize < concurrencyLevel) {
        ++sshift;
        ssize <<= 1;
    }
    // 我们这里先不要那么烧脑，用默认值，concurrencyLevel 为 16，sshift 为 4
    // 那么计算出 segmentShift 为 28，segmentMask 为 15，后面会用到这两个值
    this.segmentShift = 32 - sshift;
    this.segmentMask = ssize - 1;

    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;

    // initialCapacity 是设置整个 map 初始的大小，
    // 这里根据 initialCapacity 计算 Segment 数组中每个位置可以分到的大小
    // 如 initialCapacity 为 64，那么每个 Segment 或称之为"槽"可以分到 4 个
    int c = initialCapacity / ssize;
    if (c * ssize < initialCapacity)
        ++c;
    // 默认 MIN_SEGMENT_TABLE_CAPACITY 是 2，这个值也是有讲究的，因为这样的话，对于具体的槽上，
    // 插入一个元素不至于扩容，插入第二个的时候才会扩容
    int cap = MIN_SEGMENT_TABLE_CAPACITY; 
    while (cap < c)
        cap <<= 1;

    // 创建 Segment 数组，
    // 并创建数组的第一个元素 segment[0]
    Segment<K,V> s0 =
        new Segment<K,V>(loadFactor, (int)(cap * loadFactor),
                         (HashEntry<K,V>[])new HashEntry[cap]);
    Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize];
    // 往数组写入 segment[0]
    UNSAFE.putOrderedObject(ss, SBASE, s0); // ordered write of segments[0]
    this.segments = ss;
}
  
    
```

初始化完成，我们得到了一个 Segment 数组。

我们就当是用 new ConcurrentHashMap() 无参构造函数进行初始化的，那么初始化完成后:

- Segment 数组长度为 16，不可以扩容
- Segment[i] 的默认大小为 2，负载因子是 0.75，得出初始阈值为 1.5，也就是以后插入第一个元素不会触发扩容，插入第二个会进行第一次扩容
- 这里初始化了 segment[0]，其他位置还是 null，至于为什么要初始化 segment[0]，后面的代码会介绍
- 当前 segmentShift 的值为 32 - 4 = 28，segmentMask 为 16 - 1 = 15，姑且把它们简单翻译为移位数和掩码，这两个值马上就会用到

### [¶](#put-过程分析) put 过程分析

我们先看 put 的主流程，对于其中的一些关键细节操作，后面会进行详细介绍。

```java
public V put(K key, V value) {
    Segment<K,V> s;
    if (value == null)
        throw new NullPointerException();
    // 1. 计算 key 的 hash 值
    int hash = hash(key);
    // 2. 根据 hash 值找到 Segment 数组中的位置 j
    //    hash 是 32 位，无符号右移 segmentShift(28) 位，剩下高 4 位，
    //    然后和 segmentMask(15) 做一次与操作，也就是说 j 是 hash 值的高 4 位，也就是槽的数组下标
    int j = (hash >>> segmentShift) & segmentMask;
    // 刚刚说了，初始化的时候初始化了 segment[0]，但是其他位置还是 null，
    // ensureSegment(j) 对 segment[j] 进行初始化
    if ((s = (Segment<K,V>)UNSAFE.getObject          // nonvolatile; recheck
         (segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
        s = ensureSegment(j);
    // 3. 插入新值到 槽 s 中
    return s.put(key, hash, value, false);
}
  
        @pdai: 代码已经复制到剪贴板
    
```

第一层皮很简单，根据 hash 值很快就能找到相应的 Segment，之后就是 Segment 内部的 put 操作了。

Segment 内部是由 `数组+链表` 组成的。

```java
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
    // 在往该 segment 写入前，需要先获取该 segment 的独占锁
    //    先看主流程，后面还会具体介绍这部分内容
    HashEntry<K,V> node = tryLock() ? null :
        scanAndLockForPut(key, hash, value);
    V oldValue;
    try {
        // 这个是 segment 内部的数组
        HashEntry<K,V>[] tab = table;
        // 再利用 hash 值，求应该放置的数组下标
        int index = (tab.length - 1) & hash;
        // first 是数组该位置处的链表的表头
        HashEntry<K,V> first = entryAt(tab, index);

        // 下面这串 for 循环虽然很长，不过也很好理解，想想该位置没有任何元素和已经存在一个链表这两种情况
        for (HashEntry<K,V> e = first;;) {
            if (e != null) {
                K k;
                if ((k = e.key) == key ||
                    (e.hash == hash && key.equals(k))) {
                    oldValue = e.value;
                    if (!onlyIfAbsent) {
                        // 覆盖旧值
                        e.value = value;
                        ++modCount;
                    }
                    break;
                }
                // 继续顺着链表走
                e = e.next;
            }
            else {
                // node 到底是不是 null，这个要看获取锁的过程，不过和这里都没有关系。
                // 如果不为 null，那就直接将它设置为链表表头；如果是null，初始化并设置为链表表头。
                if (node != null)
                    node.setNext(first);
                else
                    node = new HashEntry<K,V>(hash, key, value, first);

                int c = count + 1;
                // 如果超过了该 segment 的阈值，这个 segment 需要扩容
                if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                    rehash(node); // 扩容后面也会具体分析
                else
                    // 没有达到阈值，将 node 放到数组 tab 的 index 位置，
                    // 其实就是将新的节点设置成原链表的表头
                    setEntryAt(tab, index, node);
                ++modCount;
                count = c;
                oldValue = null;
                break;
            }
        }
    } finally {
        // 解锁
        unlock();
    }
    return oldValue;
}
  
        @pdai: 代码已经复制到剪贴板
    
```

整体流程还是比较简单的，由于有独占锁的保护，所以 segment 内部的操作并不复杂。至于这里面的并发问题，我们稍后再进行介绍。

到这里 put 操作就结束了，接下来，我们说一说其中几步关键的操作。

### [¶](#初始化槽-ensuresegment) 初始化槽: ensureSegment

ConcurrentHashMap 初始化的时候会初始化第一个槽 segment[0]，对于其他槽来说，在插入第一个值的时候进行初始化。

这里需要考虑并发，因为很可能会有多个线程同时进来初始化同一个槽 segment[k]，不过只要有一个成功了就可以。

```java
private Segment<K,V> ensureSegment(int k) {
    final Segment<K,V>[] ss = this.segments;
    long u = (k << SSHIFT) + SBASE; // raw offset
    Segment<K,V> seg;
    if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u)) == null) {
        // 这里看到为什么之前要初始化 segment[0] 了，
        // 使用当前 segment[0] 处的数组长度和负载因子来初始化 segment[k]
        // 为什么要用“当前”，因为 segment[0] 可能早就扩容过了
        Segment<K,V> proto = ss[0];
        int cap = proto.table.length;
        float lf = proto.loadFactor;
        int threshold = (int)(cap * lf);

        // 初始化 segment[k] 内部的数组
        HashEntry<K,V>[] tab = (HashEntry<K,V>[])new HashEntry[cap];
        if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
            == null) { // 再次检查一遍该槽是否被其他线程初始化了。

            Segment<K,V> s = new Segment<K,V>(lf, threshold, tab);
            // 使用 while 循环，内部用 CAS，当前线程成功设值或其他线程成功设值后，退出
            while ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
                   == null) {
                if (UNSAFE.compareAndSwapObject(ss, u, null, seg = s))
                    break;
            }
        }
    }
    return seg;
}

    
```

总的来说，ensureSegment(int k) 比较简单，对于并发操作使用 CAS 进行控制。

### [¶](#获取写入锁-scanandlockforput) 获取写入锁: scanAndLockForPut

前面我们看到，在往某个 segment 中 put 的时候，首先会调用 node = tryLock() ? null : scanAndLockForPut(key, hash, value)，也就是说先进行一次 tryLock() 快速获取该 segment 的独占锁，如果失败，那么进入到 scanAndLockForPut 这个方法来获取锁。

下面我们来具体分析这个方法中是怎么控制加锁的。

```java
private HashEntry<K,V> scanAndLockForPut(K key, int hash, V value) {
    HashEntry<K,V> first = entryForHash(this, hash);
    HashEntry<K,V> e = first;
    HashEntry<K,V> node = null;
    int retries = -1; // negative while locating node

    // 循环获取锁
    while (!tryLock()) {
        HashEntry<K,V> f; // to recheck first below
        if (retries < 0) {
            if (e == null) {
                if (node == null) // speculatively create node
                    // 进到这里说明数组该位置的链表是空的，没有任何元素
                    // 当然，进到这里的另一个原因是 tryLock() 失败，所以该槽存在并发，不一定是该位置
                    node = new HashEntry<K,V>(hash, key, value, null);
                retries = 0;
            }
            else if (key.equals(e.key))
                retries = 0;
            else
                // 顺着链表往下走
                e = e.next;
        }
        // 重试次数如果超过 MAX_SCAN_RETRIES(单核1多核64)，那么不抢了，进入到阻塞队列等待锁
        //    lock() 是阻塞方法，直到获取锁后返回
        else if (++retries > MAX_SCAN_RETRIES) {
            lock();
            break;
        }
        else if ((retries & 1) == 0 &&
                 // 这个时候是有大问题了，那就是有新的元素进到了链表，成为了新的表头
                 //     所以这边的策略是，相当于重新走一遍这个 scanAndLockForPut 方法
                 (f = entryForHash(this, hash)) != first) {
            e = first = f; // re-traverse if entry changed
            retries = -1;
        }
    }
    return node;
}
  
        @pdai: 代码已经复制到剪贴板
    
```

这个方法有两个出口，一个是 tryLock() 成功了，循环终止，另一个就是重试次数超过了 MAX_SCAN_RETRIES，进到 lock() 方法，此方法会阻塞等待，直到成功拿到独占锁。

这个方法就是看似复杂，但是其实就是做了一件事，那就是获取该 segment 的独占锁，如果需要的话顺便实例化了一下 node。

### [¶](#扩容-rehash) 扩容: rehash

重复一下，segment 数组不能扩容，扩容是 segment 数组某个位置内部的数组 HashEntry<K,V>[] 进行扩容，扩容后，容量为原来的 2 倍。

首先，我们要回顾一下触发扩容的地方，put 的时候，如果判断该值的插入会导致该 segment 的元素个数超过阈值，那么先进行扩容，再插值，读者这个时候可以回去 put 方法看一眼。

该方法不需要考虑并发，因为到这里的时候，是持有该 segment 的独占锁的。

```java
// 方法参数上的 node 是这次扩容后，需要添加到新的数组中的数据。
private void rehash(HashEntry<K,V> node) {
    HashEntry<K,V>[] oldTable = table;
    int oldCapacity = oldTable.length;
    // 2 倍
    int newCapacity = oldCapacity << 1;
    threshold = (int)(newCapacity * loadFactor);
    // 创建新数组
    HashEntry<K,V>[] newTable =
        (HashEntry<K,V>[]) new HashEntry[newCapacity];
    // 新的掩码，如从 16 扩容到 32，那么 sizeMask 为 31，对应二进制 ‘000...00011111’
    int sizeMask = newCapacity - 1;

    // 遍历原数组，老套路，将原数组位置 i 处的链表拆分到 新数组位置 i 和 i+oldCap 两个位置
    for (int i = 0; i < oldCapacity ; i++) {
        // e 是链表的第一个元素
        HashEntry<K,V> e = oldTable[i];
        if (e != null) {
            HashEntry<K,V> next = e.next;
            // 计算应该放置在新数组中的位置，
            // 假设原数组长度为 16，e 在 oldTable[3] 处，那么 idx 只可能是 3 或者是 3 + 16 = 19
            int idx = e.hash & sizeMask;
            if (next == null)   // 该位置处只有一个元素，那比较好办
                newTable[idx] = e;
            else { // Reuse consecutive sequence at same slot
                // e 是链表表头
                HashEntry<K,V> lastRun = e;
                // idx 是当前链表的头节点 e 的新位置
                int lastIdx = idx;

                // 下面这个 for 循环会找到一个 lastRun 节点，这个节点之后的所有元素是将要放到一起的
                for (HashEntry<K,V> last = next;
                     last != null;
                     last = last.next) {
                    int k = last.hash & sizeMask;
                    if (k != lastIdx) {
                        lastIdx = k;
                        lastRun = last;
                    }
                }
                // 将 lastRun 及其之后的所有节点组成的这个链表放到 lastIdx 这个位置
                newTable[lastIdx] = lastRun;
                // 下面的操作是处理 lastRun 之前的节点，
                //    这些节点可能分配在另一个链表中，也可能分配到上面的那个链表中
                for (HashEntry<K,V> p = e; p != lastRun; p = p.next) {
                    V v = p.value;
                    int h = p.hash;
                    int k = h & sizeMask;
                    HashEntry<K,V> n = newTable[k];
                    newTable[k] = new HashEntry<K,V>(h, p.key, v, n);
                }
            }
        }
    }
    // 将新来的 node 放到新数组中刚刚的 两个链表之一 的 头部
    int nodeIndex = node.hash & sizeMask; // add the new node
    node.setNext(newTable[nodeIndex]);
    newTable[nodeIndex] = node;
    table = newTable;
}
  
        @pdai: 代码已经复制到剪贴板
    
```

这里的扩容比之前的 HashMap 要复杂一些，代码难懂一点。上面有两个挨着的 for 循环，第一个 for 有什么用呢?

仔细一看发现，如果没有第一个 for 循环，也是可以工作的，但是，这个 for 循环下来，如果 lastRun 的后面还有比较多的节点，那么这次就是值得的。因为我们只需要克隆 lastRun 前面的节点，后面的一串节点跟着 lastRun 走就是了，不需要做任何操作。

我觉得 Doug Lea 的这个想法也是挺有意思的，不过比较坏的情况就是每次 lastRun 都是链表的最后一个元素或者很靠后的元素，那么这次遍历就有点浪费了。不过 Doug Lea 也说了，根据统计，如果使用默认的阈值，大约只有 1/6 的节点需要克隆。

### [¶](#get-过程分析) get 过程分析

相对于 put 来说，get 就很简单了。

- 计算 hash 值，找到 segment 数组中的具体位置，或我们前面用的“槽”
- 槽中也是一个数组，根据 hash 找到数组中具体的位置
- 到这里是链表了，顺着链表进行查找即可

```java
public V get(Object key) {
    Segment<K,V> s; // manually integrate access methods to reduce overhead
    HashEntry<K,V>[] tab;
    // 1. hash 值
    int h = hash(key);
    long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
    // 2. 根据 hash 找到对应的 segment
    if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
        (tab = s.table) != null) {
        // 3. 找到segment 内部数组相应位置的链表，遍历
        for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile
                 (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
             e != null; e = e.next) {
            K k;
            if ((k = e.key) == key || (e.hash == h && key.equals(k)))
                return e.value;
        }
    }
    return null;
}
  
        @pdai: 代码已经复制到剪贴板
    
```

### [¶](#并发问题分析) 并发问题分析

现在我们已经说完了 put 过程和 get 过程，我们可以看到 get 过程中是没有加锁的，那自然我们就需要去考虑并发问题。

添加节点的操作 put 和删除节点的操作 remove 都是要加 segment 上的独占锁的，所以它们之间自然不会有问题，我们需要考虑的问题就是 get 的时候在同一个 segment 中发生了 put 或 remove 操作。

- put 操作的线程安全性。
  - 初始化槽，这个我们之前就说过了，使用了 CAS 来初始化 Segment 中的数组。
  - 添加节点到链表的操作是插入到表头的，所以，如果这个时候 get 操作在链表遍历的过程已经到了中间，是不会影响的。当然，另一个并发问题就是 get 操作在 put 之后，需要保证刚刚插入表头的节点被读取，这个依赖于 setEntryAt 方法中使用的 UNSAFE.putOrderedObject。
  - 扩容。扩容是新创建了数组，然后进行迁移数据，最后面将 newTable 设置给属性 table。所以，如果 get 操作此时也在进行，那么也没关系，如果 get 先行，那么就是在旧的 table 上做查询操作；而 put 先行，那么 put 操作的可见性保证就是 table 使用了 volatile 关键字。
- remove 操作的线程安全性。
  - remove 操作我们没有分析源码，所以这里说的读者感兴趣的话还是需要到源码中去求实一下的。
  - get 操作需要遍历链表，但是 remove 操作会"破坏"链表。
  - 如果 remove 破坏的节点 get 操作已经过去了，那么这里不存在任何问题。
  - 如果 remove 先破坏了一个节点，分两种情况考虑。 1、如果此节点是头节点，那么需要将头节点的 next 设置为数组该位置的元素，table 虽然使用了 volatile 修饰，但是 volatile 并不能提供数组内部操作的可见性保证，所以源码中使用了 UNSAFE 来操作数组，请看方法 setEntryAt。2、如果要删除的节点不是头节点，它会将要删除节点的后继节点接到前驱节点中，这里的并发保证就是 next 属性是 volatile 的。

## [¶](#concurrenthashmap---jdk-18) ConcurrentHashMap - JDK 1.8

在JDK1.7之前，ConcurrentHashMap是通过分段锁机制来实现的，所以其最大并发度受Segment的个数限制。因此，在JDK1.8中，ConcurrentHashMap的实现原理摒弃了这种设计，而是选择了与HashMap类似的数组+链表+红黑树的方式实现，而加锁则采用CAS和synchronized实现。

### [¶](#数据结构-2) 数据结构

![img](https://pdai-1257820000.cos.ap-beijing.myqcloud.com/pdai.tech/public/_images/thread/java-thread-x-concurrent-hashmap-2.png)

结构上和 Java8 的 HashMap 基本上一样，不过它要保证线程安全性，所以在源码上确实要复杂一些。

### [¶](#初始化-2) 初始化

```java
// 这构造函数里，什么都不干
public ConcurrentHashMap() {
}
public ConcurrentHashMap(int initialCapacity) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException();
    int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
               MAXIMUM_CAPACITY :
               tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
    this.sizeCtl = cap;
}
  
        @pdai: 代码已经复制到剪贴板
    
```

这个初始化方法有点意思，通过提供初始容量，计算了 sizeCtl，sizeCtl = 【 (1.5 * initialCapacity + 1)，然后向上取最近的 2 的 n 次方】。如 initialCapacity 为 10，那么得到 sizeCtl 为 16，如果 initialCapacity 为 11，得到 sizeCtl 为 32。

sizeCtl 这个属性使用的场景很多，不过只要跟着文章的思路来，就不会被它搞晕了。

### [¶](#put-过程分析-2) put 过程分析

仔细地一行一行代码看下去:

```java
public V put(K key, V value) {
    return putVal(key, value, false);
}
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    // 得到 hash 值
    int hash = spread(key.hashCode());
    // 用于记录相应链表的长度
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        // 如果数组"空"，进行数组初始化
        if (tab == null || (n = tab.length) == 0)
            // 初始化数组，后面会详细介绍
            tab = initTable();

        // 找该 hash 值对应的数组下标，得到第一个节点 f
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            // 如果数组该位置为空，
            //    用一次 CAS 操作将这个新值放入其中即可，这个 put 操作差不多就结束了，可以拉到最后面了
            //          如果 CAS 失败，那就是有并发操作，进到下一个循环就好了
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        // hash 居然可以等于 MOVED，这个需要到后面才能看明白，不过从名字上也能猜到，肯定是因为在扩容
        else if ((fh = f.hash) == MOVED)
            // 帮助数据迁移，这个等到看完数据迁移部分的介绍后，再理解这个就很简单了
            tab = helpTransfer(tab, f);

        else { // 到这里就是说，f 是该位置的头节点，而且不为空

            V oldVal = null;
            // 获取数组该位置的头节点的监视器锁
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) { // 头节点的 hash 值大于 0，说明是链表
                        // 用于累加，记录链表的长度
                        binCount = 1;
                        // 遍历链表
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            // 如果发现了"相等"的 key，判断是否要进行值覆盖，然后也就可以 break 了
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            // 到了链表的最末端，将这个新值放到链表的最后面
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) { // 红黑树
                        Node<K,V> p;
                        binCount = 2;
                        // 调用红黑树的插值方法插入新节点
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }

            if (binCount != 0) {
                // 判断是否要将链表转换为红黑树，临界值和 HashMap 一样，也是 8
                if (binCount >= TREEIFY_THRESHOLD)
                    // 这个方法和 HashMap 中稍微有一点点不同，那就是它不是一定会进行红黑树转换，
                    // 如果当前数组的长度小于 64，那么会选择进行数组扩容，而不是转换为红黑树
                    //    具体源码我们就不看了，扩容部分后面说
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    // 
    addCount(1L, binCount);
    return null;
}
    
```



### [¶](#初始化数组-inittable) 初始化数组: initTable

这个比较简单，主要就是初始化一个合适大小的数组，然后会设置 sizeCtl。

初始化方法中的并发问题是通过对 sizeCtl 进行一个 CAS 操作来控制的。

```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        // 初始化的"功劳"被其他线程"抢去"了
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // lost initialization race; just spin
        // CAS 一下，将 sizeCtl 设置为 -1，代表抢到了锁
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                if ((tab = table) == null || tab.length == 0) {
                    // DEFAULT_CAPACITY 默认初始容量是 16
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    // 初始化数组，长度为 16 或初始化时提供的长度
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    // 将这个数组赋值给 table，table 是 volatile 的
                    table = tab = nt;
                    // 如果 n 为 16 的话，那么这里 sc = 12
                    // 其实就是 0.75 * n
                    sc = n - (n >>> 2);
                }
            } finally {
                // 设置 sizeCtl 为 sc，我们就当是 12 吧
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
    
```



### [¶](#计数 addCount()) 计数 addCount()

```java
addCount(1L, binCount);
```

如何保证并发的size更新的安全性->原子性

1.cas => 加锁 性能下降，不断cas，自旋

2.分治

```java
private final void addCount(long x, int check) {
        CounterCell[] as; long b, s;
        // 判断 counterCells 是否为空，
        // 1. 如果为空，就通过 cas 操作尝试修改 baseCount 变量，对这个变量进行原子累加操作(做这个操作的意义是：如果在没有竞争的情况下，仍然采用 baseCount 来记录元素个数)
        // 2. 如果 cas 失败说明存在竞争，这个时候不能再采用 baseCount 来累加，而是通过CounterCell 来记录
        if ((as = counterCells) != null ||
            !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
            CounterCell a; long v; int m;
            boolean uncontended = true;
           // 这里有几个判断
//1. 计数表为空则直接调用 fullAddCount
//2. 从计数表中随机取出一个数组的位置为空，直接调用 fullAddCount
//3. 通过 CAS 修改 CounterCell 随机位置的值，如果修改失败说明出现并发情况（这里又用到了一种巧妙的方法），调用 fullAndCountRandom 在线程并发的时候会有性能问题以及可能会产生相同的随机数,ThreadLocalRandom.getProbe 可以解决这个问题，并且性能要比 Random 高
            if (as == null || (m = as.length - 1) < 0 ||
                (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
                !(uncontended =
                  U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
                fullAddCount(x, uncontended);
                return;
            }
            if (check <= 1) //链表长度小于等于 1，不需要考虑扩容
                return;
            s = sumCount(); // 统计ConcurrentHashMap元素个数
        }
        ...
    }
```

baseCount=0：用来记录元素个数的成员属性 => 传入值baseCount = 1

ThreadLocalRandom => 线程安全的生成随机数



### [¶](#CounterCell解释) CounterCell解释

```java
    private transient volatile int cellsBusy; // 标识当前 cell 数组是否在初始化或扩容中的CAS 标志位

    /**
     * Table of counter cells. When non-null, size is a power of 2.
     */
    private transient volatile CounterCell[] counterCells; // counterCells 数组，总数值的分值分别存在每个 cell 中
    @sun.misc.Contended static final class CounterCell {
        volatile long value;
        CounterCell(long x) { value = x; }
    }
//看到这段代码就能够明白了，CounterCell 数组的每个元素，都存储一个元素个数，而实际我们调用size 方法就是通过这个循环累加来得到的
//又是一个设计精华，大家可以借鉴； 有了这个前提，再会过去看 addCount 这个方法，就容易理解一些了
    final long sumCount() {
        CounterCell[] as = counterCells; CounterCell a;
        long sum = baseCount;
        if (as != null) {
            for (int i = 0; i < as.length; ++i) {
                if ((a = as[i]) != null)
                    sum += a.value;
            }
        }
        return sum;
    }
```

![](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/202203062357760.png)





### [¶](#fullAddCount源码)fullAddCount源码

第一次进来的位置：

cellsBusy：1表示已经有其他线程在进行扩容了

第一次线程进来的时候：ThreadA进入

![image-20220302235448532](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/202203062359251.png)

```java
            //cellsBusy=0表示没有在做初始化，通过cas更新cellsbusy的值标注当前线程正在做初始化操作
            else if (cellsBusy == 0 && counterCells == as &&
                     U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                boolean init = false;
                try {                           // Initialize table
                    if (counterCells == as) {
                        //初始化容量为2
                        CounterCell[] rs = new CounterCell[2];
                        //将x也就是元素的个数 放在指定的数组下标位置
                        rs[h & 1] = new CounterCell(x);
                        //赋值给counterCells
                        counterCells = rs;
                        //设置初始化完成标识
                        init = true;
                    }
                } finally {
                    //恢复标识
                    cellsBusy = 0;
                }
                if (init)
                    break;
            }
            else if (U.compareAndSwapLong(this, BASECOUNT, v = baseCount, v + x))
                //竞争激烈，其它线程占据cell 数组，直接累加在base变量中
                break;                          // Fall back on using base
        }
```

第二次线程线程进来的时候：ThreadB进入线程

![image-20220302235742080](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/202203062359288.png)

```java
                if ((a = as[(n - 1) & h]) == null) {
                    if (cellsBusy == 0) {            // Try to attach new Cell
                        CounterCell r = new CounterCell(x); // Optimistic create
                        if (cellsBusy == 0 &&
                            U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                            boolean created = false;
                            try {               // Recheck under lock
                                CounterCell[] rs; int m, j;
                                if ((rs = counterCells) != null &&
                                    (m = rs.length) > 0 &&
                                    rs[j = (m - 1) & h] == null) {
                                    rs[j] = r;
                                    created = true;
                                }
                            } finally {
                                cellsBusy = 0;
                            }
                            if (created)
                                break;
                            continue;           // Slot is now non-empty
                        }
                    }
                    collide = false;
                }
```

其他线程进入：

![image-20220303000045422](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/202203062359294.png)

```java
U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x)
```

直接通过cas进行修改value值

```java
private final void fullAddCount(long x, boolean wasUncontended) {
        int h;
 
        //获取当前线程的probe的值，如果值为0，则初始化当前线程的probe的值,probe就是随机数
        if ((h = ThreadLocalRandom.getProbe()) == 0) {
            ThreadLocalRandom.localInit();      // force initialization
            h = ThreadLocalRandom.getProbe();
 
            // 由于重新生成了probe，未冲突标志位设置为true
            wasUncontended = true;
        }
 
        boolean collide = false;                // True if last slot nonempty
 
        //自旋
        for (;;) {
            CounterCell[] as; CounterCell a; int n; long v;
 
            //说明counterCells已经被初始化过了，我们先跳过这个代码，先看初始化部分
            if ((as = counterCells) != null && (n = as.length) > 0) {
 
                // 通过该值与当前线程probe求与，获得cells的下标元素，和hash 表获取索引是一样的
                if ((a = as[(n - 1) & h]) == null) {
 
                    //cellsBusy=0表示counterCells不在初始化或者扩容状态下
                    if (cellsBusy == 0) {            // Try to attach new Cell
 
                        //构造一个CounterCell的值，传入元素个数
                        CounterCell r = new CounterCell(x); // Optimistic create
 
                        //通过cas设置cellsBusy标识，防止其他线程来对counterCells并发处理
                        if (cellsBusy == 0 &&
                            U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                            boolean created = false;
                            try {               // Recheck under lock
                                CounterCell[] rs; int m, j;
 
                                //将初始化的r对象的元素个数放在对应下标的位置
                                if ((rs = counterCells) != null &&
                                    (m = rs.length) > 0 &&
                                    rs[j = (m - 1) & h] == null) {
                                    rs[j] = r;
                                    created = true;
                                }
                            } finally {
                                //恢复标志位
                                cellsBusy = 0;
                            }
                            if (created)
                                break;
 
                            //说明指定cells下标位置的数据不为空，则进行下一次循环
                            continue;           // Slot is now non-empty
                        }
                    }
                    collide = false;
                }
                //说明在addCount方法中cas失败了，并且获取probe的值不为空
                else if (!wasUncontended)       // CAS already known to fail
                    //设置为未冲突标识，进入下一次自旋
                    wasUncontended = true;      // Continue after rehash
 
                //由于指定下标位置的cell值不为空，则直接通过cas进行原子累加，如果成功，则直接退出
                else if (U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))
                    break;
 
                //如果已经有其他线程建立了新的counterCells或者CounterCells大于CPU核心数（很巧妙，线程的并发数不会超过cpu核心数）
                else if (counterCells != as || n >= NCPU)
                    //设置当前线程的循环失败不进行扩容
                    collide = false;            // At max size or stale
 
                //恢复collide状态，标识下次循环会进行扩容
                else if (!collide)
                    collide = true;
 
                //进入这个步骤，说明CounterCell数组容量不够，线程竞争较大，所以先设置一个标识表示为正在扩容
                else if (cellsBusy == 0 &&
                         U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                    try {
                        if (counterCells == as) {// Expand table unless stale
                            //扩容一倍 2变成4，这个扩容比较简单
                            CounterCell[] rs = new CounterCell[n << 1];
                            for (int i = 0; i < n; ++i)
                                rs[i] = as[i];
                            counterCells = rs;
                        }
                    } finally {
                        //恢复标识
                        cellsBusy = 0;
                    }
                    collide = false;
 
                    //继续下一次自旋
                    continue;                   // Retry with expanded table
                }
 
                //继续下一次自旋
                h = ThreadLocalRandom.advanceProbe(h);
            }
            //cellsBusy=0表示没有在做初始化，通过cas更新cellsbusy的值标注当前线程正在做初始化操作
            else if (cellsBusy == 0 && counterCells == as &&
                     U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                boolean init = false;
                try {                           // Initialize table
                    if (counterCells == as) {
                        //初始化容量为2
                        CounterCell[] rs = new CounterCell[2];
                        //将x也就是元素的个数 放在指定的数组下标位置
                        rs[h & 1] = new CounterCell(x);
                        //赋值给counterCells
                        counterCells = rs;
                        //设置初始化完成标识
                        init = true;
                    }
                } finally {
                    //恢复标识
                    cellsBusy = 0;
                }
                if (init)
                    break;
            }
            else if (U.compareAndSwapLong(this, BASECOUNT, v = baseCount, v + x))
                //竞争激烈，其它线程占据cell 数组，直接累加在base变量中
                break;                          // Fall back on using base
        }
    }
```

### [¶](#CountCells图解)CountCells图解

![image-20220303000357790](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/202203062359215.png)

## [¶](transfer扩容阶段)transfer扩容阶段

> ConcurrentHashMap的扩容是可以并行扩容的

判断是否需要扩容，也就是当更新后的键值对总数 baseCount >= 阈值 sizeCtl 时，进行 rehash，这里面会有两个逻辑。

1.如果当前正在处于扩容阶段，则当前线程会加入并且协助扩容

2.如果当前没有在扩容，则直接触发扩容操作

```java
        if (check >= 0) {//如果 binCount>=0，标识需要检查扩容
            Node<K,V>[] tab, nt; int n, sc;
            //s 标识集合大小，如果集合大小大于或等于扩容阈值（默认值的 0.75）
            //并且 table 不为空并且 table 的长度小于最大容量
            while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                   (n = tab.length) < MAXIMUM_CAPACITY) {
                int rs = resizeStamp(n); //这里是生成一个唯一的扩容戳，
                if (sc < 0) { //sc<0，也就是 sizeCtl<0，说明已经有别的线程正在扩容了
                    //这 5 个条件只要有一个条件为 true，说明当前线程不能帮助进行此次的扩容，直接跳出循环
                    //sc >>> RESIZE_STAMP_SHIFT!=rs 表示比较高 RESIZE_STAMP_BITS 位生成戳和 rs 是否相等，相同
                    //sc=rs+1 表示扩容结束
                    //sc==rs+MAX_RESIZERS 表示帮助线程线程已经达到最大值了
                    //nt=nextTable -> 表示扩容已经结束
                    //transferIndex<=0 表示所有的 transfer 任务都被领取完了，没有剩余的hash 桶来给自己自己好这个线程来做 transfer
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0)
                        break;
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) //当前线程尝试帮助此次扩容，如果成功，则调用 transfer
                        transfer(tab, nt);
                }
                // 如果当前没有在扩容，那么 rs 肯定是一个正数，通过 rs<<RESIZE_STAMP_SHIFT 将 sc 设置为一个负数，+2 表示有一个线程在执行扩容
                else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                             (rs << RESIZE_STAMP_SHIFT) + 2))
                    transfer(tab, null);
                s = sumCount(); // 重新计数，判断是否需要开启下一轮扩容
            }
        }
```

### [¶](resizeStamp)resizeStamp

```java
static final int resizeStamp(int n) {
    return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
}
```

Integer.numberOfLeadingZeros 这个方法是返回无符号整数 n 最高位非 0 位前面的 0 的个数

> n = 16；
>
> Integer.numberOfLeadingZeros(16) = 5
>
> 32 - 5 = 27

resizeStamp(16) = 32795

```java
U.compareAndSwapInt(this, SIZECTL, sc, (rs << RESIZE_STAMP_SHIFT) + 2) // 执行如上代码
```

0000 0000 0000 0000 **1000 0000 0001 1011** 左移16位 =>

**1000 0000 0001 1011** 0000 0000 0000 0000 + 2 =>

**1000 0000 0001 1011** 0000 0000 0000 0010

扩容戳：

高16位代表扩容的标记

低16位代表扩容的线程数 -> 有一个线程参与了扩容

1.需要保证每次扩容的扩容戳都是唯一的

2.可以支持并发扩容

![image-20220304215313689](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/202203062359205.png)

➢ 这样来存储有什么好处呢？

1.首先在 CHM 中是支持并发扩容的，也就是说如果当前的数组需要进行扩容操作，可以 由多个线程来共同负责，这块后续会单独讲

2.可以保证每次扩容都生成唯一的生成戳，每次新的扩容，都有一个不同的 n，这个生成 戳就是根据 n 来计算出来的一个数字，n 不同，这个数字也不同

➢ 第一个线程尝试扩容的时候，为什么是+2

因为 1 表示初始化，2 表示一个线程在执行扩容，而且对 sizeCtl 的操作都是基于位运算的， 所以不会关心它本身的数值是多少，只关心它在二进制上的数值，而 sc + 1 会在 低 16 位上加 1。

### [¶](transfer)transfer

1.扩大数组的长度

2.数组迁移

ConcurrentHashMap 并没有直接加锁，而是采用 CAS 实现无锁的并发同步策略，最精华 的部分是它可以利用多线程来进行协同扩容

1、fwd:这个类是个标识类，用于指向新表用的，其他线程遇到这个类会主动跳过这个类，因 为这个类要么就是扩容迁移正在进行，要么就是已经完成扩容迁移，也就是这个类要保证线 程安全，再进行操作。

2、advance:这个变量是用于提示代码是否进行推进处理，也就是当前桶处理完，处理下一个 桶的标识

3、finishing:这个变量用于提示扩容是否结束用的

```java
if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
            stride = MIN_TRANSFER_STRIDE; // subdivide range
```

让每一个CPU执行一段数据的扩容，每一个CPU可以处理16个长度的数组

![image-20220304220428918](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/202203062359318.png)

```java
int nextIndex, nextBound;
if (--i >= bound || finishing)
    advance = false;
else if ((nextIndex = transferIndex) <= 0) {
    i = -1;
    advance = false;
}
else if (U.compareAndSwapInt
         (this, TRANSFERINDEX, nextIndex,
          nextBound = (nextIndex > stride ?
                       nextIndex - stride : 0))) {
    // nextBound = 16 // bound = 16, i = 32-1 =31
    // bound = nextBoud // i = nextIndex - 1;
    // advance = false
    bound = nextBound;
    i = nextIndex - 1;
    advance = false;
}
    private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
        int n = tab.length, stride;
        //将 (n>>>3 相当于 n/8) 然后除以 CPU 核心数。如果得到的结果小于 16，那么就使用 16
// 这里的目的是让每个 CPU 处理的桶一样多，避免出现转移任务不均匀的现象，如果桶较少的话，默认一个 CPU（一个线程）处理 16 个桶，也就是长度为 16 的时候，扩容的时候只会有一个线程来扩容
        if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
            stride = MIN_TRANSFER_STRIDE; // subdivide range
        if (nextTab == null) {            // initiating
            try {
                //新建一个 n<<1 原始 table 大小的 nextTab,也就是 32
                @SuppressWarnings("unchecked")
                Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
                nextTab = nt; //赋值给 nextTab
            } catch (Throwable ex) {      // try to cope with OOME
                //扩容失败，sizeCtl 使用 int 的最大值
                sizeCtl = Integer.MAX_VALUE;
                return;
            }
            nextTable = nextTab;//更新成员变量
            transferIndex = n; //更新成员变量
        }
        int nextn = nextTab.length; //新的 tab 的长度
        // 创建一个 fwd 节点，表示一个正在被迁移的 Node，并且它的 hash 值为-1(MOVED)，也就是前面我们在讲 putval 方法的时候，会有一个判断 MOVED 的逻辑。它的作用是用来占位，表示原数组中位置 i 处的节点完成迁移以后，就会在 i 位置设置一个 fwd 来告诉其他线程这个位置已经处理过了，具体后续还会在讲
        ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
        // 首次推进为 true，如果等于 true，说明需要再次推进一个下标（i--），反之，如果是false，那么就不能推进下标，需要将当前的下标处理完毕才能继续推进
        boolean advance = true;
        //判断是否已经扩容完成，完成就 return，退出循环
        boolean finishing = false; // to ensure sweep before committing nextTab
        //通过 for 自循环处理每个槽位中的链表元素，默认 advace 为真，通过 CAS 设置transferIndex 属性值，并初始化 i 和 bound 值，i 指当前处理的槽位序号，bound 指需要处理的槽位边界，先处理槽位 15 的节点；
        for (int i = 0, bound = 0;;) {
            // 这个循环使用 CAS 不断尝试为当前线程分配任务
 // 直到分配成功或任务队列已经被全部分配完毕
 // 如果当前线程已经被分配过 bucket 区域
 // 那么会通过--i 指向下一个待处理 bucket 然后退出该循环
            Node<K,V> f; int fh;
            while (advance) {
                int nextIndex, nextBound;
                //--i 表示下一个待处理的 bucket，如果它>=bound,表示当前线程已经分配过bucket 区域
                if (--i >= bound || finishing)
                    advance = false;
                else if ((nextIndex = transferIndex) <= 0) {//表示所有 bucket 已经被分配完毕
                    i = -1;
                    advance = false;
                }
                //通过 cas 来修改 TRANSFERINDEX,为当前线程分配任务，处理的节点区间为(nextBound,nextIndex)->(0,15)
                else if (U.compareAndSwapInt
                         (this, TRANSFERINDEX, nextIndex,
                          nextBound = (nextIndex > stride ?
                                       nextIndex - stride : 0))) {
                    bound = nextBound;
                    i = nextIndex - 1;
                    advance = false;
                }
            }
            //i<0 说明已经遍历完旧的数组，也就是当前线程已经处理完所有负责的 bucket
            if (i < 0 || i >= n || i + n >= nextn) {
                int sc;
                if (finishing) { //如果完成了扩容
                    nextTable = null; //删除成员变量
                    table = nextTab;  //删除成员变量
                    sizeCtl = (n << 1) - (n >>> 1); //更新阈值(32*0.75=24)
                    return;
                }
                // sizeCtl 在迁移前会设置为 (rs << RESIZE_STAMP_SHIFT) + 2 
// 然后，每增加一个线程参与迁移就会将 sizeCtl 加 1，
// 这里使用 CAS 操作对 sizeCtl 的低 16 位进行减 1，代表做完了属于自己的任务
                if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
           // 第一个扩容的线程，执行 transfer 方法之前，会设置 
                    //sizeCtl = (resizeStamp(n) << RESIZE_STAMP_SHIFT) + 2)后续帮其扩容的线程，执行 transfer 方法之前，会设置 sizeCtl = sizeCtl+1每一个退出 transfer 的方法的线程，退出之前，会设置 sizeCtl = sizeCtl-1那么最后一个线程退出时：必然有sc == (resizeStamp(n) << RESIZE_STAMP_SHIFT) + 2)，即 (sc - 2) == resizeStamp(n) << RESIZE_STAMP_SHIFT 
// 如果 sc - 2 不等于标识符左移 16 位。如果他们相等了，说明没有线程在帮助他们扩容了。也就是说，扩容结束了。
                    if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                        return;
                    // 如果相等，扩容结束了，更新 finising 变量
                    finishing = advance = true;
                    // 再次循环检查一下整张表
                    i = n; // recheck before commit
                }
            }
            // 如果位置 i 处是空的，没有任何节点，那么放入刚刚初始化的 ForwardingNode ”空节点“
            else if ((f = tabAt(tab, i)) == null)
                advance = casTabAt(tab, i, null, fwd);
            //表示该位置已经完成了迁移，也就是如果线程 A 已经处理过这个节点，那么线程 B 处理这个节点时，hash 值一定为 MOVED
            else if ((fh = f.hash) == MOVED)
                advance = true; // already processed
            else {
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        Node<K,V> ln, hn;
                        if (fh >= 0) {
                            int runBit = fh & n;
                            Node<K,V> lastRun = f;
                            for (Node<K,V> p = f.next; p != null; p = p.next) {
                                int b = p.hash & n;
                                if (b != runBit) {
                                    runBit = b;
                                    lastRun = p;
                                }
                            }
                            if (runBit == 0) {
                                ln = lastRun;
                                hn = null;
                            }
                            else {
                                hn = lastRun;
                                ln = null;
                            }
                            for (Node<K,V> p = f; p != lastRun; p = p.next) {
                                int ph = p.hash; K pk = p.key; V pv = p.val;
                                if ((ph & n) == 0)
                                    ln = new Node<K,V>(ph, pk, pv, ln);
                                else
                                    hn = new Node<K,V>(ph, pk, pv, hn);
                            }
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                        else if (f instanceof TreeBin) {
                            TreeBin<K,V> t = (TreeBin<K,V>)f;
                            TreeNode<K,V> lo = null, loTail = null;
                            TreeNode<K,V> hi = null, hiTail = null;
                            int lc = 0, hc = 0;
                            for (Node<K,V> e = t.first; e != null; e = e.next) {
                                int h = e.hash;
                                TreeNode<K,V> p = new TreeNode<K,V>
                                    (h, e.key, e.val, null, null);
                                if ((h & n) == 0) {
                                    if ((p.prev = loTail) == null)
                                        lo = p;
                                    else
                                        loTail.next = p;
                                    loTail = p;
                                    ++lc;
                                }
                                else {
                                    if ((p.prev = hiTail) == null)
                                        hi = p;
                                    else
                                        hiTail.next = p;
                                    hiTail = p;
                                    ++hc;
                                }
                            }
                            ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                                (hc != 0) ? new TreeBin<K,V>(lo) : t;
                            hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                                (lc != 0) ? new TreeBin<K,V>(hi) : t;
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                    }
                }
            }
        }
    }
```

1、通过数组的方式实现并发增加元素的个数

2、并发扩容，可以通过多个线程并行实现数据的迁移

3、采用高低链的方式来解决多次hash计算的问题，提升了效率

4、sizeCtl的设计，3钟状态表示

5、resizeStamp的设计，高低位的设计来实现唯一性以及多个线程的协助扩容记录

### [¶](扩容过程图解)扩容过程图解

ConcurrentHashMap 支持并发扩容，实现方式是，把 Node 数组进行拆分，让每个线程处理 自己的区域，假设 table 数组总长度是 64，默认情况下，那么每个线程可以分到 16 个 bucket。 然后每个线程处理的范围，按照倒序来做迁移

通过 for 自循环处理每个槽位中的链表元素，默认 advace 为真，通过 CAS 设置 transferIndex 属性值，并初始化 i 和 bound 值，i 指当前处理的槽位序号，bound 指需要处理的槽位边界， 先处理槽位 31 的节点； （bound,i） =(16,31) 从 31 的位置往前推动。

![image-20220304225132246](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/202203062359715.png)

假设这个时候 ThreadA 在进行 transfer

![image-20220304225242329](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/202203062359688.png)

在当前假设条件下，槽位 15 中没有节点，则通过 CAS 插入在第二步中初始化的 ForwardingNode 节点，用于告诉其它线程该槽位已经处理过了；

![image-20220304225301283](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/202203062359692.png)

### [¶](#sizeCtl 扩容退出机制)sizeCtl 扩容退出机制

```java
if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                    if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
```

每存在一个线程执行完扩容操作，就通过 cas 执行 sc-1。

接着判断(sc-2) !=resizeStamp(n) << RESIZE_STAMP_SHIFT ; 如果相等，表示当前为整个扩 容操作的 最后一个线程，那么意味着整个扩容操作就结束了；如果不想等，说明还得继续 这么做的目的，一方面是防止不同扩容之间出现相同的 sizeCtl，另外一方面，还可以避免 sizeCtl 的 ABA 问题导致的扩容重叠的情况

## [¶](#数据迁移的实现方案)数据迁移的实现方案

### [¶](#高低位原理)高低位原理

![image-20220304222908738](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/202203062359906.png)

![image-20220304231617305](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/202203062359775.png)

```java
        if (fh >= 0) {
            int runBit = fh & n;
            Node<K,V> lastRun = f;
            for (Node<K,V> p = f.next; p != null; p = p.next) {
                int b = p.hash & n;
                if (b != runBit) {
                    runBit = b;
                    lastRun = p;
                }
            }
            if (runBit == 0) {
                ln = lastRun;
                hn = null;
            }
            else {
                hn = lastRun;
                ln = null;
            }
            for (Node<K,V> p = f; p != lastRun; p = p.next) {
                int ph = p.hash; K pk = p.key; V pv = p.val;
                if ((ph & n) == 0)
                    ln = new Node<K,V>(ph, pk, pv, ln);
                else
                    hn = new Node<K,V>(ph, pk, pv, hn);
            }
            setTabAt(nextTab, i, ln);// 低位不需要变
            setTabAt(nextTab, i + n, hn); // 高位要变
            setTabAt(tab, i, fwd);
            advance = true;
        }
```

![image-20220304223445785](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/202203062359163.png)

```java
        setTabAt(nextTab, i, ln);// 低位不需要变
        setTabAt(nextTab, i + n, hn); // 高位要变，需要增加n长度的位置
```

![image-20220304223831756](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/202203062359321.png)

### [¶](#为什么要做高低位的划分)为什么要做高低位的划分

为什么这么做？

```java
f = tabAt(tab, i = (n - 1) & hash)
```

扩容以后对于同一个值，结果是不变的

### [¶](#扩容结束以后的退出机制)扩容结束以后的退出机制

```java
if (i < 0 || i >= n || i + n >= nextn) {
                int sc;
                if (finishing) {
                    nextTable = null;
                    table = nextTab;
                    sizeCtl = (n << 1) - (n >>> 1);
                    return;
                }
                if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                    if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                        return;
                    finishing = advance = true;
                    i = n; // recheck before commit
                }
            }
```

## [¶](put第三阶段)put第三阶段

如果对应的节点存在，判断这个节点的 hash 是不是等于 MOVED(-1)，说明当前节点是 ForwardingNode 节点，

意味着有其他线程正在进行扩容，那么当前现在直接帮助它进行扩容，因此调用 helpTransfer 方法

```java
else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
        Node<K,V>[] nextTab; int sc;
    // 判断此时是否仍然在执行扩容,nextTab=null 的时候说明扩容已经结束了
        if (tab != null && (f instanceof ForwardingNode) &&
            (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
            int rs = resizeStamp(tab.length);
            while (nextTab == nextTable && table == tab &&
                   (sc = sizeCtl) < 0) { //说明扩容还未完成的情况下不断循环来尝试将当前线程加入到扩容操作中
                //下面部分的整个代码表示扩容结束，直接退出循环
                //transferIndex<=0 表示所有的 Node 都已经分配了线程
                //sc=rs+MAX_RESIZERS 表示扩容线程数达到最大扩容线程数
                //sc >>> RESIZE_STAMP_SHIFT !=rs， 如果在同一轮扩容中，那么 sc 无符号右移比较高位和 rs 的值，那么应该是相等的。如果不相等，说明扩容结束了
                //sc==rs+1 表示扩容结束
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || transferIndex <= 0)
                    break;//跳出循环
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {//在低 16 位上增加扩容线程数
                    transfer(tab, nextTab);
                    break;
                }
            }
            return nextTab;
        }
        return table;
    }
```

## [¶](#put第四阶段)put第四阶段

这个方法的主要作用是，如果被添加的节点的位置已经存在节点的时候，需要以链表的方式加入到节点中 如果当前节点已经是一颗红黑树，那么就会按照红黑树的规则将当前节点加入到红黑树中

```java
            else { //进入到这个分支，说明 f 是当前 nodes 数组对应位置节点的头节点，并且不为空
                V oldVal = null;
                synchronized (f) { //给对应的头结点加锁
                    if (tabAt(tab, i) == f) { {//再次判断对应下标位置是否为 f 节点
                        if (fh >= 0) { //头结点的 hash 值大于 0，说明是链表
                            binCount = 1; //用来记录链表的长度
                            for (Node<K,V> e = f;; ++binCount) { //遍历链表
                                K ek;
                                //如果发现相同的 key，则判断是否需要进行值的覆盖
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    //默认情况下，直接覆盖旧的值
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                //一直遍历到链表的最末端，直接把新的值加入到链表的最后面
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        //如果当前的 f 节点是一颗红黑树
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            binCount = 2;
                            //则调用红黑树的插入方法插入新的值
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;//同样，如果值已经存在，则直接替换
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                }
                if (binCount != 0) {
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
```

## [¶](#put的第五阶段)put的第五阶段

判断链表的长度是否已经达到临界值 8. 如果达到了临界值，这个时候会根据当前数组的长度 来决定是扩容还是将链表转化为红黑树。也就是说如果当前数组的长度小于 64，就会先扩容。 否则，会把当前链表转化为红黑树

```java
        if (binCount != 0) { {//说明上面在做链表操作
            //如果链表长度已经达到临界值 8 就需要把链表转换为树结构
            if (binCount >= TREEIFY_THRESHOLD)
                treeifyBin(tab, i);
            if (oldVal != null) //如果 val 是被替换的，则返回替换之前的值
                return oldVal;
            break;
        }
```

### [¶](#treeifyBin)treeifyBin

在 putVal 的最后部分，有一个判断，如果链表长度大于 8，那么就会触发扩容或者红黑树的 转化操作。

```java
private final void treeifyBin(Node<K,V>[] tab, int index) {
    Node<K,V> b; int n, sc;
    if (tab != null) {
        if ((n = tab.length) < MIN_TREEIFY_CAPACITY) //tab 的长度是不是小于 64，如果是，则执行扩容

            tryPresize(n << 1);
        else if ((b = tabAt(tab, index)) != null && b.hash >= 0) { //否则，将当前链表转化为红黑树结构存储
            synchronized (b) { // 将链表转换成红黑树
                if (tabAt(tab, index) == b) {
                    TreeNode<K,V> hd = null, tl = null;
                    for (Node<K,V> e = b; e != null; e = e.next) {
                        TreeNode<K,V> p =
                            new TreeNode<K,V>(e.hash, e.key, e.val,
                                              null, null);
                        if ((p.prev = tl) == null)
                            hd = p;
                        else
                            tl.next = p;
                        tl = p;
                    }
                    setTabAt(tab, index, new TreeBin<K,V>(hd));
                }
            }
        }
    }
}
```

### [¶](tryPresize)tryPresize

```java
private final void tryPresize(int size) {
    //对 size 进 行修复 ,主 要目的是防止传入的值不是一个 2 次幂的 整数 ，然后通过tableSizeFor来讲入参转化为离该整数最近的 2 次幂
    int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY :
        tableSizeFor(size + (size >>> 1) + 1);
    int sc;
    while ((sc = sizeCtl) >= 0) {
        Node<K,V>[] tab = table; int n;
        if (tab == null || (n = tab.length) == 0) {
            n = (sc > c) ? sc : c;
            if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    if (table == tab) {
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = nt;
                        sc = n - (n >>> 2);
                    }
                } finally {
                    sizeCtl = sc;
                }
            }
        }
        else if (c <= sc || n >= MAXIMUM_CAPACITY)
            break;
        else if (tab == table) {
            int rs = resizeStamp(n);
            if (sc < 0) {
                Node<K,V>[] nt;
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
        }
    }
}
```

## [总结](https://www.how2.cn/#/2.JavaNotes/(9).gupao-lesson/3.架构师必备技术栈/01.分布式并发编程/6.ConcurrentHashMap的原理分析?id=总结)

**如果链表的长度大于8，并且node数组长度>64得时候，如果再添加数据，会把当前链表转为红黑树，当出现扩容的话，链表长度<8,红黑树又会转为链表。**

## [红黑树](https://www.how2.cn/#/2.JavaNotes/(9).gupao-lesson/3.架构师必备技术栈/01.分布式并发编程/6.ConcurrentHashMap的原理分析?id=红黑树)

![image-20220304234937725](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/202203062359238.png)

### [情况一](https://www.how2.cn/#/2.JavaNotes/(9).gupao-lesson/3.架构师必备技术栈/01.分布式并发编程/6.ConcurrentHashMap的原理分析?id=情况一)

![image-20220304235049636](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/202203062359178.png)

### [情况二](https://www.how2.cn/#/2.JavaNotes/(9).gupao-lesson/3.架构师必备技术栈/01.分布式并发编程/6.ConcurrentHashMap的原理分析?id=情况二)

![image-20220304235100418](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/202203062359351.png)

### [情况三](https://www.how2.cn/#/2.JavaNotes/(9).gupao-lesson/3.架构师必备技术栈/01.分布式并发编程/6.ConcurrentHashMap的原理分析?id=情况三)

![image-20220304235140744](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/202203062359583.png)

### [¶](#链表转红黑树-treeifybin) 链表转红黑树: treeifyBin

前面我们在 put 源码分析也说过，treeifyBin 不一定就会进行红黑树转换，也可能是仅仅做数组扩容。我们还是进行源码分析吧。

```java
private final void treeifyBin(Node<K,V>[] tab, int index) {
    Node<K,V> b; int n, sc;
    if (tab != null) {
        // MIN_TREEIFY_CAPACITY 为 64
        // 所以，如果数组长度小于 64 的时候，其实也就是 32 或者 16 或者更小的时候，会进行数组扩容
        if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
            // 后面我们再详细分析这个方法
            tryPresize(n << 1);
        // b 是头节点
        else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
            // 加锁
            synchronized (b) {

                if (tabAt(tab, index) == b) {
                    // 下面就是遍历链表，建立一颗红黑树
                    TreeNode<K,V> hd = null, tl = null;
                    for (Node<K,V> e = b; e != null; e = e.next) {
                        TreeNode<K,V> p =
                            new TreeNode<K,V>(e.hash, e.key, e.val,
                                              null, null);
                        if ((p.prev = tl) == null)
                            hd = p;
                        else
                            tl.next = p;
                        tl = p;
                    }
                    // 将红黑树设置到数组相应位置中
                    setTabAt(tab, index, new TreeBin<K,V>(hd));
                }
            }
        }
    }
}
  
        @pdai: 代码已经复制到剪贴板
    
```

### [¶](#扩容-trypresize) 扩容: tryPresize

如果说 Java8 ConcurrentHashMap 的源码不简单，那么说的就是扩容操作和迁移操作。

这个方法要完完全全看懂还需要看之后的 transfer 方法，读者应该提前知道这点。

这里的扩容也是做翻倍扩容的，扩容后数组容量为原来的 2 倍。

```java
// 首先要说明的是，方法参数 size 传进来的时候就已经翻了倍了
private final void tryPresize(int size) {
    // c: size 的 1.5 倍，再加 1，再往上取最近的 2 的 n 次方。
    int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY :
        tableSizeFor(size + (size >>> 1) + 1);
    int sc;
    while ((sc = sizeCtl) >= 0) {
        Node<K,V>[] tab = table; int n;

        // 这个 if 分支和之前说的初始化数组的代码基本上是一样的，在这里，我们可以不用管这块代码
        if (tab == null || (n = tab.length) == 0) {
            n = (sc > c) ? sc : c;
            if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    if (table == tab) {
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = nt;
                        sc = n - (n >>> 2); // 0.75 * n
                    }
                } finally {
                    sizeCtl = sc;
                }
            }
        }
        else if (c <= sc || n >= MAXIMUM_CAPACITY)
            break;
        else if (tab == table) {
            // 我没看懂 rs 的真正含义是什么，不过也关系不大
            int rs = resizeStamp(n);

            if (sc < 0) {
                Node<K,V>[] nt;
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                // 2. 用 CAS 将 sizeCtl 加 1，然后执行 transfer 方法
                //    此时 nextTab 不为 null
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            // 1. 将 sizeCtl 设置为 (rs << RESIZE_STAMP_SHIFT) + 2)
            //     我是没看懂这个值真正的意义是什么? 不过可以计算出来的是，结果是一个比较大的负数
            //  调用 transfer 方法，此时 nextTab 参数为 null
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
        }
    }
}
    
```

这个方法的核心在于 sizeCtl 值的操作，首先将其设置为一个负数，然后执行 transfer(tab, null)，再下一个循环将 sizeCtl 加 1，并执行 transfer(tab, nt)，之后可能是继续 sizeCtl 加 1，并执行 transfer(tab, nt)。

所以，可能的操作就是执行 1 次 transfer(tab, null) + 多次 transfer(tab, nt)，这里怎么结束循环的需要看完 transfer 源码才清楚。

### [¶](#数据迁移-transfer) 数据迁移: transfer

下面这个方法有点长，将原来的 tab 数组的元素迁移到新的 nextTab 数组中。

虽然我们之前说的 tryPresize 方法中多次调用 transfer 不涉及多线程，但是这个 transfer 方法可以在其他地方被调用，典型地，我们之前在说 put 方法的时候就说过了，请往上看 put 方法，是不是有个地方调用了 helpTransfer 方法，helpTransfer 方法会调用 transfer 方法的。

此方法支持多线程执行，外围调用此方法的时候，会保证第一个发起数据迁移的线程，nextTab 参数为 null，之后再调用此方法的时候，nextTab 不会为 null。

阅读源码之前，先要理解并发操作的机制。原数组长度为 n，所以我们有 n 个迁移任务，让每个线程每次负责一个小任务是最简单的，每做完一个任务再检测是否有其他没做完的任务，帮助迁移就可以了，而 Doug Lea 使用了一个 stride，简单理解就是步长，每个线程每次负责迁移其中的一部分，如每次迁移 16 个小任务。所以，我们就需要一个全局的调度者来安排哪个线程执行哪几个任务，这个就是属性 transferIndex 的作用。

第一个发起数据迁移的线程会将 transferIndex 指向原数组最后的位置，然后从后往前的 stride 个任务属于第一个线程，然后将 transferIndex 指向新的位置，再往前的 stride 个任务属于第二个线程，依此类推。当然，这里说的第二个线程不是真的一定指代了第二个线程，也可以是同一个线程，这个读者应该能理解吧。其实就是将一个大的迁移任务分为了一个个任务包。

```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;

    // stride 在单核下直接等于 n，多核模式下为 (n>>>3)/NCPU，最小值是 16
    // stride 可以理解为”步长“，有 n 个位置是需要进行迁移的，
    //   将这 n 个任务分为多个任务包，每个任务包有 stride 个任务
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range

    // 如果 nextTab 为 null，先进行一次初始化
    //    前面我们说了，外围会保证第一个发起迁移的线程调用此方法时，参数 nextTab 为 null
    //       之后参与迁移的线程调用此方法时，nextTab 不会为 null
    if (nextTab == null) {
        try {
            // 容量翻倍
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        // nextTable 是 ConcurrentHashMap 中的属性
        nextTable = nextTab;
        // transferIndex 也是 ConcurrentHashMap 的属性，用于控制迁移的位置
        transferIndex = n;
    }

    int nextn = nextTab.length;

    // ForwardingNode 翻译过来就是正在被迁移的 Node
    // 这个构造方法会生成一个Node，key、value 和 next 都为 null，关键是 hash 为 MOVED
    // 后面我们会看到，原数组中位置 i 处的节点完成迁移工作后，
    //    就会将位置 i 处设置为这个 ForwardingNode，用来告诉其他线程该位置已经处理过了
    //    所以它其实相当于是一个标志。
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);


    // advance 指的是做完了一个位置的迁移工作，可以准备做下一个位置的了
    boolean advance = true;
    boolean finishing = false; // to ensure sweep before committing nextTab

    /*
     * 下面这个 for 循环，最难理解的在前面，而要看懂它们，应该先看懂后面的，然后再倒回来看
     * 
     */

    // i 是位置索引，bound 是边界，注意是从后往前
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;

        // 下面这个 while 真的是不好理解
        // advance 为 true 表示可以进行下一个位置的迁移了
        //   简单理解结局: i 指向了 transferIndex，bound 指向了 transferIndex-stride
        while (advance) {
            int nextIndex, nextBound;
            if (--i >= bound || finishing)
                advance = false;

            // 将 transferIndex 值赋给 nextIndex
            // 这里 transferIndex 一旦小于等于 0，说明原数组的所有位置都有相应的线程去处理了
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
            else if (U.compareAndSwapInt
                     (this, TRANSFERINDEX, nextIndex,
                      nextBound = (nextIndex > stride ?
                                   nextIndex - stride : 0))) {
                // 看括号中的代码，nextBound 是这次迁移任务的边界，注意，是从后往前
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            if (finishing) {
                // 所有的迁移操作已经完成
                nextTable = null;
                // 将新的 nextTab 赋值给 table 属性，完成迁移
                table = nextTab;
                // 重新计算 sizeCtl: n 是原数组长度，所以 sizeCtl 得出的值将是新数组长度的 0.75 倍
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }

            // 之前我们说过，sizeCtl 在迁移前会设置为 (rs << RESIZE_STAMP_SHIFT) + 2
            // 然后，每有一个线程参与迁移就会将 sizeCtl 加 1，
            // 这里使用 CAS 操作对 sizeCtl 进行减 1，代表做完了属于自己的任务
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                // 任务结束，方法退出
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;

                // 到这里，说明 (sc - 2) == resizeStamp(n) << RESIZE_STAMP_SHIFT，
                // 也就是说，所有的迁移任务都做完了，也就会进入到上面的 if(finishing){} 分支了
                finishing = advance = true;
                i = n; // recheck before commit
            }
        }
        // 如果位置 i 处是空的，没有任何节点，那么放入刚刚初始化的 ForwardingNode ”空节点“
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);
        // 该位置处是一个 ForwardingNode，代表该位置已经迁移过了
        else if ((fh = f.hash) == MOVED)
            advance = true; // already processed
        else {
            // 对数组该位置处的结点加锁，开始处理数组该位置处的迁移工作
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    Node<K,V> ln, hn;
                    // 头节点的 hash 大于 0，说明是链表的 Node 节点
                    if (fh >= 0) {
                        // 下面这一块和 Java7 中的 ConcurrentHashMap 迁移是差不多的，
                        // 需要将链表一分为二，
                        //   找到原链表中的 lastRun，然后 lastRun 及其之后的节点是一起进行迁移的
                        //   lastRun 之前的节点需要进行克隆，然后分到两个链表中
                        int runBit = fh & n;
                        Node<K,V> lastRun = f;
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        if (runBit == 0) {
                            ln = lastRun;
                            hn = null;
                        }
                        else {
                            hn = lastRun;
                            ln = null;
                        }
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            if ((ph & n) == 0)
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        // 其中的一个链表放在新数组的位置 i
                        setTabAt(nextTab, i, ln);
                        // 另一个链表放在新数组的位置 i+n
                        setTabAt(nextTab, i + n, hn);
                        // 将原数组该位置处设置为 fwd，代表该位置已经处理完毕，
                        //    其他线程一旦看到该位置的 hash 值为 MOVED，就不会进行迁移了
                        setTabAt(tab, i, fwd);
                        // advance 设置为 true，代表该位置已经迁移完毕
                        advance = true;
                    }
                    else if (f instanceof TreeBin) {
                        // 红黑树的迁移
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                (h, e.key, e.val, null, null);
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }
                        // 如果一分为二后，节点数少于 8，那么将红黑树转换回链表
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                            (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                            (lc != 0) ? new TreeBin<K,V>(hi) : t;

                        // 将 ln 放置在新数组的位置 i
                        setTabAt(nextTab, i, ln);
                        // 将 hn 放置在新数组的位置 i+n
                        setTabAt(nextTab, i + n, hn);
                        // 将原数组该位置处设置为 fwd，代表该位置已经处理完毕，
                        //    其他线程一旦看到该位置的 hash 值为 MOVED，就不会进行迁移了
                        setTabAt(tab, i, fwd);
                        // advance 设置为 true，代表该位置已经迁移完毕
                        advance = true;
                    }
                }
            }
        }
    }
}
  
    
```

说到底，transfer 这个方法并没有实现所有的迁移任务，每次调用这个方法只实现了 transferIndex 往前 stride 个位置的迁移工作，其他的需要由外围来控制。

这个时候，再回去仔细看 tryPresize 方法可能就会更加清晰一些了。

### [¶](#get-过程分析-2) get 过程分析

get 方法从来都是最简单的，这里也不例外:

- 计算 hash 值
- 根据 hash 值找到数组对应位置: (n - 1) & h
- 根据该位置处结点性质进行相应查找
  - 如果该位置为 null，那么直接返回 null 就可以了
  - 如果该位置处的节点刚好就是我们需要的，返回该节点的值即可
  - 如果该位置节点的 hash 值小于 0，说明正在扩容，或者是红黑树，后面我们再介绍 find 方法
  - 如果以上 3 条都不满足，那就是链表，进行遍历比对即可

```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    int h = spread(key.hashCode());
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {
        // 判断头节点是否就是我们需要的节点
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        // 如果头节点的 hash 小于 0，说明 正在扩容，或者该位置是红黑树
        else if (eh < 0)
            // 参考 ForwardingNode.find(int h, Object k) 和 TreeBin.find(int h, Object k)
            return (p = e.find(h, key)) != null ? p.val : null;

        // 遍历链表
        while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
  
    
```



简单说一句，此方法的大部分内容都很简单，只有正好碰到扩容的情况，ForwardingNode.find(int h, Object k) 稍微复杂一些，不过在了解了数据迁移的过程后，这个也就不难了，所以限于篇幅这里也不展开说了。

## [¶](#对比总结) 对比总结

- `HashTable` : 使用了synchronized关键字对put等操作进行加锁;
- `ConcurrentHashMap JDK1.7`:  使用分段锁机制实现;
- `ConcurrentHashMap JDK1.8`: 则使用数组+链表+红黑树数据结构和CAS原子操作实现



