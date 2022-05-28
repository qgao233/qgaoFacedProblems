# 5.28

## 1 System.arraycopy线程安全问题

2022-5-28 11:39:27

>服了现在网上的东西，一堆全是复制粘贴的文章到处都是，到底有没有仔细思考里面代码的逻辑是否合理，阐述的观点是否正确，那些家伙完全没有进行思考，就一通复制。

[java中System.arraycopy是线程安全的吗？](https://segmentfault.com/q/1010000007608412)

上述链接的回答，被抄得到处都是，比如：

* [System.arraycopy详解](https://blog.csdn.net/wqy__007/article/details/115623270)
* [java线程安全的数组_java中System.arraycopy是线程安全的吗？](https://blog.csdn.net/weixin_42594427/article/details/114709155)
* [【Java集合 6】arraycopy方法的作用](https://blog.csdn.net/guorui_java/article/details/113187970) 这一篇居然还收费...

这些文章中主要描述了如何论证`System.arraycopy`是否线程安全。

关键代码如下：

```
public class ArrayCopyThreadSafe {
    private static int[] arrayOriginal = new int[1024 * 1024 * 10];
    private static int[] arraySrc = new int[1024 * 1024 * 10];
    private static int[] arrayDist = new int[1024 * 1024 * 10];
    private static ReentrantLock lock = new ReentrantLock();

    private static void modify() {
        for (int i = 0; i < arraySrc.length; i++) {
            arraySrc[i] = i + 1;
        }
    }

    private static void copy() {
        System.arraycopy(arraySrc, 0, arrayDist, 0, arraySrc.length);
    }

    private static void init() {
        for (int i = 0; i < arraySrc.length; i++) {
            arrayOriginal[i] = i;
            arraySrc[i] = i;
            arrayDist[i] = 0;
        }
    }

    private static void doThreadSafeCheck() throws Exception {
        for (int i = 0; i < 100; i++) {
            System.out.println("run count: " + (i + 1));
            init();
            Condition condition = lock.newCondition();

            new Thread(new Runnable() {
                @Override
                public void run() {
                    lock.lock();
                    condition.signalAll();
                    lock.unlock();
                    copy();
                }
            }).start();


            lock.lock();
            // 这里使用 Condition 来保证拷贝线程先已经运行了.
            condition.await();
            lock.unlock();

            Thread.sleep(2); // 休眠2毫秒, 确保拷贝操作已经执行了, 才执行修改操作.
            modify();

            if (!Arrays.equals(arrayOriginal, arrayDist)) {
                throw new RuntimeException("System.arraycopy is not thread safe");
            }
        }
    }

    public static void main(String[] args) throws Exception {
        doThreadSafeCheck();
    }
}
```

他们得出的结论是：

>如果 System.arraycopy 是线程安全的, 那么先执行拷贝操作, 再执行修改操作时, 不会影响复制结果, 因此 arrayOriginal 必然等于 arrayDist; 而如果 System.arraycopy 是线程不安全的, 那么 arrayOriginal 不等于 arrayDist.

### 1.1 疑问1

但经过我的仔细分析，先不说他们判断的依据是否正确，就说上面有关条件变量`condition`的使用就是多此一举，

既然想要实现：

* 先子线程的**复制操作** 
* 后主线程的**修改操作**

直接`sleep`会不会好一点？还需要`lock`和`condition`配合使用吗？

>普及一下lock和condition配合使用的方法（可用来实现**生产者-消费者模式**）：
>
>* 当调用`condition.await()`时，会释放当前获得的锁，但会让线程阻塞在当前条件变量上，
>* 而当调用`condition.signalAll()`时，会唤醒所有阻塞在当前条件变量的线程。

### 1.2 疑问2

接着是他们所谓的判断依据，我仔细思考后，感觉他们想表达出的观点是：如果子线程开始copy了但没copy完，此时主线程抢到cpu资源开始执行修改src数组的操作，之后，主线程与子线程互相争夺cpu，这样就导致dest(原代码中单词写错)数组的东西必然与original数组的不一样。

换句话说，就是只要在System.arraycopy复制过程中，有另外的线程去修改了src数组（共享资源），那么就导致子线程中的结果与预料的不一样。

因此，当使用System.arraycopy时，必须得加锁，若有其他线程要修改src数组，则必须获得锁才行。

这才是真正的所谓的System.arraycopy线程不安全的原因以及正确的使用方法！！！