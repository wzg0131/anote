# Java并发编程

[toc]

## 并发理论基础

![image-20221228113134135](./images/Java%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B/image-20221228113134135.png)

### 并发编程领域的三个核心问题

![image-20221228113121877](./images/Java%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B/image-20221228113121877.png)

#### 为什么需要并发？

从性能角度讲，我们**为了提高执行一定计算机任务的效率**，所以IO等待的时候不能让cpu闲着，所以我们把任务拆分交替执行，有了分时操作系统，出现了`并发`，后来cpu多核了又有了`并行`计算。[分工]

分工以后我们为了进一步提升效率和更加灵活地达到目的，所以我们要对任务进行组织编排，也就是对线程组织编排。**线程之间需要通信**，于是操作系统提供了一些让进程，线程之间通信的方式。[同步]

但是事物总不是完美的。并发和通信带来了较高的编程复杂度，同时也**出现了多线程并发操作共享资源的问题**。于是天下大势，分久必合，我们又要将**对共享资源的访问串行化**。所以我们根据现实世界的做法设计了了锁，信号量等等来补充这套体系。[互斥]

<u>总结：</u>

多线程是为了**提高性能**，但是多线程产生了两个新问题：**线程间协作**和**共享资源的并发安全问题**，需要我们解决。



### 可见性、原子性和有序性问题

![image-20221228114631665](./images/Java%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B/image-20221228114631665.png)

#### 核心矛盾：CPU，内存和IO设备的速度差异

CPU、内存、I/O 设备都在不断迭代，不断朝着更快的方向努力。但是，在这个快速发展的过程中，有一个**核心矛盾一直存在，就是这三者的速度差异**。**为了合理利用 CPU 的高性能，平衡这三者的速度差异**，计算机体系结构、操作系统、编译程序都做出了贡献，主要体现为：

1. **CPU 增加了缓存**，以均衡与内存的速度差异；
2. **操作系统增加了进程、线程**，以分时复用 CPU，进而均衡 CPU 与 I/O 设备的速度差异；
3. **编译程序优化指令执行次序**，使得缓存能够得到更加合理地利用。

但是这三个优化点却分别带来了多线程并发的三个问题：**可见性，原子性，有序性**。

#### 多核CPU缓存导致的可见性问题

<u>一个线程对共享变量的修改，另外一个线程能够立刻看到，我们称为**可见性**。</u>

在单核时代，所有的线程都是在一颗 CPU 上执行，因为**所有线程都是操作同一个 CPU 的缓存**，一个线程对缓存的写，对另外一个线程来说一定是可见的；当多个线程在不同的 CPU 上执行时，**这些线程操作的是不同的 CPU 缓存**，因此不具备可见性。

![image-20221228134841882](./images/Java%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B/image-20221228134841882.png)

<u>出现可见性的原因</u>：**共享变量在CPU缓存和内存中的值不一致**。



#### 线程切换带来的原子性问题

分时操作系统：

在一个时间片内，如果一个进程进行一个 IO 操作，该进程可以把自己标记为“休眠状态”并出让 CPU 的使用权，待文件读进内存，操作系统会把这个休眠的进程唤醒，唤醒后的进程就有机会重新获得 CPU 的使用权。

早期的操作系统基于进程来调度 CPU，不同进程间是不共享内存空间的，所以进程要做任务切换就要切换内存映射地址，**而一个进程创建的所有线程，都是共享一个内存空间的，所以线程做任务切换成本就很低了**。现代的操作系统都基于更轻量的线程来调度，现在我们提到的“任务切换”都是指“线程切换”。

<span id = "atomicity-question"><u>出现原子性的原因：</u></span>

高级语言中一条语句往往需要多条CPU指令完成，例如 `count += 1`，至少需要三条CPU指令：

1. 把count从内存加载到CPU寄存器。
2. 在寄存器执行+1操作。
3. 将结果写入内存。（缓存机制导致可能写入的是CPU缓存而不是内存）

**然而，操作系统做线程切换是可以发生在任何一条CPU指令完成时的，导致了高级语言语义中的单条指令不具有原子性**。

例如下图，两个线程都执行了`count += 1`，并且中间发生了切换，导致最终count的值是1而不是期望的2:

![image-20221228141107755](./images/Java%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B/image-20221228141107755.png)

<u>我们把一个或者多个操作在 CPU **执行的过程中不被中断的特性称为原子性。**</u>

#### 编译器优化带来的有序性问题

编译器为了优化性能，有时候会在不影响最终结果的情况下改变程序中语句的先后顺序。

例如Java（老版本JVM）利用双重检测锁创建单例：

```java
public class Singleton {
  static Singleton instance;
  static Singleton getInstance(){
    if (instance == null) {
      synchronized(Singleton.class) {
        if (instance == null)
          instance = new Singleton();//instance 增加volatile可以禁止重排序
        }
    }
    return instance;
  }
}
```

new一个实例，我们以为的步骤是：

1. 分配一块内存 M；
2. 在内存 M 上初始化 Singleton 对象；
3. 然后 M 的地址赋值给 instance 变量。

老版本的编译器在new操作的时候做了重排序：

