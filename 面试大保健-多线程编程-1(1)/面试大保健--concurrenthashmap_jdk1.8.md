
你们的大鱼哥又被虐了，额，ctm忘了具体流程，被面试官嘲笑了半天。我相信不少同学应该对ctm也有些遗忘，毕竟这个东西平时用的时候new一个挺顺手，一问原理就开始脑壳疼。

今天大鱼哥就带大家来回顾下超级面试门面--currenthashmap。话不多说让我们现在开始。

首先我们先把将这篇文章的知识储备定义在已经阅读过之前大鱼哥的hashmap那篇文章了。那我们回顾下hashmap存在的几个问题

在jdk1.7中存在的导致并发的问题，首先是扩容时产生的翻转问题，其本质就是多线程状态下进行resize后，采用的头插法导致的。在jdk1.8中采用尾插，解决了头插法导致的翻转问题，但是多线程下很多操作依旧是不安全的。

我们再来看下 在ctm中是如何解决线程不安全问题，首先讨论jdk1.7中的实现方式---段锁。然后在讨论jdk1.8中的aqs和synchronize实现。
对于一个hash，无非就几个关键点，put，get，resize。
先看下关键的几个常量

```
    private static final int MAXIMUM_CAPACITY = 1 << 30;

    private static final int DEFAULT_CAPACITY = 16;
    
    private static final float LOAD_FACTOR = 0.75f;
    
    static final int TREEIFY_THRESHOLD = 8;
    
    static final int UNTREEIFY_THRESHOLD = 6;
    
    static final int MOVED     = -1; // hash for forwarding nodes
     
    static final int TREEBIN   = -2; // hash for roots of trees
    
    static final int RESERVED  = -3; // hash for transient reservations
```

首先看下1.8 put，贴一下源码
```
 final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        //二次hash，使其分布均匀
        int hash = spread(key.hashCode());
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0)
            //tab初始化，***细讲***
                tab = initTable();
            //如果目标节点是空的
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                //通过cas的方式进行插入节点
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                  
            }
            //如果是目标节点是moved状态
            else if ((fh = f.hash) == MOVED)
                //该线程协助扩容，如何扩容下面会***细讲***
                tab = helpTransfer(tab, f);
            else {
                V oldVal = null;
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        //通过fh>0判断是否是链表
                        if (fh >= 0) {
                            binCount = 1;
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                //尾插
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        //通过f的节点属性判断是否树
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            binCount = 2;
                            //插入值
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
                //链表转树
                    if (binCount >= TREEIFY_THRESHOLD)
                    //如何转树 ，下面***细讲***
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        //node数量+1，其中需要判断是否需要transfer，transfer后面讲
        addCount(1L, binCount);
        return null;
    }
```
put 细讲-1
inittable
sizeCtl
sizeCtl ：

默认为0，用来控制table的初始化和扩容操作，具体应用在后续会体现出来。

-1 代表table正在初始化

-N 表示有N-1个线程正在进行扩容操作

其余情况：

1、如果table未初始化，表示table需要初始化的大小。

2、如果table初始化完成，表示table的容量，默认是table大小的0.75倍，居然用这个公式算0.75（n - (n >>> 2)）。

```
 private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) {
            //根据表可知 要么有其他线程已经在对其进行初始化，要么有线程在对tab进行扩容，此时等待。
            if ((sc = sizeCtl) < 0)
                Thread.yield(); // lost initialization race; just spin
            //如果检测到sc=0 说明没有完成初始化，则进行初始化
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    if ((tab = table) == null || tab.length == 0) {
                    //用 16 进行初始化
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;
                        ////sc的值改变为 n - 0.25n，也就是 0.75n
                        //所以说sizectl完成初始化就是0.75*tab.size
                        sc = n - (n >>> 2);
                    }
                } finally {
                    sizeCtl = sc;
                }
                break;
            }
        }
        return tab;
    }
```
put 细讲-2 treeifyBin

```
   private final void treeifyBin(Node<K,V>[] tab, int index) {
        Node<K,V> b; int n, sc;
        if (tab != null) {
            //如果这个tab本身就小于64，就没必要转红黑树，直接扩容
            if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
                //这里有个扩容（打个tag）,下面细讲
                tryPresize(n << 1);
            else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
                synchronized (b) {
                    //转红黑树算法
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

trypresize 即预先的resize，后面还是需要调用transfer方法实现：
```
 private final void tryPresize(int size) {
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
                        //扩容核心 ，打个tag，后面会再次用到，后面的时候仔细讲
                        transfer(tab, nt);
                }
                else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                             (rs << RESIZE_STAMP_SHIFT) + 2))
                    transfer(tab, null);
            }
        }
    }
