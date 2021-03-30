大家好，很高兴又和大家见面了。我是大鱼哥。
今天我们就讲一个关键字，对，我们的最难关键字之一--volatile。这个关键字透露着神秘和恐惧。似乎这个关键字有什么魔力，让人寸步难行。
到底这个关键字有什么用，也始终困扰着大鱼哥。今天让我们一起来学习volatile这个魔鬼关键字。
在学习这个关键字之前，我们得先了解下在操作系统层面上，cpu和内存是如何协同工作的。
## 操作系统的缓存不一致
我们都知道一个复杂的计算任务需要内存和cpu共同参与，在早期cpu的运算速度和内存的读写速度是一个级别。但是随着技术的发展，cpu的运算速度已经远远快于内存的读写速度。为了弥补这中间的速度差，在cpu和内存中间引入了缓存，即每一个核都有自己的高速缓存。而这些缓存其实并不是完全同步的。所以就引入了缓存一致性问题。在解决这个问题的时候，引入了很多协议比如msi，mesi，mosi等等。
## jvm虚拟机的缓存不一致
在jvm虚拟机中同样面临虚拟机层面的缓存不一致问题，所以jvm根据这种情况建立了java 缓存模型 jmm。

![image](https://upload-images.jianshu.io/upload_images/4222138-96ca2a788ec29dc2.png?imageMogr2/auto-orient/strip|imageView2/2/w/579/format/webp)

根据对jvm的内存分布的理解，我们知道栈是线程私有的，所以栈是不存在数据共享问题的，但是有时候变量在方法区或者在堆区，此时就需要共享一个数据。而从前面的图可以发现，每个内存其实都是主内存的一个备份，如果工作内存一直不去同步主内存，那么就会存在错误计算的问题，如何大家都保持和主内存同步呢。

这里需要用到我们的volatile关键字。volatile最大的作用的就是可见性，而可见性就是需要禁止指令重排。指令重排就会涉及 as-if-serial 和 happens before。
但这个话不能倒过来讲，实现指令重拍只有使用volatile。

好，我们提取下关键字

可见性，指令重排，as-if-serial， happens before

首先如何保证可见性。
volatile修饰的可见性实现
1. 首先 线程1首先读取到了被volatile修饰的变量
1. 然后 线程2对这个变量进行了修改，注意这里是直接set值，并不能做++运算，后面会说
1. 接着 主内存通知所有线程这个变量在自己的缓存中失败，请重新取
1. 最后 线程1重新获取了变量值，对变量赋值后写入主存

##### volatile==指令重排==
    首先什么是指令重排，指令重排是指在执行程序时，为了提高性能，编译器和处理器常常会对指令做重排序。
    volatile如何做到不让重排的？
 1.     在每个volatile写操作的前面插入一个StoreStore屏障；
 2.     在每个volatile写操作的后面插入一个StoreLoad屏障；
 2.     在每个volatile读操作的后面插入一个LoadLoad屏障；
 .      在每个volatile读操作的后面插入一个LoadStore屏障。

1. StoreStore屏障：禁止上面的普通写和下面的volatile写重排序；

1. StoreLoad屏障：防止上面的volatile写与下面可能有的volatile读/写重排序

1. LoadLoad屏障：禁止下面所有的普通读操作和上面的volatile读重排序

1. LoadStore屏障：禁止下面所有的普通写操作和上面的volatile读重排序

##### as-if-serial
    语义，即如果你是单线程执行，无论我怎么进行重排序吗，都不会影响到最终的结果。这里暗含如果你只遵守as-if-serial 那么不能保证在多线程下我进行重排序是否会影响到最终结果。
    
##### ==happens before==
    语义，如果你遵守happens before，我保证多线程见可见性。如何遵守。这里jmm提出了几种方式。
######     程序顺序规则： 对于单个线程中的每个操作，前继操作happens-before于该线程中的任意后续操作。
######     监视器锁规则： 对一个锁的解锁，happens-before于随后对这个锁的加锁。
######     volatile变量规则： 对一个volatile域的写，happens-before于任意后续对这个volatile域的读。
######     传递性： 如果A happens-before B，且B happens-before C，那么A happens-before C。

为什么LONG 加上volatile就可以是原子的？

volatile作为锁？

