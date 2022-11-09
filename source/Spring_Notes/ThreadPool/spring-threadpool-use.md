# Spring : ThreadPoolTaskExecutor 线程池的使用

------

## 方法说明

| 方法                                      | 说明                                                         |
| ----------------------------------------- | ------------------------------------------------------------ |
| setCorePoolSize                           | 设置线程池核心线程数量，会一直存在。可以在运行时修改。若未设置，则默认为 1 。 |
| getCorePoolSize                           | 获取核心线程数量                                             |
| setMaxPoolSize                            | 设置最大线程数量。可以在运行时修改。若未设置，则默认为 Integer.MAX_VALUE。 |
| getMaxPoolSize                            | 获取最大线程数量                                             |
| setKeepAliveSeconds                       | 设置线程空闲后的存活时间，单位秒。可以在运行时修改。若未设置，则默认为 60 。 |
| getKeepAliveSeconds                       | 获取线程存活时间                                             |
| setQueueCapacity                          | 设置任务队列容量。正数，则用 LinkedBlockingQueue 作为队列；其他情况下用 SynchronousQueue 做队列。若未设置，则默认为 Integer.MAX_VALUE。 |
| setAllowCoreThreadTimeOut                 | 设置是否允许核心线程超时。若允许，核心线程超时后，会被销毁。默认为不允许（fasle）。 |
| setTaskDecorator                          | 设置任务装饰器                                               |
| getThreadPoolExecutor                     | 获取包装的底层 java.util.concurrent.ThreadPoolExecutor       |
| getPoolSize                               | 获取线程池线程数量                                           |
| getActiveCount                            | 获取线程池活动中的线程数量                                   |
| setWaitForTasksToCompleteOnShutdown       | 设置shutdown时是否等到所有任务完成再真正关闭                 |
| setAwaitTerminationSeconds                | 当 setWaitForTasksToCompleteOnShutdown(true)时，setAwaitTerminationSeconds 设置在 shutdown 之后最多等待多长时间后再真正关闭线程池 |
| initialize                                | 初始化线程池                                                 |
| execute                                   | 执行任务                                                     |
| submit                                    | 提交任务，返回 Future                                        |
| shutdown                                  | 关闭/销毁线程池                                              |
| destroy                                   | 关闭/销毁线程池，底层直接调用的 shutdown                     |
| getThreadPoolExecutor().getQueue().size() | 获取任务队列中任务数量，即未执行的任务数量                   |

## 示例1: 使用入门

```java
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ThreadPoolExecutor;

public class TestThreadPool {

    public static void main(String[] args) throws InterruptedException {

        ThreadPoolTaskExecutor threadPoolTaskExecutor = new ThreadPoolTaskExecutor();
        threadPoolTaskExecutor.setThreadNamePrefix("xxxx-");
        threadPoolTaskExecutor.setCorePoolSize(1);
        threadPoolTaskExecutor.setMaxPoolSize(5);
        threadPoolTaskExecutor.setQueueCapacity(10000);
        threadPoolTaskExecutor.setKeepAliveSeconds(300);
        threadPoolTaskExecutor.setRejectedExecutionHandler(new ThreadPoolExecutor.DiscardPolicy() {
            @Override
            public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
                System.out.println("丢弃");
            }
        });
        threadPoolTaskExecutor.initialize();

        int taskNum = 10;
        CountDownLatch latch =  new CountDownLatch(taskNum);

        for (int i=0; i < taskNum; i++) {
            final int taskId = i;
            System.out.println("-- submit task " + taskId);
            threadPoolTaskExecutor.execute(() -> {
                try {
                    System.out.printf("task: %s, thread: %s, start at %d\n", taskId, Thread.currentThread().getName(), System.currentTimeMillis()/1000);
                    Thread.sleep(2_000);
                    System.out.printf("task: %s, thread: %s, end at %d\n", taskId,  Thread.currentThread().getName(), System.currentTimeMillis()/1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    latch.countDown();
                }
            });
        }

        latch.await();
        threadPoolTaskExecutor.shutdown();
    }
}
```

运行结果：

```plain
-- submit task 0
-- submit task 1
-- submit task 2
-- submit task 3
-- submit task 4
task: 0, thread: xxxx-1, start at 1578795725
-- submit task 5
-- submit task 6
-- submit task 7
-- submit task 8
-- submit task 9
task: 0, thread: xxxx-1, end at 1578795727
task: 1, thread: xxxx-1, start at 1578795727
task: 1, thread: xxxx-1, end at 1578795729
task: 2, thread: xxxx-1, start at 1578795729
task: 2, thread: xxxx-1, end at 1578795731
task: 3, thread: xxxx-1, start at 1578795731
task: 3, thread: xxxx-1, end at 1578795733
task: 4, thread: xxxx-1, start at 1578795733
task: 4, thread: xxxx-1, end at 1578795735
task: 5, thread: xxxx-1, start at 1578795735
task: 5, thread: xxxx-1, end at 1578795737
task: 6, thread: xxxx-1, start at 1578795737
task: 6, thread: xxxx-1, end at 1578795739
task: 7, thread: xxxx-1, start at 1578795739
task: 7, thread: xxxx-1, end at 1578795741
task: 8, thread: xxxx-1, start at 1578795741
task: 8, thread: xxxx-1, end at 1578795743
task: 9, thread: xxxx-1, start at 1578795743
task: 9, thread: xxxx-1, end at 1578795745
```

