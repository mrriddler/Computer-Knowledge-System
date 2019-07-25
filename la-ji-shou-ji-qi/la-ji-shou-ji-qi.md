# 垃圾收集器

  
堆管理器已经很好的管理起了堆上的内容，但是在编写程序的时候还是要对堆上内容人为的进行控制。这样编写程序即不高效、又容易出错，优化是必须的，而这种优化就叫做垃圾收集器\(Garbage Collector\)，主旨就是将程序员从内存管理的劳动中解放出来。

GC有两种管理方式：**引用计数、跟踪式**。而管理方式指的是如何识别是否对象该被回收。而跟踪式又有多种算法实现，下文就逐个聊聊这些内容。

## 引用计数\(Reference-Counting-Collector\)

引用计数这种方式比较简单。对于每个要管理的对象，记录一个被引用的计数，当被引用，计数就加一，当不被引用，计数就减一。如果计数为零，则可回收。记录所有管理对象被引用的计数自然需要一个常数级寻址的数据结构，一般使用的是哈希表，以被管理的对象的地址为键，其计数为值。

引用计数有很多种优化方式，主要就是防止引用计数频繁被更改，比如一个被管理对象如果在被引用后，马上不被引用，这一对的引用计数增减操作就可以成对省去。

实际上上文提到了引用计数的一大缺点，需要频繁更新引用计数。引用计数还有一大缺点，就是引用形成环路造成的循环引用，一般通过介入弱引用来解决此问题。

引用计数可以认为手动控制\(MRC\)，也可以自动管理\(ARC\)。

引用计数不仅可以用来管理内存，操作系统的文件资源也可以用引用计数方式管理，比如说文件描述符\(fd\)。

## 跟踪式\(Trace-Collector\)

跟踪式就是将所有被管理的对象以引用关系组织成一张有向图，然后从必定不需回收的节点作为根节点\(root\)出发，trace其引用关系，遍历所有可达节点。只要节点可达就不需回收，如果不可达就要回收。

跟踪式为自动管理，有以下几种算法：

* 标记-清除\(Mark&Sweep\)

这是最经典的算法，将整个过程分为两个阶段\(phase\)：标记和清除，首先先trace出所有被引用的节点，然后标记它们。接着在清除阶段，回收所有未被标记节点。这种算法有个严重的问题就是容易造成外部碎片。而这就影响了内存管理的一大指标使用率，所以有下文的算法在其基础上进行优化。

* 标记-压缩\(Mark&Compact\)

类似标记-清除算法，标记-压缩同样需要标记出所有被引用节点。但是下一步并不是光简单的清除未被引用的节点，还需要将被引用的节点压缩到内存上的一片连续区域，来优化外部碎片问题。

* 复制\(Copying\)

这种算法将内存分为两大空间，每次只使用一个空间。当触发回收，将使用空间中所有不需回收的对象复制到另一未使用空间，清除前一使用空间所有对象，然后交换两空间角色，使用后一空间。很明显，这种算法在不需回收对象较少时更高效，并且需要实现高效的复制。并且由于复制到另一空间，可以优化掉外部碎片问题。当然，永远使用内存的一半，也造成了严重的资源使用限制。

* 增量\(Incremental Collecting\)

由于跟踪式是集中式的触发，这种独占式处理迫使其他任务停顿，会影响到其他重要任务的实时性，这种独占式触发导致其他任务停顿叫做stop the world。而这种算法就是将内存分为若干个空间并且使用多线程，每次只回收一个或多个空间，回收完切换线程去处理其他任务，下次回收在本次回收基础上继续。这样可以使跟踪式成为并发式，并发处理跟踪式的回收和其他任务。这种算法缺点也很明显，由于切换线程等开销，会加大跟踪式的整体开销。

* 分代\(Generational Collecting\)

这种算法按被管理对象的生命周期长短使用不同的算法对其进行管理。如果几次GC触发回收仍都不需回收的对象被认为是常驻内存的对象，被称作老生代对象\(old generation\)，会被放入老生代区域，使用mark&compact算法对其进行管理。而刚被分配的对象，被称作年轻代对象\(young generation\)，会被放入年轻代区域，使用copy算法对其进行管理。

## JVM中的GC

