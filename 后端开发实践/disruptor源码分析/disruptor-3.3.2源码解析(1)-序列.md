## disruptor-3.3.2源码解析(1)-序列

作者：大飞

 

- **Disruptor中的序列-Sequence：**

​    disruptor中较为重要的一个类是Sequence。我们设想下，在disruptor运行过程中，事件发布者(生产者)和事件处理者(消费者)在ringbuffer上相互追逐，由什么来标记它们的相对位置呢？它们根据什么从ringbuffer上发布或者处理事件呢？就是这个Sequence-序列。

​    我们看一下这个类的源代码，先看结构：

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **class** LhsPadding{ 
2.   **protected** **long** p1, p2, p3, p4, p5, p6, p7; 
3. } 
4. **class** Value **extends** LhsPadding{ 
5.   **protected** **volatile** **long** value; 
6. } 
7. **class** RhsPadding **extends** Value{ 
8.   **protected** **long** p9, p10, p11, p12, p13, p14, p15; 
9. } 
10.  
11. **public** **class** Sequence **extends** RhsPadding{ 
12.   **static** **final** **long** INITIAL_VALUE = -1L; 
13.   **private** **static** **final** Unsafe UNSAFE; 
14.   **private** **static** **final** **long** VALUE_OFFSET; 
15.   **static**{ 
16. ​    UNSAFE = Util.getUnsafe(); 
17. ​    **try**{ 
18. ​      VALUE_OFFSET = UNSAFE.objectFieldOffset(Value.**class**.getDeclaredField("value")); 
19. ​    }**catch** (**final** Exception e){ 
20. ​      **throw** **new** RuntimeException(e); 
21. ​    } 
22.   } 
23.   /** 
24.    \* 默认初始值为-1 
25.    */ 
26.   **public** Sequence(){ 
27. ​    **this**(INITIAL_VALUE); 
28.   } 
29.  
30.   **public** Sequence(**final** **long** initialValue){ 
31. ​    UNSAFE.putOrderedLong(**this**, VALUE_OFFSET, initialValue); 
32.   } 

​    我们可以注意到两点：

​       1.通过Sequence的一系列的继承关系可以看到，它真正的用来计数的域是value，在value的前后各有7个long型的填充值，这些值在这里的作用是做cpu cache line填充，防止发生伪共享。

​       2.value域本身由volatile修饰，而且又看到了Unsafe类，大概猜到是要做原子操作了。

 

​    继续看一下Sequence中的方法： 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **long** get(){ 
2.   **return** value; 
3. } 
4. /** 
5.  \* ordered write 
6.  \* 在当前写操作和任意之前的读操作之间加入Store/Store屏障 
7.  */ 
8. **public** **void** set(**final** **long** value){ 
9.   UNSAFE.putOrderedLong(**this**, VALUE_OFFSET, value); 
10. } 
11. /** 
12.  \* volatile write 
13.  \* 在当前写操作和任意之前的读操作之间加入Store/Store屏障 
14.  \* 在当前写操作和任意之后的读操作之间加入Store/Load屏障 
15.  */ 
16. **public** **void** setVolatile(**final** **long** value){ 
17.   UNSAFE.putLongVolatile(**this**, VALUE_OFFSET, value); 
18. } 
19.  
20. **public** **boolean** compareAndSet(**final** **long** expectedValue, **final** **long** newValue){ 
21.   **return** UNSAFE.compareAndSwapLong(**this**, VALUE_OFFSET, expectedValue, newValue); 
22. } 
23.  
24. **public** **long** incrementAndGet(){ 
25.   **return** addAndGet(1L); 
26. } 
27.  
28. **public** **long** addAndGet(**final** **long** increment){ 
29.   **long** currentValue; 
30.   **long** newValue; 
31.   **do**{ 
32. ​    currentValue = get(); 
33. ​    newValue = currentValue + increment; 
34.   }**while** (!compareAndSet(currentValue, newValue)); 
35.   **return** newValue; 
36. } 

​    可见Sequence是一个"原子"的序列。

​    总结一下：Sequence是一个做了缓存行填充优化的原子序列。

 

​    再看个FixedSequenceGroup类： 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **final** **class** FixedSequenceGroup **extends** Sequence{ 
2.    
3.   **private** **final** Sequence[] sequences; 
4.  
5.   **public** FixedSequenceGroup(Sequence[] sequences){ 
6. ​    **this**.sequences = Arrays.copyOf(sequences, sequences.length); 
7.   } 
8.  
9.   @Override 
10.   **public** **long** get(){ 
11. ​    **return** Util.getMinimumSequence(sequences); 
12.   } 
13.  
14.   @Override 
15.   **public** **void** set(**long** value){ 
16. ​    **throw** **new** UnsupportedOperationException(); 
17.   } 
18.    
19.   ... 
20.  
21. } 

​     FixedSequenceGroup相当于包含了若干序列的一个包装类，尽管本身继承了Sequence，但只是重写了get方法，获取内部序列组中最小的序列值，但其他的"写"方法都不支持。

 

- **上面看了序列的内容，框架中也针对序列的使用，提供了专门的功能接口Sequencer：**

​    Sequencer接口扩展了Cursored和Sequenced。先看下这两个货：

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **interface** Cursored{ 
2.   **long** getCursor(); 
3. } 

​    Cursored接口只提供了一个获取当前序列值(游标)的方法。

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **interface** Sequenced{ 
2.   /** 
3.    \* 数据结构中事件槽的个数(就是RingBuffer的容量) 
4.    */ 
5.   **int** getBufferSize(); 
6.   /** 
7.    \* 判断是否还有给定的可用容量。 
8.    */ 
9.   **boolean** hasAvailableCapacity(**final** **int** requiredCapacity); 
10.   /** 
11.    \* 获取剩余容量。 
12.    */ 
13.   **long** remainingCapacity(); 
14.   /** 
15.    \* 申请下一个序列值，用来发布事件。 
16.    */ 
17.   **long** next(); 
18.   /** 
19.    \* 申请下N个序列值，用来发布事件。 
20.    */ 
21.   **long** next(**int** n); 
22.   /** 
23.    \* 尝试申请下一个序列值用来发布事件，这个是无阻塞的方法。 
24.    */ 
25.   **long** tryNext() **throws** InsufficientCapacityException; 
26.   /** 
27.    \* 尝试申请下N个序列值用来发布事件，这个是无阻塞的方法。 
28.    */ 
29.   **long** tryNext(**int** n) **throws** InsufficientCapacityException; 
30.   /** 
31.    \* 在给定的序列值上发布事件，当填充好事件后会调用这个方法。 
32.    */ 
33.   **void** publish(**long** sequence); 
34.   /** 
35.    \* 在给定的序列返回上发布事件，当填充好事件后会调用这个方法。 
36.    */ 
37.   **void** publish(**long** lo, **long** hi); 
38. } 

​    最后看下Sequencer接口： 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **interface** Sequencer **extends** Cursored, Sequenced{ 
2.   /** 序列初始值 */ 
3.   **long** INITIAL_CURSOR_VALUE = -1L; 
4.   /** 
5.    \* 声明一个序列，这个方法在初始化RingBuffer的时候被调用。 
6.    */ 
7.   **void** claim(**long** sequence); 
8.   /** 
9.    \* 判断一个序列是否被发布，并且发布到序列上的事件是可处理的。非阻塞方法。 
10.    */ 
11.   **boolean** isAvailable(**long** sequence); 
12.   /** 
13.    \* 添加一些追踪序列到当前实例，添加过程是原子的。 
14.    \* 这些控制序列一般是其他组件的序列，当前实例可以通过这些 
15.    \* 序列来查看其他组件的序列使用情况。 
16.    */ 
17.   **void** addGatingSequences(Sequence... gatingSequences); 
18.   /** 
19.    \* 移除控制序列。 
20.    */ 
21.   **boolean** removeGatingSequence(Sequence sequence); 
22.   /** 
23.    \* 基于给定的追踪序列创建一个序列栅栏，这个栅栏是提供给事件处理者 
24.    \* 在判断Ringbuffer上某个事件是否能处理时使用的。 
25.    */ 
26.   SequenceBarrier newBarrier(Sequence... sequencesToTrack); 
27.   /** 
28.    \* 获取控制序列里面当前最小的序列值。 
29.    */ 
30.   **long** getMinimumSequence(); 
31.   /** 
32.    \* 获取RingBuffer上安全使用的最大的序列值。 
33.    \* 具体实现里面，这个调用可能需要序列上从nextSequence到availableSequence之间的值。 
34.    \* 如果没有比nextSequence大的可用序列，会返回nextSequence - 1。 
35.    \* 为了保证正确，事件处理者应该传递一个比最后的序列值大1个单位的序列来处理。 
36.    */ 
37.   **long** getHighestPublishedSequence(**long** nextSequence, **long** availableSequence); 
38.  
39.   /* 
40.    \* 通过给定的数据提供者和控制序列来创建一个EventPoller 
41.    */ 
42.   <T> EventPoller<T> newPoller(DataProvider<T> provider, Sequence...gatingSequences); 
43. } 

 

​    这里要注意一下：

​       Sequencer接口的很多功能是提供给事件发布者用的。

​       通过Sequencer可以得到一个SequenceBarrier，这货是提供给事件处理者用的。

 

​    **框架中针对Sequencer接口提供了2种实现：SingleProducerSequencer和MultiProducerSequencer。**

​    看这两个类之前，先看下它们的基类AbstractSequencer：

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **abstract** **class** AbstractSequencer **implements** Sequencer{ 
2.  
3.   **private** **static** **final** AtomicReferenceFieldUpdater<AbstractSequencer, Sequence[]> SEQUENCE_UPDATER = 
4. ​      AtomicReferenceFieldUpdater.newUpdater(AbstractSequencer.**class**, Sequence[].**class**, "gatingSequences"); 
5.   **protected** **final** **int** bufferSize; 
6.   **protected** **final** WaitStrategy waitStrategy; 
7.   **protected** **final** Sequence cursor = **new** Sequence(Sequencer.INITIAL_CURSOR_VALUE); 
8.   **protected** **volatile** Sequence[] gatingSequences = **new** Sequence[0]; 
9.  
10.   **public** AbstractSequencer(**int** bufferSize, WaitStrategy waitStrategy){ 
11. ​    **if** (bufferSize < 1){ 
12. ​      **throw** **new** IllegalArgumentException("bufferSize must not be less than 1"); 
13. ​    } 
14. ​    **if** (Integer.bitCount(bufferSize) != 1){ 
15. ​      **throw** **new** IllegalArgumentException("bufferSize must be a power of 2"); 
16. ​    } 
17. ​    **this**.bufferSize = bufferSize; 
18. ​    **this**.waitStrategy = waitStrategy; 
19.   } 
20.  
21.   @Override 
22.   **public** **final** **long** getCursor(){ 
23. ​    **return** cursor.get(); 
24.   } 
25.  
26.   @Override 
27.   **public** **final** **int** getBufferSize(){ 
28. ​    **return** bufferSize; 
29.   } 
30.  
31.   @Override 
32.   **public** **final** **void** addGatingSequences(Sequence... gatingSequences){ 
33. ​    SequenceGroups.addSequences(**this**, SEQUENCE_UPDATER, **this**, gatingSequences); 
34.   } 
35.  
36.   @Override 
37.   **public** **boolean** removeGatingSequence(Sequence sequence){ 
38. ​    **return** SequenceGroups.removeSequence(**this**, SEQUENCE_UPDATER, sequence); 
39.   } 
40.  
41.   @Override 
42.   **public** **long** getMinimumSequence(){ 
43. ​    **return** Util.getMinimumSequence(gatingSequences, cursor.get()); 
44.   } 
45.  
46.   @Override 
47.   **public** SequenceBarrier newBarrier(Sequence... sequencesToTrack){ 
48. ​    **return** **new** ProcessingSequenceBarrier(**this**, waitStrategy, cursor, sequencesToTrack); 
49.   } 
50.  
51.   @Override 
52.   **public** <T> EventPoller<T> newPoller(DataProvider<T> dataProvider, Sequence... gatingSequences){ 
53. ​    **return** EventPoller.newInstance(dataProvider, **this**, **new** Sequence(), cursor, gatingSequences); 
54.   } 
55. } 

​    可见，基类基本上的作用就是管理追踪序列和关联当前序列。

 

​    先看下SingleProducerSequencer，还是先看结构：

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. bstract **class** SingleProducerSequencerPad **extends** AbstractSequencer{ 
2.   **protected** **long** p1, p2, p3, p4, p5, p6, p7; 
3.   **public** SingleProducerSequencerPad(**int** bufferSize, WaitStrategy waitStrategy){ 
4. ​    **super**(bufferSize, waitStrategy); 
5.   } 
6. } 
7. **abstract** **class** SingleProducerSequencerFields **extends** SingleProducerSequencerPad{ 
8.   **public** SingleProducerSequencerFields(**int** bufferSize, WaitStrategy waitStrategy){ 
9. ​    **super**(bufferSize, waitStrategy); 
10.   } 
11.   **protected** **long** nextValue = Sequence.INITIAL_VALUE; 
12.   **protected** **long** cachedValue = Sequence.INITIAL_VALUE; 
13. } 
14.  
15. **public** **final** **class** SingleProducerSequencer **extends** SingleProducerSequencerFields{ 
16.   **protected** **long** p1, p2, p3, p4, p5, p6, p7; 
17.  
18.   **public** SingleProducerSequencer(**int** bufferSize, **final** WaitStrategy waitStrategy){ 
19. ​    **super**(bufferSize, waitStrategy); 
20.   } 

​    又是缓存行填充，真正使用的值是nextValue和cachedValue。

 

​    再看下里面的方法实现：

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. @Override 
2. **public** **boolean** hasAvailableCapacity(**final** **int** requiredCapacity){     
3.   **long** nextValue = **this**.nextValue; 
4.   **long** wrapPoint = (nextValue + requiredCapacity) - bufferSize; 
5.   **long** cachedGatingSequence = **this**.cachedValue; 
6.   **if** (wrapPoint > cachedGatingSequence || cachedGatingSequence > nextValue){ 
7. ​    **long** minSequence = Util.getMinimumSequence(gatingSequences, nextValue); 
8. ​    **this**.cachedValue = minSequence; 
9. ​    **if** (wrapPoint > minSequence){ 
10. ​      **return** **false**; 
11. ​    } 
12.   } 
13.   **return** **true**; 
14. } 

​    hasAvailableCapacity方法可以这样理解：

​       当前序列的nextValue + requiredCapacity是事件发布者要申请的序列值。

​       当前序列的cachedValue记录的是之前事件处理者申请的序列值。

​       想一下一个环形队列，事件发布者在什么情况下才能申请一个序列呢？

​       事件发布者当前的位置在事件处理者前面，并且不能从事件处理者后面追上事件处理者(因为是环形)，

​       即 事件发布者要申请的序列值大于事件处理者之前的序列值 且 事件发布者要申请的序列值减去环的长度要小于事件处理者的序列值 

​       如果满足这个条件，即使不知道当前事件处理者的序列值，也能确保事件发布者可以申请给定的序列。

​       如果不满足这个条件，就需要查看一下当前事件处理者的最小的序列值(因为可能有多个事件处理者)，如果当前要申请的序列值比当前事件处理者的最小序列值大了一圈(从后面追上了)，那就不能申请了(申请的话会覆盖没被消费的事件)，也就是说没有可用的空间(用来发布事件)了，也就是hasAvailableCapacity方法要表达的意思。

 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. @Override 
2. **public** **long** next(){ 
3.   **return** next(1); 
4. } 
5.  
6. @Override 
7. **public** **long** next(**int** n){ 
8.   **if** (n < 1) { 
9. ​    **throw** **new** IllegalArgumentException("n must be > 0"); 
10.   } 
11.   **long** nextValue = **this**.nextValue; 
12.   **long** nextSequence = nextValue + n; 
13.   **long** wrapPoint = nextSequence - bufferSize; 
14.   **long** cachedGatingSequence = **this**.cachedValue; 
15.   **if** (wrapPoint > cachedGatingSequence || cachedGatingSequence > nextValue){ 
16. ​    **long** minSequence; 
17. ​    **while** (wrapPoint > (minSequence = Util.getMinimumSequence(gatingSequences, nextValue))){ 
18. ​      LockSupport.parkNanos(1L); // TODO: Use waitStrategy to spin? 
19. ​    } 
20. ​    **this**.cachedValue = minSequence; 
21.   } 
22.   **this**.nextValue = nextSequence; 
23.   **return** nextSequence; 
24. } 

​    next方法是真正申请序列的方法，里面的逻辑和hasAvailableCapacity一样，只是在不能申请序列的时候会阻塞等待一下，然后重试。 

 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. @Override 
2. **public** **long** tryNext() **throws** InsufficientCapacityException{ 
3.   **return** tryNext(1); 
4. } 
5.  
6. @Override 
7. **public** **long** tryNext(**int** n) **throws** InsufficientCapacityException{ 
8.   **if** (n < 1){ 
9. ​    **throw** **new** IllegalArgumentException("n must be > 0"); 
10.   } 
11.   **if** (!hasAvailableCapacity(n)){ 
12. ​    **throw** InsufficientCapacityException.INSTANCE; 
13.   } 
14.   **long** nextSequence = **this**.nextValue += n; 
15.   **return** nextSequence; 
16. } 

​    tryNext方法是next方法的非阻塞版本，不能申请就抛异常。 

 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. @Override 
2. **public** **long** remainingCapacity(){ 
3.   **long** nextValue = **this**.nextValue; 
4.   **long** consumed = Util.getMinimumSequence(gatingSequences, nextValue); 
5.   **long** produced = nextValue; 
6.   **return** getBufferSize() - (produced - consumed); 
7. } 

​    remainingCapacity方法就是环形队列的容量减去事件发布者与事件处理者的序列差。 

 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. @Override 
2. **public** **void** claim(**long** sequence){ 
3.   **this**.nextValue = sequence; 
4. } 

​    claim方法是声明一个序列，在初始化的时候用。 

 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. @Override 
2. **public** **void** publish(**long** sequence){ 
3.   cursor.set(sequence); 
4.   waitStrategy.signalAllWhenBlocking(); 
5. } 
6. @Override 
7. **public** **void** publish(**long** lo, **long** hi){ 
8.   publish(hi); 
9. } 

​    发布一个序列，会先设置内部游标值，然后唤醒等待的事件处理者。

 

​    看下剩下的方法： 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. @Override 
2. **public** **boolean** isAvailable(**long** sequence){ 
3.   **return** sequence <= cursor.get(); 
4. } 
5. @Override 
6. **public** **long** getHighestPublishedSequence(**long** lowerBound, **long** availableSequence){ 
7.   **return** availableSequence; 
8. } 

 

​    再看下MultiProducerSequencer，还是先看结构：

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **final** **class** MultiProducerSequencer **extends** AbstractSequencer{ 
2.   **private** **static** **final** Unsafe UNSAFE = Util.getUnsafe(); 
3.   **private** **static** **final** **long** BASE = UNSAFE.arrayBaseOffset(**int**[].**class**); 
4.   **private** **static** **final** **long** SCALE = UNSAFE.arrayIndexScale(**int**[].**class**); 
5.   **private** **final** Sequence gatingSequenceCache = **new** Sequence(Sequencer.INITIAL_CURSOR_VALUE); 
6.   // availableBuffer是用来记录每一个ringbuffer槽的状态。 
7.   **private** **final** **int**[] availableBuffer; 
8.   **private** **final** **int** indexMask; 
9.   **private** **final** **int** indexShift; 
10.  
11.   **public** MultiProducerSequencer(**int** bufferSize, **final** WaitStrategy waitStrategy){ 
12. ​    **super**(bufferSize, waitStrategy); 
13. ​    availableBuffer = **new** **int**[bufferSize]; 
14. ​    indexMask = bufferSize - 1; 
15. ​    indexShift = Util.log2(bufferSize); 
16. ​    initialiseAvailableBuffer(); 
17.   } 

​    MultiProducerSequencer内部多了一个availableBuffer，是一个int型的数组，size大小和RingBuffer的Size一样大，用来追踪Ringbuffer每个槽的状态，构造MultiProducerSequencer的时候会进行初始化，availableBuffer数组中的每个元素会被初始化成-1。

 

​    再看下里面的方法实现： 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. @Override 
2. **public** **boolean** hasAvailableCapacity(**final** **int** requiredCapacity){ 
3.   **return** hasAvailableCapacity(gatingSequences, requiredCapacity, cursor.get()); 
4. } 
5. **private** **boolean** hasAvailableCapacity(Sequence[] gatingSequences, **final** **int** requiredCapacity, **long** cursorValue){ 
6.   **long** wrapPoint = (cursorValue + requiredCapacity) - bufferSize; 
7.   **long** cachedGatingSequence = gatingSequenceCache.get(); 
8.   **if** (wrapPoint > cachedGatingSequence || cachedGatingSequence > cursorValue){ 
9. ​    **long** minSequence = Util.getMinimumSequence(gatingSequences, cursorValue); 
10. ​    gatingSequenceCache.set(minSequence); 
11. ​    **if** (wrapPoint > minSequence){ 
12. ​      **return** **false**; 
13. ​    } 
14.   } 
15.   **return** **true**; 
16. } 

​    逻辑和前面SingleProducerSequencer内部一样，区别是这里使用了cursor.get()，里面获取的是一个volatile的value值。 

 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. @Override 
2. **public** **long** next(**int** n){ 
3.   **if** (n < 1){ 
4. ​    **throw** **new** IllegalArgumentException("n must be > 0"); 
5.   } 
6.   **long** current; 
7.   **long** next; 
8.   **do**{ 
9. ​    current = cursor.get(); 
10. ​    next = current + n; 
11. ​    **long** wrapPoint = next - bufferSize; 
12. ​    **long** cachedGatingSequence = gatingSequenceCache.get(); 
13. ​    **if** (wrapPoint > cachedGatingSequence || cachedGatingSequence > current){ 
14. ​      **long** gatingSequence = Util.getMinimumSequence(gatingSequences, current); 
15. ​      **if** (wrapPoint > gatingSequence){ 
16. ​        LockSupport.parkNanos(1); // TODO, should we spin based on the wait strategy? 
17. ​        **continue**; 
18. ​      } 
19. ​      gatingSequenceCache.set(gatingSequence); 
20. ​    }**else** **if** (cursor.compareAndSet(current, next)){ 
21. ​      **break**; 
22. ​    } 
23.   }**while** (**true**); 
24.   **return** next; 
25. } 

​    逻辑还是一样，区别是里面的增加当前序列值是原子操作。

 

​    其他的方法都类似，都能保证多线程下的安全操作，唯一有点不同的是publish方法，看下：

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. @Override 
2. **public** **void** publish(**final** **long** sequence){ 
3.   setAvailable(sequence); 
4.   waitStrategy.signalAllWhenBlocking(); 
5. } 
6. **private** **void** setAvailable(**final** **long** sequence){ 
7.   setAvailableBufferValue(calculateIndex(sequence), calculateAvailabilityFlag(sequence)); 
8. } 
9. **private** **void** setAvailableBufferValue(**int** index, **int** flag){ 
10.   **long** bufferAddress = (index * SCALE) + BASE; 
11.   UNSAFE.putOrderedInt(availableBuffer, bufferAddress, flag); 
12. } 
13. **private** **int** calculateAvailabilityFlag(**final** **long** sequence){ 
14.   **return** (**int**) (sequence >>> indexShift); 
15. } 
16. **private** **int** calculateIndex(**final** **long** sequence){ 
17.   **return** ((**int**) sequence) & indexMask; 
18. } 

​    方法中会将当前序列值的可用状态记录到availableBuffer里面，而记录的这个值其实就是sequence除以bufferSize，也就是当前sequence绕buffer的圈数。 

 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. @Override 
2. **public** **boolean** isAvailable(**long** sequence){ 
3.   **int** index = calculateIndex(sequence); 
4.   **int** flag = calculateAvailabilityFlag(sequence); 
5.   **long** bufferAddress = (index * SCALE) + BASE; 
6.   **return** UNSAFE.getIntVolatile(availableBuffer, bufferAddress) == flag; 
7. } 
8. @Override 
9. **public** **long** getHighestPublishedSequence(**long** lowerBound, **long** availableSequence){ 
10.   **for** (**long** sequence = lowerBound; sequence <= availableSequence; sequence++){ 
11. ​    **if** (!isAvailable(sequence)){ 
12. ​      **return** sequence - 1; 
13. ​    } 
14.   } 
15.   **return** availableSequence; 
16. } 

​    isAvailable方法也好理解了，getHighestPublishedSequence方法基于isAvailable实现。

​       

 

- **大概了解了Sequencer的功能和实现，接下来看一下序列相关的一些类：**

​    首先看下SequenceBarrier这个接口： 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **interface** SequenceBarrier{ 
2.  
3.   /** 
4.    \* 等待一个序列变为可用，然后消费这个序列。. 
5.    \* 这货明显是给事件处理者使用的。 
6.    */ 
7.   **long** waitFor(**long** sequence) **throws** AlertException, InterruptedException, TimeoutException; 
8.   /** 
9.    \* 获取当前可以读取的序列值。 
10.    */ 
11.   **long** getCursor(); 
12.   /** 
13.    \* 当前栅栏是否发过通知。 
14.    */ 
15.   **boolean** isAlerted(); 
16.   /** 
17.    \* 通知事件处理者状态变化，然后停留在这个状态上，直到状态被清除。 
18.    */ 
19.   **void** alert(); 
20.   /** 
21.    \* 清楚通知状态。 
22.    */ 
23.   **void** clearAlert(); 
24.   /** 
25.    \* 检测是否发生了通知，如果已经发生了抛出AlertException异常。 
26.    */ 
27.   **void** checkAlert() **throws** AlertException; 
28. } 

​    来看一下SequenceBarrier的实现ProcessingSequenceBarrier： 

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **final** **class** ProcessingSequenceBarrier **implements** SequenceBarrier{ 
2.   //等待策略。 
3.   **private** **final** WaitStrategy waitStrategy; 
4.   //这个域可能指向一个序列组。 
5.   **private** **final** Sequence dependentSequence; 
6.   **private** **volatile** **boolean** alerted = **false**; 
7.   **private** **final** Sequence cursorSequence; 
8.   **private** **final** Sequencer sequencer; 
9.   **public** ProcessingSequenceBarrier(**final** Sequencer sequencer, 
10. ​                   **final** WaitStrategy waitStrategy, 
11. ​                   **final** Sequence cursorSequence, 
12. ​                   **final** Sequence[] dependentSequences){ 
13. ​    **this**.sequencer = sequencer; 
14. ​    **this**.waitStrategy = waitStrategy; 
15. ​    **this**.cursorSequence = cursorSequence; 
16. ​    **if** (0 == dependentSequences.length){ 
17. ​      dependentSequence = cursorSequence; 
18. ​    }**else**{ 
19. ​      dependentSequence = **new** FixedSequenceGroup(dependentSequences); 
20. ​    } 
21.   } 
22.   @Override 
23.   **public** **long** waitFor(**final** **long** sequence) 
24. ​    **throws** AlertException, InterruptedException, TimeoutException{ 
25. ​    //先检测报警状态。 
26. ​    checkAlert(); 
27. ​    //然后根据等待策略来等待可用的序列值。 
28. ​    **long** availableSequence = waitStrategy.waitFor(sequence, cursorSequence, dependentSequence, **this**); 
29. ​    **if** (availableSequence < sequence){ 
30. ​      **return** availableSequence; //如果可用的序列值小于给定的序列，那么直接返回。 
31. ​    } 
32. ​    //否则，要返回能安全使用的最大的序列值。 
33. ​    **return** sequencer.getHighestPublishedSequence(sequence, availableSequence); 
34.   } 
35.   @Override 
36.   **public** **long** getCursor(){ 
37. ​    **return** dependentSequence.get(); 
38.   } 
39.   @Override 
40.   **public** **boolean** isAlerted(){ 
41. ​    **return** alerted; 
42.   } 
43.   @Override 
44.   **public** **void** alert(){ 
45. ​    alerted = **true**; //设置通知标记 
46. ​    waitStrategy.signalAllWhenBlocking();//如果有线程以阻塞的方式等待序列，将其唤醒。 
47.   } 
48.   @Override 
49.   **public** **void** clearAlert(){ 
50. ​    alerted = **false**; 
51.   } 
52.   @Override 
53.   **public** **void** checkAlert() **throws** AlertException{ 
54. ​    **if** (alerted){ 
55. ​      **throw** AlertException.INSTANCE; 
56. ​    } 
57.   } 
58. } 

​    再看一个SequenceGroup类：

Java代码 [![收藏代码](https://www.iteye.com/images/icon_star.png)](javascript:void())

1. **public** **final** **class** SequenceGroup **extends** Sequence{ 
2.  
3.   **private** **static** **final** AtomicReferenceFieldUpdater<SequenceGroup, Sequence[]> SEQUENCE_UPDATER = 
4. ​      AtomicReferenceFieldUpdater.newUpdater(SequenceGroup.**class**, Sequence[].**class**, "sequences"); 
5.   **private** **volatile** Sequence[] sequences = **new** Sequence[0]; 
6.  
7.   **public** SequenceGroup(){ 
8. ​    **super**(-1); 
9.   } 
10.   /** 
11.    \* 获取序列组中最小的序列值。 
12.    */ 
13.   @Override 
14.   **public** **long** get(){ 
15. ​    **return** Util.getMinimumSequence(sequences); 
16.   } 
17.   /** 
18.    \* 将序列组中所有的序列设置为给定值。 
19.    */ 
20.   @Override 
21.   **public** **void** set(**final** **long** value){ 
22. ​    **final** Sequence[] sequences = **this**.sequences; 
23. ​    **for** (**int** i = 0, size = sequences.length; i < size; i++){ 
24. ​      sequences[i].set(value); 
25. ​    } 
26.   } 
27.   /** 
28.    \* 添加一个序列到序列组，这个方法只能在初始化的时候调用。 
29.    \* 运行时添加的话，使用addWhileRunning(Cursored, Sequence) 
30.    */ 
31.   **public** **void** add(**final** Sequence sequence){ 
32. ​    Sequence[] oldSequences; 
33. ​    Sequence[] newSequences; 
34. ​    **do**{ 
35. ​      oldSequences = sequences; 
36. ​      **final** **int** oldSize = oldSequences.length; 
37. ​      newSequences = **new** Sequence[oldSize + 1]; 
38. ​      System.arraycopy(oldSequences, 0, newSequences, 0, oldSize); 
39. ​      newSequences[oldSize] = sequence; 
40. ​    } **while** (!SEQUENCE_UPDATER.compareAndSet(**this**, oldSequences, newSequences)); 
41.   } 
42.   /** 
43.    \* 将序列组中出现的第一个给定的序列移除。 
44.    */ 
45.   **public** **boolean** remove(**final** Sequence sequence){ 
46. ​    **return** SequenceGroups.removeSequence(**this**, SEQUENCE_UPDATER, sequence); 
47.   } 
48.   /** 
49.    \* 获取序列组的大小。 
50.    */ 
51.   **public** **int** size(){ 
52. ​    **return** sequences.length; 
53.   } 
54.   /** 
55.    \* 在线程已经开始往Disruptor上发布事件后，添加一个序列到序列组。 
56.    \* 调用这个方法后，会将新添加的序列的值设置为游标的值。 
57.    */ 
58.   **public** **void** addWhileRunning(Cursored cursored, Sequence sequence){ 
59. ​    SequenceGroups.addSequences(**this**, SEQUENCE_UPDATER, cursored, sequence); 
60.   } 
61. } 

​    还有一个SequenceGroups类，是针对SequenceGroup的帮助类，里面提供了addSequences和removeSequence方法，都是原子操作。

 

- **最后总结：**

​    **上面看了这么多序列的相关类，其实只需要记住三点：**

​       **1.真正的序列是Sequence。**

​       **2.事件发布者通过Sequencer的大部分功能来使用序列。**

​       **3.事件处理者通过SequenceBarrier来使用序列。**

 