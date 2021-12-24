## ConcurrentHashMap.put

```java
   
final V putVal(K key, V value, boolean onlyIfAbsent){
        //控制k 和 v 不能为null
        if (key == null || value == null) throw new NullPointerException();
    	//通过spread方法，可以让高位也能参与进寻址运算。
        int hash = spread(key.hashCode());
        int binCount = 0;
         //binCount表示当前k-v 封装成node后插入到指定桶位后，在桶位中的所属链表的下标位置
        //0 表示当前桶位为null，node可以直接放着
        //2 表示当前桶位已经可能是红黑树
        for (Node<K,V>[] tab = table;;) {
            //f 表示桶位的头结点
            //n 表示散列表数组的长度
            //i 表示key通过寻址计算后，得到的桶位下标
            //fh 表示桶位头结点的hash值
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0)
                //tab初始化
                tab = initTable();
            //头结点为空。当前桶位现在还没有数据
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                //通过CAS来设置值 成功break 跳出自旋 失败表示当前有竞继续自旋执行其他的逻辑
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            //表示当前桶位的头结点 为 FWD结点，表示目前map正处于扩容过程中
            else if ((fh = f.hash) == MOVED)
                //去帮助扩容
                tab = helpTransfer(tab, f);
            else {
                V oldVal = null;
                //头节点加锁
                synchronized (f) {
                    //判断头节点是不是之前的头节点
                    if (tabAt(tab, i) == f) {
                        //条件成立说明当前的节点是链表
                        if (fh >= 0) {
                            binCount = 1;
                             //1.当前插入key与链表当中所有元素的key都不一致时，当前的插入操作是追加到链表的末尾，binCount表示链表长度
                            //2.当前插入key与链表当中的某个元素的key一致时，当前插入操作可能就是替换了。binCount表示冲突位置（binCount - 1）
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                //当前有冲突
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                //无冲突
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {
                                    //尾插法
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                           //强制设置 binCount的值
                            binCount = 2;
                           //p 表示红黑树中如果与你插入节点的key 有冲突节点的话 ，则putTreeVal 方法 会返回冲突节点的引用。
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                }
                
                //说明当前桶位不为null，可能是红黑树 也可能是链表
                if (binCount != 0) {
                    //如果binCount>=8 表示处理的桶位一定是链表
                    if (binCount >= TREEIFY_THRESHOLD)
                        //调用转化链表为红黑树的方法
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        addCount(1L, binCount);
        return null;
    }
```

​      

## ConcurrentHashMap.initTable

```java
    private final Node<K,V>[] initTable() {
        //tab 引用map.table
        //sc sizeCtl的临时值
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) {
             //大概率就是-1，表示其它线程正在进行创建table的过程，当前线程没有竞争到初始化table的锁。
            if ((sc = sizeCtl) < 0)
                Thread.yield();
            //1.sizeCtl = 0，表示创建table数组时 使用DEFAULT_CAPACITY为大小
            //2.如果table未初始化，表示初始化大小
            //3.如果table已经初始化，表示下次扩容时的 触发条件（阈值）
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                     //重新判断防止其它线程已经初始化完毕了，然后当前线程再次初始化..导致丢失数据。
                    //条件成立，说明其它线程都没有进入过这个if块，当前线程就是具备初始化table权利了。
                    if ((tab = table) == null || tab.length == 0) {
                        //sc大于0 创建table时 使用 sc为指定大小，否则使用 16 默认值.
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        // 将这个数组赋值给 table，table 是 volatile 的
                        table = tab = nt;
                        //n >>> 2  => 等于 1/4 n     n - (1/4)n = 3/4 n => 0.75 * n
                        //sc 0.75 n 表示下一次扩容时的触发条件。
                        sc = n - (n >>> 2);
                    }
                } finally {
                    //1.如果当前线程是第一次创建map.table的线程话，sc表示的是 下一次扩容的阈值
                    //2.表示当前线程 并不是第一次创建map.table的线程，当前线程进入到else if 块 时，将
                    //sizeCtl 设置为了-1 ，那么这时需要将其修改为 进入时的值。
                    sizeCtl = sc;
                }
                break;
            }
        }
        return tab;
    }
```