## 示例2: 如果工作中线程数量达到 CorePoolSize ，任务队列已满，对于新任务会新开线程执行。总线程数不会超过 MaxPoolSize

这也意味着，即使任务队列不为空，部分后来的任务，会先被执行。

```java
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ThreadPoolExecutor;

public class TestThreadPool {

    public static void main(String[] args) throws InterruptedException {

        ThreadPoolTaskExecutor threadPoolTaskExecutor = new ThreadPoolTaskExecutor();
        threadPoolTaskExecutor.setThreadNamePrefix("xxxx-");
        threadPoolTaskExecutor.setCorePoolSize(1);
        threadPoolTaskExecutor.setMaxPoolSize(5);
        threadPoolTaskExecutor.setQueueCapacity(5);
        threadPoolTaskExecutor.setKeepAliveSeconds(300);
        threadPoolTaskExecutor.setRejectedExecutionHandler(new ThreadPoolExecutor.DiscardPolicy() {
            @Override
            public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
                System.out.println("丢弃");
            }
        });
        threadPoolTaskExecutor.initialize();

        int taskNum = 10;
        CountDownLatch latch =  new CountDownLatch(taskNum);

        for (int i=0; i < taskNum; i++) {
            final int taskId = i;
            System.out.println("-- submit task " + taskId);
            threadPoolTaskExecutor.execute(() -> {
                try {
                    System.out.printf("task: %s, thread: %s, start at %d\n", taskId, Thread.currentThread().getName(), System.currentTimeMillis()/1000);
                    Thread.sleep(2_000);
                    System.out.printf("task: %s, thread: %s, end at %d\n", taskId,  Thread.currentThread().getName(), System.currentTimeMillis()/1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    latch.countDown();
                }
            });
        }

        latch.await();
        threadPoolTaskExecutor.shutdown();
    }
}
```

执行结果：

```plain
-- submit task 0
-- submit task 1
-- submit task 2
-- submit task 3
-- submit task 4
task: 0, thread: xxxx-1, start at 1578795897
-- submit task 5
-- submit task 6
-- submit task 7
task: 6, thread: xxxx-2, start at 1578795897
task: 7, thread: xxxx-3, start at 1578795897
-- submit task 8
-- submit task 9
task: 8, thread: xxxx-4, start at 1578795897
task: 9, thread: xxxx-5, start at 1578795897
task: 6, thread: xxxx-2, end at 1578795899
task: 0, thread: xxxx-1, end at 1578795899
task: 7, thread: xxxx-3, end at 1578795899
task: 2, thread: xxxx-1, start at 1578795899
task: 1, thread: xxxx-2, start at 1578795899
task: 9, thread: xxxx-5, end at 1578795899
task: 8, thread: xxxx-4, end at 1578795899
task: 5, thread: xxxx-4, start at 1578795899
task: 4, thread: xxxx-5, start at 1578795899
task: 3, thread: xxxx-3, start at 1578795899
task: 2, thread: xxxx-1, end at 1578795901
task: 1, thread: xxxx-2, end at 1578795901
task: 5, thread: xxxx-4, end at 1578795901
task: 4, thread: xxxx-5, end at 1578795901
task: 3, thread: xxxx-3, end at 1578795901
```

## 示例3: 若 MaxPoolSize 小于 CorePoolSize，会报异常

```java
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ThreadPoolExecutor;

public class TestThreadPool {

    public static void main(String[] args) throws InterruptedException {

        ThreadPoolTaskExecutor threadPoolTaskExecutor = new ThreadPoolTaskExecutor();
        threadPoolTaskExecutor.setThreadNamePrefix("xxxx-");
        threadPoolTaskExecutor.setCorePoolSize(8);
        threadPoolTaskExecutor.setMaxPoolSize(5);
        threadPoolTaskExecutor.setQueueCapacity(5);
        threadPoolTaskExecutor.setKeepAliveSeconds(300);
        threadPoolTaskExecutor.setRejectedExecutionHandler(new ThreadPoolExecutor.DiscardPolicy() {
            @Override
            public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
                System.out.println("丢弃");
            }
        });
        threadPoolTaskExecutor.initialize();
    }
}
```

运行时报错如下：

```plain
Exception in thread "main" java.lang.IllegalArgumentException
    at java.util.concurrent.ThreadPoolExecutor.<init>(ThreadPoolExecutor.java:1314)
```



( 本文完 )