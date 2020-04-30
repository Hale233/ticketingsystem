# 列车售票的可线性化并发数据结构课程设计报告

------

## 题目要求

------

每位学生使用Java语言设计并完成一个用于列车售票的可线性化并发 数据结构：TicketingDS类，该类实现TicketingSystem接口，同时提供 TicketingDS ( routenum , coachnum , seatnum , stationnum , threadnum ) ; 构造函数。 其中， routenum是车次总数（缺省为20个），coachnum是列车的车厢数目（缺省为15个），seatnum是每节车厢的座位数（缺省为100个）， stationnum是每个车次经停站的数量（缺省为10个，含始发站和终点站），threadnum 是并发购票的线程数（缺省为96个）。每个线程是一个票务代理，按照80%查询余票，15%购票和5% 退票的比率反复调用TicketingDS类的三种方法若干次（缺省为总共500000次）。按照线程数为4， 8， 16， 32， 64,  96个的情况分别给出每种方法调用的平均执行时间，同时计算系统的总吞吐率（单位时间内完成的方法调用总数）。

正确性要求

- 每张车票都有一个唯一的编号tid，不能重复。
- 每一个tid的车票只能出售一次。退票后，原车票的tid作废。
- 每个区段有余票时，系统必须满足该区段的购票请求。
- 车票不能超卖，系统不能卖无座车票。
- 查询余票的约束放松到静态一致性。查询结果允许不精确，但是在某个车次查票过程中没有其他购票和退票的“ 静止”状态下， 该车次查询余票的结果必须准确。

## 分析思路

------

​	首先必须要保证程序的正确性，那么肯定需要从锁的方面来考虑。其次，要让程序吞吐率尽可能的高，这里就涉及到对整个程序的优化问题。而程序的优化和实现程序的正确性不能单独分开来讲，两者是互相影响的，比如一个大粒度的锁直接将并行变为串行访问，那么程序肯定能正确，但是性能必然不太高。所以接下来我将展示我是怎么考虑这之间的trade-off，然后对设想的验证。

​    整体上来看，程序调用实际上是针对每个车次做相应的三种操作，所以我们只需要考虑对车次进行分析即可，这里我定义了一个RouteDS的类来完成对指定车次的操作，其次在进行操作的时候，最终也会落实到对每个座位进行访问，这里最小粒度访问的实体单位其实是座位号，对应程序中的SeatDS类。



   ### 查询操作

------

​    整个过程中80%为查询操作，所以对于查询方面做优化必然对整个系统吞吐量的提升有非常大的帮助。而基于这个目的，我的考虑是从两方面来着手优化查询操作：

1. 查询操作可不可以不加锁。
2. 查询时能不能不去遍历整个车次座位。

​     针对问题一：题目要求是查询保证静态一致性，也即是在某个车次查票过程中没有其他购票和退票的“ 静止”状态下， 该车次查询余票的结果必须准确，而有其他操作时，只要结果是可能出现的都可返回。首先上锁的目的是为了保护共享资源的访问，而实际上对于查询操作来讲，并不会修改共享资源，同时也只需要静态一致性，所以理论上我们可以对查询操作不加锁。

​     针对问题二：如果不去遍历整个车次的话，只是将区间余票结果缓存下来，再访问时不需要去遍历15*100个座位上对应区间的情况，那么理论上是可以大大提高查询效率的。所以等价于要解决的问题是：谁来写缓存，缓存怎么保证是正确的。

​     谁来写缓存？这里我们首先明确哪些操作会改变缓存：1.购票操作 2.退票操作。那么基于这个原则来考虑，写缓存的任务就交给购票和退票。也许购票和退票操作只占20%，而查询占80%，那么让查询来写看起来能更快速的写满，但实际上查询来写会出现问题，因为查询的整个流程是先查缓存，查不到就去遍历，如果此时同时有AB两线程查询1-2区间的余票，而于此同时C线程在购买1-2车间上的票，假设当前只有1张1-2的余票，那么有可能A后写入1进去，把B之前写入的0覆盖了，这样后续再有线程访问该区间时，直接从缓存中得到余票数为1，显然不正确。所以基于查询操作我们必须无锁来做的话，那么查询必不能去修改缓存。实际上假设我们有10个车站，那么排列起来，总共只有(1+9)*9/2=45种组合，那么理论上购票操作也很快能把缓存写满供查询访问，实际上经过测试统计出来查询操作缓存命中率为86%，效果很好。                                 

