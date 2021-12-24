### 数据结构

![2](https://wsx666-1302523054.cos.ap-nanjing.myqcloud.com/image/2021-04-03/2%20.png)

+ JDK 7与 JDK8 的差异
  + jdk 7 是数组加链表组成，jdk8是数组加链表加红黑树组成。
  + jdk 7插入时时头插法。jdk8是尾插法

> jdk7 时间复杂度取决于链表的长度，为 **O(n)**,在 Java8 中，当链表中的元素达到了 8 个时，会将链表转换为红黑树，在这些位置进行查找的时候可以降低时间复杂O(logN)**。

### HashMap为什么是线程不安全的

+ 在JDK1.7中，当并发执行扩容操作时会造成环形链和数据丢失的情况。
+ 在JDK1.8中，在并发执行put操作时会发生数据覆盖的情况
+ fast-fail 如果在使用迭代器的过程中有其他线程修改了map，那么将抛出 ConcurrentModificationgException，这就是所谓 fail-fast 策略

### 源码解析

+ 确定哈希桶数组索引位置

  + 取key的hashCode值
  + 高位运算
  + 取模运算

  ```java
  static final int hash(Object key) {   //jdk1.8 & jdk1.7
       int h;
       // h = key.hashCode() 为第一步 取hashCode值
       // h ^ (h >>> 16)  为第二步 高位参与运算
       return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
  }
  
  static int indexFor(int h, int length) {  //jdk1.7的源码，jdk1.8没有这个方法，但是实现原理一样的
       return h & (length-1);  //第三步 取模运算
  }
  ```

  ​		对于任意给定的对象，只要它的hashCode()返回值相同，那么程序调用方法一所计算得到的Hash码值总是相同的。把hash值对数组长度取模运算，这样一来，元素的分布相对来说是比较均匀的。通过h & (table.length -1)来得到该对象的保存位，而HashMap底层数组的长度总是2的n次方，这是HashMap在速度上的优化。当length总是2的n次方时，h& (length-1)运算等价于对length取模，也就是h%length，但是&比%具有更高的效率。

  ​		在JDK1.8的实现中，优化了高位运算的算法，通过hashCode()的高16位异或低16位实现的：(h = k.hashCode()) ^ (h >>> 16)，主要是从速度、功效、质量来考虑的，这么做可以在数组table的length比较小的时候，也能保证考虑到高低Bit都参与到Hash的计算中，同时不会有太大的开销。

  ![preview](https://wsx666-1302523054.cos.ap-nanjing.myqcloud.com/image/2021-04-03/8e8203c1b51be6446cda4026eaaccf19_r%20.jpg)

+ put方法

  ![img](https://wsx666-1302523054.cos.ap-nanjing.myqcloud.com/image/2021-04-03/58e67eae921e4b431782c07444af824e_1440w%20.png)

  ①.判断键值对数组table[i]是否为空或为null，否则执行resize()进行扩容；

  ②.根据键值key计算hash值得到插入的数组索引i，如果table[i]==null，直接新建节点添加，转向⑥，如果table[i]不为空，转向③；

  ③.判断table[i]的首个元素是否和key一样，如果相同直接覆盖value，否则转向④，这里的相同指的是hashCode以及equals；

  ④.判断table[i] 是否为treeNode，即table[i] 是否是红黑树，如果是红黑树，则直接在树中插入键值对，否则转向⑤；

  ⑤.遍历table[i]，判断链表长度是否大于8，大于8的话把链表转换为红黑树，在红黑树中执行插入操作，否则进行链表的插入操作；遍历过程中若发现key已经存在直接覆盖value即可；

  ⑥.插入成功后，判断实际存在的键值对数量size是否超多了最大容量threshold，如果超过，进行扩容。

  ```java
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
      	// 步骤①：tab为空则创建
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        // 步骤②：计算index，并对null做处理 
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
           // 步骤③：节点key存在，直接覆盖value
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
          	// 步骤④：判断该链为红黑树
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                // 步骤⑤：该链为链表
                for (int binCount = 0; ; ++binCount) {
                  	//尾插法
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                      //链表长度大于8转换为红黑树进行处理
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                   //key 存在直接覆盖
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
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
        ++modCount
       // 步骤⑥：超过最大容量 就扩容
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
  ```

+ resize

  ​		使用的是2次幂的扩展(指长度扩为原来2倍)，所以，元素的位置要么是在原位置，要么是在原位置再移动2次幂的位置。看下图可以明白这句话的意思，n为table的长度，图（a）表示扩容前的key1和key2两种key确定索引位置的示例，图（b）表示扩容后key1和key2两种key确定索引位置的示例，其中hash1是key1对应的哈希与高位运算结果。

  ![preview](https://wsx666-1302523054.cos.ap-nanjing.myqcloud.com/image/2021-04-03/a285d9b2da279a18b052fe5eed69afe9_r%20.jpg)

  元素在重新计算hash之后，因为n变为2倍，那么n-1的mask范围在高位多1bit(红色)，因此新的index就会发生这样的变化：

  ![preview](https://wsx666-1302523054.cos.ap-nanjing.myqcloud.com/image/2021-04-03/b2cb057773e3d67976c535d6ef547d51_r%20.jpg)

```java
    final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            if (oldCap >= MAXIMUM_CAPACITY) {
                // 超过最大值就不再扩充了，就只好随你碰撞去吧
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
           // 没超过最大值，就扩充为原来的2倍
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; 
        }
        else if (oldThr > 0) // new HashMap(initialCapacity) 初始化后，第一次 put 的时候
            newCap = oldThr;
        else {               // 对应使用 new HashMap() 初始化后，第一次 put 的时候
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        // 计算新的resize上限
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;

      // 用新的数组大小初始化新的数组
       Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
      // 如果是初始化数组，到这里就结束了，返回 newTab 即可
      table = newTab;
        if (oldTab != null) {
          	// 开始遍历原数组，进行数据迁移。
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                  	//如果该数组位置上只有单个元素，那就简单了，简单迁移这个元素就可以了
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                       //如果是红黑树
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { 
                       // 这块是处理链表的情况，
                      // 需要将此链表拆成两个链表，放到新的数组中，并且保留原来的先后顺序
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
														//原索引
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                           //原索引+oldCap放到
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        //
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                      	//新索引的位置
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

+ get 
  + 计算 key 的 hash 值，根据 hash 值找到对应数组下标: hash & (length-1)
  + 判断数组该位置处的元素是否刚好就是我们要找的，如果不是，走第三步
  + 判断该元素类型是否是 TreeNode，如果是，用红黑树的方法取数据，如果不是，走第四步
  + 遍历链表，直到找到相等(==或equals)的 key

  ```java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}
  ```



```java
    final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        // 判断第一个节点是不是就是需要的
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        if ((e = first.next) != null) {
            // 判断是否是红黑树
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);   // 链表遍历
        do {
            if (e.hash == hash &&
                ((k = e.key) == key || (key != null && key.equals(k))))
                return e;
        } while ((e = e.next) != null);
    }
}
return null;
}
```
### 1.7 环形链表

​		map初始化为一个长度为2的数组，loadFactor=0.75，threshold=2*0.75=1，也就是说当put第二个key的时候，map就需要进行resize。

​		通过设置断点让线程1和线程2同时debug到transfer方法(3.3小节代码块)的首行。注意此时两个线程已经成功添加数据。放开thread1的断点至transfer方法的“Entry next = e.next;” 这一行；然后放开线程2的的断点，让线程2进行resize。结果如下图

![preview](https://wsx666-1302523054.cos.ap-nanjing.myqcloud.com/image/2021-04-03/fa10635a66de637fe3cbd894882ff0c7_r%20.jpg)



![preview](https://wsx666-1302523054.cos.ap-nanjing.myqcloud.com/image/2021-04-03/d39d7eff6e8e04f98f5b53bebe2d4d7f_r%20.jpg)



​	注意，Thread1的 e 指向了key(3)，而next指向了key(7)，其在线程二rehash后，指向了线程二重组后的链表。

​	线程一被调度回来执行，先是执行 newTalbe[i] = e， 然后是e = next，导致了e指向了key(7)，而下一次循环的next = e.next导致了next指向了key(3)

![preview](https://wsx666-1302523054.cos.ap-nanjing.myqcloud.com/image/2021-04-03/5f3cf5300f041c771a736b40590fd7b1_r%20.jpg)

​	e.next = newTable[i] 导致 key(3).next 指向了 key(7)。注意：此时的key(7).next 已经指向了key(3)， 环形链表就这样出现了