
---



Java 多线程编程允许在一个进程中并发执行多个任务，每条线程可以独立执行不同的任务，提高 CPU 利用率和程序效率。

---

## 1. 线程与进程

- **线程（Thread）**：进程中的单一执行流。
    
- **进程（Process）**：包含由操作系统分配的内存空间和一个或多个线程。线程不能独立存在，必须隶属于某个进程。
    
- **多线程的优势**：资源开销小，能够高效执行并发任务。
    

---

## 2. 线程生命周期

线程从创建到结束经历多个状态：

1. **新建状态（New）**
    
    - 使用 `new Thread()` 或实现 `Runnable` 创建线程对象。
        
    - 调用 `start()` 前处于新建状态。
        
2. **就绪状态（Runnable）**
    
    - 调用 `start()` 后进入就绪队列，等待 CPU 调度。
        
3. **运行状态（Running）**
    
    - 获取 CPU 资源，执行 `run()` 方法。
        
    - 可进入阻塞、就绪或死亡状态。
        
4. **阻塞状态（Blocked）**
    
    - **等待阻塞**：`wait()` 方法
        
    - **同步阻塞**：获取 `synchronized` 锁失败
        
    - **其他阻塞**：`sleep()`、`join()`、I/O 请求等
        
5. **死亡状态（Terminated）**
    
    - 任务完成或出现终止条件，线程结束。
        

---

## 3. 线程优先级

- Java 线程优先级范围：`1 (MIN_PRIORITY)` ~ `10 (MAX_PRIORITY)`，默认值 `5 (NORM_PRIORITY)`。
    
- 高优先级线程更容易获得 CPU 时间，但不保证严格执行顺序。
    
- 设置方法：`setPriority(int priority)`
    

---

## 4. 创建线程的三种方式

### 4.1 实现 Runnable 接口

```java
class MyRunnable implements Runnable {
    public void run() {
        System.out.println("Thread running");
    }
}

Thread t = new Thread(new MyRunnable(), "Thread-1");
t.start();
```

- **优点**：线程类可以继承其他类。
    
- **特点**：`run()` 方法执行线程逻辑，必须通过 `start()` 启动。
    

---

### 4.2 继承 Thread 类

```java
class MyThread extends Thread {
    public void run() {
        System.out.println("Thread running");
    }
}

MyThread t = new MyThread();
t.start();
```

- **优点**：调用 `this` 即可获得当前线程，编写简单。
    
- **缺点**：线程类无法继承其他类。
    

---

### 4.3 Callable + Future

```java
class MyCallable implements Callable<Integer> {
    public Integer call() {
        return 123;
    }
}

FutureTask<Integer> ft = new FutureTask<>(new MyCallable());
Thread t = new Thread(ft);
t.start();
int result = ft.get();  // 获取返回值
```

- **优点**：支持返回值和抛出异常。
    
- **适用场景**：需要线程执行结果或异常处理。
    

---

## 5. Thread 类常用方法

|方法|描述|
|---|---|
|`start()`|启动线程，调用 `run()` 方法|
|`run()`|线程执行体，需重写|
|`setName(String name)`|设置线程名|
|`setPriority(int priority)`|设置线程优先级|
|`setDaemon(boolean on)`|设置守护线程|
|`join(long millisec)`|等待线程终止最多 millis 毫秒|
|`interrupt()`|中断线程|
|`isAlive()`|判断线程是否活跃|

**静态方法**

|方法|描述|
|---|---|
|`yield()`|暂停当前线程，执行其他线程|
|`sleep(long millisec)`|让当前线程休眠指定时间|
|`holdsLock(Object x)`|判断当前线程是否持有对象锁|
|`currentThread()`|返回当前线程对象|
|`dumpStack()`|打印线程堆栈信息|

---

## 6. 多线程关键概念

- **线程同步**：防止多个线程同时访问共享资源引发数据不一致。
    
- **线程间通信**：如 `wait()` / `notify()` / `notifyAll()`。
    
- **线程死锁**：两个或多个线程相互等待资源，导致程序停止。
    
- **线程控制**：挂起、停止、恢复线程。
    

---

## 7. 多线程使用注意事项

1. **合理创建线程**：过多线程会降低效率，CPU 花在上下文切换上的时间增加。
    
2. **理解并发执行**：程序执行不是串行，需注意线程安全和同步问题。
    
3. **守护线程**：用于后台任务，JVM 在所有用户线程结束后自动退出。
    

---

## 8. 小结对比

|特性|Runnable|Thread|Callable + Future|
|---|---|---|---|
|继承能力|可以继承其他类|无法继承其他类|可以继承其他类|
|是否有返回值|否|否|是|
|是否可抛异常|可捕获|可捕获|可抛出异常|
|使用复杂度|中|简单|稍复杂|

---

**多线程核心要点**：

- 理解线程生命周期和状态转换
    
- 合理设置线程优先级
    
- 掌握三种线程创建方式及适用场景
    
- 注意线程安全和同步问题
    
- 避免线程过多导致效率下降
    