## ConcurrentHashMap.addCount

```java
    private final void addCount(long x, int check) {
         //as 表示 LongAdder.cells
        //b 表示LongAdder.base
        //s 表示当前map.table中元素的数量
        CounterCell[] as; long b, s;
        
        //条件一：true->表示cells已经初始化了，当前线程应该去使用hash寻址找到合适的cell 去累加数据
        //       false->表示当前线程应该将数据累加到 base
        //条件二：false->表示写base成功，数据累加到base中了，当前竞争不激烈，不需要创建cells
        //       true->表示写base失败，与其他线程在base上发生了竞争，当前线程应该去尝试创建cells。
        if ((as = counterCells) != null ||
            !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
            //1.true->表示cells已经初始化了，当前线程应该去使用hash寻址找到合适的cell 去累加数据
            //2.true->表示写base失败，与其他线程在base上发生了竞争，当前线程应该去尝试创建cells。
            //a 表示当前线程hash寻址命中的cell
            CounterCell a;
             //v 表示当前线程写cell时的期望值
            long v;
             //m 表示当前cells数组的长度
            int m;
              //true -> 未竞争  false->发生竞争
            boolean uncontended = true;
            //条件一：as == null || (m = as.length - 1) < 0
            //true-> 表示当前线程是通过 写base竞争失败 然后进入的if块，就需要调用fullAddCount方法去扩容 或者 重试.. LongAdder.longAccumulate
            //条件二：a = as[ThreadLocalRandom.getProbe() & m]) == null   前置条件：cells已经初始化了
            //true->表示当前线程命中的cell表格是个空，需要当前线程进入fullAddCount方法去初始化 cell，放入当前位置. 
            //条件三：!(uncontended = U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x)
            //      false->取反得到false，表示当前线程使用cas方式更新当前命中的cell成功
            //      true->取反得到true,表示当前线程使用cas方式更新当前命中的cell失败，需要进入fullAddCount进行重试 或者 扩容 cells。
            if (as == null || (m = as.length - 1) < 0 ||
                (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
                !(uncontended =
                  U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
                fullAddCount(x, uncontended);
                return;
            }
            if (check <= 1)
                return;
             //获取当前散列表元素个数，这是一个期望值
            s = sumCount();
        }
        if (check >= 0) {
            //tab 表示map.table
            //nt 表示map.nextTable
            //n 表示map.table数组的长度
            //sc 表示sizeCtl的临时值
            Node<K,V>[] tab, nt; int n, sc;
           /**
             * sizeCtl < 0
             * 1. -1 表示当前table正在初始化（有线程在创建table数组），当前线程需要自旋等待..
             * 2.表示当前table数组正在进行扩容 ,高16位表示：扩容的标识戳   低16位表示：（1 + nThread） 当前参与并发扩容的线程数量
             *
             * sizeCtl = 0，表示创建table数组时 使用DEFAULT_CAPACITY为大小
             *
             * sizeCtl > 0
             *
             * 1. 如果table未初始化，表示初始化大小
             * 2. 如果table已经初始化，表示下次扩容时的 触发条件（阈值）
             */
            
            //自旋
            //条件一：s >= (long)(sc = sizeCtl)
            //       true-> 1.当前sizeCtl为一个负数 表示正在扩容中..
            //              2.当前sizeCtl是一个正数，表示扩容阈值
            //       false-> 表示当前table尚未达到扩容条件
            //条件二：(tab = table) != null
            //       恒成立 true
            //条件三：(n = tab.length) < MAXIMUM_CAPACITY
            //       true->当前table长度小于最大值限制，则可以进行扩容。
            while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                   (n = tab.length) < MAXIMUM_CAPACITY) {
               //扩容批次唯一标识戳
                //16 -> 32 扩容 标识为：1000 0000 0001 1011
                int rs = resizeStamp(n);
               //条件成立：表示当前table正在扩容
                //当前线程理论上应该协助table完成扩容
                if (sc < 0) {
                     //条件一：(sc >>> RESIZE_STAMP_SHIFT) != rs
                    //      true->说明当前线程获取到的扩容唯一标识戳 非 本批次扩容
                    //      false->说明当前线程获取到的扩容唯一标识戳 是 本批次扩容
                    //条件二： JDK1.8 中有bug jira已经提出来了 其实想表达的是 =  sc == (rs << 16 ) + 1
                    //        true-> 表示扩容完毕，当前线程不需要再参与进来了
                    //        false->扩容还在进行中，当前线程可以参与
                    //条件三：JDK1.8 中有bug jira已经提出来了 其实想表达的是 = sc == (rs<<16) + MAX_RESIZERS
                    //        true-> 表示当前参与并发扩容的线程达到了最大值 65535 - 1
                    //        false->表示当前线程可以参与进来
                    //条件四：(nt = nextTable) == null
                    //        true->表示本次扩容结束
                    //        false->扩容正在进行中
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0)
                        break;
                    
                    //前置条件：当前table正在执行扩容中.. 当前线程有机会参与进扩容。
                    //条件成立：说明当前线程成功参与到扩容任务中，并且将sc低16位值加1，表示多了一个线程参与工作
                    //条件失败：1.当前有很多线程都在此处尝试修改sizeCtl，有其它一个线程修改成功了，导致你的sc期望值与内存中的值不一致 修改失败
                    //        2.transfer 任务内部的线程也修改了sizeCtl。
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                        //协助扩容线程，持有nextTable参数
                        transfer(tab, nt);
                }
                //条件成立，说明当前线程是触发扩容的第一个线程，在transfer方法需要做一些扩容准备工作
                else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                             (rs << RESIZE_STAMP_SHIFT) + 2))
                    //触发扩容条件的线程 不持有nextTable
                    transfer(tab, null);
                s = sumCount();
            }
        }
    }
```

