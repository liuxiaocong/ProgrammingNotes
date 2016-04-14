内存

#1 volatile的内存语义
当声明变量为volatile后，对这个变量的读/写将会很特别，为了深入理解volatile实现的原理，接下来学习volatile的内存语义和volatile的内存语义的实现

## 1.1 volatile的特性

理解volatile的特性的一个好办法是把对volatile变量的单个读/写，看成是使用同一个所对这些单个读/写操作做了同步。例如：

volatile代码实例

    public class VolatileFeatureExample {

        volatile long v1 = 0L;//声明volatile的变量

        public void set(long l) {//volatile 的set
            v1 = l;
        }

        public void getAndIncrement() {//复合volatile的读/写
            v1++;
        }

        public long get() {//volatile的读
            return v1;
        }
    }


上面代码示例等同于下面同步后的代码

    class VolatileFeatureExample2 {

        long v1 = 0L;//声明普通变量

        public synchronized void set(long l) {//同步的写
            v1 = l;
        }

        public void getAndIncrement() {
            long temp = get();//同步的读
            temp += 1L;//普通的写
            set(temp);//同步写
        }

        public synchronized long get() {//同步的写
            return v1;
        }
    }


锁的happens-before规则保证释放锁和获取锁的两个线程之间的内存可见性，这意味着：**对一个volatile变量的写，总是能看到（任意线程）对这个volatile变量最后的写入。**

而所的临界区的执行具有原子性，意味着即使是对64数据的变量类型，只要他们是volatile的，对该变量的读/写都具有原子性，**但是多个volatile变量的操作，类似volatile++这种复合操作，不具有原子性**。

所以总得volatile变量自身具有如下特性：
- 可见性 对一个volatile变量的读，总是可以看到任意线程对这个volatile变量的写
- 原子性 对任意单个volatile变量的读/写都具有原子性，但类似于volatile++这种复合的volatile操作，不具有原子性

## 1.2 volatile写-读建立的关系

从JSR-133(JDK5)开始，volatile变量的写-读可以实现线程之间的通信,从内存语义来讲，volatile的的写读操作与所的释放-获取具有相同的效果，即volatile的写于锁的释放具有相同的语义，volatile的都与锁的获取具有相同的语义。如：


    class VolatileExample {
         int a = 0;              
         boolean flag = false;

        public void writer() {
            a = 1;                   //步骤1
            flag = true;             //步骤2
        }


        public void reader() {

            if (flag) {                    //步骤3
                int i = a;                 //步骤4
                System.out.println(i);     
            }
        }
    }


           new Thread(new Runnable() {//A
                @Override
                public void run() {
                    volatileExample.writer();
                }
            }).start();


            new Thread(new Runnable() {//B
                @Override
                public void run() {
                    volatileExample.reader();
                }
            }).start();


*`如果`* **线程A执行writer后线程B执行reader**，根据happens befor规则，这个过程建立的happens bofore关系如下：

- 1 happens bofore 2
- 3 happens bofore 4
- 根据volatile的内存语义，2  happens bofore 3 (读 before 写)
- 根据 happens bofore的传递性，1  happens bofore 4














>未完
