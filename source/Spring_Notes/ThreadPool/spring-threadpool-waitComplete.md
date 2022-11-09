# Spring : ThreadPoolTaskExecutor 线程池等待所有任务完成的几种方式

------

## 方式1: 使用 CountDownLatch

示例代码: 

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

## 方式2: 使用 Future

示例代码: 

```java
package spring_thread_pool;

import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.Future;
import java.util.concurrent.ThreadPoolExecutor;

public class TestThreadPool {

    public static void main(String[] args) throws InterruptedException, ExecutionException {

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
        List<Future> taskFutureList = new ArrayList<>();

        for (int i=0; i < taskNum; i++) {
            final int taskId = i;
            System.out.println("-- submit task " + taskId);
            Future future = threadPoolTaskExecutor.submit(() -> {
                 try {
                     System.out.printf("task: %s, thread: %s, start at %d\n", taskId, Thread.currentThread().getName(), System.currentTimeMillis()/1000);
                     Thread.sleep(2_000);
                     System.out.printf("task: %s, thread: %s, end at %d\n", taskId,  Thread.currentThread().getName(), System.currentTimeMillis()/1000);
                 } catch (InterruptedException e) {
                     e.printStackTrace();
                 }
            });
            taskFutureList.add(future);
        }

        for (Future future : taskFutureList) {
            future.get();
        }
        threadPoolTaskExecutor.shutdown();
    }
}
```

## 方式3: setWaitForTasksToCompleteOnShutdown(true)

示例代码: 

```java
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

import java.util.concurrent.ExecutionException;
import java.util.concurrent.ThreadPoolExecutor;

public class TestThreadPool {

    public static void main(String[] args) throws InterruptedException, ExecutionException {

        ThreadPoolTaskExecutor threadPoolTaskExecutor = new ThreadPoolTaskExecutor();
        threadPoolTaskExecutor.setThreadNamePrefix("xxxx-");
        threadPoolTaskExecutor.setCorePoolSize(1);
        threadPoolTaskExecutor.setMaxPoolSize(5);
        threadPoolTaskExecutor.setQueueCapacity(10);
        threadPoolTaskExecutor.setKeepAliveSeconds(300);
        threadPoolTaskExecutor.setRejectedExecutionHandler(new ThreadPoolExecutor.DiscardPolicy() {
            @Override
            public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
                System.out.println("丢弃");
            }
        });
        // 设置 shutdown 时先等待所有任务处理完
        threadPoolTaskExecutor.setWaitForTasksToCompleteOnShutdown(true);
        threadPoolTaskExecutor.initialize();

        int taskNum = 10;

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
                }
            });
        }

        threadPoolTaskExecutor.shutdown();
    }
}
```



( 本文完 )