## ConcurrentHashMap.transfer

![JDK1.8 ConcurrentHashMap并发扩容](https://wsx666-1302523054.cos.ap-nanjing.myqcloud.com/win/JDK1.8%20ConcurrentHashMap%E5%B9%B6%E5%8F%91%E6%89%A9%E5%AE%B9.png)

```java
 private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
        //n 表示扩容之前table数组的长度
        //stride 表示分配给线程任务的步长
        int n = tab.length, stride;
        //方便讲解源码  stride 固定为 16
        if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
            stride = MIN_TRANSFER_STRIDE; // subdivide range


        //条件成立：表示当前线程为触发本次扩容的线程，需要做一些扩容准备工作
        //条件不成立：表示当前线程是协助扩容的线程..
        if (nextTab == null) {            // initiating
            try {
                //创建了一个比扩容之前大一倍的table
                @SuppressWarnings("unchecked")
                Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
                nextTab = nt;
            } catch (Throwable ex) {      // try to cope with OOME
                sizeCtl = Integer.MAX_VALUE;
                return;
            }
            //赋值给对象属性 nextTable ，方便协助扩容线程 拿到新表
            nextTable = nextTab;
            //记录迁移数据整体位置的一个标记。index计数是从1开始计算的。
            transferIndex = n;
        }

        //表示新数组的长度
        int nextn = nextTab.length;
        //fwd 节点，当某个桶位数据处理完毕后，将此桶位设置为fwd节点，其它写线程 或读线程看到后，会有不同逻辑。
        ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
        //推进标记
        boolean advance = true;
        //完成标记
        boolean finishing = false; // to ensure sweep before committing nextTab

        //i 表示分配给当前线程任务，执行到的桶位
        //bound 表示分配给当前线程任务的下界限制
        int i = 0, bound = 0;
        //自旋
        for (;;) {
            //f 桶位的头结点
            //fh 头结点的hash
            Node<K,V> f; int fh;


            /**
             * 1.给当前线程分配任务区间
             * 2.维护当前线程任务进度（i 表示当前处理的桶位）
             * 3.维护map对象全局范围内的进度
             */
            while (advance) {
                //分配任务的开始下标
                //分配任务的结束下标
                int nextIndex, nextBound;

                //CASE1:
                //条件一：--i >= bound
                //成立：表示当前线程的任务尚未完成，还有相应的区间的桶位要处理，--i 就让当前线程处理下一个 桶位.
                //不成立：表示当前线程任务已完成 或 者未分配
                if (--i >= bound || finishing)
                    advance = false;
                //CASE2:
                //前置条件：当前线程任务已完成 或 者未分配
                //条件成立：表示对象全局范围内的桶位都分配完毕了，没有区间可分配了，设置当前线程的i变量为-1 跳出循环后，执行退出迁移任务相关的程序
                //条件不成立：表示对象全局范围内的桶位尚未分配完毕，还有区间可分配
                else if ((nextIndex = transferIndex) <= 0) {
                    i = -1;
                    advance = false;
                }
                //CASE3:
                //前置条件：1、当前线程需要分配任务区间  2.全局范围内还有桶位尚未迁移
                //条件成立：说明给当前线程分配任务成功
                //条件失败：说明分配给当前线程失败，应该是和其它线程发生了竞争吧
                else if (U.compareAndSwapInt
                         (this, TRANSFERINDEX, nextIndex,
                          nextBound = (nextIndex > stride ?
                                       nextIndex - stride : 0))) {

                    bound = nextBound;
                    i = nextIndex - 1;
                    advance = false;
                }
            }

            //CASE1：
            //条件一：i < 0
            //成立：表示当前线程未分配到任务
            if (i < 0 || i >= n || i + n >= nextn) {
                //保存sizeCtl 的变量
                int sc;
                if (finishing) {
                    nextTable = null;
                    table = nextTab;
                    sizeCtl = (n << 1) - (n >>> 1);
                    return;
                }

                //条件成立：说明设置sizeCtl 低16位  -1 成功，当前线程可以正常退出
                if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                    //1000 0000 0001 1011 0000 0000 0000 0000
                    //条件成立：说明当前线程不是最后一个退出transfer任务的线程
                    if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                        //正常退出
                        return;

                    finishing = advance = true;
                    i = n; // recheck before commit
                }
            }
            //前置条件：【CASE2~CASE4】 当前线程任务尚未处理完，正在进行中

            //CASE2:
            //条件成立：说明当前桶位未存放数据，只需要将此处设置为fwd节点即可。
            else if ((f = tabAt(tab, i)) == null)
                advance = casTabAt(tab, i, null, fwd);
            //CASE3:
            //条件成立：说明当前桶位已经迁移过了，当前线程不用再处理了，直接再次更新当前线程任务索引，再次处理下一个桶位 或者 其它操作
            else if ((fh = f.hash) == MOVED)
                advance = true; // already processed
            //CASE4:
            //前置条件：当前桶位有数据，而且node节点 不是 fwd节点，说明这些数据需要迁移。
            else {
                //sync 加锁当前桶位的头结点
                synchronized (f) {
                    //防止在你加锁头对象之前，当前桶位的头对象被其它写线程修改过，导致你目前加锁对象错误...
                    if (tabAt(tab, i) == f) {
                        //ln 表示低位链表引用
                        //hn 表示高位链表引用
                        Node<K,V> ln, hn;

                        //条件成立：表示当前桶位是链表桶位
                        if (fh >= 0) {
                            //lastRun
                            //可以获取出 当前链表 末尾连续高位不变的 node
                            int runBit = fh & n;
                            Node<K,V> lastRun = f;
                            for (Node<K,V> p = f.next; p != null; p = p.next) {
                                int b = p.hash & n;
                                if (b != runBit) {
                                    runBit = b;
                                    lastRun = p;
                                }
                            }
                            //条件成立：说明lastRun引用的链表为 低位链表，那么就让 ln 指向 低位链表
                            if (runBit == 0) {
                                ln = lastRun;
                                hn = null;
                            }
                            //否则，说明lastRun引用的链表为 高位链表，就让 hn 指向 高位链表
                            else {
                                hn = lastRun;
                                ln = null;
                            }
                            for (Node<K,V> p = f; p != lastRun; p = p.next) {
                                int ph = p.hash; K pk = p.key; V pv = p.val;
                                if ((ph & n) == 0)
                                  	//构造的低位链表
                                    ln = new Node<K,V>(ph, pk, pv, ln);
                                else
                                  	//构造的高位链表
                                    hn = new Node<K,V>(ph, pk, pv, hn);
                            }
                          	//低位链表设置到原位置
                            setTabAt(nextTab, i, ln);
                          	//低位链表设置到新的位置
                            setTabAt(nextTab, i + n, hn);
                          	//就位置设置成fwd节点
                            setTabAt(tab, i, fwd);
                          	//推进标记
                            advance = true;
                        }
                        //条件成立：表示当前桶位是 红黑树 代理结点TreeBin
                        else if (f instanceof TreeBin) {
                            //转换头结点为 treeBin引用 t
                            TreeBin<K,V> t = (TreeBin<K,V>)f;

                            //低位双向链表 lo 指向低位链表的头  loTail 指向低位链表的尾巴
                            TreeNode<K,V> lo = null, loTail = null;
                            //高位双向链表 lo 指向高位链表的头  loTail 指向高位链表的尾巴
                            TreeNode<K,V> hi = null, hiTail = null;


                            //lc 表示低位链表元素数量
                            //hc 表示高位链表元素数量
                            int lc = 0, hc = 0;

                            //迭代TreeBin中的双向链表，从头结点 至 尾节点
                            for (Node<K,V> e = t.first; e != null; e = e.next) {
                                // h 表示循环处理当前元素的 hash
                                int h = e.hash;
                                //使用当前节点 构建出来的 新的 TreeNode
                                TreeNode<K,V> p = new TreeNode<K,V>
                                    (h, e.key, e.val, null, null);

                                //条件成立：表示当前循环节点 属于低位链 节点
                                if ((h & n) == 0) {
                                    //条件成立：说明当前低位链表 还没有数据
                                    if ((p.prev = loTail) == null)
                                        lo = p;
                                    //说明 低位链表已经有数据了，此时当前元素 追加到 低位链表的末尾就行了
                                    else
                                        loTail.next = p;
                                    //将低位链表尾指针指向 p 节点
                                    loTail = p;
                                    ++lc;
                                }
                                //当前节点 属于 高位链 节点
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
## ConcurrentHashMap.get

```java
    public V get(Object key) {
        Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
        //扰动函数获取桶位 
     
        int h = spread(key.hashCode());
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (e = tabAt(tab, (n - 1) & h)) != null) {
            //对比头结点hash与查询key的hash是否一致
            //条件成立：说明头结点与查询Key的hash值 完全一致
            if ((eh = e.hash) == h) {
                //完全比对 查询key 和 头结点的key
                //条件成立：说明头结点就是查询数据
                if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                    return e.val;
            }
            //条件成立：
            //1.-1  fwd 说明当前table正在扩容，且当前查询的这个桶位的数据 已经被迁移走了
            		//调用ForwardingNode.find节点来查找
            //2.-2  TreeBin节点，需要使用TreeBin 提供的find 方法查询。
                    //TreeBin.find节点来查找
            else if (eh < 0)
                return (p = e.find(h, key)) != null ? p.val : null;
            while ((e = e.next) != null) {
                if (e.hash == h &&
                    ((ek = e.key) == key || (ek != null && key.equals(ek))))
                    return e.val;
            }
        }
        return null;
    }
