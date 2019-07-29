# 算法火花

本篇主要涉及到学习算法时的方法论。

## 学习已有的经典算法

在学习算法的时候，我们都是以学习经典算法的方式来学习算法。深入研究过算法的同学跟我说，学习算法要学习算法背后的思想。然而，这话还是像古寺里和尚说出来的话，好像算法需要“悟”，必须修炼多年才能理解。话听上去好像有道理的样子，做的时候依然做不好。最后落得，经典确实算法记下来了。出现了和经典算法一样的问题时，我直接将经典算法套上就解决了，自我感觉算法很强。出现了经典算法的变种，傻眼了，这时候就知道自己算法能力根本就不行。

这是重新思考自己学习算法的过程和方式的好机会。可以问自己一个问题，从已有的经典算法中能学到些什么？

算法设计。当遇到新的问题时，运用所学，设计出一个算法。

严谨的思维。算法设计其中一步就是，数学证明算法的正确性。锻炼出严谨的思维，对其他领域也会很有帮助。

经典算法的运用。遇到经典算法可以解决的时候，熟练使用已被证明过高效、正确的算法。

### **算法设计**

Algorithm Design by JON KLEINBERG EVA TARDOS 一书中第一章第一节如是说：

> As an opening topic, we look at an algorithmic problem that nicely illustrates many of the themes we will be emphasizing. It is motivated by some very natural and practical concerns, and from these we formulate a clean and simple statement of a problem. The algorithm to solve the problem is very clean as well, and most of our work will be spent in proving that it is correct and giving an acceptable bound on the amount of time it takes to terminate with an answer.

书中这一节以Stable Matching为例，说明了算法设计的过程。基本上来说，就是从复杂的现实情况中，简化问题，设计出解决问题的算法，证明算法的正确性，计算算法的时间复杂度。而最重的部分是证明正确性和计算时间复杂度。感兴趣的可以看这里：[http://zhangxiaoyang.me/categories/intro-to-algorithms-tutorial/intro-to-algorithms-tutorial-1.html](http://zhangxiaoyang.me/categories/intro-to-algorithms-tutorial/intro-to-algorithms-tutorial-1.html)

### **严谨的思维**

编程之美的推荐序如是说：

> 要想把程序写好，需要学习好一定的基础知识，包括编程语言、数据结构和算法。程序写得好的人通常都有缜密的思维能力和良好的数理基础，而且熟悉编程环境和编程工具。

我们工作的时候，很少会出现自己实现一个算法，或者说设计一个算法。那学习算法就没有用了吗？NONONO！算法锻炼出的严谨的思维就是高手的护身法宝。

### **经典算法的运用**

对于排序算法，就很明显了，对数据的排序是很常见的。

其他的经典应用，比如说，文本编辑器\(Ctrl+F\)查找特定模式，大部分上是BoyerMoore算法。

再比如说，打开手机导航的应用。从出发地址，去往目的地址。这就是图论中，求两地最近距离的最好应用。当然，现实情况复杂，肯定不是跑一遍Dijkstra就能解决的。

再比如说，社交应用中的推荐。通过图论的强连通分量，找出你属于的“群体”，然后推荐你所属“群体”的共同点。

当然，还有更多的算法运用在我们的生活中。值得一提的是，我见过最有意思的算法就是死锁的鸵鸟算法，像鸵鸟一样把头埋在沙子里，假装问题没有发生。生活中最常用的“算法”。

> 这几篇算法文章，需要有一定的算法基础，文章中几乎没有实现。包括算法导论，虽然对于学习算法来说，确实是导论，但是看之前，最好要看过别的算法的书籍，才能对里面的讲解有更深的理解。
>
> 愿你在文中可以找到自己长久所搜寻的。

## 引用

有很大启发的算法自学笔记 [http://zhangxiaoyang.me/category.html](http://zhangxiaoyang.me/category.html)

Princeton Theory of Algorithms Lecture [https://www.cs.princeton.edu/courses/archive/spring13/cos423/lectures.php](https://www.cs.princeton.edu/courses/archive/spring13/cos423/lectures.php)

MIT Introduction To Algorithms,Fall 2011 [https://www.youtube.com/watch?v=HtSuA80QTyo&list=PLUl4u3cNGP61Oq3tWYp6V\_F-5jb5L2iHb](https://www.youtube.com/watch?v=HtSuA80QTyo&list=PLUl4u3cNGP61Oq3tWYp6V_F-5jb5L2iHb)

MIT Introduction To Algorithms,Fall 2005 [http://open.163.com/special/opencourse/algorithms.html](http://open.163.com/special/opencourse/algorithms.html)

MIT 6.046J Design and Analysis of Algorithms,Spring 2015 [https://www.youtube.com/watch?v=2P-yW7LQr08&list=PLUl4u3cNGP6317WaSNfmCvGym2ucw3oGp](https://www.youtube.com/watch?v=2P-yW7LQr08&list=PLUl4u3cNGP6317WaSNfmCvGym2ucw3oGp)

Harvard CSCIE119 [http://www.fas.harvard.edu/~cscie119/lectures/](http://www.fas.harvard.edu/~cscie119/lectures/)

算法例题-1 [http://www.acmerblog.com/data-structure-algorithm-6107.html](http://www.acmerblog.com/data-structure-algorithm-6107.html)

算法例题-2 [http://www.geeksforgeeks.org/fundamentals-of-algorithms/\#DynamicProgramming](http://www.geeksforgeeks.org/fundamentals-of-algorithms/#DynamicProgramming)  