![](https://raw.githubusercontent.com/haoheipi/picGo/master/img/20200430203804.png)
​        查询的整体流程如下图：

![](https://raw.githubusercontent.com/haoheipi/picGo/master/img/20200430203925.png)




 ### 购票、退票操作

------

​    经过上面查询操作的分析知：购票、退票除了它们自身的逻辑操作外，还需要承担写缓存的任务。所以基于这一点再来考虑设计这两个操作要解决的问题。

     1. 购票和退票是否加锁？如果需要的话锁的粒度在哪个层面上？
        2. 什么时候去写缓存？怎么写缓存能保证缓存的正确性？

​    针对问题一：首先一个座位的乘车区间在没有退票的情况下是不能买两次的，所以购票操作必须要加锁。退票该座位时，也有可能出现重复退票，所以也要保持互斥。那么我们加锁的粒度是在车次层面上还是座位层面上，这个就要和我们接下的缓存机制联合来看。

​    针对问题二：首先购票操作必须要从前开始往后遍历，实际上对于购票操作来讲，当查询到有该座位有余票时，即可返回，但为了完成写缓存的任务，该操作还需接着往后遍历到结尾，然后在写缓存。而为了保证缓存的正确性，假设我们只是在车座层次上加锁，那么当后续的遍历判断座位区间时，如果有退票，那么写缓存的结果将少一。实际上，可以分析出缓存和整个的退票、购票过程中是非常耦合，很容易出现缓存的结果不正确的问题。所以我最终考虑的结果，是个购票和退票加个大力度锁，锁在整个操作过程，而不是单单车座层面。而主要的目标转化为怎么利用已经缓存的结果去减少购票操作的阻塞时间。

​     实际设计的购票、退票具体的流程如下：

<img src="/Users/haoheipi/Downloads/未命名表单 (1).png" alt="未命名表单 (1)" style="zoom:50%;" />



    ### 正确性验证

------

对于查询余票的方法，前面已经证明了，只要确保查询方法不会读出中间状态的值时，就能满足静态一致性，而不必考虑锁的粒度。

对于购票和退票方法，由于方法是互斥的，同一时间只会有一个线程访问相应方法，所以购票退票方法是无锁的。当该线程成功买到一个区间的票后，将占用相应区间，那么之后所有的相关区间（与该区间重叠）都不再可占用，更无法购票，因此不会出现同一个座位一个车站两个人坐的情况。而退票方法也是如此，多个线程退同一个座位同一个区间的时候，只会有一个线程成功。而先操作，并成功修改区间的线程的方法调用，其可线性化点也一定在其他线程的方法调用之前。

这是针对一个座位的情况，接下来要考虑对列车所有座位整体的执行情况。由于所有线程都是从前向后按固定的顺序遍历的，因此如果一个方法调用发现一个座位无法占用，那一定是前面有方法调用先占用了该座位，其可线性化点也一定在该方法之前；而如果一个座位可以占用，要么是之前没有占用过，要么是有退票的方法退掉了该座位，其可线性化点也一定在该方法之前。同样地，对于退票方法也可以这样推演。这样说的话，我们就可以按照占用或者释放占用的时刻作为可线性化点。

针对缓存正确性的情况，首先只有购票和退票才会对缓存结果有影响，而我们对购票和退票是使用的一个大粒度锁，缓存的读写是被该锁包括在内，所以缓存的读写也是可线性化的。



### 细节方面优化

------

1. 锁是选用sychronized还是Reentrantlock？

   Synchronized：底层使用指令码方式来控制锁的，映射成字节码指令就是增加来两个指令：monitorenter和monitorexit。当线程执行遇到monitorenter指令时会尝试获取内置锁，如果获取锁则锁计数器+1，如果没有获取锁则阻塞；当遇到monitorexit指令时锁计数器-1，如果计数器为0则释放锁。

   Lock：底层是CAS乐观锁，依赖AbstractQueuedSynchronizer类，把所有的请求线程构成一个CLH队列。而对该队列的操作均通过Lock-Free（CAS）操作。

   而在JDK1.6之后，对synchronized优化，根据不同情形出现了偏向锁、轻量锁、对象锁，自旋锁（或自适应自旋锁）等。

   所以在96线程并发访问下，经过测试使用synchronized比Lock锁耗时少了一半。我觉得原因应该是synchronized的自适应自旋让线程没得到锁的时候不必让出CPU进行上下文切换，而其他操作也能很快完成，所以线程也能在自旋后及时得到锁。

2. coach和seat虽然是一个二维数组，但是可以转化为一维数组，节省内存开销和访问效率。

3. 初始化HashMap、List变量时，都预先分配好空间，减少扩容时的效率下降问题。

   等等。



## 性能测试

------

| ThreadNum (50WTestNum) | Total Execution Time(ms) | Throughput Rate(kop/s) | Query AVE Time(ns) | Buy AVE Time(ns) | Refund AVE Time(ns) |
| :--------------------- | :----------------------- | :--------------------- | :----------------- | :--------------- | :------------------ |
| 1                      | 2459                     | 203                    | 2227               | 16222            | 13812               |
| 4                      | 2800                     | 714                    | 331                | 6086             | 4407                |
| 8                      | 3156                     | 1267                   | 343                | 2803             | 1861                |
| 16                     | 3101                     | 2579                   | 232                | 1098             | 742                 |
| 32                     | 3448                     | 4640                   | 104                | 564              | 942                 |
| 64                     | 3668                     | 8742                   | 55                 | 347              | 367                 |
| 96                     | 3938                     | 12188                  | 34                 | 274              | 263                 |



## 总结

------

在整个过程设计中，不断在保证正确性的情况下做优化。然而多线程下正确性又是非常难观察到，这就需要非常小心的做设计，大多时候也只能形式化的来判断正确性。有时候有一些点子，但是实际认真思考下来发现也并不能保证正确性。其次一点是，因为这门课是并发编程，实际上我先是从无锁、乐观锁、悲观锁的粒度来做优化，但是我发现我的设计从这方面最后也只能优化到10秒左右。后来我改变思路，从利用缓存的角度来考虑，这个时候又发现原来的锁机制并不能保证程序的正确性，所以一直做调整思考，最后快到deadline了就直接粗粒度实现。后续再做优化的话还是可以再把锁机制进行调整，其次一些细节方面也可以大大提高速度。比如我在购票时候多加了个cache判空的操作，程序就从6s提升到4s。

这次大作业收获还是挺大的，首先深入了解了很多数据结构，其次也从整体上学习并发程序设计。而并发设计不仅仅是这次售票系统，也可以推广到很多方方面面，踩了这么多坑之后也一定会对以后相关的程序设计提供宝贵的经验。