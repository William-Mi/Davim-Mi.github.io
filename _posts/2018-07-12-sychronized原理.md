# synchronized关键字  

## synchronized修饰代码块  
示例代码如下：
```java
package com.davim.test.concurrent;

public class SynchronizedDemo {
    public void method() {
        synchronized (this) {
            System.out.println("Method 1 start");
        }
    }
}
```
反编译对应的class文件，结果如下：  
<a href="https://davim-mi.github.io/">
  <img src="/images/posts/2018-07-12/synchronized_code_block.png">
</a>  
- **monitorenter**  

JVM标准中解释如下：  
The objectref must be of type reference.
Each object is associated with a monitor. A monitor is locked if
and only if it has an owner. The thread that executes monitorenter
attempts to gain ownership of the monitor associated with
objectref, as follows:
1. If the entry count of the monitor associated with objectref is
zero, the thread enters the monitor and sets its entry count to
one. The thread is then the owner of the monitor.
2. If the thread already owns the monitor associated with objectref,
it reenters the monitor, incrementing its entry count.
3. If another thread already owns the monitor associated with
objectref, the thread blocks until the monitor's entry count is
zero, then tries again to gain ownership.

翻译如下：  
每个对象有一个监视器（monitor），此监视器只能被他的占有者锁住，当monitor被占用时就会处于锁定状态，线程执行monitorenter指令时尝试获取monitor的所有权，过程如下：

1. 如果monitor的进入数为0，则该线程进入monitor，然后将进入数设置为1，该线程即为monitor的所有者。

2. 如果线程已经占有该monitor，只是重新进入，则进入monitor的进入数加1.

3. 如果其他线程已经占用了monitor，则该线程进入阻塞状态，直到monitor的进入数为0，再重新尝试获取monitor的所有权  

- **monitorexit**  

JVM标准中解释如下：  
The objectref must be of type reference.
The thread that executes monitorexit must be the owner of the
monitor associated with the instance referenced by objectref.
The thread decrements the entry count of the monitor associated
with objectref. If as a result the value of the entry count is zero, the
thread exits the monitor and is no longer its owner. Other threads
that are blocking to enter the monitor are allowed to attempt to do
so

翻译如下：  
执行monitorexit的线程必须是objectref所对应的monitor的所有者。
指令执行时，monitor的进入数减1，如果减1后进入数为0，那线程退出monitor，不再是这个monitor的所有者。其他被这个monitor阻塞的线程可以尝试去获取这个 monitor 的所有权。  

通过这两段描述，我们应该能很清楚的看出Synchronized的实现原理，Synchronized的语义底层是通过一个monitor的对象来完成，其实wait/notify等方法也依赖于monitor对象，这就是为什么只有在同步的块或者方法中才能调用wait/notify等方法，否则会抛出java.lang.IllegalMonitorStateException的异常的原因  
***
## sychronized修饰方法  
示例代码如下：
```java
package com.davim.testgilde;

public class SynchronizedMethod {
    public synchronized void method() {
        System.out.println("Hello World!");
    }
}
```
反编译对应的class文件，结果如下：  
<a href="https://davim-mi.github.io/">
  <img src="/images/posts/2018-07-12/synchronized_method.png">
</a>  

从反编译的结果来看，方法的同步并没有通过指令monitorenter和monitorexit来完成（理论上其实也可以通过这两条指令来实现），不过相对于普通方法，其常量池中多了ACC_SYNCHRONIZED标示符。JVM就是根据该标示符来实现方法的同步的：当方法调用时，调用指令将会检查方法的 ACC_SYNCHRONIZED 访问标志是否被设置，如果设置了，执行线程将先获取monitor，获取成功之后才能执行方法体，方法执行完后再释放monitor。在方法执行期间，其他任何线程都无法再获得同一个monitor对象。 其实本质上没有区别，只是方法的同步是一种隐式的方式来实现，无需通过字节码来完成

