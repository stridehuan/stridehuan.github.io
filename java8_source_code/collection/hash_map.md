# HashMap源码解析

## 几个关键的对象属性

``` java
/**
 * The table, initialized on first use, and resized as
 * necessary. When allocated, length is always a power of two.
 * 用来存放 bucket 的数组（链表的头节点或红黑树的根节点）
 * 对table进行扩容时，就是将数组长度翻倍
 */
transient Node<K,V>[] table;

/**
 * The number of key-value mappings contained in this map.
 * 键值对的个数
 */
transient int size;

/**
 * The next size value at which to resize (capacity * load factor).
 * 键值对个数达到这个阈值后就对table进行扩容
 */
int threshold;

/**
 * The load factor for the hash table.
 * 用来计算threshold
 */
final float loadFactor;
```

## 几个关键的静态属性

``` java
    /**
     * The default initial capacity - MUST be a power of two.
     * table的默认长度
     */
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

    /**
     * The maximum capacity, used if a higher value is implicitly specified
     * table的最大长度
     */
    static final int MAXIMUM_CAPACITY = 1 << 30;

    /**
     * The load factor used when none specified in constructor.
     * 默认的loadFactor
     */
    static final float DEFAULT_LOAD_FACTOR = 0.75f;

    /**
     * The bin count threshold for using a tree rather than list for a bin.
     * 树化阈值，bucket的队列长度超过这个阈值后就要树化（还要考虑MIN_TREEIFY_CAPACITY）
     */
    static final int TREEIFY_THRESHOLD = 8;

    /**
     * The bin count threshold for untreeifying a (split) bin
     * 反树化阈值，当bucket的大小低于这个值的时候就要反树化
     */
    static final int UNTREEIFY_THRESHOLD = 6;

    /**
     * The smallest table capacity for which bins may be treeified.
     * 当table达到这个长度后，才允许对超出阈值的bucket进行树化
     */
    static final int MIN_TREEIFY_CAPACITY = 64;
```

## 关键方法putVal

``` java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // 如果table没有初始化，则尝试对table进行初始化
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    // 对hashCode进行取模操作（通过位计算实现），找到bucket，如果bucket中的node为空，新建队列的首节点，放到bucket中
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    // 如果bucket中的node不为空，尝试解析队列或树的节点属性
    else {
        // e是被key命中的node
        Node<K,V> e; K k;
        // 如果队列首节点（树的根节点）被key属性命中，则将p赋值给e
        if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        // 如果第一个节点没有命中，且p是树节点，则尝试在红黑树中尝试查找，如果命中，则返回这个Node，如果没有命中，则new一个TreeNode放到红黑树中
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        // 如果第一个节点没有命中，且p是链表节点，则尝试遍历链表，尝试查找节点，如果命中，则返回这个Node，如果没有命中，则new一个Node放到队列中
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    // 如果队列长度超过树化阈值，尝试树化成红黑树
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        // 如果节点命中，尝试替换oldNode中的value，并返回更新后的oldNode
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    // 代码走到这一步，说明没有命中已经存在的key，也就是新增了bucket的节点
    // 自增size，如果size超过了阈值，则调用resize方法对table进行扩容
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

### putVal中使用的其它的方法

- putTreeVal 方法

通过循环的方式，递归查询，尝试命中key，如果不命中，尝试新增加树节点。

- treeifyBin 方法：

如果 table 的 size 没有达到 MIN_TREEIFY_CAPACITY（64），尝试对 table 进行扩容（size翻倍）

如果没有达到 MIN_TREEIFY_CAPACITY，对 bucket 进行树化

## 对于一些设计的原因分析

### 为什么要设置一个 MIN_TREEIFY_CAPACITY

随着 HashMap 中元素（键值对）数量的增长，bucket 中的链表就会变长，从而影响查询和插入效率。

为了解决上面这个问题，HashMap 会做两件事：table 的扩容（增加bucket，对数据进行打散） 和 bucket 的树化（改变 bucket）。

MIN_TREEIFY_CAPACITY 这个静态变量的作用是限制 bucket 的树化，也就是说只有当 table 的长度达到 MIN_TREEIFY_CAPACITY 后，才会在 bucket 链表过长的时候对 bucket 进行树化。

引入 MIN_TREEIFY_CAPACITY 相当于在 早期优先使用 table 扩容的方式来降低 哈希冲突，从而解决 bucket 队列过长的问题。

那为什么不优先去对 bucket 做树化呢？

因为在 bucket 已经树化的情况下，如果再对 table 进行扩容，这个时候需要将旧 bucket 中的部分数据转移到新 bucket 上，这就意味着需要对 已经树化的 bucket 进行重构，成本比较高。

如果 bucket 还没有树化，这个时候 bucket 还是链表的结构，做 bucket 间的数据转移，只是将一个链表的数据转移到另一个链表，成本相对低一些。

因此，在 HashMap 的早期优先使用 table 扩容的方式 解决队列过长的问题，可以避免 bucket 中红黑树的频繁重构。

### 为什么 HashMap 中 table 的长度始终是 2 的 n 次方

目前分析到这种设计在两个方面是有优势的 ：

1. 对 2 的 n 次方做求模运算的时候，可以简单地转化为 位 操作，提高执行效率。
2. 每次对 table 扩容都是将 table 长度翻倍，这样在扩容的时候每个 bucket 平均只需要迁移一半的数据节点，其实相当于是对每个 bucket 做一次分裂。




[上一页](../index.md)
