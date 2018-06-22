---
title: wait、notify、notifyAll和Condition
date: 2018/06/22 09:23:00
---

经典的生产者-消费者模型是线程之间协作的典型应用：当队列满时，生产者需要等待队列有空间才能继续往里面放入商品，而在等待期间，生产者必须释放对临界资源（即队列）的占用权。因为生产者如果不释放对临界资源的占用权，那么消费者就无法消费队列中的商品，就不会让队列有空间，那么生产者就会一直无限等待下去。因此，一般情况下，当队列满时，会让生产者交出对临界资源的占用权，并进入挂起状态。然后等待消费者消费了商品，然后消费者通知生产者队列有空间了。同样的，当队列空时，消费者也必须等待，等待生产者通知它队列中有商品了。这种互相通信的过程就是线程间的协作。

Java中线程协作的最常见的两种方式：利用`Object.wait()`、`Object.notify()`和使用`Condition`
<!-- more -->
## wait notify notifyAll

1. `wait()`、`notify()`、`notifyAll()`方法是本地方法，并且为final方法，无法被重写
2. 调用某个对象的`wait()`方法能让当前线程阻塞，并且当前线程必须拥有此对象的monitor（即锁）。

    因此调用`wait()`方法必须在同步块或者同步方法中进行(synchronized块或者synchronized方法)。
    
    调用某个对象的`wait()`方法，相当于让当前线程交出此对象的monitor，然后进入等待状态，等待后续再次获得此对象的锁（Thread类中的sleep方法使当前线程暂停执行一段时间，从而让其他线程有机会继续执行，但它并不释放对象锁）
    
3. 调用某个对象的`notify()`方法能够唤醒一个正在等待这个对象的monitor的线程，如果有多个线程都在等待这个对象的monitor，则只能唤醒其中一个线程，具体唤醒哪个线程则不得而知。

    同样的，调用某个对象的`notify()`方法，当前线程也必须拥有这个对象的monitor，因此调用`notify()`方法必须在同步块或者同步方法中进行(synchronized块或者synchronized方法)

4. 调用`notifyAll()`方法能够唤醒所有正在等待这个对象的monitor的线程

    `notify()`和`notifyAll()`方法只是唤醒等待该对象的monitor的线程，并不决定哪个线程能够获取到monitor
    
尤其要注意一点，一个线程被唤醒不代表立即获取了对象的monitor，只有等调用完`notify()`或者`notifyAll()`并退出synchronized块，释放对象锁后，其余线程才可获得锁执行

```java
public class MonitorTest {
    public static Object object = new Object();

    public static void main(String[] args) throws InterruptedException {
        Thread1 thread1 = new Thread1();
        Thread2 thread2 = new Thread2();

        thread1.start();

        Thread.sleep(200);

        thread2.start();
    }

    static class Thread1 extends Thread {
        @Override
        public void run() {
            synchronized (object) {
                try {
                    object.wait();
                } catch (InterruptedException e) {
                }
                System.out.println("线程" + Thread.currentThread().getName() + "获取到了锁");
            }
        }
    }

    static class Thread2 extends Thread {
        @Override
        public void run() {
            synchronized (object) {
                object.notify();
                System.out.println("线程" + Thread.currentThread().getName() + "调用了object.notify()");
            }
            System.out.println("线程" + Thread.currentThread().getName() + "释放了锁");
        }
    }
}
```

上面实例的运行结果必定是：

```
线程Thread-1调用了object.notify()
线程Thread-1释放了锁
线程Thread-0获取到了锁
```

## Condition

Condition用来替代传统Object的`wait()`、`notify()`实现线程间的协作，相比使用Object的`wait()`、`notify()`，使用Condition的`await()`、`signal()`这种方式实现线程间协作更加安全和高效。

- Condition是一个接口，基本的方法就是`await()`和`signal()`方法
- Condition依赖于Lock接口，生成一个Condition的基本代码是lock.newCondition()
- 调用Condition的`await()`和`signal()`方法，都必须在lock保护之内，就是说必须在`lock.lock()`和`lock.unlock()`之间才可以使用

## 两者的比较

Object和Condition在使用形式和实现的功能上都非常类似，但这里面有一个最大的问题就是synchronized方式对应的wait、notify不能有多个谓词条件，Lock对应的Condition await signal则可以有多个谓词条件：

```java
private static ReentrantLock lock = new ReentrantLock();
	
private static Condition notEmpty = lock.newCondition();
private static Condition notFull = lock.newCondition();
```

没有多个谓词条件带来的问题在于：

例如队列已满，所有的生产者线程阻塞，某个时刻消费者消费了一个元素，则需要唤醒某个生产者线程，而通过Object notify方式唤醒的线程不能确保一定就是一个生产者线程，因为notify是随机唤醒某一个正在该synchronized对应的锁上面通过wait方式阻塞的线程，如果这时正好还有消费者线程也在阻塞中，则很可能唤醒的是一个消费者线程；signalAll更是会唤醒所有在对应锁上通过wait方式阻塞的线程，而不管是生产者还是消费者线程。

来看下面的生产者消费者例子：

