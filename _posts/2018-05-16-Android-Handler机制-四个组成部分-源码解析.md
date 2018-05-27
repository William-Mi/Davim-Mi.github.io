# Android Handler机制，四个组成部分及源码解析  

## 概述
Handler 、 Looper 、Message 、MessageQueue等组成了Android异步消息处理机制  
其中，异步消息处理线程启动后会进入一个无限循环体之中，每循环一次，从其内部的消息队列中取出一个消息，然后回调相应的消息处理函数，执行完成一个消息后则继续循环。若消息队列为空，线程则会阻塞等待。  
Looper负责的就是创建一个MessageQueue，然后进入一个无限循环体不断从该MessageQueue中读取消息，而Message的创建者就是一个或多个Handler。  

## Looper
Class used to run a message loop for a thread. Threads by default do not have a message loop associated with them; to create one, call prepare() in the thread that is to run the loop, and then loop() to have it process messages until the loop is stopped.  

Most interaction with a message loop is through the Handler class.  

This is a typical example of the implementation of a Looper thread, using the separation of prepare() and loop() to create an initial Handler to communicate with the Looper.  
```java
class LooperThread extends Thread {
      public Handler mHandler;

      public void run() {
          Looper.prepare();

          mHandler = new Handler() {
              public void handleMessage(Message msg) {
                  // process incoming messages here
              }
          };

          Looper.loop();
      }
  }
  ```  
### Looper创建
 ```java
private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }
```   
* 创建一个MessageQueue实例
* 获取当前线所在的线程  

### Looper.prepare( )
Initialize the current thread as a looper. This gives you a chance to create handlers that then reference this looper, before actually starting the loop. Be sure to call loop() after calling this method, and end it by calling quit().  
```java
private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
```  
sThreadLocal是一个ThreadLocal对象，可以在一个线程中存储变量。
首先判断looper所在的当前线程中是否设置了Looper；  
* 如果设置了，直接抛出异常，这也就说明了Looper.prepare()方法不能被调用两次，同时也保证了一个线程中只有一个Looper实例  
* 如果没有设置，为当前线程创建一个Looper实例  

### Looper.loop( )
Run the message queue in this thread. Be sure to call quit() to end the loop.  

```java
public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;

        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

            // This must be in a local variable, in case a UI event sets the logger
            Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }

            msg.target.dispatchMessage(msg);

            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }

            // Make sure that during the course of dispatching the
            // identity of the thread wasn't corrupted.
            final long newIdent = Binder.clearCallingIdentity();
            if (ident != newIdent) {
                Log.wtf(TAG, "Thread identity changed from 0x"
                        + Long.toHexString(ident) + " to 0x"
                        + Long.toHexString(newIdent) + " while dispatching to "
                        + msg.target.getClass().getName() + " "
                        + msg.callback + " what=" + msg.what);
            }

            msg.recycleUnchecked();
        }
    }
```  
    
执行流程如下：
1. public static Looper myLooper() {
    return sThreadLocal.get();
    }
    方法直接返回了sThreadLocal存储的Looper实例，如果me为null则抛出异常，也就是说looper方法必须在prepare方法之后运行。  
2. 获取looper实例中的MessageQueue
3. 进入无限循环
4. 取出一条消息，如果没有消息则阻塞
5. 调用 msg.target.dispatchMessage(msg);把消息交给msg的target的dispatchMessage方法去处理。Msg的target其实就是handler对象  

## Handler
A Handler allows you to send and process Message and Runnable objects associated with a thread's MessageQueue. Each Handler instance is associated with a single thread and that thread's message queue. When you create a new Handler, it is bound to the thread / message queue of the thread that is creating it -- from that point on, it will deliver messages and runnables to that message queue and execute them as they come out of the message queue.  

There are two main uses for a Handler: (1) to schedule messages and runnables to be executed as some point in the future; and (2) to enqueue an action to be performed on a different thread than your own.  

Scheduling messages is accomplished with the post(Runnable), postAtTime(Runnable, long), postDelayed(Runnable, Object, long), sendEmptyMessage(int), sendMessage(Message), sendMessageAtTime(Message, long), and sendMessageDelayed(Message, long) methods. The post versions allow you to enqueue Runnable objects to be called by the message queue when they are received; the sendMessage versions allow you to enqueue a Message object containing a bundle of data that will be processed by the Handler's handleMessage(Message) method (requiring that you implement a subclass of Handler).   

When posting or sending to a Handler, you can either allow the item to be processed as soon as the message queue is ready to do so, or specify a delay before it gets processed or absolute time for it to be processed. The latter two allow you to implement timeouts, ticks, and other timing-based behavior.  

When a process is created for your application, its main thread is dedicated to running a message queue that takes care of managing the top-level application objects (activities, broadcast receivers, etc) and any windows they create. You can create your own threads, and communicate back with the main application thread through a Handler. This is done by calling the same post or sendMessage methods as before, but from your new thread. The given Runnable or Message will then be scheduled in the Handler's message queue and processed when appropriate.  