```java
import java.util.concurrent.*;  
  
// ==================== 1. 实现 Runnable ====================/**  
 * 实现 Runnable 接口的线程任务示例  
 * 优点：可以继承其他类，灵活  
 */  
class RunnableTask implements Runnable {  
    private String name; // 线程名称  
  
    public RunnableTask(String name) {  
        this.name = name;  
    }  
  
    @Override  
    public void run() {  
        // run() 方法定义线程执行体  
        for (int i = 1; i <= 5; i++) {  
            System.out.println("Runnable " + name + " running: " + i);  
            try {  
                Thread.sleep(200); // 线程休眠，模拟耗时任务  
            } catch (InterruptedException e) {  
                e.printStackTrace();  
            }  
        }  
        System.out.println("Runnable " + name + " finished.");  
    }  
}  
  
// ==================== 2. 继承 Thread ====================/**  
 * 继承 Thread 类的线程任务示例  
 * 优点：使用简单，可直接调用 this 获得当前线程  
 * 缺点：不能再继承其他类  
 */  
class ThreadTask extends Thread {  
    public ThreadTask(String name) {  
        super(name); // 设置线程名称  
    }  
  
    @Override  
    public void run() {  
        for (int i = 1; i <= 5; i++) {  
            System.out.println("Thread " + getName() + " running: " + i);  
            try {  
                Thread.sleep(200); // 线程休眠，模拟耗时任务  
            } catch (InterruptedException e) {  
                e.printStackTrace();  
            }  
        }  
        System.out.println("Thread " + getName() + " finished.");  
    }  
}  
  
// ==================== 3. Callable + Future ====================  
/**  
 * 实现 Callable 接口的线程任务示例  
 * 优点：可以返回结果、抛出异常  
 */  
class CallableTask implements Callable<Integer> {  
    private int n; // 累加上限  
  
    public CallableTask(int n) {  
        this.n = n;  
    }  
  
    @Override  
    public Integer call() throws Exception {  
        int sum = 0;  
        for (int i = 1; i <= n; i++) {  
            sum += i;  
            System.out.println("Callable adding: " + i);  
            Thread.sleep(80); // 模拟耗时操作  
        }  
        return sum; // 返回累加结果  
    }  
}  
  
  
// ==================== 4. 线程同步示例 ====================/**  
 * 计数器示例，演示线程安全  
 */  
class Counter {  
    private int count = 0;  
  
    // synchronized 确保同一时刻只有一个线程修改 count    public synchronized void increment() {  
        count++;  
    }  
  
    public int getCount() {  
        return count;  
    }  
}  
  
/**  
 * 多线程操作计数器的任务  
 */  
class SyncTask implements Runnable {  
    private Counter counter;  
  
    public SyncTask(Counter counter) {  
        this.counter = counter;  
    }  
  
    @Override  
    public void run() {  
        // 每个线程对 counter 增加 1000 次  
        for (int i = 0; i < 1000; i++) {  
            counter.increment();  
        }  
    }  
}  
  
// ==================== 5. 线程间通信示例 ====================/**  
 * 生产者-消费者模型示例  
 * 使用 wait() / notify() 实现线程间通信  
 */  
class ProducerConsumer {  
    private int data;  
    private boolean available = false; // 标记数据是否可用  
  
    public synchronized void produce(int value) throws InterruptedException {  
        // 如果数据未被消费，生产者等待  
        while (available) wait();  
        data = value;  
        System.out.println("Produced: " + data);  
        available = true;  
        notify(); // 通知消费者  
    }  
  
    public synchronized void consume() throws InterruptedException {  
        // 如果没有数据，消费者等待  
        while (!available) wait();  
        System.out.println("Consumed: " + data);  
        available = false;  
        notify(); // 通知生产者  
    }  
}  
  
// ==================== 主程序 ====================public class duoxiancheng {  
    public static void main(String[] args) throws Exception {  
        System.out.println("=== Runnable & Thread Demo ===");  
        // Runnable 示例  
        RunnableTask rTask1 = new RunnableTask("R1");  
        Thread t1 = new Thread(rTask1);  
        t1.setPriority(Thread.MAX_PRIORITY); // 设置线程优先级为最高  
        t1.start();  
  
        // Thread 示例  
        ThreadTask t2 = new ThreadTask("T2");  
        t2.setPriority(Thread.MIN_PRIORITY); // 设置线程优先级为最低  
        t2.start();  
  
        // 等待 t1、t2 执行完毕再继续  
        t1.join();  
        t2.join();  
  
        System.out.println("=== Callable + Future Demo ===");  
  
  
        // Callable + Future 示例  
        CallableTask cTask = new CallableTask(5);  
        FutureTask<Integer> ft = new FutureTask<>(cTask);  
        Thread t3 = new Thread(ft);  
        t3.start();  
        int result = ft.get(); /* 获取子线程返回值，ft.get()是阻塞方法，会等待子线程执行完毕并返回结果，当  
        CallableTask 的 call() 方法执行完毕，ft.get() 方法会返回 call() 方法的返回值，即子线程的计算结果。  
        */        System.out.println("Callable result: " + result);  
  
        System.out.println("=== Synchronized Counter Demo ===");  
  
  
        // 同步计数器示例  
        Counter counter = new Counter();  
        Thread t4 = new Thread(new SyncTask(counter));  
        Thread t5 = new Thread(new SyncTask(counter));  
        t4.start();  
        t5.start();  
        t4.join();  
        t5.join();  
        System.out.println("Counter final value: " + counter.getCount());  
  
        System.out.println("=== Producer-Consumer Demo ===");  
        // 生产者消费者示例  
        ProducerConsumer pc = new ProducerConsumer();  
        Thread producer = new Thread(() -> {  
            try {  
                for (int i = 1; i <= 5; i++) pc.produce(i);  
            } catch (InterruptedException e) {  
                e.printStackTrace();  
            }  
        });  
        Thread consumer = new Thread(() -> {  
            try {  
                for (int i = 1; i <= 5; i++) pc.consume();  
            } catch (InterruptedException e) {  
                e.printStackTrace();  
            }  
        });  
        producer.start();  
        consumer.start();  
        producer.join();  
        consumer.join();  
  
        System.out.println("Main thread finished.");  
    }  
}
```

---

