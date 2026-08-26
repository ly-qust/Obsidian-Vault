# 1.
Java 六种线程状态：

NEW：已创建，未启动
RUNNABLE：可运行，包括等待 CPU 和正在运行
BLOCKED：等待 synchronized 监视器锁
WAITING：无限期等待
TIMED_WAITING：限时等待
TERMINATED：执行结束

口诀：
新建、可运行、阻塞、等待、限时等待、终止。


# 2.sleep wait join
| 方法        | 所属       | 作用           | 是否释放已持有的锁       | 状态                          |
| --------- | -------- | ------------ | --------------- | --------------------------- |
| `sleep()` | `Thread` | 当前线程休眠一段时间   | 不释放             | `TIMED_WAITING`             |
| `wait()`  | `Object` | 等待其他线程通知     | 释放调用对象的监视器锁     | `WAITING` / `TIMED_WAITING` |
| `join()`  | `Thread` | 当前线程等待目标线程结束 | 不应理解为释放当前持有的业务锁 | `WAITING` / `TIMED_WAITING` |
关键区别：

```
Thread.sleep(1000); // 当前线程暂停，不释放锁

object.wait();      // 必须在 synchronized(object) 内调用，
                    // 释放 object 的监视器锁，等待 notify/notifyAll

thread.join();      // 当前线程等待 thread 执行结束
```

记忆：

```
sleep：
当前线程休眠；不释放锁；限时等待。

wait：
等待通知；必须持有对应对象的监视器锁；
会释放该对象的锁。

join：
当前线程等待另一个线程执行结束。

不要混淆：
sleep 是“我休息一会儿”；
wait 是“我释放锁并等待通知”；
join 是“我等另一个线程结束”。
```


# 3.易错题
- `sleep()` 会释放 `synchronized` 锁吗？为什么？
- `wait()` 和 `sleep()` 最大的区别是什么？
- 主线程调用 `t.join()`，究竟是谁等待谁？主线程通常进入什么状态？
sleep：
当前线程暂停一段时间；
不释放已经持有的锁；
进入 TIMED_WAITING。

wait：
当前线程等待通知；
必须持有对应对象的监视器锁；
会释放该对象的监视器锁；
wait() 进入 WAITING；
wait(时间) 进入 TIMED_WAITING。

join：
调用 join 的线程等待目标线程结束；
t.join() 表示当前线程等待 t；
join() 进入 WAITING；
join(时间) 进入 TIMED_WAITING。