```

### ForwardingNode.find

```java
        Node<K,V> find(int h, Object k) {
            // loop to avoid arbitrarily deep recursion on forwarding nodes
            //tab 一定不为空
            Node<K,V>[] tab = nextTable;
            outer: for (;;) {
                //n 表示为扩容而创建的 新表的长度
                //e 表示在扩容而创建新表使用 寻址算法 得到的 桶位头结点
                Node<K,V> e; int n;

                //条件一：永远不成立
                //条件二：永远不成立
                //条件三：永远不成立
                //条件四：在新扩容表中 重新定位 hash 对应的头结点
                //true -> 1.在oldTable中 对应的桶位在迁移之前就是null
                //        2.扩容完成后，有其它写线程，将此桶位设置为了null
                if (k == null || tab == null || (n = tab.length) == 0 ||
                    (e = tabAt(tab, (n - 1) & h)) == null)
                    return null;

                //前置条件：扩容后的表 对应hash的桶位一定不是null，e为此桶位的头结点
                //e可能为哪些node类型？
                //1.node 类型
                //2.TreeBin 类型
                //3.FWD 类型

                for (;;) {
                    //eh 新扩容后表指定桶位的当前节点的hash
                    //ek 新扩容后表指定桶位的当前节点的key
                    int eh; K ek;
                    //条件成立：说明新扩容 后的表，当前命中桶位中的数据，即为 查询想要数据。
                    if ((eh = e.hash) == h &&
                        ((ek = e.key) == k || (ek != null && k.equals(ek))))
                        return e;

                    //eh<0
                    //1.TreeBin 类型    2.FWD类型（新扩容的表，在并发很大的情况下，可能在此方法 再次拿到FWD类型..）
                    if (eh < 0) {
                        if (e instanceof ForwardingNode) {
                            tab = ((ForwardingNode<K,V>)e).nextTable;
                            continue outer;
                        }
                        else
                            //说明此桶位 为 TreeBin 节点，使用TreeBin.find 查找红黑树中相应节点。
                            return e.find(h, k);
                    }

                    //前置条件：当前桶位头结点 并没有命中查询，说明此桶位是 链表
                    //1.将当前元素 指向链表的下一个元素
                    //2.判断当前元素的下一个位置 是否为空
                    //   true->说明迭代到链表末尾，未找到对应的数据，返回Null
                    if ((e = e.next) == null)
                        return null;
                }
            }
        }
