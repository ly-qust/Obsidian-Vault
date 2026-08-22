### 推荐口述：
`AtomicInteger` 是基于 CAS 实现的原子类，不需要显式加锁，适合计数器等单变量的简单原子更新。不过高竞争时，CAS 失败重试会消耗 CPU。  
`synchronized` 是 Java 的隐式锁，进入同步代码时自动获取锁，退出或发生异常时自动释放，支持可重入，适合保护多个操作组成的完整临界区。  
`ReentrantLock` 是显式可重入锁，需要手动加锁，并在 `finally` 中解锁。相比 `synchronized`，它还支持 `tryLock`、超时获取、可中断获取、公平锁和多个 `Condition`，适合需要精细控制锁的场景。
实际选择时，简单原子更新使用 Atomic 类；普通复杂临界区优先考虑 `synchronized`；需要高级锁控制时选择 `ReentrantLock`。


AtomicInteger：
CAS + 简单单变量原子更新；
高竞争时重试消耗 CPU。

synchronized：
隐式加锁、自动释放、可重入；
适合保护完整临界区。

ReentrantLock：
显式加锁、finally 解锁、可重入；
支持 tryLock、可中断、公平锁、多个 Condition；
适合精细控制锁。


易错：
1. CAS 能保证一次“比较并交换”的原子性，
   不是 CAS 不能保证原子性。

2. ReentrantLock 不是解决 ABA 的工具，
   ABA 可使用 AtomicStampedReference 解决。

3. synchronized 保护的是完整临界区，
   不要把它表述成“执行多个事务”。


### 0.
指 `AtomicInteger`、`AtomicLong`、`AtomicReference` 等类。
- **本质**：基于 **CAS（Compare-And-Swap）** 实现的无锁原子操作。
- **常见方法**：`getAndIncrement()`、`compareAndSet()`、`getAndUpdate()` 等。
- **和 `synchronized` 的区别**：不加锁，适合**读多写少、竞争不激烈**的场景；竞争激烈时 CAS 自旋会消耗 CPU。
- **和 `volatile` 的区别**：`volatile` 只保证可见性，不能保证复合操作（如 `i++`）的原子性；Atomic 类能。
### 1.
AtomicInteger：CAS 重试，无须 lock/unlock
ReentrantLock：显式加锁，必须 finally 解锁

### 2.
ReentrantLock 是可重入锁：
同一个线程可以重复获取同一把锁，不会把自己锁死。

ReentrantLock 不会在方法结束时自动释放：
每次 lock() 都必须对应一次 unlock()，
通常将 unlock() 放在 finally 中。


### 3.ReentrantLock 与synchronized的区别
synchronized：
隐式加锁、自动释放，写法简单。

ReentrantLock：
显式 lock/unlock，必须在 finally 中解锁；
支持可中断、tryLock/超时、公平锁、多个 Condition；
适合需要精细控制锁的场景。

### 4.需要立即尝试、超时获取或可中断获取锁：
选择 ReentrantLock。

tryLock(...) 返回 true：
成功获取锁，执行完必须在 finally 中 unlock()。

tryLock(...) 返回 false：
没有获取锁，不能调用 unlock()

### 5.
简单、短小的单变量原子更新：
优先考虑 AtomicInteger。

多个操作或多个变量需要整体保护：
选择 synchronized / ReentrantLock。

AtomicInteger 不一定永远比锁快：
高竞争时 CAS 失败重试会消耗 CPU。



### 6.`AtomicInteger` 为什么是线程安全的？它和 `volatile int` 有什么区别？
AtomicInteger 的线程安全：
volatile 保证读取到较新的值，
CAS 保证比较并更新的原子性，
原子类方法在 CAS 失败后通常循环重试。

volatile int：
只保证可见性和有序性，
不能保证 i++ 的原子性。


### 7.CAS 有哪些问题？什么是 ABA 问题？如何解决？
CAS 能保证一次“比较并交换”的原子性。

CAS 的主要问题：
1. 高竞争下，循环重试消耗 CPU；
2. 存在 ABA 问题；
3. 更适合简单的单变量更新，复杂临界区通常使用锁。

ABA：
值从 A → B → A，
CAS 只看当前值，可能误以为从未改变。

解决 ABA：
使用版本号，例如 AtomicStampedReference。（每次操作时都检查版本号而不是值）


### 8.`AtomicInteger` 和 `synchronized` 有什么区别？各自适合什么场景？
AtomicInteger：
基于 CAS，无须显式加锁；
适合单变量的简单原子更新。

synchronized：
隐式加锁、自动释放；
适合保护多个操作或多个共享变量组成的完整临界区。

不要说“CAS 一定比锁快”：
高竞争下 CAS 重试也会产生较大成本。


### 9.`ReentrantLock` 和 `synchronized` 有什么区别？为什么使用 `ReentrantLock` 时必须在 `finally` 中解锁？

synchronized：
隐式获取锁、自动释放锁，使用简单。

ReentrantLock：
显式调用 lock() / unlock()；
支持 tryLock、超时获取、可中断获取、公平锁、
多个 Condition，锁控制能力更丰富。

unlock() 必须放在 finally：
保证业务代码即使抛出异常也能释放锁，
避免其他线程一直无法获取锁。

两者都是可重入锁。


### 10

|场景|优先考虑|原因|
|---|---|---|
|简单计数|`AtomicInteger`|原子自增，代码简单|
|单变量简单 CAS 更新|Atomic 类|乐观更新|
|多个操作或变量一起修改|`synchronized` / `ReentrantLock`|保护完整临界区|
|只要求共享状态及时可见|`volatile`|保证可见性、有序性|
|超时或尝试获取锁|`ReentrantLock`|支持 `tryLock()`|
|可中断获取锁|`ReentrantLock`|支持 `lockInterruptibly()`|
|复杂锁控制|`ReentrantLock`|支持公平锁和多个 `Condition`|