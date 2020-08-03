## 前言

其实起这个名字有点言过其实，自然是不可能在一篇文章中讲清多线程的所有问题。我的本意是想把多线程中的很多零碎知识串联到一起。

不知道大家在学习多线程的时候有没有一种感觉，知识很零碎，很多概念性的东西，例如：原子性，可见性，有序性，volatile，Synchronized，CAS等等，网上有很多关于这些知识的讲解，但是总觉得没能把这些知识结合起来，volatile到底解决了什么问题？Happens-before限制了JVM的什么行为？这里面总归是有某些联系的。

我尝试着在这篇文章中把这些关键知识点理清，并将它们串联起来，找到他们的内在联系，让初学者能更好地学习和理解多线程的概念。但是这篇文章不会涉及这些知识点的用法，具体的用法网上有很多资料，可以自行查阅。



## 从多线程的三大难点说起

多线程程序为啥难写，其实原因很简单，因为在多线程环境下，程序执行的三大特性可能会被破坏，哪三大特性：自然是原子性，可见性和有序性。

来看一段程序，这段程序是对类中的变量进行累加操作。

```java
class AddTest{
    private Integer count = 0;
    public void add(){
        count++;
    }

    public static void main(String[] args) throws InterruptedException {
        AddTest addTest = new AddTest();
        ExecutorService executorService = Executors.newFixedThreadPool(10);
        /**向线程池提交10个任务，每个任务会把count加1000**/
        for(int i=0; i<10; i++){
            executorService.execute(()->{
                for(int j=0; j<1000; j++)
                    addTest.add();
            });
        }
        /** 向线程池提交1个任务，把count加10000次
      	executorService.execute(()->{
                for(int j=0; j<10000; j++)
                    addTest.add();
            });
         **/
         
        //等待线程池中的线程执行完毕
        executorService.shutdown();
        while (!executorService.awaitTermination(1,TimeUnit.SECONDS)){
            Thread.yield();
        }
        System.out.println("最终结果: "+addTest.count);
    }  
}
```

通过运行结果我们可以发现，当向线程池提交10个任务去累加时，最终结果总不是10000，而是一个小于10000的数，但是通过向线程池提交一个任务去执行累加操作，结果却总是符合我们预期的。所以我们猜测还是在并发执行累加的过程中出现了问题。

### 原子性

提到原子性，大家大概会联想到数据库事务的原子性，其实两者是很类似的。

在Java中，原子性并不是指一组操作不可被CPU中断，而是说即使一组操作被CPU中断了，也必须保证这一组未完成操作的中间状态对外是不可见的，违反了这一原则，就会出现意想不到的结果。

但是JVM只能保证一些基本操作是原子性的，例如：一个最简单的i++操作，对于JVM来说，包含了三个操作指令：

1. 从主内存中读取i的值到自己线程的本地内存。
2. 在自己的本地内存执行累加操作
3. 最终将结果刷新的主内存。

JVM只能保证1，2和3这三个单独的操作是原子性的，组合起来JVM就不能保证了，加入有两个线程同时从主内存读取了i=0的值，然后分别累加1000次，再把结果刷新到主存时，两者都是将1000刷新到主存，这明显不对。

所以这要怎么解决呢？这就需要我们开发人员在代码层面上保证其原子性。

### 可见性

可见性并不是由Java多线程直接引起的，实际上是因为计算机CPU的多级缓存架构引起的。我们知道现代CPU为了提高执行的效率，都有数据进行了缓存，一般来说，CPU都有三级缓存，每颗CPU都有自己独立的一级和二级缓存，然后公用三级缓存，在对主存的数据进行读写时，实际上是将主存的数据读到自己的缓存 ，或者将缓存中的数据刷新到主存。这就是问题所在，因为缓存的存在，导致CPU在执行的时候看到的不是主存中最新的数据，自然就会产生问题。

那么怎么解决可见性问题呢？其实CPU厂商也在思考这些问题，也制定了一些协议用来约束CPU的缓存行为，例如：MESI协议等，通过提供一些原语指令来进行数据的强制读取和刷新。

但是Java是一门高级语言，旨在屏蔽硬件对软件开发带来的问题，所以Java对计算机缓存体系进行了抽象，提出了主存和线程本地存储的概念，同时也提供了Java层面上对CPU缓存进行限制的原语，我个人觉得，这些原语指令就是对计算机多缓存协议的抽象和封装。例如我们经常用到的Synchronized，volatile等等。

除了Java提供的一些原语指令，JVM在提出了一些规则，用来限制厂商在设计JVM时必须要遵守的一些规定，否则，程序在多线程环境下将会发生意想不到的结果，这就是happens-before原则。

### 有序性

有序性大部分都是指的CPU在执行时会将一些指令进行重排序，但排序过后的执行结果在单线程环境下不会有任何区别，即CPU只对不存在数据依赖的指令进行重排序。

除了CPU，JVM在编译时也会对一些指令进行重排序，在单线程环境下，重排序并不会对我们的程序执行结果有任何影响，但是在一些极端的环境下还是会有问题，下面来看一个非常经典的案例：懒汉式的双重检验锁单例模式（这个大家一定要会写，面试常考）

```java
public class SingletonObject {

    private static SingletonObject instance = null;  //这里有没有问题？？？？？

    public static SingletonObject getInstance() {
        if (instance == null) { // 第一次检查
            synchronized (Singleton.class) { 
                if (instance == null) { // 第二次检查
                    instance = new SingletonObject();
                }
            }
        }
        return instance;
    }
}
```

这里为什么要两次检查instance是否为null呢，是因为防止多个线程同时进行第一个判断中，如此就会产生多个对象，也就不能叫单例模式了。