```java
    public Handler() {  
        this(null, false);  
	}  
    public Handler(Callback callback, boolean async) {  
        if (FIND_POTENTIAL_LEAKS) {  
            final Class<? extends Handler> klass = getClass();  
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&  
                    (klass.getModifiers() & Modifier.STATIC) == 0) {  
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +  
                    klass.getCanonicalName());  
            }  
        }  
  
        mLooper = Looper.myLooper();  
        if (mLooper == null) {  
            throw new RuntimeException(  
                "Can't create handler inside thread that has not called Looper.prepare()");  
        }  
        mQueue = mLooper.mQueue;  
        mCallback = callback;  
        mAsynchronous = async;  
    }  
```  
    
通过Looper.myLooper()获取了当前线程保存的Looper实例，然后就可以获得在这个Looper实例中保存的MessageQueue（消息队列），这样就保证了handler的实例与我们Looper实例中MessageQueue关联上了。  
 
### sendMessage
```java
    /**
     * Pushes a message onto the end of the message queue after all pending messages
     * before the current time. It will be received in {@link #handleMessage},
     * in the thread attached to this handler.
     *  
     * @return Returns true if the message was successfully placed in to the 
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.
     */
    public final boolean sendMessage(Message msg)
    {
        return sendMessageDelayed(msg, 0);
    }
```  
    
```java
    public final boolean sendEmptyMessageDelayed(int what, long delayMillis) {  
     Message msg = Message.obtain();  
     msg.what = what;  
     return sendMessageDelayed(msg, delayMillis);  
 }
```
```java
    public final boolean sendMessageDelayed(Message msg, long delayMillis)
    {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }  
```  
```java
    /**
     * Enqueue a message into the message queue after all pending messages
     * before the absolute time (in milliseconds) <var>uptimeMillis</var>.
     * <b>The time-base is {@link android.os.SystemClock#uptimeMillis}.</b>
     * Time spent in deep sleep will add an additional delay to execution.
     * You will receive it in {@link #handleMessage}, in the thread attached
     * to this handler.
     * 
     * @param uptimeMillis The absolute time at which the message should be
     *         delivered, using the
     *         {@link android.os.SystemClock#uptimeMillis} time-base.
     *         
     * @return Returns true if the message was successfully placed in to the 
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.  Note that a
     *         result of true does not mean the message will be processed -- if
     *         the looper is quit before the delivery time of the message
     *         occurs then the message will be dropped.
     */
    public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }
```


```java
    private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```  
enqueueMessage中首先为meg.target赋值为this，【Looper的loop方法会取出每个msg然后交给msg的target的dispatchMessage(msg)方法去处理消息】，也就是把当前的handler作为msg的target属性，最终会调用MessageQueue的enqueueMessage的方法，也就是说handler发出的消息，最终会保存到Looper中的MessageQueue中去  

==现在已经很清楚了Looper会调用prepare()和loop()方法，在当前执行的线程中保存一个Looper实例，这个实例会保存一个MessageQueue对象，然后当前线程进入一个无限循环中去，不断从MessageQueue中读取Handler发来的消息。然后再回调创建这个消息的handler中的dispathMessage方法==  

### post(Runnable r)
```java
    /**
     * Causes the Runnable r to be added to the message queue.
     * The runnable will be run on the thread to which this handler is 
     * attached. 
     *  
     * @param r The Runnable that will be executed.
     * 
     * @return Returns true if the Runnable was successfully placed in to the 
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.
     */
    public final boolean post(Runnable r)
    {
       return  sendMessageDelayed(getPostMessage(r), 0);
    }
```
    
==产生一个Message对象，可以new  ，也可以使用Message.obtain()方法；两者都可以，但是更建议使用obtain方法，因为Message内部维护了一个Message池用于Message的复用，避免使用new 重新分配内存==
```java
private static Message getPostMessage(Runnable r) {
        Message m = Message.obtain();
        m.callback = r;
        return m;
    }
```  

在getPostMessage中，得到了一个Message对象，然后将我们创建的Runable对象作为callback属性，赋值给了此message  

### dispatchMessage( )
```java
public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }
```  
最后调用了handleMessage方法  
```java
    /**
     * Subclasses must implement this to receive messages.
     */
    public void handleMessage(Message msg) {
    }
```

这是一个空方法，因为消息的最终回调是由我们控制的，我们在创建handler的时候都是复写handleMessage方法，然后根据msg.what进行消息处理

## 总结
1. 首先Looper.prepare()在本线程中保存一个Looper实例，然后该实例中保存一个MessageQueue对象；因为Looper.prepare()在一个线程中只能调用一次，所以MessageQueue在一个线程中只会存在一个。

2. Looper.loop()会让当前线程进入一个无限循环，不端从MessageQueue的实例中读取消息，然后回调msg.target.dispatchMessage(msg)方法。

3. Handler的构造方法，会首先得到当前线程中保存的Looper实例，进而与Looper实例中的MessageQueue关联。

4. Handler的sendMessage方法，会给msg的target赋值为handler自身，然后加入MessageQueue中。

5. 在构造Handler实例时，我们会重写handleMessage方法，也就是msg.target.dispatchMessage(msg)最终调用的方法。

