提起GC，我想第二个在脑中的词汇不是java就是jvm。接下来就聊聊GC在java中的历史和运用，主要涉及四种GC。这些实际运用的GC会自由搭配上文提到的几种算法。首先是历史最悠久的Serial GC、Parallel GC，随后是CMS，最后是最年轻的G1。实际上，generation算法提出的比较早，涉及提到的四种GC都是在generation算法基础上设计的。

### **Serial & Parallel**

serial和parallel对old generation回收算法就是如上文所说的mark&compact，这并不是serial或parallel的重点，其真正有意思的是young generation使用的copy算法，其copy算法并不是简单的将空间分为两个区域。而是分为eden、survivor空间，而survivor又分为from、to两个子空间。新分配的对象会被加入eden空间，eden空间满后触发GC，会将eden空间中存活下来的对象copy到survivor中的from或to子空间中。而from或to子空间满后触发GC，会被copy到另一子空间中，也就是from和to子空间会符合copy算法的要求，其中一个空间为空。当对象多次触发GC都存活下来就会被放到old generation去。young generation触发的GC叫做minor GC。

而serial和parallel分别就是单线程串行和多线程并行。整体的空间布局如下：

![](../.gitbook/assets/hotspot_heap_structure.png)

其中除了提到的young generation中的eden、survivor，还有old generation和permanent generation。permanent generation包含类和对象的元数据，也会被回收，触发的GC叫做major GC。

### **CMS\(Concurrent-Mark-Sweep\)**

CMS使用的young generation算法类似parallel GC，其有意思的地方是对old generation的回收。对young generation的回收必须是stop the world，而对old generation的回收CMS尽力做到能多concurrent就多concurrent。CMS对old generation的回收分为下面几个阶段：

* initial mark：对root节点标记，这阶段会stop the world。
* concurrent mark：对所有存活节点标记，这阶段不会stop the world。
* remark：会检查新增加、新删除的对象并对它们重新标记，这个阶段会stop the world。
* concurrent sweep：回收未被标记的对象，这个阶段不会stop the world。

注意，整个回收过程不一定会进行compact，但是如果外部碎片较多，就需要在concurrent sweep后进行compact。

### **G1\(Garbage-First\)**

G1是个比较复杂的GC，这里来聊一下整体思想，对具体细节感兴趣的同学还请移步官方文档。

G1结合了上面后三个算法。首先，G1使用了incremental算法，将空间划分成若干个大小相等的空间，G1管这些空间叫region。而每次回收并不需要全量回收所有region，可以只回收部分region。根据用户设置的stop the world时间和已知的region空间大小，G1可以预估出回收多少region，每次stop the world G1只回收这个数量的region，然后切换到其他程序。这样，G1整体就可以成为一个并发式的GC了。当然，用户设置的时间，并不是绝对时间，只是参考时间。incremental算法给了G1一个可以控制stop the world时间的能力。

而由于incremental算法的限制，每次触发不一定能将所有需要回收的region全部回收完，所以造就了collection context这一概念，触发collection要继续上次的collection context继续回收region，并且这次标记完的记号不需要下次再完全重标一遍，这也包含在collection context之中。实际上，这一概念是本人编造出来帮助理解的，官方文档没有说太清楚，但是context这一概念肯定会以某种算法、形式实现、存在。这也就是，incremental命名的真正来源，每次接着上次回收的base继续回收。

如果G1每次回收只回收有限个region，可不可以挑出垃圾最多的region来回收？当然可以，但是这样需要统计所有region需要清理的垃圾，这太费时间了。不如，将incremental算法和generation算法结合起来，呈现出这样的一种效果，这就是G1所做的。将incremental算法和generation算法结合起来以后，region可以为eden region、survivor region、old region、humongous region\(大对象\)、available region\(可用\)\(humongous region和available region这里就不多聊了\)。region被分generation后，最常被回收的region自然是young region，然后是old region，虽然old region的垃圾一般会比young region少，但是G1也会挑选old region中垃圾最多的region回收。整体上来看，G1就形成了会挑选垃圾最多的region来回收。相比与以上几种GC回收整个空间的垃圾，G1做到了在有限的时间内，回收有限空间内最多的垃圾。G1拥有最高的性价比，以高回收垃圾效率为目标，所以被命名为Garbage First！

整体的空间布局如下：

![](../.gitbook/assets/g1_space.png)

G1会触发两种collection，一种是young collection，一种是mixed collection。望文生义，young collection是针对young generation发起的回收，而mixed collection是针对young和old generation发起的回收。