```java
public class Pnc1 {
    private int queueSize = 10;
    private PriorityBlockingQueue<Integer> queue = new PriorityBlockingQueue<>(queueSize);

    public static void main(String[] args) {
        Pnc1 pnc1 = new Pnc1();
        Producer producer1 = pnc1.new Producer();
        Consumer consumer1 = pnc1.new Consumer();
        Producer producer2 = pnc1.new Producer();
        Consumer consumer2 = pnc1.new Consumer();

        producer1.start();
        consumer1.start();
        producer2.start();
        consumer2.start();
    }

    class Consumer extends Thread {
        @Override
        public void run() {
            try {
                consume();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

        private void consume() throws InterruptedException {
            Random random = new Random();
            while (true) {
                Thread.sleep(random.nextInt(2000));
                synchronized (queue) {
                    while (queue.size() == 0) {
                        try {
                            System.out.println(Thread.currentThread().getName() + " 队列空，等待数据");
                            queue.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                            queue.notify();
                        }
                    }
                    queue.poll();
                    queue.notify();
                    System.out.println(Thread.currentThread().getName() + " 从队列取走一个元素，队列剩余" + queue.size() + "个元素");
                }
            }
        }
    }

    class Producer extends Thread {
        @Override
        public void run() {
            try {
                produce();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

        private void produce() throws InterruptedException {
            Random random = new Random();
            while (true) {
                Thread.sleep(random.nextInt(1));
                synchronized (queue) {
                    while (queue.size() == queueSize) {
                        try {
                            System.out.println(Thread.currentThread().getName() + " 队列满，等待有空闲空间");
                            queue.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                            queue.notify();
                        }
                    }
                    queue.offer(1);
                    queue.notify();
                    System.out.println(Thread.currentThread().getName() + " 向队列中插入一个元素，队列剩余空间：" + (queueSize - queue.size()));
                }
            }
        }
    }
}
```

运行结果不确定，但是会有如下的输出：

```
...
Thread-0 队列满，等待有空闲空间
Thread-2 队列满，等待有空闲空间
Thread-3 从队列取走一个元素，队列剩余9个元素
Thread-0 向队列中插入一个元素，队列剩余空间：0
Thread-2 队列满，等待有空闲空间
Thread-0 队列满，等待有空闲空间
Thread-1 从队列取走一个元素，队列剩余9个元素
...
```

我们可以看到，当Thread-0向队列中插入一个元素，这时队列是满的，但是这时唤醒的却是Thread-2生产者线程，当Thread-2执行`queue.wait()`发生阻塞从而让出锁的后，Thread-0又获得了锁，等到Thread-0执行`queue.wait()`发生阻塞从而让出锁的后，Thread-1消费者线程才被唤醒获得锁。

我们发现，使用Object wait notify方法无法准确唤醒响应的线程，造成了一定资源和性能的浪费。

如果使用Condition则不会有这样的问题，看下面的例子：

```java
public class Pnc2 {
    private int queueSize = 10;
    private PriorityBlockingQueue<Integer> queue = new PriorityBlockingQueue<>(queueSize);
    private Lock lock = new ReentrantLock();
    private Condition notFull = lock.newCondition();
    private Condition notEmpty = lock.newCondition();

    public static void main(String[] args) {
        Pnc2 pnc2 = new Pnc2();
        Producer producer1 = pnc2.new Producer();
        Consumer consumer1 = pnc2.new Consumer();
        Producer producer2 = pnc2.new Producer();
        Consumer consumer2 = pnc2.new Consumer();

        producer1.start();
        consumer1.start();
        producer2.start();
        consumer2.start();
    }

    class Consumer extends Thread {
        @Override
        public void run() {
            try {
                consume();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

        private void consume() throws InterruptedException {
            Random random = new Random();
            while (true) {
                Thread.sleep(random.nextInt(1000));
                lock.lock();
                try {
                    while (queue.size() == 0) {
                        try {
                            System.out.println(Thread.currentThread().getName() + " 队列空，等待数据");
                            notEmpty.await();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                    queue.poll();
                    notFull.signal();
                    System.out.println(Thread.currentThread().getName() + " 从队列中取走一个元素，队列剩余" + queue.size() + "个元素");
                } finally {
                    lock.unlock();
                }
            }
        }
    }

    class Producer extends Thread {
        @Override
        public void run() {
            try {
                produce();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

        private void produce() throws InterruptedException {
            Random random = new Random();
            while (true) {
                Thread.sleep(random.nextInt(1));
                lock.lock();
                try {
                    while (queue.size() == queueSize) {
                        try {
                            System.out.println(Thread.currentThread().getName() + " 队列满，等待有空闲空间");
                            notFull.await();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                    queue.offer(1);
                    notEmpty.signal();
                    System.out.println(Thread.currentThread().getName() + " 向队列中插入一个元素，队列剩余空间：" + (queueSize - queue.size()));
                } finally {
                    lock.unlock();
                }
            }
        }
    }
}
```

来看输出的结果：

```
...
Thread-0 队列满，等待有空闲空间
Thread-2 队列满，等待有空闲空间
Thread-1 从队列中取走一个元素，队列剩余9个元素
Thread-0 向队列中插入一个元素，队列剩余空间：0
Thread-0 队列满，等待有空闲空间
Thread-1 从队列中取走一个元素，队列剩余9个元素
Thread-2 向队列中插入一个元素，队列剩余空间：0
Thread-2 队列满，等待有空闲空间
Thread-1 从队列中取走一个元素，队列剩余9个元素
Thread-0 向队列中插入一个元素，队列剩余空间：0
Thread-0 队列满，等待有空闲空间
Thread-3 从队列中取走一个元素，队列剩余9个元素
Thread-2 向队列中插入一个元素，队列剩余空间：0
Thread-2 队列满，等待有空闲空间
Thread-3 从队列中取走一个元素，队列剩余9个元素
Thread-0 向队列中插入一个元素，队列剩余空间：0
Thread-0 队列满，等待有空闲空间
...
```

我们看到，当队列满了以后，不会再唤醒另外的生产者线程，而是精确唤醒消费者线程。这就是Object wait nofity和Condition的区别。




> https://www.cnblogs.com/dolphin0520/p/3920385.html
> 
> https://my.oschina.net/u/174366/blog/608509