```

put 细讲-3 addCount
```
private final void addCount(long x, int check) {
        CounterCell[] as; long b, s;
        if ((as = counterCells) != null ||
            !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
            CounterCell a; long v; int m;
            boolean uncontended = true;
            if (as == null || (m = as.length - 1) < 0 ||
                (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
                !(uncontended =
                  U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
                fullAddCount(x, uncontended);
                return;
            }
            if (check <= 1)
                return;
            s = sumCount();
        }
        if (check >= 0) {
            Node<K,V>[] tab, nt; int n, sc;
            while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                   (n = tab.length) < MAXIMUM_CAPACITY) {
                int rs = resizeStamp(n);
                if (sc < 0) {
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0)
                        break;
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    //trancefer 关键点 ***单独细讲***
                        transfer(tab, nt);
                }
                else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                             (rs << RESIZE_STAMP_SHIFT) + 2))
                    transfer(tab, null);
                s = sumCount();
            }
        }
    }

```


concurrenthashmap是相对难以理解的，其中最难以理解的部分是transfer，笔者也花了好几天功夫进行研究和学习。
要明确几个问题：

首先在迁移的时候是多线程的，如何实现多线程，透露下是通过多任务包的形式。那么这里这里需要提出几个问题：如何分解任务？如何组合任务，如何校验？

其次，普通的tab迁移/链表迁移/红黑树迁移方法其实是有很大差异的，到底是如何迁移的这里需要重点关注。

最后，几个标志位 ：transferIndex，ForwardingNode，advance，runBit，lastRun，sizeCtl需要重点掌握。

最后的最后：在进行扩容的时候，如果出现get会怎么样？

好，让我们走进transfer 源码，看看大佬是怎么写代码的。

```
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
        //n表示tab的长度
        int n = tab.length, stride;
        //单核cpu下，stride是tab的长度，多核cpu下是(n>>>3)/NCPU，最小值是 16
        //stride表示步长，即转移的单个任务的分到的长度
        if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
            stride = MIN_TRANSFER_STRIDE; // subdivide range
        //外围只能运行一个线程过来的时候，nextTab为null，开始其他时候都是带值进入方法
        if (nextTab == null) {            // initiating
            try {
                //扩容成原来长度的两倍
                Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
                nextTab = nt;
            } catch (Throwable ex) {      // try to cope with OOME
                sizeCtl = Integer.MAX_VALUE;
                return;
            }
            //全局变量 表示nexttab引用
            nextTable = nextTab;
            //全局变量，tab长度
            transferIndex = n;
        }
        //nextTable的长度
        int nextn = nextTab.length;
        //一个奇葩节点，他的hash为moved，并指向下一个tab。他有什么用呢，后面会讲到
        ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
        boolean advance = true;
        boolean finishing = false; // to ensure sweep before committing nextTab
        for (int i = 0, bound = 0;;) {
            Node<K,V> f; int fh;
            //这个循环主要是通过cas给本线程分配任务
            while (advance) {
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
                    bound = nextBound;
                    i = nextIndex - 1;
                    advance = false;
                }
            }
            //这里判断是不是已经彻底完成了，以及完成后的一些操作
            if (i < 0 || i >= n || i + n >= nextn) {
                int sc;
                //只有结束了线程才能退出
                if (finishing) {
                    nextTable = null;
                    table = nextTab;
                    sizeCtl = (n << 1) - (n >>> 1);
                    return;
                }
                //遍历
                if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                    if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                        return;
                    finishing = advance = true;
                    i = n; // recheck before commit
                }
            }
            //没有完成且节点是空的，如果是空的插入fwd，表示正在处理中
            else if ((f = tabAt(tab, i)) == null)
                advance = casTabAt(tab, i, null, fwd);
            //判断这个节点已经被处理
            else if ((fh = f.hash) == MOVED)
                advance = true; // already processed
            //转移桶
            else {
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        Node<K,V> ln, hn;
                        //迁移链表
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
                                //高低位，即通过原hash值，ph是p.hash，是随机化key.hashcode 得到，高低位是利用之前的(n-1)&hash 重新使用n&hash计算，分离高低位。
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
                        //迁移红黑树
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
get 方法
记住get方法永不阻塞。
```
    public V get(Object key) {
        Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
        int h = spread(key.hashCode());
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (e = tabAt(tab, (n - 1) & h)) != null) {
            if ((eh = e.hash) == h) {
                if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                    return e.val;
            }
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

一张解决问题的图片
![image](https://img-blog.csdnimg.cn/20190602184408984.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1pPS0VLQUk=,size_16,color_FFFFFF,t_70)
