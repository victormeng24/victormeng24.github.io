---
title: Leetcode-1114-Print-In-Order
date: 2020-12-31 11:42:51
tags:
- 算法
- 多线程
---
# 题目描述
我们提供了一个类：

public class Foo {
  public void first() { print("first"); }
  public void second() { print("second"); }
  public void third() { print("third"); }
}
三个不同的线程将会共用一个 Foo 实例。

线程 A 将会调用 first() 方法
线程 B 将会调用 second() 方法
线程 C 将会调用 third() 方法
请设计修改程序，以确保 second() 方法在 first() 方法之后被执行，third() 方法在 second() 方法之后被执行。
示例 1:
输入: [1,2,3]()
输出: "firstsecondthird"
解释: 
有三个线程会被异步启动。
输入 [1,2,3]() 表示线程 A 将会调用 first() 方法，线程 B 将会调用 second() 方法，线程 C 将会调用 third() 方法。
正确的输出是 "firstsecondthird"。
示例 2:
输入: [1,3,2]()
输出: "firstsecondthird"
解释: 
输入 [1,3,2]() 表示线程 A 将会调用 first() 方法，线程 B 将会调用 third() 方法，线程 C 将会调用 second() 方法。
正确的输出是 "firstsecondthird"。
提示：
尽管输入中的数字似乎暗示了顺序，但是我们并不保证线程在操作系统中的调度顺序。
你看到的输入格式主要是为了确保测试的全面性。

来源：力扣（LeetCode）
链接：https://leetcode-cn.com/problems/print-in-order
著作权归领扣网络所有。商业转载请联系官方授权，非商业转载请注明出处。
# 题解
刷Leetcode又换回java了，感觉还是java提供的基础数据结构更加丰富，不足是java写UT真心没有Golang方便，这里分享一个非常好用的Java刷LC的[工具][5]，支持一键生成Solution和UT的代码骨架，让你专注于实现算法逻辑。
回到这道题，easy的题目有很多宝藏啊，这道题一道题可以练习不少多线程的相关知识，这里总结几种解题思路。
## Semaphore信号量
利用两个信号量保证并发调用顺序
```js
class Foo {
    Semaphore two;
    Semaphore three;

    public Foo() {
        two = new Semaphore(0);
        three = new Semaphore(0);
    }

    public void first(Runnable printFirst) throws InterruptedException {

        // printFirst.run() outputs "first". Do not change or remove this line.
        printFirst.run();
        two.release();
    }

    public void second(Runnable printSecond) throws InterruptedException {

        // printSecond.run() outputs "second". Do not change or remove this line.
        two.acquire();
        printSecond.run();
        three.release();
    }

    public void third(Runnable printThird) throws InterruptedException {

        // printThird.run() outputs "third". Do not change or remove this line.
        three.acquire();
        printThird.run();
    }
}
```
## CountDownLatch
利用CountDownLatch计数保证并发访问，
```js
class Foo {

    CountDownLatch c2;
    CountDownLatch c3;

    public Foo() {
        c2 = new CountDownLatch(1);
        c3 = new CountDownLatch(1);
    }

    public void first(Runnable printFirst) throws InterruptedException {

        // printFirst.run() outputs "first". Do not change or remove this line.
        printFirst.run();
        c2.countDown();
    }

    public void second(Runnable printSecond) throws InterruptedException {

        // printSecond.run() outputs "second". Do not change or remove this line.
        c2.await();
        printSecond.run();
        c3.countDown();
    }

    public void third(Runnable printThird) throws InterruptedException {

        // printThird.run() outputs "third". Do not change or remove this line.
        c3.await();
        printThird.run();
    }
}
```
## 阻塞队列
利用阻塞队列放入令牌的方式控制访问顺序。
```js
class Foo {

    BlockingQueue blockingQueue1;
    BlockingQueue blockingQueue2;

    public Foo() {
        blockingQueue1 = new SynchronousQueue();
        blockingQueue2 = new SynchronousQueue();
    }

    public void first(Runnable printFirst) throws InterruptedException {

        // printFirst.run() outputs "first". Do not change or remove this line.
        printFirst.run();
        blockingQueue1.put("Go");
    }

    public void second(Runnable printSecond) throws InterruptedException {

        // printSecond.run() outputs "second". Do not change or remove this line.
        blockingQueue1.take();
        printSecond.run();
        blockingQueue2.put("Go");
    }

    public void third(Runnable printThird) throws InterruptedException {

        // printThird.run() outputs "third". Do not change or remove this line.
        blockingQueue2.take();
        printThird.run();
    }
}
```
# 思考
一题多解，对于实践JUC包中的各种数据结构非常有帮助，很有价值的一道题。

[5]:	https://github.com/helloShen/leetcode-helper "代码框架"