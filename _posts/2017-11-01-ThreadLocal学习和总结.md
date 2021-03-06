### ThreadLocal的理解和使用场景
在一个进程中多个线程共享一个变量的时候，这个变量可能是全局的或者是static的，但是当其中某一个线程对这个共享变量修改了，那么也会影响到其他的线程，而且多个线程对这个共享变量的访问存在激烈的竞争，假设有这么一个需求，一个变量被多个线程共享，保证在每个线程中都能访问且相互不影响，比如说现在有个Logger模块打印每个线程的特定信息,这个Logger模块在每个线程中都没有依赖关系，它输出的日志信息相对于各个线程都是独立的，但是这个Logger的打印功能在多个线程中都复用，那么是不是可以让这个Logger的对象在每个线程中都有一个拷贝副本，等到必要的时候每个线程都会获取和自己当前线程相关的Logger对象来打印日志，对其他线程并无影响。

Java提供的ThreadLocal类可以解决这个问题，ThreadLocal会为变量在每个线程中都有一份拷贝副本，每个线程只会访问自己内部的那个副本从而达到线程隔离的效果。通过下面的代码来简单理解它的好处，

```java
static class MyLogger{
        private static ThreadLocal<MyLogger> sThreadLocal = new ThreadLocal<MyLogger>();

        private MyLogger(){

        }

        public static MyLogger getInstance(){
            MyLogger myLogger = sThreadLocal.get();

            if (myLogger == null){
                myLogger = new MyLogger();

                sThreadLocal.set(myLogger);
            }

            return myLogger;
        }

        public void printMsg(){
            System.out.println(Thread.currentThread().getName()+": printMsg");
        }
    }

```
上面定义的MyLogger类，Logger通过单例模式来获取对象，在多线程的条件下平时我们可能会通过双重锁的形式保证对象的正确性，但是这里我们不会用同步锁的形式，因为这个Logger对象相对于每个线程它是独立的，并不会相互依赖，所以没有互斥的概念。在上面的Logger类中定义了一个全局的变量 sThreadLocal,在getInstance方法中首先通过ThreadLocal的get方法来获得Logger对象，如果它不为null就返回，如果是null的话就new一个对象出来然后再将这个对象set到ThreadLocal中，ThreadLocal会将这个对象和当前线程绑定。下面的代码就是创建几个线程，创建一个共有的runnable对象，run方法中就是调用Logger的打印当前线程的信息：

```java
    private static Runnable shareRunnable = new Runnable() {
        @Override
        public void run() {
            MyLogger.getInstance().printMsg();
        }
    };

    public static void main(String[] args) {
        // TODO: 2017/11/14 write your code below
        for (int i = 0; i < 3; i++) {
            new Thread(shareRunnable).start();
        }

    }

```
运行的结果：

```java
Thread-0: printMsg
Thread-2: printMsg
Thread-1: printMsg
```

在Android的消息循环机制中，Looper对象的获取也采用了ThreadLocal技术，它保证了一个线程中有且仅有一个Looper，在ActivityThread的main函数中看到Looper的初始化：
```java
        Looper.prepareMainLooper();
        ...
        Looper.loop();

```
Looper的prepareMainLooper方法会调用它内部的prepare方法：
```java
    public static void prepareMainLooper() {
        prepare(false);
        synchronized (Looper.class) {
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            sMainLooper = myLooper();
        }
    }

    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
```
看到prepare方法中他会检查ThreadLocal中是否已经有了Looper对象，如果有的话就抛出异常，如果没有的话创建一个Looper对象并且设置到ThreadLocal中，这和上面Logger获取它的对象方式很相似。

### ThreadLocal背后做了什么
透过源码来理解ThreadLocal是如何实现线程局部存储的。
首先从它的get方法入手：
```java
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null)
                return (T)e.value;
        }
        return setInitialValue();
    }
```
得到当前线程来获取一个ThreadLocalMap对象，这个ThreadLocalMap是ThreadLocal的内部类，它是一个自定义的散列表，初始情况下这个map是null，所以会走到setInitialValue方法中：
```java
    private T setInitialValue() {
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }
```
这里通过createMap函数创建了这个map，找打对应的ThreadLocalMap的构造函数：
```java
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }

    ThreadLocalMap(ThreadLocal firstKey, Object firstValue) {
            table = new Entry[INITIAL_CAPACITY];
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
            table[i] = new Entry(firstKey, firstValue);
            size = 1;
            setThreshold(INITIAL_CAPACITY);
    }
```
在ThreadLocalMap中维护了一个Entry数组，这个Entry继承自WeakReference，以ThreadLocal为key，以ThreadLocal要存储的对象为value，在上面的构造函数中可以看到它先初始化了一个Entry数组，通过ThreadLocal的nextHashCode散列函数得到它在数组中的索引，然后在数组中找到这个索引的位置i，并且将新创建出来的Entry节点插入到i对应的位置中。当创建完ThreadLocalMap对象之后返回到createMap函数中，它将新创建的对象赋值给Thread的全局变量threadLocals，**也就是说每一个线程内部都会维护一个ThreadLocalMap**，当我们从ThreadLocal中getMap的方法是就是获取当前线程的ThreadLocalMap对象。