---
layout: default

---

Tags : ZAZE

---

[TOC]

---


## 消息机制篇


### 0. 消息驱动机制
Android 扩展了线程的退出机制，在启动线程时，在线程内部创建一个消息队列， 然后让线程进入无限循环；
在这个无限循环中，线程会不断的检查消息队列中是否是消息。如果需要执行某个任务，就向线程的消息队列中发送消息，循环检测到有消息到来 便会获取这个消息 进而执行它。如果没有则线程进入等待状态。

### 1. 浅见Java层

```
- Looper调用prepare(),在当前执行的线程中生成仅且一个Looper实例，这个实例包含一个MessageQueue对象
- 调用loop()方法,在当前线程进行无限循环，不断从MessageQueue中读取Handler发来的消息。
- 生成Handler实例同时获取到当前线程的Looper对象 
- sendMessage方法, 将自身赋值给msg.target, 并将消息放入MessageQueue中
- 回调创建这个消息的handler中的dispathMessage方法，
```
```
构建自己Looper线程，体验消息驱动。(可以直接使用Android 提供的HandlerThread)
Looper 线程的特点就是 run方法执行完成之后不会推出 而是进入一个loop消息循环。
class LooperThread extends Thread {
    public Handler mHandler;
    
    public void run() {
        Looper.prepare();
        mHandler = new Handler() {
            public void handleMessage(Message msg) {
            // 处理消息
            }
        };
        Looper.loop();
    }
}
```

```
穿插一下ThreadLocal(线程局部变量)
实质是线程的变量值的一个副本
而他存取的数据，就是的当前线程的数据

public class ThreadLocal<T> {
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null)
                return (T)e.value;
        }
        return setInitialValue();
    }
}
```

#### 1.1 消息(Message)

```
建议使用Message.obtain()方法生成Message对象; 因为Message内部维护了一个Message池用于Message的复用，避免使用new 重新分配内存
```

```
public final class Message implements Parcelable {


}
```



#### 1.2 消息循环(Looper)

```
- prepare()与当前线程绑定。
- loop()方法中，通过MessageQueue.next()获取消息(消息驱动)，交给Message.target.dispatchMessage分发处理(target指Handler, 在Handler.enqueueMessage中被赋值)。
```

```
public class Looper {
    // 定义一个线程局部对象存储Looper对象
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
    final MessageQueue mQueue;
    final Thread mThread;
    
    /**
     * 创建了MessageQueue对象
     * 建立了MessageQueue和NativeMessageQueue的关系
     * 初始化了Native层的Looper
     * 持有当前线程的引用
     * 
     **/
    private Looper() {
        // 创建了MessageQueue对象
        mQueue = new MessageQueue();
        // 将当前线程状态标记为run
        mRun = true;
        // 存储当前线程
        mThread = Thread.currentThread();
    }
    
    
    /**
     *  prepare()只能被调用一次, 否则直接抛出异常
     *  Looper对象存入ThreadLocal ,保证一个线程中只有一个Looper
     **/
    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
    
    /**
     * loop 必须在prepare()后执行，否则报错
     **/
    public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        .....
        for(;;) {
            Message msg = queue.next(); // might block
            ......
            msg.target.dispatchMessage(msg);
            ......
        }
    }
}
```

#### 1.3 消息队列(MessageQueue)

```
java层实例化对象时, 同时native层也完成初始化（NativeMessageQueue）
```

![image_1c9odn8dlocl1lnh18mkahqbdo9.png-85.5kB][MessageQueue于NativeMessageQueue]

```
public class MessageQueue {
    boolean mQuitAllowed = true; // 默认允许推出, 没看到置为false的地方
    private int mPtr; // used by native code

    private native void nativeInit();
    
    MessageQueue() {
        // 执行Native层初始化操作
        nativeInit();
    }
    
}


```


```
MessageQueue含有个Message队列 mMessages

boolean enqueueMessage(Message msg, long when) {
    ...
    Message p = mMessages;
    boolean needWake;
    // 先和消息队列的头进行比对
    if (p == null || when == 0 || when < p.when) {
        // 若新消息执行时间小于队列头消息执行时间, 将新消息next指向当前的队列头, 新消息放到队列头
        msg.next = p;
        mMessages = msg;
    } else { message in the queue.
        needWake = mBlocked && p.target == null && msg.isAsynchronous();
        Message prev;
        // 循环当前消息队列 挑选出 第一个比新消息执行时间晚的消息
        for (;;) {
            prev = p;
            p = p.next;
            if (p == null || when < p.when) {
                break;
            }
            if (needWake && p.isAsynchronous()) {
                needWake = false;
            }
        }
        // 交换位置
        msg.next = p;
        prev.next = msg;
    }

    // We can assume mPtr != 0 because mQuitting is false.
    if (needWake) {
        nativeWake(mPtr);
    }
}

```


