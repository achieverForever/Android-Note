#### Message - 携带数据&维护Message pool
1. 一个Message包括了以下几个重要的字段:
    ```java
    class Message {
        
        /************** 以下用于存放Message的内容 ************************/

        int what;       // 消息的代码
        int arg1;       // arg1和arg2是用于携带轻量级的数据
        int arg2;    
        Object obj;     // 一个Message可以携带任意对象，前提是该obj要实现Parcelable
        Bundle data;    // 还可以携带一个Bundle 
        Runnable callback;  // 也可以携带一个Runnable
        
        /************* 以下用于Message的投递 *****************************/
        Handler target;     // 指明Message要发送给谁

        /*************** 以下用于维护Message poll ************************/
        Message next;               // 将Message pool中的Message链接起来
        static Message sPool;       // 指向Message pool(实际上是一个linked list)下一个将要返回给client使用的Message
        static int sPoolSize;       // 记录当前Message pool中可用的Message数
        
        // 其他省略。。。
    
    }
    ```
    
    **Note**: Handler可以向MessageQueue投递Message和Runnable对象，其实Runnable是封装在Message的callback字段中的。所以MessageQueue中的都是Message对象。
    
2. Message pool的实现
    - `obtain()` 获取一个Message，如果Message pool不为空，则直接从sPool指向的位置取出一个Message返回；否则新创建一个Message.
    ```java
    /**
     * Return a new Message instance from the global pool. Allows us to
     * avoid allocating new objects in many cases.
     */
    public static Message obtain() {
        synchronized (sPoolSync) {  // 用于保护sPool和sPoolSize
            if (sPool != null) {    // Message pool不空，直接返回sPool所指向的Message
                Message m = sPool; 
                sPool = m.next;     // sPool指向下一个位置
                m.next = null;
                sPoolSize--;
                return m;
            }
        }
        return new Message();       // Message pool为空，返回一个新的Message
    }
     ```

    - `recycle()` 
    ```java
    /**
     * Return a Message instance to the global pool.  You MUST NOT touch
     * the Message after calling this function -- it has effectively been
     * freed.
     */
    public void recycle() {
        clearForRecycle();      // 清除对其他对象的引用，使得其他对象能被GC回收，并让Message准备好循环使用

        synchronized (sPoolSync) {
            if (sPoolSize < MAX_POOL_SIZE) {
                next = sPool;   // 调整sPool指针
                sPool = this;
                sPoolSize++;
            }
        }
    }
    ```

#### MessageQueue 
1. 每个Looper线程都有一个MessageQueue，Looper负责在MessageQueue抽出一个Message，交给Handler处理，而Handler也可以通过`post*()`和`send*()`往MessageQueue中投递消息
2. 几个关键方法：
    `Message next();`  从MessageQueue中取出一个Message；
    `boolean enqueueMessage(Message msg, long when);`  将一个Message放到MessageQueue中，必要时，如当MessageQueue由empty变为non-empty的时候唤醒Event queue？

    `void nativePollOnce(int ptr, int timeoutMillis);`  C函数，轮询？
    
    下面一一剖析：
    `MessageQueue.next()`
    ```java
    Message next() {
        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        int nextPollTimeoutMillis = 0;
        for (;;) {
            // 不懂，跳过
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }

            // 在nextPollTimeoutMillis毫秒之后轮询what？ 
            nativePollOnce(mPtr, nextPollTimeoutMillis);

            synchronized (this) {
                // Try to retrieve the next message.  Return if found.
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                if (msg != null && msg.target == null) {
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    // 遇到了SyncBarrier(target为null的Message)，同步的Message都被挂起了，然而异步的Message不受影响，尝试继续找下一个异步的Message
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {
                    if (now < msg.when) {
                        // Next message is not ready.  Set a timeout to wake up when it is ready.
                        // 这个Message使用了delay，还没到触发的时候呢？设置nextPollTimeoutMillis，当这个Message变为ready的时候再Poll一次
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // Got a message.
                        // 从MessageQueue中取出一个Message
                        mBlocked = false;
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        if (false) Log.v("MessageQueue", "Returning message: " + msg);
                        msg.markInUse();    // 标记这个Message为使用中，使得不会被Message pool拿出去recycle
                        return msg;
                    }
                } else {
                    // No more messages.
                    // 木有Message了~
                    nextPollTimeoutMillis = -1;
                }

                // Process the quit message now that all pending messages have been handled.
                if (mQuitting) {
                    dispose();
                    return null;
                }
                
                // 以下涉及到PendingIdleHandler，不懂，跳过。。。
        }
    }
    
    ```
    
    MessageQueue.enqueueMessage()
    ```java
    boolean enqueueMessage(Message msg, long when) {

        synchronized (this) {

            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            if (p == null || when == 0 || when < p.when) {
                // 新的head，如果现在blocked唤醒event queue。因为MessageQueue有empty变为non-empty了，有消息了喂~
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                // 在MessageQueue中间插入一个Message，当且仅当现在blocked，有一个SyncBarrier在MessageQueue头部，而且新插入的Message为异步Message时，才需要唤醒event queue；因为MessageQueue由停滞状态变为有消息了。
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                for (;;) {  // 根据Message的when字段在MessageQueue中找到插入位置
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }

            if (needWake) {
                nativeWake(mPtr);   // 唤醒event queue，具体是啥我也不知道~
            }
        }
        return true;
    }    
    ```
    
