---
title: 《深入理解Java虚拟机》---垃圾收集器与内存分配策略
date: 2016-08-23 22:43:15
tags: [读书笔记, JVM, 垃圾收集, 内存分配, GC]
---
程序计数器、虚拟机栈、本地方法栈这三个区域随线程而生，随线程而灭，这几个区域的内存分配和回收都具备确定性，在这几个区域内就不需要过多考虑回收的问题，因为方法结束或者线程结束时，内存自然就跟着回收了。而Java堆和方法区则不一样，一个接口中的多个实现类需要的内存可能不一样，一个方法中的多个分支需要的内存也可能不一样，我们只有在程序处于运行期间时才能知道会创建哪些对象，这部分内存的分配和回收都是动态的，垃圾收集器所关注的是这部分内存。
<!--more-->
## 对象已死吗
垃圾收集器在对堆进行回收前，第一件事情就是要确定这些对象中哪些还“存活”着，哪些已经“死去”。
### 引用计数法
所谓引用计数法即是给对象添加一个引用计数器，每当有一个地方引用它时，计数器值就加1；当引用失效时，计数器值就减1；任何时刻计数器为0的对象就是不可能再被使用的。
引用计数法的实现简单，判定效率也很高。但是，主流的Java虚拟机里面没有选用这种算法来进行内存管理的。其中最主要的原因是它很难解决对象之间相互循环引用的问题。
代码说话：
```
public class ReferenceCountingGC {
	public Object instance = null;
    private static  final int _1MB = 1024 * 1024;
    private byte[] bigSize = new byte[2 * _1MB];
    
    public static void testGC() {
    	ReferenceCountingGC objA = new ReferenceCountingGC();
        ReferenceCountingGC objB = new ReferenceCountingGC();
        objA.instance = objA;
        objB.instance = objB;
        
        objA = null;
        objB = null;
        System.gc();
    }
}
```
上述代码中，objA和objB这两个对象已经没法被访问了，但是它们因为相互引用着对方，导致它们的引用计数都不为0，于是引用计数器无法通知GC收集器回收它们。
### 可达性分析算法
一些主流的应用实现便是运用的这种算法。这个算法的基本思想就是通过一系列的称为“GC Roots”的对象作为起始点，从这些节点开始向下搜索，搜索所走过的路径称为引用链，当一个对象到GC Roots没有任何引用链相连时（用图论的话来说，就是从GC Roots到这个对象不可达），则证明此对象是不可用的。
![5.png](https://raw.githubusercontent.com/lirui1992/MarkdownPhotos/master/res/5.png)      ![6.png](https://raw.githubusercontent.com/lirui1992/MarkdownPhotos/master/res/6.png)
上图中，object1-4均为可用对象，object5-7为不可用对象。
可以作为GC Roots的对象包括下面几种：
1. 虚拟机栈（栈帧中的本地变量表）中引用的对象；
2. 方法区中类静态属性引用的对象；
3. 方法区中常量引用的对象；
4. 本地方法栈中JNI引用的对象。

在JDK 1.2之后，Java对引用的概念进行了扩充，将引用分为强引用、软引用、弱引用、虚引用4种，这4种引用强度依次逐渐减弱。
### 最后的挣扎
即使在可达性分析算法中不可达的对象，也并非“非死不可”的，这时候它们暂时处于“缓刑”阶段，要真正宣告一个对象死亡，至少要经历两次标记过程：如果对象在进行可达性分析后发现没有与GC Roots相连接的引用链，那它将会被第一次标记并且进行一次筛选，筛选的条件是此对象是否有必要执行finalize()方法。当对象没有覆盖finalize()方法，或者finalize方法已经被虚拟机调用过，虚拟机将这两种情况都视为“没有必要执行”。
finalize()方法是对象逃脱死亡命运的最后一次机会，如果对象要在finalize()中拯救自己，只要重新与引用链上的任何一个对象建立关联即可。
### 方法区的回收
在HotSpot虚拟机中，方法区也被称作永久代，在方法区中进行垃圾回收的性价比一般比较低。在队中，尤其是在新生代中，常规应用进行一次垃圾收集一般可以回收70%~95%的空间，而永久代的回收效率远低于此。
永久代的垃圾回收主要包括两部分内容：废弃变量和无用的类。回收废弃变量与回收堆中的对象十分类似。而判断一个类是否是“无用的类”需要满足以下三个条件：
1. 该类所有的实例都已经被回收，也就是Java堆中不存在该类的任何实例；
2. 加载该类的ClassLoader已经被回收；
3. 该类对应的java.lang.Class对象没有在任何地方被引用，无法在任何地方通过反射访问该类的方法。

## 垃圾收集算法
### 标记-清除算法
该算法分为“标记”和“清除”两个阶段：首先标记处所有需要回收的对象，在标记完成之后统一回收所有被标记的对象。
该算法简单易实施，但是有两个不足：
1. 效率问题，标记和清除的效率都不高；
2. 空间问题，标记清除后会产生大量不连续的内存碎片，空间碎片太多可能会导致以后在程序运行过程中需要分配大对象时，无法找到足够的连续内存而不得不提前触发另一次垃圾收集动作。

### 复制算法
现在的商业虚拟机多采用这种收集算法来回收新生代，新生代中的对象98%是“朝生夕死”的，所以不需要按照1:1的比例来划分内存空间，而是将内存划分为一块较大的Eden空间和两块较小的Survivor空间，这两块空间的大小比例是8：1。当回收时，将Eden和Survivor中还存活着的对象一次性地复制到另外一块Survivor空间上，最后清理掉Eden和刚才用过的Survivor空间。当Survivor空间不够用时，需要依赖其他内存（这里指老年代）进行分配担保。
### 标记整理算法
复制算法在对象存活率较高时就要进行较多的复制操作，效率将会变低，所以老年代一般不能直接选用这种算法。
根据老年代的特点，有人提出了一种“标记-整理”算法，标记过程仍然与“标记-清除”算法一样，但后续步骤不是直接对可回收对象进行清理，而是让所有存活的对象都向一端移动，然后直接清理掉端边界以外的内存。

当前的商业虚拟机都采用“分代收集”算法，即将Java堆分为新生代和老年代，新生代采用复制算法，老年代使用“标记-清理”或者“标记-整理”算法来进行回收。
## HotSpot的算法实现
### 枚举根节点
在用可达性分析算法判断对象是否需要被收集时，不可以出现分析过程中对象引用关系还在不断变化的情况，该点不满足的话分析结果准确性就无法得到保证。这点事导致GC进行时必须停顿所有Java执行线程的其中一个重要原因。
当前的主流Java虚拟机使用的都是准确式GC，当执行系统停顿下来后，并不需要一个不漏地检查完所有执行上下文和全局的引用位置。在HotSpot的实现中，是使用一组称为OopMap的数据结构来在类加载完成的时候对这些GC Roots进行标记，这样，GC在扫描的时候就可以直接得知这些信息了。
### 安全点 && 安全区域
程序执行时并非在所有的地方都能停顿下来开始GC，只有在到达安全点时才能暂停。安全点机制保证了程序执行时，在不太长的时间内就会遇到可进入GC的安全点。但是，程序不执行的时候呢？例如，当前线程没有获得CPU时间，这时候线程无法响应JVM的中断请求，这时候就需要安全区域来解决这个问题。
安全区域是指在一段代码片段中，引用关系不会发生变化。在这个区域中的任意位置开始GC都是安全的。我们可以把安全区域看做是被扩展了的安全点。
## 内存分配策略
内存分配策略包括以下几条：
1. 对象优先在Eden区域分配：大多是情况下，对象会优先在Eden区域进行分配，当Eden区域没有足够空间进行分配时，虚拟机将发起一次Minor GC。
2. 大对象直接进入老年代：所谓大对象，即是指需要大量连续内存空间的Java对象。写程序时应避免生成“朝生夕死”的“短命大对象”。经常出现大对象容易导致内存还有不少空间时就提前触发垃圾收集以获取足够的连续空间来“安置”它们。
3. 长期存活的对象将进入老年代：当新生代的对象年龄达到某个阈值时，就将会晋升到老年代。
4. 动态对象年龄判定：虚拟机并不是永远地要求对象的年龄必须达到了某个阈值才能晋升老年代，如果Survivor空间中相同年龄所有对象大小总和大于Survivor空间的一半，年龄大于或等于该年龄的对象就可以直接进入老年代。
5. 空间分配担保：在进行Minor GC之前，虚拟机会先检查老年代最大可用的连续空间是否大于新生代所有对象空间，如果这个条件成立，那么Minor GC可以确保是安全的。否则，会检查虚拟是否允许担保失败，如果允许，会检查老年代最大可用连续空间大小是否大于历次晋升到老年代对象的平均大小；如果大于，就尝试进行一次Minor GC；如果小于或者不予许担保失败，则要进行一次Full GC。