```
next()方法

 Message next() {
     int nextPollTimeoutMillis = 0;
     for (;;) {
        // 
        nativePollOnce(ptr, nextPollTimeoutMillis);
     }
 }
```


#### 1.4 消息处理器(Handler)

Handle是Looper线程的消息处理器, 承担了发送消息和处理消息两部分工作。


- 构造函数

```
final MessageQueue mQueue;
final Looper mLooper;
final Callback mCallback;
IMessenger mMessenger; // 用于跨进程发送消息
public Handler() {
    ....
    // 将 Looper、MessageQueue和Handler关联到一起
    mLooper = Looper.myLooper();
    if (mLooper == null) {
    // 必须调用Looper.prepare()后才能使用
        throw new RuntimeException(
            "Can't create handler inside thread that has not called Looper.prepare()");
    }
    mQueue = mLooper.mQueue;
    mCallback = null;
}
```

- Handler.sendMessage()

```
将自身赋值给msg.target, 并将消息放入MessageQueue中
```

- Handler.post(new Runnable())

```
实际是发送了一条消息,此处的Runnable并没有创建线程，只是作为一个callback使用

public final boolean post(Runnable r){
   return  sendMessageDelayed(getPostMessage(r), 0);
}

private static Message getPostMessage(Runnable r) {
    Message m = Message.obtain();
    m.callback = r;
    return m;
}
```

- **Handler.dispatchMessage()消息分配**

```
以下源码可以看出, 当使用post()发送消息时, 最后会调用runnable.run()回调。sendMessage()则是执行handleMessage()， 这个就是我们构建对象时重写的方法

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
private static void handleCallback(Message message) {
    message.callback.run();
}
```

### 2. 深入native层


#### 2.1 Looper.cpp

与java 层的Looper完全不同
主要是负责监听管道

```
Looper::Looper(bool allowNonCallbacks) :
        mAllowNonCallbacks(allowNonCallbacks), mSendingMessage(false),
        mResponseIndex(0), mNextMessageUptime(LLONG_MAX) {
    int wakeFds[2];
    // 创建一个管道
    int result = pipe(wakeFds);
    // 管道读端
    mWakeReadPipeFd = wakeFds[0];
    // 管道写端
    mWakeWritePipeFd = wakeFds[1];
    // 给读管道设置非阻塞的flag, 保证在没有缓冲区可读的情况下立即返回而不阻塞。
    result = fcntl(mWakeReadPipeFd, F_SETFL, O_NONBLOCK);
    // 同理给写管道设置非阻塞
    result = fcntl(mWakeWritePipeFd, F_SETFL, O_NONBLOCK);
#ifdef LOOPER_USES_EPOLL
    // 分配epoll实例，注册唤醒管道并监听
    mEpollFd = epoll_create(EPOLL_SIZE_HINT);
    // 定义监听事件
    struct epoll_event eventItem;
    memset(& eventItem, 0, sizeof(epoll_event));
    // 监听EPOLLIN类型事件, 该事件表示对应的文件有数据可读
    eventItem.events = EPOLLIN;
    eventItem.data.fd = mWakeReadPipeFd;
    // 注册监听事件, 这里只监听mWakeReadPipeFd
    result = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, mWakeReadPipeFd, & eventItem);
#else
    // Add the wake pipe to the head of the request list with a null callback.
    struct pollfd requestedFd;
    requestedFd.fd = mWakeReadPipeFd;
    requestedFd.events = POLLIN;
    mRequestedFds.push(requestedFd);

    Request request;
    request.fd = mWakeReadPipeFd;
    request.callback = NULL;
    request.ident = 0;
    request.data = NULL;
    mRequests.push(request);

    mPolling = false;
    mWaiters = 0;
#endif

#ifdef LOOPER_STATISTICS
    mPendingWakeTime = -1;
    mPendingWakeCount = 0;
    mSampledWakeCycles = 0;
    mSampledWakeCountSum = 0;
    mSampledWakeLatencySum = 0;

    mSampledPolls = 0;
    mSampledZeroPollCount = 0;
    mSampledZeroPollLatencySum = 0;
    mSampledTimeoutPollCount = 0;
    mSampledTimeoutPollLatencySum = 0;
#endif
}
```

