---
title: java-thread
date: 2020-04-11 17:19:31
tags: JVM Thread
categories: JVM
---

# 一、线程状态
```
public enum State {
    //新建状态，未启动
    NEW,
    //就绪状态，在JVM中运行，但是在操作系统中可能是等待执行。
    RUNNABLE,
    //阻塞状态，表示线程阻塞于锁
    BLOCKED,
    //等待状态，需要等待其他线程做出一些特定动作（通知或中断）
    WAITING,
   //超时等待，可以在指定的时间后自行返回
   TIMED_WAITING,
   //表示该线程已经执行完毕。
   TERMINATED;
}
```
![状态流转](2020-04-11-java-thread/线程状态流转.png)

Java线程与Linux进程状态的对应关系：  
1. new，是Thread对象的状态，此时和Linux的进程还没有关系。
2. Runnable对应linux的Running。
3. BLOCKED、WAITING、TIMED_WAITING，对应task_intermptible
4. TERMINATED 对应task_stoped

<!--more-->  

sleep
```
Thread类的方法
public static native void sleep(long millis)
public static native void sleep(long millis, int nanos)//sleep(毫秒, 纳秒)

总结：
    1.线程暂时停止运行，但是未失去对锁的拥有
    2.线程等待指定的时间后，该线程不一定会立马/确定运行
    3.线程等待，进入_EntryList队列中
```

join
```
Waits at most millis milliseconds for this thread to die. A timeout of 0 means to wait forever.

public final void join(){
    join(0);
}

public final **synchronized** void join(long millis) throw InterruptException{
    long base = System.currentTimeMillis();
    long now = 0;
    if(millis < 0){
        throw new IllegalArgumentException();
    }
    if(millis == 0){
        while(isAlive()){//isAlive()，当前线程是否还活着
            wait(0);//object.wait(0);
        }
    }else{
        while(isAlive()){
            long delay = millis - now;
            if(delay <=0 ){
                break;
            }
            wait(delay);
            now = System.currentTimeMillis() - base;
        }
    }
}

join总结：
    1、join方法为synchronized方法，所以当线程执行时，会获取该thread对象的锁
    2、所以执行join方法就已经获取了thread对象的锁，再执行wait方法，就不会抛出异常了。
    3、使执行object.join()方法的线程等待
    4、主线程和子线程之间使用，主线程执行，那么主线程将阻塞等待
        
使用
public class Join {
    public static void main(String[] args) {
        Thread thread = new JoinThread();
        thread.start();
        try {
            //主线程等待thread的业务处理完了之后再向下运行  
            thread.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        for(int i = 0; i < 5; i++){
            System.out.println(Thread.currentThread().getName()+" -- " + i);
        }
    }
}

class JoinThread extends Thread{
    @Override
    public void run() {
        for(int i = 0; i < 5; i++){
            System.out.println(Thread.currentThread().getName() + " -- "+i);
            try {
                sleep(500);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
//---------------运行结果---------------------
//主线程等待JoinThread执行完再执行
Thread-0 -- 0
Thread-0 -- 1
Thread-0 -- 2
Thread-0 -- 3
Thread-0 -- 4
main -- 0
main -- 1
main -- 2
main -- 3
main -- 4
```

yield
```
a hint to the scheduler that the current thread is willing to yield its
current use of a processor. The schedular is free to ignore this hint.
一个线索、示意调度器当前线程将会放弃他当前的处理器使用权

public static native void yield();
总结：
    yield()方法不会使线程失去资源，只是失去了处理器使用权，但是时机不好把控，取决于操作系统
    public static void main(String[] args){
        String object = "abc";
        new Thread(new Runnable(){
            public void run(){
                synchronized(object){
                    System.out.println("A1");
                    sleep(4000);
                    System.out.println("A3");
                    Thread.currrentThread.yield();
                    System.out.println("A4");
                }
            }
        }).start();
        
        new Thread(new Runnable(){
            public void run(){
                synchronized(object){
                    System.out.println("a");
                }
            }
        }).start();
    }

执行结果：
    A1
    A3
    A4
    a
    线程让步，并没有执行其他线程，未释放锁

使用
    public static void main(String[] args){
        new Thread(new Runnable(){
            public void run(){
                System.out.println("A1");
                sleep(4000);
                System.out.println("A3");
                Thread.currrentThread.yield();
                System.out.println("A4");
            }
        }).start();
        
        new Thread(new Runnable(){
            public void run(){
                System.out.println("a");
            }
        }).start();
    }
    
执行结果
    例：A1
        A3
        a
        A4

```

# 二、创建线程方式

* 继承Thread类，并复写run方法
* 实现Runnable接口，复写run方法
* 创建FutureTask对象，创建Callable子类对象，复写call(相当于run)方法，将其传递给FutureTask对象（相当于一个Runnable）
* 线程池

1、Thread类
```
public class MyThread extends Thread{ 
    @Override 
    public void run() { 
        System.out.println(Thread.currentThread().getName()); 
    }
}
public class MyThreadTest { 
    public static void main(String[] args) { 
        // 创建线程 
        MyThread thread = new MyThread(); 
        // 启动线程 
        thread.start(); }
}
```
**为什么需要start()才能启动线程？**      
1. new Thread()：此时只是创建了Thread对象，没有实际去创建线程
2. thread.start()：启动线程，实际操作的是调用Linux创建进程，直到进程获取CPU资源才会执行。
    1. 内部执行native start0()方法
    2. start0()方法内部会调用OS创建进程的方法
    3. start0()方法执行后，会调用run()方法

2、Runnable接口
```
public class MyRunnable implements Runnable{ 
    @Override 
    public void run() { 
        System.out.println(Thread.currentThread().getName()); 
    }
}
public class MyRunnableTest { 
    public static void main(String[] args) { 
        MyRunnable myRunnable = new MyRunnable(); 
        // 创建线程 
        Thread thread = new Thread(myRunnable); 
        // 启动线程 
        thread.start(); 
    }
}
```

3、Callable
```
public class MyCallable implements Callable { 
    @Override 
    public Integer call() throws Exception { 
        System.out.println(Thread.currentThread().getName()); 
        return 99; 
    }
}
public class MyCallableTest { 
    public static void main(String[] args) { 
        FutureTask futureTask = new FutureTask<>(new MyCallable()); 
        // 创建线程 
        Thread thread = new Thread(futureTask); 
        // 启动线程 
        thread.start(); 
        // 结果返回 
        try { 
            Thread.sleep(1000); 
            System.out.println("返回的结果是：" + futureTask.get()); 
        } catch (Exception e) { 
            e.printStackTrace(); 
        } 
    }
}
```

4、线程池
```
public class MyRunnable implements Runnable{ 
    @Override 
    public void run() { 
        System.out.println(Thread.currentThread().getName()); 
    }
}
public class SingleThreadExecutorTest { 
    public static void main(String[] args) { 
        ExecutorService executorService = Executors.newSingleThreadExecutor(); 
        MyRunnable myRunnable = new MyRunnable(); 
        for(int i = 0; i < 10; i++){ 
            executorService.execute(myRunnable); 
        } 
        System.out.println("=======任务开始======="); 
        executorService.shutdown(); 
    }
}
```


三、linux进程和java线程
![线程进程关系](2020-04-11-java-thread/线程进程关系.png)

![jvm-linux](2020-04-11-java-thread/jvm线程-Linux进程.png)
jvm线程对应Linux的一个进程