```

### TreeBin.find

> TreeBin维护了两种数据结构  链表和红黑树  当在写锁状态时在链表查询。

```java
        final Node<K,V> find(int h, Object k) {
            if (k != null) {
                //e 表示循环迭代的当前节点   迭代的是first引用的链表
                for (Node<K,V> e = first; e != null; ) {
                    //s 保存的是lock临时状态
                    //ek 链表当前节点 的key
                    int s; K ek;
                    //(WAITER|WRITER) => 0010 | 0001 => 0011
                    //lockState & 0011 != 0 条件成立：说明当前TreeBin 有等待者线程 或者 目前有写操作线程正在加锁
                    if (((s = lockState) & (WAITER|WRITER)) != 0) {
                        if (e.hash == h &&
                            ((ek = e.key) == k || (ek != null && k.equals(ek))))
                            return e;
                        e = e.next;
                    }
                    //前置条件：当前TreeBin中 等待者线程 或者 写线程 都没有
                    //条件成立：说明添加读锁成功
                    else if (U.compareAndSwapInt(this, LOCKSTATE,s,s + READER)) {
                        TreeNode<K,V> r, p;
                        try {
                            //查询操作
                            p = ((r = root) == null ? null :
                                 r.findTreeNode(h, k, null));
                        } finally {
                            //w 表示等待者线程
                            Thread w;
                            //U.getAndAddInt(this, LOCKSTATE, -READER) == (READER|WAITER)
                            //1.当前线程查询红黑树结束，释放当前线程的读锁 就是让 lockstate 值 - 4
                            //(READER|WAITER) = 0110 => 表示当前只有一个线程在读，且“有一个线程在等待”
                            //当前读线程为 TreeBin中的最后一个读线程。

                            //2.(w = waiter) != null 说明有一个写线程在等待读操作全部结束。
                            if (U.getAndAddInt(this, LOCKSTATE, -READER) ==
                                (READER|WAITER) && (w = waiter) != null)
                                //使用unpark 让 写线程 恢复运行状态。
                                LockSupport.unpark(w);
                        }
                        return p;
                    }
                }
            }
            return null;
        }
```