```
static pthread_once_t gTLSOnce = PTHREAD_ONCE_INIT;
sp<Looper> Looper::getForThread() {
    // initTLSKey 指代 initTLSKey()函数 内部调用pthread_key_create(...)创建线程局部对象
    // PTHREAD_ONCE_INIT  在Linex 线程模型中表示在本线程只执行一次
    int result = pthread_once(& gTLSOnce, initTLSKey);
    LOG_ALWAYS_FATAL_IF(result != 0, "pthread_once failed");
    // 返回线程局部对象中的Looper对象
    return (Looper*)pthread_getspecific(gTLSKey);
}

void Looper::setForThread(const sp<Looper>& looper) {
    sp<Looper> old = getForThread(); // also has side-effect of initializing TLS
    // 增加新引用的计数
    if (looper != NULL) {
        looper->incStrong((void*)threadDestructor);
    }
    // 将新的Looper对象设置到线程局部对象
    pthread_setspecific(gTLSKey, looper.get());
    // 减少老引用的计数
    if (old != NULL) {
        old->decStrong((void*)threadDestructor);
    }
}
```

#### 2.2 android_os_MessageQueue.cpp

这里其实也实现了和Java层Looper.prepare()方法相同的功能 

- 构造函数

```
NativeMessageQueue::NativeMessageQueue() {
    // 查询是否存在
    mLooper = Looper::getForThread();
    if (mLooper == NULL) {
        // 没有就创建Looper
        mLooper = new Looper(false);
        Looper::setForThread(mLooper);
    }
}
```

- nativeInit()

```
/**
 * 创建一个NativeMessageQueue对象
 * 将Java层与Native层的MessageQueue关联起来
 * 从而在Java层中可以通过mPtr成员变量来访问Native层的NativeMessageQueue对象
 * [obj]: 表示调用native 的Java类
 **/
static void android_os_MessageQueue_nativeInit(JNIEnv* env, jobject obj) {
    // JNI层创建一个NativeMessageQueue对象
    NativeMessageQueue* nativeMessageQueue = new NativeMessageQueue();
    if (! nativeMessageQueue) {
        jniThrowRuntimeException(env, "Unable to allocate native queue");
        return;
    }
    // 将Java层与Native层的MessageQueue关联起来
    android_os_MessageQueue_setNativeMessageQueue(env, obj, nativeMessageQueue);
}

static void android_os_MessageQueue_setNativeMessageQueue(JNIEnv* env, jobject messageQueueObj,
        NativeMessageQueue* nativeMessageQueue) {
        // SetIntField是JNI层向Java传输数据的方法。
        // 这里将nativeMessageQueue的地址保存到Java层MessageQueue对象的mPtr成员变量中
    env->SetIntField(messageQueueObj, gMessageQueueClassInfo.mPtr,
             reinterpret_cast<jint>(nativeMessageQueue));
}

static struct {
    // jfieldID表示Java层成员变量的名字,因此这里指代Java层MessageQueue对象的mPtr成员变量
    jfieldID mPtr;   // native object attached to the DVM MessageQueue
} gMessageQueueClassInfo;
```

## ~！@#￥%……&*（

### HandleThread

```
主要和Handle结合使用来处理异步任务

- 实质是一个Thread, 继承自Thread
- 他拥有自己的Looper对象
- 必须调用HandleThread.start()方法, 因为在run中创建Looper对象
```

### IdleHandler

* 使用场景:

```
- 主线程加载完页面之后，去加载一些二级界面;
- 管理一些任务, 空闲时触发执行队列。
```

* Handler线程空闲时执行:

```
- 线程的消息队列为空
- 消息队列头部的处理时间未到
```

* 使用

```
可以参考ActivityThread类中的空闲时执行gc流程

class IdleForever implements MessageQueue.IdleHandler {
    /**
     * @return true : 保持此Idle一直在Handler中, 每次线程执行完后都会在这执行.
     */
    @Override
    public boolean queueIdle() {
        Log.d("", "我,天地难葬!");
        return true;
    }

}

class IdleOnce implements MessageQueue.IdleHandler {
    /**
     * @return false : 执行一次后就从Handler线程中remove掉。
     */
    @Override
    public boolean queueIdle() {
        Log.d("", "我真的还想再活五百年~!!!");
        return false;
    }
}
```

## 参考资料

以上部分图片和解读说明摘自以下参考资料。

> **<< Android的设计和实现:卷I>>**
> **<<深入理解Android: 卷I>>**


[back](./)

------
作者 : [口戛口崩月危.Z][author]

[author]: https://zaze359.github.io
[MessageQueue于NativeMessageQueue]: http://static.zybuluo.com/zaze/kbfxaf2elx70xzzpc1ue4n8m/image_1c9odn8dlocl1lnh18mkahqbdo9.png
