# Java多线程之Volatile关键字

## 1. Volatile语义

通常，Volatile关键字用于多线程优化方面，被用来阻止(伪)编译器认为的无法“被代码本身”改变的代码（变量/对象）进行优化。

## 2. Java中的“volatile”关键字

在JDK1.2之前，JMM(Java Memory Modle)的实现总是从**主存**（即共享内存）读取变量，是不需要进行特别的注意的。而当前的JMM，线程可以把变量保存**本地内存**（如机器的寄存器）中，而不是直接在主存中进行读写。这可能造成一个线程在主存中修改了一个变量的值，而另外一个线程还继续使用它在寄存器中的变量值的拷贝，造成**数据的不一致**。

在Java中，为了解决这个问题，就使用了“volatile”关键字。它指示JVM，这个变量是不稳定的，每次使用这个变量都要到主存中进行读取。

## 3. volatile关键字的可见性

**volatile 修饰的成员变量**在每次被线程访问时，都强迫**从主存（共享内存）中重读该成员变量的值**。而且，当成员变量发生变化时，**强迫线程将变化值回写到主存（共享内存）**。这样在任何时刻，**两个不同的线程总是看到某个成员变量的同一个值**，这样也就保证了同步数据的**可见性**。

```java
public class VolatileExample {
//  volatile boolean flag = true;
    boolean flag = true;
	public static void main(String[] args) throws InterruptedException {
		VolatileExample n = new VolatileExample();
		n.foo();
		Thread.sleep(1000);
		n.flag = false;
	}
	
	void foo() {
		new Thread(()->{
			while (flag) {
            }
			System.out.println("end");
		}).start();
	}

}	
```

运行结果将出现**死循环**，这是因为isRunning变量虽然被修改但是没有被写到主存中，这样该线程在本地内存中的值一直为true，这样就导致了死循环的产生。

解决办法就是注释中的加上volatile关键字。

**但是，当我们在while循环里加上输出语句或者sleep方法，不管flag是否被加上volatile，过一段时间死循环也会停止。**

加上输出语句：

```java
void foo() {
		new Thread(()->{
			while (flag) {
                System.out.println("foo");
            }
			System.out.println("end");
		}).start();
	}
```

加上sleep方法：

```java
void foo() {
		new Thread(()->{
			while (flag) {
                //try{}catch(){}
                Thread.sleep(1000);
            }
			System.out.println("end");
		}).start();
	}
```

**这是因为**，JVM会尽力保证内存的可见性，即便这个变量没有加上同步关键字。换句话说，只要CPU有时间，那么JVM就回去抢占CPU来更新这个资源。而volatile关键字是强制保证线程的可见性，JVM是尽力保证，这个保证的前提就是CPU没有事情做。一开始的代码一直处于死循环中，CPU一直处于占用状态，JVM就无法强制要求CPU分时间取最新的变量。而**加了输出或者sleep语句，CPU就有可能有时间去保证内存的可见性，所以while循环一样可以终止。**

## 4. volatile关键字能保证原子性吗？

### 4.1 原子性

> 并发编程中的原子性：一个操作是不可中断的，要么全部成功要么全部失败，且一旦操作开始，就不会被其它线程所干扰。

```java
int a = 10; //1
a++; //2
int b=a; //3
a = a+1; //4
```

上面这四个语句中，只有**第1个语句是原子操作**，将10赋值给线程工作内存的变量a，而语句2，实际上包含了三个操作：1.读取变量a的值；2.对a进行＋1的操作；3.将计算后的值再赋值给变量a，这三个操作无法构成原子操作。语句3、4同理。

JMM定义了8种操作都是原子的，不可再分

1. lock(锁定)：作用于主内存中的变量，它把一个变量标识为一个线程独占的状态；

2. unlock(解锁):作用于主内存中的变量，它把一个处于锁定状态的变量释放出来，释放后的变量才可以被其他线程锁定

3. read（读取）：作用于主内存的变量，它把一个变量的值从主内存传输到线程的工作内存中，以便后面的load动作使用；

4. load（载入）：作用于工作内存中的变量，它把read操作从主内存中得到的变量值放入工作内存中的变量副本
5. use（使用）：作用于工作内存中的变量，它把工作内存中一个变量的值传递给执行引擎，每当虚拟机遇到一个需要使用到变量的值的字节码指令时将会执行这个操作；
6. assign（赋值）：作用于工作内存中的变量，它把一个从执行引擎接收到的值赋给工作内存的变量，每当虚拟机遇到一个给变量赋值的字节码指令时执行这个操作；
7. store（存储）：作用于工作内存的变量，它把工作内存中一个变量的值传送给主内存中以便随后的write操作使用；
8. write（操作）：作用于主内存的变量，它把store操作从工作内存中得到的变量的值放入主内存的变量中。

### 4.2 无法保证原子性

**volatile无法保证对变量原子性的，synchronized满足原子性。**

```java
public class VolatileExample {
    private static volatile int counter = 0;

    public static void main(String[] args) {
        for (int i = 0; i < 10; i++) {
            Thread thread = new Thread(new Runnable() {
                @Override
                public void run() {
                    for (int i = 0; i < 10000; i++)
                        counter++;
                }
            });
            thread.start();
        }
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(counter);
    }
}
```

上述代码理想结果是10*10000=100000，但是每次运行都是小于100000的，这是因为**volatile不能保证原子性**，counter++并不是一个原子操作。线程A读取counter到工作内存中，其它线程会对这个值已经做了自增操作，那么A的值就已经过期了，因此总结果必然是小于100000的。

使用synchronized关键字

```java
public class ao implements Runnable{
    private static int count;

    public ao(){
        count = 0;
    }

    @Override
    public void run() {
        synchronized (this){
            for (int i = 0; i < 10000; i++) {
                  count++;
            }
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(count);
        }
    }
}

public class vo {
    public static void main(String[] args) {
        ao o = new ao();
        for (int i = 0; i <10 ; i++) {
            Thread thread = new Thread(o,"th"+i);
            thread.start();
        }

    }
}
```

## 5. synchronized关键字和volatile关键字比较

- **volatile关键字**是线程同步的**轻量级实现**，所以**volatile性能肯定比synchronized关键字要好**。但是**volatile关键字只能用于变量而synchronized关键字可以修饰方法以及代码块**。synchronized关键字在JavaSE1.6之后进行了主要包括为了减少获得锁和释放锁带来的性能消耗而引入的偏向锁和轻量级锁以及其它各种优化之后执行效率有了显著提升，**实际开发中使用synchronized关键字还是更多一些**。
- **多线程访问volatile关键字不会发生阻塞，而synchronized关键字可能会发生阻塞**。
- **volatile关键字能保证数据的可见性，但不能保证数据的原子性。synchronized关键字两者都能保证。**
- **volatile关键字用于解决变量在多个线程之间的可见性，而ynchronized关键字解决的是多个线程之间访问资源的同步性。**

- synchronized：具有原子性、有序性和可见性；
- volatile：具有有序性和可见性；