#### Handler 投递消息&处理消息
1. Hander用于投递和处理Message和Runnable(被封装成Message)。每个Handler都关联一个线程和该线程的MessageQueue，当创建Handler的时候，默认是Handler是与创建它的线程关联起来的。所以Handler只能向创建它的线程的MessageQueue投递消息或者处理该MessageQueue上的消息。
2. Handler的用处有二：
    - 调度Message或者Runnable使得他们在某个时刻被处理或执行；
    - 将一个Message或者Runnable投递给另一个线程执行，例如在WorkerThread中完成Bitmap的下载，然后将显示的操作封装在一个Message或者Runnable中，投递给UI Thread执行。因为Android规定，不同从其他线程访问UI Thread的数据，否则会crash!!!

3. 关键方法与接口:
    - `Handler.Callback interface`  声明了`handleMessage(msg)`，用于处理Looper分派过来的Message。
    - `void dispatchMessage(Message msg)`  由Looper调用，用于分派Message给Handler
    - `boolean post*()`  投递一个Runnable到与Handler关联线程的MessageQueue
    - `boolean send*Message()`  投递一个Message到与Handler关联的MessageQueue
    
    **Note**: post*()和send*Message()都需要调用MessageQueue.enqueueMessage()。

#### Looper 维护一个线程的MessageQueue&负责从MessageQueue中提取消息
1. 重要的字段:
    ```java
public final class Looper {

    // 使用TLS(线程本地存储)来存放Looper，所以每个线程只有一个Looper
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
    private static Looper sMainLooper;  // guarded by Looper.class

    final MessageQueue mQueue;          // 要维护的MessageQueue
    final Thread mThread;               // 关联的线程
}
    ```
    
2. 一个普通的线程默认是没有消息循环的，要加上消息循环的功能，必要要变为Looper线程。可以调用`Looper.prepare()`创建一个MessageQueue，并创建一个该线程本地的唯一一个Looper实例管理MessageQueue，然后调用`Loop.loop()`不断地loop MessageQueue，从MessageQueue中提取Message，分发给关联的Handler进行处理。

3. 重点方法剖析
    - `Looper.prepare(boolean quitAllowed);`  
    ```java
    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {   // TLS中已经存在Looper实例了，还prepare个毛线啊~
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));  // 在TLS中创建一个Looper的实例
    }
    ```
    - `private Looper.Looper(boolean quitAllowed);`  声明为私有，因为应该从prepare处调用 
    ```java
    private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed); // 创建一个MessageQueue
        mThread = Thread.currentThread();       // 关联当前线程
    }
    ```
    - `loop()`  在当前线程中运行消息循环，即从MessageQueue中取出一个Message，然后分派给与该Message关联的Handler进行处理
    ```java
    /**
     * Run the message queue in this thread. Be sure to call
     * {@link #quit()} to end the loop.
     */
    public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;

        // 不懂。。。
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

        for (;;) {
            Message msg = queue.next(); // 调用MessageQueue.next()，提取出下一个Message，若没有ready的Message，阻塞。。。
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }
            
            msg.target.dispatchMessage(msg);    // 调用与此消息关联的Handler的dispatchMessage()，将此Message分发给该Handler进行处理

            // 将此Message丢回Message poll中准备重用
            msg.recycle();
        }
    }            
    ```
    