其实这里是有问题的，我已经在注释中标明了。至于原因我们先来回顾一下JVM创建对象的过程，大致分为三步：

1. 在堆中创建一个对象
2. 对对象进行初始化
3. 将栈中的变量指向堆中的对象。

需要注意的是，一旦第三步执行完成，那么instance就不为null了，如果JVM对2和3进行调换顺序，那么可能后面的线程就能执行判断instance不为null，直接返回，但此时实际instance还未完成初始化动作，那么其他线程在使用该对象的时候很有可能会抛出空指针异常。所以解决方式就是改为 `private static volatile SingletonObject instance = null` 。加了volatile就可以避免重排序问题吗，这个关键字到底是干嘛的，我们后面再说。

## Java对多线程的约束

既然多线程程序中可能会存在这三大问题，那么当然得要有解决方案啊，所以这里得有个概念，我们Java程序员所接触到的一些同步关键字或者同步原语都是为了解决上面这三大问题而存在的。下面来详细说说。

Java在jdk层面上提供了很多同步指令，以便我们在编写代码时能正确地约束程序和JVM的行为。

### synchronized

每个开发者最先接触到的同步指令应该就是synchronized了，我们知道使用synchronized修饰的方法，在同一时间只能有同一个线程在执行，因为在执行临界代码时，必须要先获取互斥锁才可以，所以这也就保证了执行临界代码的原子性。而且，在前一个线程执行完之后，对一个变量的修改，下一个线程在获取到锁之后能看到，这也就保证了可见性，其实真正的语义是是在获取到锁资源后，会强制将CPU缓存无效，从主存中获取最新的值，在释放锁资源时，强制将本地线程缓存刷新到主存中。

### volatile

前面我们说过，volatile可以禁止JVM进行指令重排序，实际上，volatile同样也具有可见性的语义，线程在对一个volatile的变量进行读写后，另一个线程都能看见最新的更改。至于具体的原因，则是我们将要介绍的happens-before原则。

### happens-before原则

 happens-before原则是用来限制厂商在设计JVM时需要遵守的一些规范，在这些规则下，我们可以保证程序的可见性。

 happens-before原则的真正意义在于，前一个操作的结果对后一个线程是可见的。

1. **程序的顺序性规则**

   这条规则是指在一个线程中，按照程序顺序，前面的操作 Happens-Before 于后续的任意操作。

2. **volatile 变量规则**

   这条规则是指对一个 volatile 变量的写操作， Happens-Before 于后续对这个 volatile 变量的读操作。

3. **管程锁定规则**

   这条规则是指对一个锁的解锁 Happens-Before 于后续对这个锁的加锁。

   要理解这个规则，就首先要了解“管程指的是什么”。管程是一种通用的同步原语，在 Java 中指的就是 synchronized，synchronized 是 Java 里对管程的实现。

   管程中的锁在 Java 里是隐式实现的，例如下面的代码，在进入同步块之前，会自动加锁，而在代码块执行完会自动释放锁，加锁以及释放锁都是编译器帮我们实现的

   ```java
   synchronized (this) { //此处自动加锁
     // x是共享变量,初始值=10
     if (this.x < 12) {
       this.x = 12; 
     }  
   } //此处自动解锁
   ```

​      	可以这样理解：假设 x 的初始值是 10，线程 A 执行完代码块后 x 的值会变成 12（执行完自动释放锁），线程 B 进入代码块时，能够看到线程 A 对 x 的写操作，也就是线程 B 能够看到 x==12。

4. **线程启动规则**

   它是指主线程 A 启动子线程 B 后，子线程 B 能够看到主线程在启动子线程 B 前的操作。

5. **线程的终止规则**

   它是指主线程 A 等待子线程 B 完成（主线程 A 通过调用子线程 B 的 join() 方法实现），当子线程 B 完成后（主线程 A 中 join() 方法返回），主线程能够看到子线程的操作。当然所谓的“看到”，指的是对共享变量的操作。

6. **线程中断规则**

   对线程interrupt()方法的调用happens-before于被中断线程代码检测中断。既可以通过Thread.interrupt()检测到是否被中断。

7. **传递特性**

这条规则是指如果 A Happens-Before B，且 B Happens-Before C，那么 A Happens-Before C。

这个规则大有玄机，我们来看一段代码：

```java
//实例代码
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
    }
  }
}
```

<img src="/Applications/program/common/java-blog/JavaBlog/Java核心基础/img/Happens-before传递规则.png" alt="传递规则" style="zoom:50%;" />

1. “x=42” Happens-Before 写变量 “v=true” ，这是规则 1 的内容；

2. 写变量“v=true” Happens-Before 读变量 “v=true”，这是规则 2 的内容 。

再根据这个传递性规则，我们得到结果：“x=42” Happens-Before 读变量“v=true”。这意味着什么呢？如果线程 B 读到了“v=true”，那么线程 A 设置的“x=42”对线程 B 是可见的。

发现了什么吗，另一个线程在对一个volatile变量读取之后，能看到前一个线程对X的更新，前提是，操作是在对volatile变量操作之前。

这就是ReentrantLock即AQS的核心，为什么ReentrantLock能保证原子性和可见性，这就是原因。

### CAS

CAS是Java提供的一种无锁方案，我们无须对CAS变量做额外的同步操作即可保证其正确性，那么原因在哪儿呢？

其实，CAS是利用了CPU提供的一种原语，其核心在于，若存在多个线程在不同的CPU核心上同时对一个变量进行写操作，那么一定只有一个能执行成功，其他CPU会收到执行失败的通知，当执行失败后，我们在Java层面上再次读取最新的值，再去执行更新操作，一直到执行成功为止，注意，这个操作是在Java层面完成的，并不是CPU直接再去重试，很多人都有这个误解。