1. 分配一块内存 M；
2. <u>将 M 的地址赋值给 instance 变量</u>；
3. 最后在内存 M 上初始化 Singleton 对象。

这会导致线程A将内存地址赋值给变量但还未初始化，另一个线程判断不为空返回，这时候访问对象可能导致空指针异常。



### JAVA内存模型如何解决可见性和有序性问题

[JMM](http://www.cs.umd.edu/~pugh/java/memoryModel/jsr-133-faq.html)的方案是**按需禁用缓存（可见性）以及编译优化（有序性）**，只有程序员知道什么时候需要禁用，JAVA提供的方法包括：`volatile`，`synchronized`和`final`三个关键字以及**Happens-Before规则**。

<u>Java内存模型底层怎么实现的？</u>

主要是**通过内存屏障(memory barrier)禁止重排序的**，即时编译器根据具体的底层体系架构，将这些内存屏障替换成具体的 CPU 指令。对于编译器而言，内存屏障将限制它所能做的重排序优化。而对于处理器而言，内存屏障将会导致缓存的刷新操作。比如，对于volatile，编译器将在volatile字段的读写操作前后各插入一些内存屏障。

<u>为什么定义Java内存模型？</u>

现代计算机体系大部是采用的对称多处理器的体系架构。每个处理器均有独立的寄存器组和缓存，多个处理器可同时执行同一进程中的不同线程，这里称为处理器的乱序执行。在Java中，不同的线程可能访问同一个共享或共享变量。如果任由编译器或处理器对这些访问进行优化的话，很有可能出现无法想象的问题，这里称为编译器的重排序。除了处理器的乱序执行、编译器的重排序，还有内存系统的重排序。因此Java语言规范引入了Java内存模型，通过定义多项规则对编译器和处理器进行限制，主要是针对可见性和有序性。

参考文档：

[JAVA内存模型一些问题总结](http://ifeve.com/jmm-faq/)

[JSR-133](https://www.cs.umd.edu/~pugh/java/memoryModel/jsr133.pdf)

#### volatile关键字

`volatile`在C语言里最原始的语义就是**禁用CPU缓存**，表示告诉编译器，**被`volatile`修饰的变量都必须从内存中读写**。（可见性）

在Java1.5版本，JMM对volatile的语义进行了增强：**volatile变量的写操作对读操作可见**（Happens-Before规则约束了编译优化，或者说**禁用编译重排序**）。（有序性）

例子:

```java
class VolatileExample {
  int x = 0;
  volatile boolean v = false;
  public void writer() {
    x = 42;
    v = true;
  }
  public void reader() {
    if (v == true) {
      	// 这里x会是多少呢？
        //jdk5以前可能是0也可能是42，jdk5以后是42
    }
  }
}
```

#### Happens-Before规则

Happens-Before 并不是说前面一个操作发生在后续操作的前面，它真正要表达的是：**前面一个（对共享变量的）操作的结果对后续操作是==可见的==**。就像有心灵感应的两个人，虽然远隔千里，一个人心之所想，另一个人都看得到。Happens-Before 规则就是要保证线程之间的这种“心灵感应”。所以比较正式的说法是：**Happens-Before 约束了编译器的优化行为**，虽允许编译器优化，但是要求编译器优化后一定遵守 Happens-Before 规则。（保证有序性）

<u>Happens-Before的7个规则：</u>

1. **程序次序规则**：在**一个线程内**，按照程序代码顺序，书写在前面的操作Happens-Before于书写在后面的操作。准确地说，应该是控制流顺序而不是程序代码顺序，因为要考虑分支、循环等结构。 
2. **管程锁定规则**：一个unlock操作Happens-Before于后面对**同一个锁**的lock操作。这里必须强调的是同一个锁，而"后面"是指时间上的先后顺序。 
3. **volatile变量规则**：对**一个volatile变量**的写操作Happens-Before于后面对这个变量的读操作，这里的"后面"同样是指时间上的先后顺序。
4. **线程启动规则**：Thread对象的`start()`方法Happens-Before于此线程的每一个动作。 
5. **线程终止规则**：线程中的所有操作都Happens-Before于对此线程的终止检测，我们可以通过`Thread.join()`方法结束、`Thread.isAlive()`的返回值等手段检测到线程已经终止执行。
6. **线程中断规则**：对线程`interrupt()`方法的调用Happens-Before于被中断线程的代码检测到中断事件的发生，可以通过`Thread.interrupted()`方法检测到是否有中断发生。
7. **对象终结规则**：**一个对象**的初始化完成(构造函数执行结束)Happens-Before于它的`finalize()`方法的开始。

<u>Happens-Before的1个特性：</u>传递性。A Happens-Before B，B Happens-Before C可以推出A Happens-Before C。



#### final关键字

`final`告诉编译器这个变量不会变，可以随便优化。（java1.5以前可能会优化出问题：[例子](http://www.cs.umd.edu/~pugh/java/memoryModel/jsr-133-faq.html#finalWrong)）

final修饰的实例字段涉及到新建对象的发布问题。**当一个对象包含final修饰的实例字段时，其他线程能够看到已经初始化的final实例字段**，这是安全的。在 1.5 以后 Java 内存模型对 final 类型变量的重排进行了约束。现在只要我们提供正确构造函数没有“逸出”，就不会出问题了。逸出指在构造函数中将this复制给全局变量，例如：

```java
// 错误的构造函数
public FinalFieldExample() { 
  x = 3;
  y = 4;
  // 此处就是讲this逸出，
  global.obj = this;
}
```

### 互斥锁

#### 解决原子性问题

一个或者多个操作在 CPU 执行的过程中不被中断的特性，称为“原子性”。

原子性问题的源头是**线程切换**（通过CPU 中断），但是禁用线程切换不能完全解决[原子性问题](#atomicity-question)：

禁用线程切换，只能保证每个CPU核心上的线程可以不间断地执行，对单核来说具有原子性，但是**不能保证同一处代码同一时刻只有一个线程在执行**，即对于多核CPU来说不满足原子性（例如两个线程可以同时执行同一段代码，修改同一个共享变量），存在线程安全的问题。

<u>如何保证原子性？</u>

“**同一时刻只有一个线程执行**”这个条件我们称之为**互斥**。如果我们能够保证**对共享变量的修改是互斥的**，那么，无论是单核 CPU 还是多核 CPU，就都能保证原子性了。

<u>锁模型：</u>

![image-20221228200207018](./images/Java%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B/image-20221228200207018.png)

#### synchronized关键字

锁的一种实现，隐藏了加锁释放锁，通过代码块来划分临界区。

例子：

```java
class SafeCalc {
  	long value = 0L;
  	synchronized void addOne() {//基于管程Happens-Before规则，多线程调用addOne不再有问题
    	value += 1;
  	}
    long get() {//get方法没有加锁，所以可见性不能得到保障
    	return value;
  	}
}
```

上面的例子中，一般对一个变量的setter方法加锁，也要对getter方法加锁。

#### 锁和受保护资源的关系

受保护资源和锁之间的关联关系是 **N:1** 的关系。**一把锁可以保护多个资源**，但是**不能用多把锁保护一个资源**，例如：

```java
class SafeCalc {
  	static long value = 0L;
  	synchronized static void addOne() {
    	value += 1;
  	}
    long get() {
    	return value;
  	}
}
```

上面例子中，两个锁分别是`SafeCalc.class`和`this`，**两个临界区之间没有互斥关系**，临界区 `addOne()` 对 `value` 的修改对临界区 get() 也没有可见性保证，这就导致并发问题了。



<u>一把锁保护多个资源的使用方式：</u>

- 如果多个资源不相关，可以每个资源一把锁，减小锁的粒度能提高并发性能。
- 如果多个资源相关，就要选择粒度更大的锁，这个锁应该能覆盖所有资源。

多个资源的关联关系，其实是一种原子性，本质上是多个资源间有一致性的要求，**操作的中间状态对外不可见**。



#### 如何预防死锁？

<u>死锁：</u>**一组互相竞争资源的线程因互相等待，导致“永久”阻塞的现象**。

![image-20221228204242272](./images/Java%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B/image-20221228204242272.png)

<u>一个死锁的例子：</u>

```java
class Account {
    private int balance;

    // 转账
    void transfer(Account target, int amt) {
        // 锁定转出账户
        synchronized (this) {<1>
            // 锁定转入账户
            synchronized (target) {<2>
                if (this.balance > amt) {
                    this.balance -= amt;
                    target.balance += amt;
                }
            }
        }
    }
}
```

如果两个`Account`对象同时在两个CPU上执行`transfer`方法，分别持有自身对象的锁，并等待对方账号对象的锁，就会产生死锁。



<u>解决死锁问题最好的办法还是规避死锁</u>。**死锁产生的条件**需要以下四个都发生：

1. 互斥，共享资源 X 和 Y 只能被一个线程占用；
2. 占有且等待，线程 T1 已经取得共享资源 X，在等待共享资源 Y 的时候，不释放共享资源 X；
3. 不可抢占，其他线程不能强行抢占线程 T1 占有的资源；循环等待，
4. 线程 T1 等待线程 T2 占有的资源，线程 T2 等待线程 T1 占有的资源，就是循环等待。

<u>只需要破坏2,3,4任意一个条件，就可以避免死锁，具体如何做呢？</u>

1. 对于“占用且等待”这个条件，我们可以**一次性申请所有的资源**，这样就不存在等待了。
2. 对于“不可抢占”这个条件，占用部分资源的线程进一步申请其他资源时，**如果申请不到，可以主动释放它占有的资源(设置超时时间)**，这样不可抢占这个条件就破坏掉了。（**sychronized不能做到**，需要用JUC包中其他的锁）
3. 对于“循环等待”这个条件，可以**靠按序申请资源来预防**。所谓按序申请，是指资源是有线性顺序的，申请的时候可以先申请资源序号小的，再申请资源序号大的，这样线性化后自然就不存在循环了。

<span id = "lock-demo"><u>预防死锁的转账例子：</u></span>

```java
class Allocator {//单例
    private List<Object> als =
            new ArrayList<>();
    // 一次性申请所有资源：将“申请两个资源”作为一个临界区
    synchronized boolean apply(Object from, Object to){<1>
        if(als.contains(from) ||
                als.contains(to)){
            return false;
        } else {
            als.add(from);
            als.add(to);
        }
        return true;
    }
    // 归还资源
    synchronized void free(
            Object from, Object to){
        als.remove(from);
        als.remove(to);
    }
}
class Account {
    private Allocator allocator;
    private int balance;

    void transfer(Account target, int amt) {
        //自旋直到拿到锁
        while (!allocator.apply(this, target)) ;

        try {
            //只有一个线程能进到这里，不担心死锁
            synchronized (this) {//balance可能有别的方法在用,所以还得加锁
                synchronized (target) {
                    if (this.balance > amt) {
                        this.balance -= amt;
                        target.balance += amt;
                    }
                }
            }
        } finally {
            allocator.free(this, target);
        }
    }
}
```

<1>: 这个例子**跟直接使用Account.class作为锁的区别**是：粒度更小，Allocator的`apply`虽然也是全局串行的，但好在这个方法基本不耗时，**只有“获取多个资源”这个临界区，没有锁住业务操作**，所以可以A转B，C转D并行。



### 等待-通知机制

在前面转账的例子中，为了“获取两个账号资源”，需要不停**自旋**，**如果并发量很大或者执行apply方法比较耗时的话，会非常消耗CPU**。

```java
 //自旋直到拿到锁
        while (!allocator.apply(this, target)) ;
```

在这种场景下，最好的方案应该是：如果线程要求的条件（同时获得两个账号资源）不满足，则**线程阻塞自己，进入等待状态**；当线程要求的条件（同时获得两个账号资源）满足后，**通知等待的线程重新执行**。其中，使用**<u>线程阻塞的方式就能避免循环等待消耗 CPU 的问题</u>**。

<u>一个完整的**等待 - 通知**机制：</u>**线程首先获取互斥锁，当线程要求的条件不满足时，==释放互斥锁，进入等待（阻塞）状态==；当要求的条件满足时，通知等待的线程，==重新获取互斥锁，并从阻塞的地方开始执行==**。



#### <span id = "wait-notify">用 synchronized 实现等待 - 通知机制</span>

在Java中，等待-通知机制有多种实现，比如语言内置的`synchronized` 配合 `wait()`、`notify()`、`notifyAll()` 这三个方法就能实现。

**<u>原理:</u>**

每个锁(Object对象)都有一个自己**锁等待队列/锁池**，存放竞争锁的线程的ID:

![image-20221229140358257](./images/Java%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B/image-20221229140358257.png)

某个线程获取锁后进入临界区，如果这时候“不满足条件”，就调用`wait()`（这里的锁是this，所以调用的是`this.wati()`）阻塞自己，进入**对象等待队列/等待池**，**并释放锁（消除对象头中的锁标志）**，这样其他线程就有机会获得锁。

当条件满足后，同过调用锁的`notify()`或`notifyAll()`方法，通知对象等待队列中其他阻塞的线程（**转移到锁等待队列重新竞争锁**），条件**曾今**满足过（可能等到这个线程抢到锁的时候条件又不满足了）：

![image-20221229141039226](./images/Java%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B/image-20221229141039226.png)

NOTE: `wait()`、`notify()`、`notifyAll()` 这三个方法**能够被调用的前提是已经获取了相应的互斥锁**，所以只能在 synchronized{}内部被调用，否则会抛出`java.lang.IllegalMonitorStateException`。



<u>**等待-通知的编程范式**</u>：

```java
while(condition()) {
    wait();
}
```

经典写法，这种范式很好解决了“**条件曾经满足过**”这个问题，因为wait()返回时，有可能又不满足条件了。

以[预防死锁的转账例子](#lock-demo)，使用等待-通知机制改进：

```java
class Allocator {
    private List<Object> als;

    // 一次性申请所有资源
    synchronized void apply(
            Object from, Object to) {
        // 经典写法
        while (als.contains(from) || als.contains(to)) {
            try {
                wait();//阻塞
            } catch (InterruptedException ignored) {
            }
        }
        als.add(from);
        als.add(to);
    }

    // 归还资源
    synchronized void free(
            Object from, Object to) {
        als.remove(from);
        als.remove(to);
        notifyAll();//通知所有阻塞的线程
    }
}
```



**<u>notify和notifyAll的选择：</u>**

**`notify()`是随机通知等待池中的一个线程**，有时候可能会导致线程一直挂起的问题：

在上面例子中，假设线程1申请到AB资源，线程2申请到CD，线程3申请AB，线程4申请CD，因为条件不满足，3和4会进入等待池。如果这时候线程1归还了资源，并用notify()随机唤醒了线程4：这时候4还是不满足，又回到等待池，**而此时本来AB资源没有被使用，线程1却处于挂起状态，没有机会被唤醒**。

所以：

- 如果所有阻塞的线程等待条件是不同的，如上面例子，应该用`notifyAll()`；
- 满足以下所有条件，则可以用`notify()`：
  - 所有等待线程拥有相同的等待条件；
  - 有等待线程被唤醒后，执行相同的操作；
  - 只需要唤醒一个线程。




**<u>`Object#wait()`和`Thread#sleep()`都会阻塞线程，他们的区别？</u>**

1. `wait()`被调用的前提是获取锁，`sleep()`不需要。
2. `wait()`会释放锁，`sleep()`不会。
3. `sleep()`方法需要指定等待的时间









### 写代码的时候需要考虑的三个问题(宏观)

并发编程中我们需要注意的问题：安全性问题、活跃性问题和性能问题。

#### 线程安全

理论上线程安全的程序，就要避免出现<u>原子性问题、可见性问题和有序性问题</u>。

<u>什么情况下需要考虑程序的线程安全呢？</u>

只有一种情况需要：**存在共享数据并且该数据会发生变化**，通俗地讲就是**有多个线程会同时读写同一数据**。



<u>竞态条件：</u>

竞态条件(Race Condition)指的是**程序的执行结果依赖线程执行的顺序**。在**并发环境里，线程的执行顺序是不确定的**，如果程序存在竞态条件问题，那就意味着程序执行的结果是不确定的，而执行结果不确定这可是个大 Bug。

举个例子：

```java
class Account {
    private int balance;//共享变量

    void transfer(
            Account target, int amt) {
        if (this.balance > amt) {
            this.balance -= amt;
            target.balance += amt;
        }
    }
}
```

如果现在balance=200，有两个线程都要转出150，同时执行到`if (this.balance > amt)`都判断为`true`，会导致结果出错。这就是程序执行结果依赖线程的先后顺序。

也可以理解为，**程序依赖某个状态变量，而这个变量会被多个线程读写**：

```java
if (状态变量 满足 执行条件) {
  执行操作
}
```

 <u>如何保证线程安全？</u>加锁（互斥）。



#### 活跃性问题

所谓活跃性问题，指的是**某个操作无法执行下去**。我们常见的“死锁”就是一种典型的活跃性问题，当然除了**死锁**外，还有两种情况，分别是“**活锁**”和“**饥饿**”。

<u>死锁：</u> 线程会互相等待，技术上的表现形式是线程**永久地“阻塞”**。

<u>活锁：</u>

有时线程虽然**没有发生阻塞**，但仍然会存在执行不下去的情况，这就是所谓的“活锁”。

解决“活锁”的方案很简单，谦让时，尝试等待一个随机的时间就可以了。

<u>饥饿：</u>

指的是线程因无法访问**所需资源**而无法执行下去的情况。可能是线程优先级低，也可能是持有锁的线程执行时间过长。

解决“饥饿”问题的方案很简单，有三种方案：一是保证资源充足，二是公平地分配资源，三就是避免持有锁的线程长时间执行。



#### 性能问题

“锁”的过度使用可能导致串行化的范围过大，这样就不能够发挥多线程的优势了。

<u>两个优化思路：</u>

第一，既然使用锁会带来性能问题，那最好的方案自然就是**使用无锁的算法和数据结构**了。在这方面有很多相关的技术，例如：

- 线程本地存储 (Thread Local Storage, TLS)
- 写入时复制 (Copy-on-write)
- 乐观锁；
- Java 并发包里面的原子类也是一种无锁的数据结构；
- Disruptor 则是一个无锁的内存队列，性能都非常好

第二，**减少锁持有的时间**。互斥锁本质上是将并行的程序串行化，所以要增加并行度，一定要减少持有锁的时间。这个方案具体的实现技术也有很多，例如：

- **使用细粒度的锁**，一个典型的例子就是 Java 并发包里的 ConcurrentHashMap，它使用了所谓分段锁的技术（这个技术后面我们会详细介绍）；
- 还可以使用**读写锁**，也就是读是无锁的，只有写的时候才会互斥。



### Monitor: 管程

管程（Monitor）指的是**管理共享变量以及对共享变量的操作过程**，让他们支持并发，是一种概念。翻译为 Java 领域的语言，就是管理类的成员变量和成员方法，让这个类是线程安全的。有三种管程模型：Hasen 模型、Hoare 模型和 MESA 模型。**Java 管程的实现参考的也是 MESA 模型**。

在并发编程领域，有两大核心问题：一个是**互斥**，即同一时刻只允许一个线程访问共享资源；另一个是**同步**，即线程之间如何通信、协作。这两大问题，管程都是能够解决的。

#### MESA模型如何保证互斥和同步

![image-20221229203306228](./images/Java%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B/image-20221229203306228.png)

<u>MESA管程模型：</u>

1. 将共享变量及其对共享变量的操作统一封装起来。当多个线程同时试图进入管程内部时，**只允许一个线程进入**，其他线程则在`入口等待队列`中等待。（互斥）
2. **每个条件变量都对应有一个等待队列**，当线程不满足`条件变量X`时通过调用**条件变量的`wati()`**进入`条件变量的等待队列`。（同步，等待-通知机制中的等待/阻塞）
3. 另一个线程进入管程后，如果条`件变量X`满足了，会通过调用**条件变量的`notify()`**让**条件变量的等待队列中的线程**回到`入口等待队列`重新竞争。（同步，等待-通知机制中的通知）

条件变量和条件变量等待队列的作用就是解决线程**同步**问题。

NOTE：条件变量是否满足，是由我们的代码来决定的，即通过`wait()`和`notify()`来控制。



三个管程模型的区别主要在体现在<u>当条件满足后，如何通知相关线程</u>（同步）：

- Hasen 模型里面，要求 notify() 放在代码的最后，这样 T2 通知完 T1 后，T2 就结束了，然后 T1 再执行，这样就能保证同一时刻只有一个线程执行。
- Hoare 模型里面，T2 通知完 T1 后，T2 阻塞，T1 马上执行；等 T1 执行完，再唤醒 T2，也能保证同一时刻只有一个线程执行。但是相比 Hasen 模型，T2 多了一次阻塞唤醒操作。
- MESA 管程里面，T2 通知完 T1 后，T2 还是会接着执行，T1 并不立即执行，仅仅是从条件变量的等待队列进到入口等待队列里面。这样做的好处是 notify() 不用放到代码的最后，T2 也没有多余的阻塞唤醒操作。**但是也有个副作用，就是当 T1 再次执行的时候，可能曾经满足的条件，现在已经不满足了，所以需要以循环方式检验条件变量。**

NOTE: Hasen 模型和Hoare 模型保证了等待的线程一定会被通知到，而MESA 模型是等待的线程回到入口队列，不一定有机会执行，因此**MESA模型的`wait()`方法是带有超时参数的**。



#### java实现的管程

两套实现：

- 语言内置的synchronized

  Java 参考了 MESA 模型，语言内置的管程（synchronized）对 MESA 模型进行了精简。MESA 模型中，条件变量可以有多个，Java 语言内置的管程里只有一个条件变量（锁对象本身）。synchronized 关键字修饰的代码块，在编译期会自动生成相关加锁和解锁的代码（使用的是可重入锁），但是仅支持一个条件变量；

- JUC包中新的实现

  而 Java SDK 并发包实现的管程支持多个条件变量（`Condition`），不过并发包里的锁，需要开发人员自己进行加锁和解锁操作。



<u>原理：</u>

- **锁对象的对象头中有绑定了一个monitor（管程）**，所有线程访问共享资源（获取锁）的适合，需要先拥有monitor（进入管程）；
- 调用了条件变量`wait()`的线程，会进入该条件变量的等待队列（等待池）；
- 条件变量`notify()`或`signal()`被调用后，该条件变量等待队列中的线程就不再阻塞，重新进入monitor的锁池。
- 条件变量可以是**锁对象本身（synchronized块中）**；也可以是**Condition对象**。



### 线程

#### 通用的线程生命周期

![image-20230104141825073](./images/Java%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B/image-20230104141825073.png)

1. **初始状态**，指的是线程已经被创建，但是还不允许分配 CPU 执行。这个状态属于编程语言特有的，不过这里所谓的被创建，仅仅是在编程语言层面被创建，**而在操作系统层面，真正的线程还没有创建**。
2. **可运行状态**，指的是线程可以分配 CPU 执行。在这种状态下，真正的操作系统线程已经被成功创建了，所以可以分配 CPU 执行。
3. 当有空闲的 CPU 时，操作系统会将其分配给一个处于可运行状态的线程，被分配到 CPU 的线程的状态就转换成了**运行状态**。
4. <u>运行状态的线程如果调用一个阻塞的 API（例如以阻塞方式读文件）或者等待某个事件（例如条件变量），那么线程的状态就会转换到**休眠状态**，==同时释放 CPU 使用权==，休眠状态的线程永远没有机会获得 CPU 使用权。当等待的事件出现了，线程就会从休眠状态转换到可运行状态。</u>
5. 线程执行完或者出现异常就会进入**终止状态**，终止状态的线程不会切换到其他任何状态，进入终止状态也就意味着线程的生命周期结束了。

Java 语言里则把可运行状态和运行状态合并了，这两个状态在操作系统调度层面有用，而 JVM 层面不关心这两个状态，因为 JVM 把线程调度交给操作系统处理了。

#### JAVA中线程的生命周期

JAVA线程可以处于以下状态之一：

- **NEW** 

  尚未启动的线程处于此状态。

- **RUNNABLE** 

  可运行线程的线程状态。处于可运行状态的线程正在 Java 虚拟机中执行，<u>但它可能正在等待来自操作系统的其他资源</u>，例如处理器。

- **BLOCKED** 

  阻塞等待监视器锁的线程的线程状态。处于阻塞状态的线程**正在等待监视器锁进入同步块/方法**或在调用`Object.wait()`(后重新进入同步块/方法。

- **WAITING** 

  等待线程的线程状态。由于调用以下方法之一，线程处于等待状态：

  - `Object.wait()`

  - `Thread.join()`

  - `LockSupport.park()`

    处于等待状态的线程正在等待另一个线程执行特定操作。例如，已对某个对象调用`Object.wait()`的线程正在等待另一个线程对该对象调用`Object.notify()`或`Object.notifyAll()` ;调用`Thread.join()`的线程正在等待指定线程终止。

- **TIMED_WAITING** 

  具有指定**等待时间**的等待线程的线程状态。线程由于调用了以下方法之一而处于定时等待状态，并具有指定的正等待时间：

  - `Thread.sleep(millis)`
  - `Object.wait(timeout)`
  - `Thread.join(millis)`
  - `LockSupport.parkNanos(blocker, deadline)`
  - `LockSupport.parkUntil(deadline)`

- **TERMINATED** 

  已退出的线程处于此状态。

在操作系统层面，Java 线程中的 BLOCKED、WAITING、TIMED_WAITING 是一种状态，即前面我们提到的休眠状态。也就是说只要 Java 线程处于这三种状态之一，那么这个线程就永远没有 CPU 的使用权。

![image-20230104143408691](./images/Java%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B/image-20230104143408691.png)

##### RUNNABLE 与 BLOCKED 的状态转换

只有线程等待 synchronized 的隐式锁这种场景会转换成BLOCKED。

<u>线程**调用阻塞式 API** 时，是否会转换到 BLOCKED 状态呢？</u>

**在操作系统层面，线程是会转换到*休眠状态*的**，但是**在 JVM 层面，Java 线程的状态不会发生变化，也就是说 Java 线程的状态会依然保持 *RUNNABLE 状态***。**JVM 层面并不关心操作系统调度相关的状态**，因为在 JVM 看来，等待 CPU 使用权（操作系统层面此时处于可执行状态）与等待 I/O（操作系统层面此时处于休眠状态）没有区别，都是在等待某个资源，所以都归入了 RUNNABLE 状态。

而我们平时所谓的 Java 在调用阻塞式 API 时，线程会阻塞，指的是操作系统线程的状态，并不是 Java 线程的状态。



##### 从 NEW 到 RUNNABLE 状态

NEW 状态的线程，不会被操作系统调度。当调用`Thread.start()`后转换为RUNABLE。

```java
    //java.lang.Thread#start
	public synchronized void start() {
        if (threadStatus != 0)//state "NEW"
            throw new IllegalThreadStateException();

        group.add(this);

        boolean started = false;
        try {
            start0();
            started = true;
        } finally {
            try {
                if (!started) {
                    group.threadStartFailed(this);
                }
            } catch (Throwable ignore) {
                /* do nothing. If start0 threw a Throwable then
                  it will be passed up the call stack */
            }
        }
    }
	//由虚拟机实现，调用操作系统的线程API创建真正的线程
	private native void start0();
```

##### 从 RUNNABLE 到 TERMINATED 状态

线程**执行完`run()`方法**或者**执行过程抛出异常**都会导致终止。<u>Thread提供了两个方法用于中断`run()`的执行：</u>

- **stop()** 已废弃

  `stop()` 方法强制停止线程。使用 Thread.stop 停止线程会导致它解锁所有已锁定的监视。如果之前受这些监视器保护的任何对象处于不一致状态，则损坏的对象对其他线程可见，可能导致任意行为。

- **interrupt()**

  `interrupt()` 方法仅仅是通知线程，线程有机会执行一些后续操作，同时也可以无视这个通知。<u>被 interrupt 的线程，是怎么收到通知的呢？</u>一种是**异常**，另一种是**主动检测**。

  - **当线程 A 处于 WAITING、TIMED_WAITING 状态时，如果其他线程调用线程 A 的 interrupt() 方法**，会使线程 A 返回到 RUNNABLE 状态，同时线程 A 的代码会**触发 InterruptedException 异常**。上面我们提到转换到 WAITING、TIMED_WAITING 状态的触发条件，都是调用了类似 `wait()`、`join()`、`sleep()` 这样的方法，我们看这些方法的签名，发现都会 throws InterruptedException 这个异常。这个异常的触发条件就是：其他线程调用了该线程的 interrupt() 方法。
  - 当线程 A 处于 RUNNABLE 状态时，并且阻塞在 java.nio.channels.InterruptibleChannel 上时，如果其他线程调用线程 A 的 interrupt() 方法，线程 A 会触发 java.nio.channels.ClosedByInterruptException 这个异常；而阻塞在 java.nio.channels.Selector 上时，如果其他线程调用线程 A 的 interrupt() 方法，线程 A 的 java.nio.channels.Selector 会立即返回。
  - 上面这两种情况属于被中断的线程通过异常的方式获得了通知。还有一种是**主动检测**，如果线程处于 RUNNABLE 状态，并且没有阻塞在某个 I/O 操作上，例如中断计算圆周率的线程 A，这时就得依赖线程 A 主动检测中断状态了。**如果其他线程调用线程 A 的 interrupt() 方法，那么线程 A 可以通过 isInterrupted() 方法，检测是不是自己被中断了**。

##### InterruptedException 

在触发 InterruptedException 异常的同时，JVM 会同时**把线程的中断标志位清除**，所以这个时候`th.isInterrupted()`返回的是 `false`。



#### 线程问题排查工具

jstack

Java VisualVM



#### 创建多少线程才是合适的？

在并发编程领域，提升性能本质上就是提升硬件的利用率，再具体点来说，就是提升 I/O 的利用率和 CPU 的利用率。操作系统解决硬件利用率问题的对象往往是单一的硬件设备，而我们的并发程序，往往需要 CPU 和 I/O 设备相互配合工作，也就是说，**我们需要解决 CPU 和 I/O 设备综合利用率的问题**。解决方法就是多线程。

线程也不是越多越好，线程切换需要成本。

- CPU密集型程序

  理论上纯计算的应用，每个逻辑核心一个线程就是最优的，再多创建的线程也只会增加线程切换的成本，不过在工程上，线程的数量一般会设置为“**CPU 核数 +1**”，这样的话，当线程因为偶尔的内存页失效或其他原因导致阻塞时，这个额外的线程可以顶上，从而保证 CPU 的利用率。

- IO密集型程序

  IO密集型需要评估CPU计算和I/O操作的耗时比，最佳线程数 =CPU 核数 * [ 1 +（I/O 耗时 / CPU 耗时）]

如果一台机器跑了多个程序，每个程序都有自己的线程池，需要根据实际情况确定。



#### 线程调用栈

每个线程都有自己独立的调用栈：

![image-20230104171914745](./images/Java%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B/image-20230104171914745.png)

**局部变量是放到调用栈里的，不会和其他线程共享，所以线程安全（线程封闭技术）**：

![image-20230104172038735](./images/Java%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B/image-20230104172038735.png)

### 如何用面向对象思想写好并发程序

#### 封装共享变量

将共享变量作为对象属性封装在内部，对所有公共方法制定并发访问策略；对共享变量进行封装，要避免“逸出”，所谓“逸出”简单讲就是共享变量逃逸到对象的外面。

对于不会发生变化的共享变量，用 final 关键字来避免并发问题。

```java
public class Counter {
    private static final long MAX = 10000;
  	private long value;
  	synchronized long get(){
    	return value;
  	}
  	synchronized long addOne(){
    	return ++value;
  	}
}
```

#### 识别多个共享变量间的约束条件

如果多个共享变量存在约束，一般需要用`if`来判断，很可能会出现**竞态条件**。例如下面的例子没法满足约束条件：

```java
//库存下限要小于库存上限
public class SafeWM {
  // 库存上限
  private final AtomicLong upper = new AtomicLong(0);
  // 库存下限
  private final AtomicLong lower = new AtomicLong(0);
  // 设置库存上限
  void setUpper(long v){
    // 检查参数合法性
    if (v < lower.get()) {
      throw new IllegalArgumentException();
    }
    upper.set(v);
  }
  // 设置库存下限
  void setLower(long v){
    // 检查参数合法性
    if (v > upper.get()) {
      throw new IllegalArgumentException();
    }
    lower.set(v);
  }
  // 省略其他业务代码
}
```

假如原本上下限是(2,10)，如果两个线程同时执行了`setUpper(7)`和`setLower(5)`,它们都能通过校验语句，导致结果变成(7,5)不满足约束。

<u>如何解决这个问题呢？</u>

- 方法1：使用`synchronized`给`setLower`和`setUpper`加锁，牺牲性能。

- 方法2：将上下限封装到一个对象中，一起操作：

  ```java
  public class Boundary {
      private final lower;
      private final upper;
      //线程封闭，保证不会破坏约束
      public Boundary(long lower, long upper) {
          if(lower >= upper) {
              // throw exception
          }
          this.lower = lower;
          this.upper = upper;
      }
  }
  public class SafeWM {
      private volatile Boundary boundary;
  	void setBoundary(Boundary boundary) {
          this.boundary = boundary;
      }
      Boundary getBoundary() {
          return this.boundary;
      }
  }
  ```

  

#### 制定并发访问策略

- **避免共享**：避免共享的技术主要是利于线程本地存储以及为每个任务分配独立的线程。

- **不变模式**：这个在 Java 领域应用的很少，但在其他领域却有着广泛的应用，例如 Actor 模式、CSP 模式以及函数式编程的基础都是不变模式。
- **管程及其他同步工具**：Java 领域万能的解决方案是管程，但是对于很多特定场景，使用 Java 并发包提供的读写锁、并发容器等同步工具会更好。



## JDK并发工具类

### JUC中的管程：Lock & Condition

Java SDK 并发包通过 Lock 和 Condition 两个接口来实现管程，其中 **Lock 用于解决互斥问题，Condition 用于解决同步问题**。

既然 Java 从语言层面已经实现了管程了，那为什么还要在 SDK 里提供另外一种实现呢？区别在哪里？



### 信号量Semaphore



### ReadWriteLock

### StampedLock

### CountDownLatch

### CyclicBarrier

### 并发容器

### 原子类

### Executor和线程池







## Java多线程

### 多线程基础

#### 为什么需要多线程？

摩尔定律失效，所以需要多核+分布式。

多 CPU 核心意味着同时操作系统有更多的并行计算资源可以使用。操作系统以线程作为基本的调度单元。

单线程是最好处理不过的。**线程越多，管理复杂度越高**。

#### Java中线程的创建过程

![image-20221227203041733](./images/Java%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B/image-20221227203041733.png)

#### 线程状态

![image-20221228105538324](./images/Java%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B/image-20221228105538324.png)

![image-20221228105516908](./images/Java%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B/image-20221228105516908.png)

#### 等待唤醒机制

### Java线程的使用

使用Thread的方式：

```java
```