* Young-Collection

当JVM从young generation中的eden region分配空间，如果分配失败，也就是eden region已满就会触发young collection。为了继续上次young collection进行回收，要从young collection context中挑选出需要回收的eden region和survivor region。其中，eden region会被copy到survivor region，而survivor region根据其生命周期长短，可能会被copy到old region\(promoted\)，也可能会留在下次young collection，也可能会被放到mixed collection中去回收。

* Marking-Cycle

当已使用空间达到threshold，会启动marking cycle，结束后会触发mixed collection，而marking cycle就是帮mixed collection对old generation进行标记。

marking cycle类似CMS，分为以下几个阶段：initial mark、root region scanning、concurrent marking、remark、cleanup。initial mark会对root节点标记，这个阶段会stop the world，此阶段还会捎带\(piggybacked，吐槽一下，Oracle原文档这个词用出了境界\)上一次young collection。root region scanning会对root节点引用的old generation对象进行标记，这个阶段不会stop the world。concurrent marking会标记出heap上所有存活的对象，这个阶段不会stop the world。remark会检查新增加、新删除的对象并对它们重新标记，这个阶段会stop the world。cleanup会将挑选出一些需要被回收的old region作为候选交给接下来的mixed collection，这个阶段会stop the world。类似CMS，marking cycle会尽量concurrent起来。

* Mixed-Collection

mixed collection会从marking cycle获得的候选，根据被设置的参数，挑出一些合理的、垃圾最多\(lowest "liveness"\)的old region，和一些young region一起进行回收。不过这里要注意，虽然上文说过，young generation适合用copy算法、old generation适合用mark&compact算法，但是G1在这个阶段要回收的old generation已经是被精挑细选出的有大量垃圾region，直接使用copy算法就好。也就是说G1对old generation是相当于进行了mark&compact。不过，这个mark&compact是通过mark和copy hybrid出来的。

实际上，不光是old generation，所有region被copy的时候，都相当于是copy算法，所以G1不需要额外处理外部碎片问题。

## 比较\(comparison\)

相比于引用计数管理方式和跟踪式管理方式，引用计数不保存引用关系，而跟踪式会保存明确的引用关系，自然就不会出现循环引用。跟踪式是集中式的触发，会引起stop the world，虽然跟踪式尽力做到concurrent，但是无法避免stop the world，而引用计数是单点式的触发。相比较而言，集中式的stop the world容易造成明显的卡顿，相信大家的看过微博上一张各个语言参加运动会的图片，java本来遥遥领先，触发full GC以后，名落孙山。引用计数和跟踪式都可以自动管理，这一点上这两种方式半斤八两。如果整体比较性能的话，很难得出孰胜孰劣，在不同情况下，面对不同系统各有利弊。对于实时交互系统来说，个人认为引用计数单点式触发更胜一筹。对于内存充裕的情况下，跟踪式更胜一筹。对于内存紧张的情况下，引用计数更胜一筹。

## 引用

GC概述：[https://www.ibm.com/developerworks/cn/java/j-lo-JVMGarbageCollection/](https://www.ibm.com/developerworks/cn/java/j-lo-JVMGarbageCollection/)

Serial & Parallel & CMS： [http://www.alexleo.click/java-%E5%96%9D%E6%9D%AF%E5%92%96%E5%95%A1%EF%BC%8C%E8%81%8A%E9%BB%9E-gc%EF%BC%88%E4%B8%80%EF%BC%89-%E5%9F%BA%E7%A4%8E%E6%A6%82%E5%BF%B5/](http://www.alexleo.click/java-%E5%96%9D%E6%9D%AF%E5%92%96%E5%95%A1%EF%BC%8C%E8%81%8A%E9%BB%9E-gc%EF%BC%88%E4%B8%80%EF%BC%89-%E5%9F%BA%E7%A4%8E%E6%A6%82%E5%BF%B5/)

G1官方文档-1：[http://www.oracle.com/technetwork/tutorials/tutorials-1876574.html](http://www.oracle.com/technetwork/tutorials/tutorials-1876574.html)

G1官方文档-2：[http://www.oracle.com/technetwork/articles/java/g1gc-1984535.html](http://www.oracle.com/technetwork/articles/java/g1gc-1984535.html)

G1深入了解：[http://blog.jobbole.com/109170/](http://blog.jobbole.com/109170/)

