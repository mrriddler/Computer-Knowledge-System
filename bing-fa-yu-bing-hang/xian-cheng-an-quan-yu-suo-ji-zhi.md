# 线程安全与锁机制

线程安全与锁机制以笔者的经验来讲，是编程中较为困难的一部分。这倒不光是因为其出现的bug不易察觉，而是归结于线程安全抽象层级比较低，其思维方式完全是计算机的运行方式，离人脑思维的方式相差甚远，人脑不适应从计算机运行的角度去思考。

本篇文章从独特的角度去理解线程安全与锁机制，并且把一些松散的锁机制知识点总结起来，统一理解。

## 线程安全\(Thread Safety\)

[线程安全](https://en.wikipedia.org/wiki/Thread_safety)没有一个绝对统一的定义，有些定义甚至是互斥的。本篇文章从理解的角度给出一个宽泛的定义，线程安全指的是，一段程序多次多线程执行结果一致，且和单线程执行结果一致。这也就是强调无论计算机如何去安排多线程工作，导致最后的执行操作的潜在顺序是一定的，不会违反开发者的意图。

而若是想要保证线程安全，最常使用的是锁机制。不过，锁机制只是最常用的一种方式。下文以不同方式分类：

* 原子操作\(atomic\)：将操作\(operation\)约束成执行过程中不可以切换线程的操作。比如，`std::atomic`。
* 内存光栅\(memory barrier\)、volatile：CPU可能动态改变不相关指令顺序，编译器也可能会为了优化改变数据的操作指令顺序或存储方式，而内存光栅和volatile分别针对CPU、编译器，约束其指令顺序、数据存储方式，保持原有现场。内存光栅保证其前后的指令顺序前后一定，CPU不可以动态交换指令越过屏障。volatile保证修饰的数据的操作指令顺序、存储方式不变。
* 不可变数据\(immutable data\)：不可变数据保证数据不可修改，这样对于数据来说，是线程安全的。
* TLS\(Thread Local Storage\)：线程自己的存储，既然是线程自己独有的，自然是线程安全的。比如，`pthread_setspecific`、`pthread_getspecific`
* 锁机制：内容较多，下文慢慢道来。

## 锁机制

理解锁机制，可以借用生活中的红绿灯。锁机制实质上都是用**信号\(signal\)**去提示操作，这也是因为并没有实质上用物理手段去阻止操作，只通过信号去提示操作。就像红绿灯一样，没有实际的墙阻止汽车启动，而是用信号去提示汽车。绿灯获取\(accquire\)，红灯释放\(release\)。

锁的种类琳琅满目，从不同维度划分多种锁，下文从不同维度分类：

* 互斥锁\(mutex\)、信号量\(semaphore\)：通过使其他线程睡眠保证，只有一个线程访问临界区\(critical section\)。信号量实际上是互斥锁的一个多元版本，而二元信号量和互斥锁还是有不同的，在有些操作系统中，互斥锁只能在同一线程上获取和释放，而二元信号量没有这种限制。比如，`pthread_mutex_t`、`sem_t`。
* 自旋锁\(spin lock\)：自旋锁和互斥锁达到同样的效果，不过，自旋锁不会使其线程睡眠，只会使其他线程在while循环中忙等待。那么，相对于互斥锁，自旋锁更加适合锁占用时间少的场景。比如，`pthread_spinlock_t`。
* 读写锁\(read-write\)：形成了写独占\(exclusive\)、读共享\(shared\)，在写稀疏、读频繁的场景下，能够有优势。比如，`pthread_rwlock_t`。
* 重入锁\(reentrant lock\)：单个线程下，多次获取和释放，仍是安全的。比如，`ReeTrantLock`。
* 同步块\(synchronized\)：同步块会被翻译成enter、leave，不同语言中的实现都是针对数据的地址添加一把重入锁，enter的时候获取，leave的时候释放。比如，`synchronized`。
* 条件锁\(condition lock\)：条件变量和锁互相协作，条件提供阻塞等待机制而不是锁，锁确保条件变量线程安全。比如，`pthread_cond_t`、`std::condition_variable`。

### 条件锁

编写条件变量和锁互相协作有个独特的模式，条件变量和数据为了保证线程安全要在临界区内，条件变量还会出现[虚唤醒\(Spurious Wakeup\)](https://en.wikipedia.org/wiki/Spurious_wakeup)。虚唤醒指的是即使条件没有达成没有发出pthread\_cond\_signal，pthread\_cond\_wait也有可能接到通知，所以才有了while循环检查。

系统默认使用者会用这种模式，pthread\_cond\_wait内部会在堵塞的时候释放锁，并在接到pthread\_cond\_signal通知后重新获取锁。这代表如果违反了此模式，根据不同平台实现可能会出现死锁的情况。

此模式代码如下：

```text
pthread_mutex_init(&_mutex, NULL);
pthread_cond_init(&_cond, NULL);
bool flag = flase;
​
void wait() {
  pthread_mutex_lock(&_mutex);
        
     while (flag) {
      pthread_cond_wait(&_cond, &_mutex);
     }
​
     pthread_mutex_unlock(&_mutex);
}
​
void signal() {
  pthread_mutex_lock(&_mutex);
  flag = true;
  pthread_cond_signal(&_cond);
    pthread_mutex_unlock(&_mutex);
}
```

### 乐观锁和悲观锁

乐观锁和悲观锁指的不是一种锁，是种数据库事务的并发机制。乐观锁认为事务冲突不会经常发生，直接对数据进行修改并用数据版本控制，等待事务提交时通过数据版本判断是否过期，如果版本过期就会重试乃至回滚。而悲观锁认为事务冲突会经常发生，并发时就直接加锁保护。

### 双重检查锁\(Double-checked locking\)

双重检查锁指的不是一种锁，而是种锁的使用模式，为保证单例模式线程安全的模式。这种模式只存在java懒加载单例模式中。双重检查锁通过双重检查减少锁的开销。代码如下：

```text
public class Singleton {
    private static Singleton instance;
    private Singleton (){}
    public static Singleton getSingleton() {
        if (null == instance) {                         
            synchronized (Singleton.class) {
                if (null == instance) {       
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

不过，这种模式之所以经典，是因为引入双重检查为了优化反而引入了问题。instance = new Singletion\(\)这行执行起来有三步：

1. 分配实例内存。
2. 调用Singleton的构造方法。
3. 将instance指向实例\(会通过checked检查\)。

而java执行此行的时候，编译器为了优化可能会交换2、3步骤的顺序。如果在多线程下，交换顺序后，就可能发生还未调用过构造方法就直接使用instance的情况。解决方法就是添加volatile，代码如下：

```text
public class Singleton {
    private volatile static Singleton instance;
    private Singleton (){}
    public static Singleton getSingleton() {
        if (null == instance) {                         
            synchronized (Singleton.class) {
                if (null == instance) {       
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

### 释放时机

锁机制只要获取，就要释放，要对等调用。对于重入锁，获取和释放也要对等。而常见的问题就是遇到了异常，锁获取却没有释放。一些语言中提供defer这样的语法，保证函数执行到最后也能释放资源，而没有这样的语法若是强制要求每次在try-catch-finaly里面添加锁释放，常在河边走，总会湿鞋。推荐使用scope技巧，c++代码如下：

```text
template <typename LockType>
class ScopedLock {
  public:
    explicit ScopedLock(LockType& lock)
        : lock_(lock), islocked_(false) {
        lock();
    }
​
    ~ScopedLock() {
        if (islocked_) unlock();
    }
​
    void lock() {
        if (!islocked_ && lock_.lock()) {
            islocked_ = true;
        }
    }
​
    void unlock() {
        if (islocked_) {
            lock_.unlock();
            islocked_ = false;
        }
    }
  private:
    LockType& lock_;
    bool islocked_;
  }
```

### 锁性能

在并发编程下，如果出现了性能问题，锁往往是脑中的罪魁祸首。锁也因为这点和性能差划上了等号，然而，任何的性能问题都要具体问题具体分析，锁不等于性能差，合理使用锁，可以保证性能不会出现问题。

[原文请移步。](http://preshing.com/20111118/locks-arent-slow-lock-contention-is/)这篇文章提示我们要使用锁要关注指标：锁竞争等待占比。

为了方便理解，将线程分为工作、等待锁两种阶段，线程不是在工作就是在等待锁。假如，一个线程获取锁以后，工作阶段持续非常长，就会导致其他线程都在等待锁，这种状况多线程的性能将会非常不理想，锁竞争等待占比越大，性能会越差劲，到最后多线程不如单线程。

## 引用

程序员的自我修养—链接、装载与库

