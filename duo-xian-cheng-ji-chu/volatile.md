# Volatile

在多线程并发编程中 `synchronized` 和 `volatile` 都扮演着重要的角色，`volatile` 是轻量级的 `synchronized`，它在多处理器开发中保证了共享变量的“可见性”。

**内存可见性**

* 内存可见性，指的是线程之间的可见性，当一个线程修改了共享变量时，另一个线程可以读取到这个修改后的值。
* 所谓内存可见性，指的是当一个线程对`volatile`修饰的变量进行写操作（比如step 2）时，会立即把该线程对应的本地内存中的共享变量的值刷新到主内存；当一个线程对`volatile`修饰的变量进行读操作（比如step 3）时，会立即把该线程对应的本地内存置为无效，从主内存中读取共享变量的值。

```text
    public class VolatileDemo {
        int a = 0;
        volatile boolean flag = false;

        public void writer() {
            a = 1; // step 1
            flag = true; // step 2
        }

        public void reader() {
            if (flag) { // step 3
                System.out.println(a); // step 4
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        VolatileDemo volatileDemo = new VolatileDemo();
        Thread a = new Thread(new Runnable() {
            @Override
            public void run() {
                volatileDemo.writer();
            }
        });

        Thread b = new Thread(new Runnable() {
            @Override
            public void run() {
                volatileDemo.reader();
            }
        });

        a.start();
        b.start();
    }
```

这个时候的输出结果为：`1`

而如果flag变量没有用volatile修饰，在step 2，线程A的本地内存里面的变量就不会立即更新到主内存，那随后线程B也同样不会去主内存拿最新的值，仍然使用线程B本地内存缓存的变量的值a = 0，flag = false。

**禁止重排序**

为优化程序性能，对原有的指令执行顺序进行优化重新排序。重排序可能发生在多个阶段，比如编译重排序、CPU重排序等。

在JSR-133之前的旧的Java内存模型中，是允许volatile变量与普通变量重排序的。那上面的案例中，可能就会被重排序成下列时序来执行：

* 线程A写volatile变量，step 2，设置flag为true；
* 线程B读同一个volatile，step 3，读取到flag为true；
* 线程B读普通变量，step 4，读取到 a = 0；
* 线程A修改普通变量，step 1，设置 a = 1；

可见，如果volatile变量与普通变量发生了重排序，虽然volatile变量能保证内存可见性，也可能导致普通变量读取错误。

所以在旧的内存模型中，volatile的写-读就不能与锁的释放-获取具有相同的内存语义了。为了提供一种比锁更轻量级的线程间的通信机制，JSR-133专家组决定增强volatile的内存语义：严格限制编译器和处理器对volatile变量与普通变量的重排序。

**volatile的非原子性**

* `volatile`虽然增加了实例变量在多个线程之间的可见性，但是却不具备通同步性，也就是不具备原子性。
* 使用`volatile`关键字之后，可以强制从公共内存中读取变量的值。
* `Volatile`只能保证对单个`volatile`变量的读/写具有原子性。

**volatile的用途**

volatile的主要使用场合是在多个线程中可以感知到实例变量被更改了，也就是多线程读取共享变量时可以获得最新的值来使用。

在保证内存可见性这一点上，`volatile`有着与锁相同的内存语义，所以可以作为一个“轻量级”的锁来使用。但由于`volatile`仅仅保证对单个`volatile`变量的读/写具有原子性，而锁可以保证整个临界区代码的执行具有原子性。所以在功能上，锁比`volatile`更强大；在性能上，volatile更有优势。

比如我们熟悉的单例模式，其中有一种实现方式是“双重锁检查”，比如这样的代码：

```text
public class Singleton {

    private static volatile Singleton instance; // 不使用volatile关键字

    // 双重锁检验
    public static Singleton getInstance() {
        if (instance == null) { // 第7行
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton(); // 第10行
                }
            }
        }
        return instance;
    }
}
```

**synchronized和volatile对比**

* volatile是线程同步的轻量级的实现，所以volatile性能比synchronized要好，并且volatile只能修饰变量，而synchronized可以修饰方法以及代码块。
* 多线程访问volatile不会发生阻塞，而synchronized会发生阻塞。
* volatile能保证数据的可见性，但不能保证原子性。而synchronized可以保证原子性，也可以间接保证可见性，因为synchronized会将私有内存和公共内存中的数据做同步，所以更加推荐使用synchronized。
* volatile的关注点在于解决变量在多个线程之间的可见性，而synchronized的关注点在于解决多个线程之间访问资源的同步性。

