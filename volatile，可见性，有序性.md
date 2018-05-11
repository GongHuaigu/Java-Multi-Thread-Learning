# volatile,可见性，有序性
### volatile的特性
- 可见性：对一个volatile变量的读，总能获取其他任意线程对该变量最后的写入。
- 有序性：JMM会限制volatile变量相关的编译器重排序和处理器重排序。
### 内存语义的的实现
##### 1.可见性的实现基于volatile的读取，写入两个操作的内存语义。
- **volatile写的内存语义**：当写入一个volatile变量的时候，JMM将线程工作内存中的该变量的值刷新到主内存。
- **volatile读的内存语义**：当读取一个volatile变量的时候，JMM首先将该线程工作内存中的这个变量设置为无效，迫使该线程重新从主内存获取最新的有效值。
- **两者结合起来，就实现了，volatile变量的可见性，因为一个线程去读取volatile变量的时候获取的肯定是最新的值。**
##### 2.有序性的实现基于JMM针对编译器制定的volatile重排序表
能否重排序 | - | 第二个操作 | -
---|---|---|---
第一个操作 | 普通读/写 | volatile读 | volatile写
普通读/写 |  |  | NO
volatile读 | NO | NO | NO
volatile写 |  | NO | NO
##### 3.最核心的部分还是内存屏障的使用，内存屏障实现了volatile读写的内存语义，也实现了重排序表
**理解JMM如何实现volatile的两层内存语义的关键是==内存屏障==。volatile的两层内存语义都是使用内存屏障来实现的。**    
###### 首先，对4中内存屏障的介绍： 内存屏障用于控制特定条件下的重排序和内存可见性问题。

- LoadLoad屏障：对于这样的语句Load1; LoadLoad; Load2，在Load2及后续读取操作要读取的数据被访问前，保证Load1要读取的数据被读取完毕。
- StoreStore屏障：对于这样的语句Store1; StoreStore; Store2，在Store2及后续写入操作执行前，保证Store1的写入操作对其它处理器可见。
- LoadStore屏障：对于这样的语句Load1; LoadStore; Store2，在Store2及后续写入操作被刷出前，保证Load1要读取的数据被读取完毕。
- StoreLoad屏障：对于这样的语句Store1; StoreLoad; Load2，在Load2及后续所有读取操作执行前，保证Store1的写入对所有处理器可见。它的开销是四种屏障中最大的。        在大多数处理器的实现中，这个屏障是个万能屏障，兼具其它三种内存屏障的功能。

###### 再看看JMM内存屏障的插入策略
- volatile写操作前面插入一个StoreStore屏障  
对比StoreStore屏障的定义，这里的volatile写是那个Store2,该屏障保证在volatile写执行之前，Store1的写入操作对其他处理器可见，那么可以得出，该屏障不仅保证了Store1已经执行完毕（有序性），也保证了可见性。
- 每个volatile写操作的后面插入一个StoreLoad屏障
对比StoreLoad屏障的定义，这里的volatile写是那个Store1,该屏障也保证了有序性和可见性。其他都是类似的。
- 每个volatile读操作后面插入一个LoadLoad屏障。
- 每个volatile读操作后面插入一个LoadStore屏障。  


##### 4.从CPU的角度，看看可见性如何实现
```
//假设有个volatile变量instance,对其进行写操作
instance = new Singleton();
```
在X86处理器下看看其对应的汇编代码的一部分：
```
lock add1 $0*0
```
lock前缀的指令在多核处理器下引发了两件事：
- 当前处理器缓存行的数据写回系统内存
- 使其他处理器缓存行中缓存的该内存地址的数据无效
