## Android面试

Android基础原理

- 绘制原理

  - Choreographer 

  - 刷新机制

  - 绘制流程

  - 子线程更新UI

- 进程间通信

  - Linux的基本知识

    - 进程隔离：虚拟地址空间，进程间的数据不共享

    - 进程空间：用户空间/内核空间

    - 系统调用：用户态/内核态

    - Binder驱动：负责进程之间的Binder通信

  - 传统IPC机制的通信原理

    - 发送方进程通过系统调用（copy_from_user）将要发送的数据存拷贝到内核缓存区中。

    - 接收方开辟一段内存空间，内核通过系统调用（copy_to_user）将内核缓存区中的数据拷贝到接收方的内存缓存区。

  - Binder的优势

    - 性能角度：只需要一次拷贝
    - Binder借助了内存映射，在内核空间和接收方用户空间的数据缓存区之间做了一层内存映射。
    - 安全性：身份校验
    - Binder通信原理：
    
  - 类比网络通信：Client、Server、Binder驱动和ServiceManager![img](https://api2.mubu.com/v3/document_image/960384d8-3451-4df2-89b5-ca52d31e8190-2012723.jpg)
    
  - 基本通信过程：Client向SerivceManger获取到Server的Binder引用，Client通过Binder引用向Server发起具体请求。
    - ServiceManager

      - 和Zygote进程一样，是Android系统的守护进程之一

      - ServiceManager是比较特殊的服务，所有应用都能直接使用，因为ServiceManager对于Client端来说Handle句柄是固定的，都是0，所以ServiceManager服务并不需要查询，可以直接使用。

      - binder_context_mgr_node![img](https://api2.mubu.com/v3/document_image/AA3330F559BC4DDC1609631850.jpg)

      - 为什么有两棵binder-refs红黑树
    - refs_by_node红黑树，主要是给binder驱动往用户空间写数据使用的。相对的refs_by_desc主要是为了用户空间往binder驱动写数据使用的，当用户空间已经获得Binder驱动为其创建的binder_ref引用句柄后，就可以通过binder_get_ref从refs_by_desc找到响应binder_ref，进而找到目标binder_node。
    
  - Binder驱动
    
        - Binder驱动其实跟硬件没有什么关系，它就是一段运行在内核空间的代码，通过一个叫"/dev/binder"的文件在内核空间和用户空间来回搬数据。
  
- 它负责Binder节点的建立，负责Binder在进程间的传递，负责对Binder引用的计数管理，在不同进程间传输数据包等底层操作。
    - binder通信的内存大小限制。（1M和128k）

      - binder_open 中会通过mmap 内存映射 **128 k** 的内存空间，但是这仅仅在 ServiceManager 进程中，其它Binder 服务进程 BINDER_VM_SIZE ((1 * 1024 * 1024) - sysconf(SC_PAGE_SIZE) * 2) 的内存空间，SC_PAGE_SIZE 为 一个 page 页大小， 估摸着为**1M - 8K**的样子。
    - binder线程池默认最大数量（15）
    - AndroidBinder权限检查之clearCallingIdentify
    
  - clearCallingIdentity作用是清空远程调用端的uid和pid，用当前本地进程的uid和pid替代
    
  - 场景分析![img](https://api2.mubu.com/v3/document_image/b73f0275-103b-4e32-8836-8c4f804ae5ae-2012723.jpg)
  
- AIDL
  
  - aidl 架构![img](https://api2.mubu.com/v3/document_image/2dec716a-92ee-4fab-8a4d-7587de49d553-2012723.jpg)
  - Binder IPC 属于 C/S 结构，Client 部分是用户代码，用户代码最终会调用 Binder Driver 的 transact 接口，Binder Driver 会调用 Server，这里的 Server 与 service 不同，可以理解为 Service 中 onBind 返回的 Binder 对象
  - AIDL做了什么
  
    - AIDL 自动生成了 Stub 类
  - 在 Service 端继承 Stub 类，Stub 类中实现了 onTransact 方法实现了「解包」的功能
    - 在 Client 端使用 Stub 类的 Proxy 对象，该对象实现了「组包」并且调用 transact 的功能
  - oneway
    - 在AIDL中写代码时，如果接口标记了oneway，表示Server端**串行化处理(从异步队列中拿出消息指令一个个分发)、异步调用**。这个关键字主要是用于修改远程调用的行为，就如上面的两个图一样。非oneway关键字的AIDL类，**客户端需要挂起线程等待休眠，相当于调用了Sleep函数**。例如WMS 、AMS等相关系统Binder调用都是oneway的。
  
- 事件分发

  - 流程图![img](https://api2.mubu.com/v3/document_image/ea500efd-bb84-4d9f-8f12-3264fe8be759-2012723.jpg)

  - InputManagerService![img](https://api2.mubu.com/v3/document_image/6463e77f-b885-4990-92eb-b01c2fdca158-2012723.jpg)

  - ViewRootImpl.setView()函数非常重要，该函数也正是ViewRootImpl本身职责的体现：

    - 1.链接WindowManager和DecorView的纽带，更广一点可以说是Window和View之间的纽带；

    - 2.完成View的绘制过程，包括measure、layout、draw过程；

    - 3.向DecorView分发收到的用户发起的InputEvent事件。

  - 事件分发的最终目的是：获知事件是否被消费

    - 如果 child 是 ViewGroup，那么实际执行的就是 ViewGroup 重写的 dispatchTouchEvent 方法。该方法内可以判断，是否在当前层级拦截当前事件、或是递给下一级。

    - 如果 child 是不再有 child 的 View 或 ViewGroup，那么实际执行的就是 View 类实现的 super.dispatchTouchEvent 方法。该方法内可以判断，如果 View enabled 并且实现了 onTouchListener，且 onTouch 返回 true，那么不执行 onTouchEvent，并直接返回结果。否则执行 onTouchEvent。

- Handler

  - MessageQueue的创建：nativeInit 会创建一个native层的MessageQueue，并将引用地址返回给Java层的mPtr变量中

  - MessageQueue的数据结构：单向链表

  - 消息屏障：postSyncBarrier，插入一个无target的message，这个屏障之后的所有同步消息都不会执行。

  - IdleHandler

  - nativePollOnce(mPtr, nextPollTimeoutMillis)：通过Native层的MessageQueue阻塞nextPollTimeoutMillis毫秒的时间。调用native层的MessageQueue的Looper的epoll_wait，是Linux下多路复用IO接口的增强版本。总结一下就是，Java层的阻塞是通过native层的epoll监听文件描述符的写入事件来实现的。

  - MessageQueuer入队列时会唤醒等待的线程nativeWakeup(mPtr) 。write(mWakeEventFd, &inc, sizeof(uint64_t)),写入了一个1，这个时候epoll就能监听到事件，也就被唤醒了。

  - 回调执行顺序：1、message的callback 2、handler的callback、handleMessage()

  - Looper#loop方法为何不会导致ANR？nativePollOnce细节。eventfd和epoll机制了解么？

     eventfd 是 Linux 系统中一个用来通知事件的文件描述符，基于内核向用户空间应用发送通知的机制，可以有效地被用来实现用户空间事件驱动的应用程序。简而言之：eventfd 就是用来触发事件通知，它只有一个系统调用接口：表示打开一个 eventfd 文件并返回文件描述符，支持 epoll/poll/select 操作。

- 四大组件

  - Zygote进程启动时做了哪些事

    - 初始化DDMS
    - 注册Zygote进程的socket通信
    - 初始化Zygote的各种类、资源文件、opengl、类库、text资源等
    - 初始化完成之后fork出SystemServer进程
    - 循环等待客户端发来的 socket 请求（请求 socket 连接和请求 fork 应用进程）

  - Application启动流程

    - AMS是如何确认Application启动完成的？关键条件是什么（zygote返给AMS的pid；应用的ActivityThread#main方法中会向AMS上报Application的binder对象）？
    - Application#constructor、Application#onCreate、Application#attach他们的执行顺序（132）

  - Activity

    - 启动模式

    - ![](D:\Downloads\IMG_0016.png)

    - taskAffinity，allowTaskReparting的用法

    - Activity生命周期

    - onSaveInstanceState/onRestoreInstanceState

      - 这两个方法，主要用于在Activity被意外杀死的情况下进行界面数据存储与恢复。什么叫意外杀死呢？
        如果你主动点击返回键、调用finish方法、从多任务列表清除后台应用等等，这些操作表示用户想要完整得退出activity，那么就没有必要保留界面数据了，所以也不会调用这两个方法。而当应用被系统意外杀死，或者系统配置更改导致的activity销毁，这个时候当用户返回activity时，期望界面的数据还在，则会通过回调onSaveInstanceState方法来保存界面数据，而在activity重新创建并运行的时候调用onRestoreInstanceState方法来恢复数据。事实上，onRestoreInstanceState方法的参数和onCreate方法的参数是一致的，只是他们两个方法回调的时机不同。因此，判断是否执行的关键因素就是用户是否期望返回该activity时界面数据仍然存在。

        其onSaveInstanceState方法会在什么时候被执行，有这么几种情况： 
          1、当用户按下HOME键时。 这是显而易见的，系统不知道你按下HOME后要运行多少其他的程序，自然也不知道activity A是否会被销毁，故系统会调用onSaveInstanceState，让用户有机会保存某些非永久性的数据。以下几种情况的分析都遵循该原则
          2、长按HOME键，选择运行其他的程序时。
          3、按下电源按键（关闭屏幕显示）时。
          4、从activity A中启动一个新的activity时。
          5、屏幕方向切换时，例如从竖屏切换到横屏时。 

          在屏幕切换之前，系统会销毁activity A，在屏幕切换之后系统又会自动地创建activity A，所以onSaveInstanceState一定会被执行  总而言之，onSaveInstanceState的调用遵循一个重要原则，即当系统“未经你许可”时销毁了你的activity，则onSaveInstanceState会被系统调用，这是系统的责任，因为它必须要提供一个机会让你保存你的数据（当然你不保存那就随便你了）。至于onRestoreInstanceState方法，需要注意的是，onSaveInstanceState方法和onRestoreInstanceState方法“不一定”是成对的被调用的，onRestoreInstanceState被调用的前提是，activity A“确实”被系统销毁了，而如果仅仅是停留在有这种可能性的情况下，则该方法不会被调用，例如，当正在显示activity A的时候，用户按下HOME键回到主界面，然后用户紧接着又返回到activity A，这种情况下activity A一般不会因为内存的原因被系统销毁，故activity A的onRestoreInstanceState方法不会被执行。 另外，onRestoreInstanceState的bundle参数也会传递到onCreate方法中，你也可以选择在onCreate方法中做数据还原。
      
    - Activity#setContentView的具体过程

      - 在Activity中调用setContentView（实际调用PhoneWindow#setContentView）
      - 新建DecorView实例
      - 设置界面主题（requestFeature）
      - 确定主题界面（layoutResource = [R.layout.xxx](http://r.layout.xxx/)）
      - 在主题界面抽取内容ViewGroup（mContentParent = findViewById）
      - 将我们自己创建的布局界面和Android提供的内容mContentParent打包进Scene
      - 通过LayoutInflater解析布局，将布局转化为View
      - 将view添加到mContentParent中
      - 将整个界面装载到系统界面中

  - Broadcast/LocalBroadcast

  - ContentProvider

  - Service

- Context

  -  Context 的类图![img](https://api2.mubu.com/v3/document_image/eb3016f4-0815-4b9b-988f-82ecbf8327fa-2012723.jpg)
  
-  ContextImpl
  
  -  Context类型
  
- Context创建时机
  -  https://juejin.cn/post/6887514585629196295
    -  context是从哪里来的？AMS！AMS是系统级进程，拥有访问系统级操作的权利，应用程序的启动受AMS的调控，在程序启动的过程中，AMS会把一个“凭证”通过跨进程通信给到我们的应用程序，我们的程序会把这个“凭证”封装成context，并提供一系列的接口，这样我们的程序也就可以很方便地访问系统资源了。
  -  Activity ： ActivityThread.performLaunchActivity() -> createBaseContextForActivity - > attach() -> attachBaseContext
  -  Service：ActivityThread.handleCreateService()
  -  Application：LoadedApk.makeApplication()
  -  Activity、Service、Application 的 Context 初始化流程大致是这样：首先创建 ContextImpl 实例，接着调用 ContextWrapper.attachBaseContext() 赋值给 mBase，ContextImpl 用于实现 Context 的核心功能，而 Context 的继承类则增加差异性功能。
  
- SharedPreference

  - 基本结构
    - 本质是一个以 键值对（key-value）的方式保存数据的xml文件，其保存在/data/data/shared_prefs目录下。

  - 读操作的优化
    - 第一次通过Context.getSharedPreferences()进行初始化时，对xml文件进行一次读取，并将文件内所有内容（即所有的键值对）缓到内存的一个Map中

  - 写操作的优化
    - Editor+apply()

  - 保证线程安全：
    - 三把锁：mLock、mWrtingToDiskLock、mEditorLock

  - apply()引发的anr

    - 在apply()方法中，首先会创建一个等待锁，根据源码版本的不同，最终更新文件的任务会交给QueuedWork.singleThreadExecutor()单个线程或者HandlerThread去执行，当文件更新完毕后会释放锁。

    - 但当Activity.onStop()以及Service处理onStop等相关方法时，则会执行 QueuedWork.waitToFinish()等待所有的等待锁释放，因此如果SharedPreferences一直没有完成更新任务，有可能会导致卡在主线程，最终超时导致ANR。

  - 文件损坏&备份机制
    - SharedPreferences的写入操作正式执行之前，首先会对文件进行备份，将初始文件重命名为增加了一个.bak后缀的备份文件：这之后，尝试对文件进行写入操作，写入成功时，则将备份文件删除；反之，若因异常情况（比如进程被杀）导致写入失败，进程再次启动后，若发现存在备份文件，则将备份文件重名为源文件，原本未完成写入的文件就直接丢弃。

- 虚拟机

  - Java虚拟机和Dalvik虚拟机的不同

    - Java虚拟机运行的是Java字节码，Dalvik虚拟机运行的是dalvik字节码

    - Dalvik可执行文件体积更小。dx工具负责将JAVA字节码转换为Dalvik字节码

    - JVM基于栈，DVM基于寄存器

  - ART和Dalvik的不同

    - JIT（Just In Time，即时编译技术）和AOT(Ahead Of Time，预编译技术)两种编译模式

    - Dalvik虚拟机执行的是dex字节码，ART虚拟机执行的是本地机器码

    - 总的来说ART就是“空间换时间”

- Bitmap

  - https://mp.weixin.qq.com/s/RTRkNXOzrtb0T7FuG-vIwA

  - inBitmap

    - 主要就是指的复用内存块，不需要在重新给这个bitmap申请一块新的内存,避免了一次内存的分配和回收，从而改善了运行效率。 

  - 跨进程传递大图（Bundle#putBinder）

    常规的intent传递数据，在startActivity时将Bundle的allowFds 设置成了false, 然后就会将 Bitmap直接写到 

    Parcel 缓冲区。如果通过 bundle.putBinder形式传递Bitmap，会开辟一个块共享匿名内存用来存Bitmap的数据，

    而Parcel 缓冲区只是存储 FD 。

- 动画

- Apk打包流程。R文件最终会生成什么文件？aapt的作用是什么？

- LocalBroadcastReceiver为何比BroadCastReceiver速度快，LocalBroadcastReceiver的实现。

- 讲下leakCanary原理。为什么不用虚引用？引用队列里面存的是什么？内存数据是如何dump出来的？

- Glide

- ArrayMap和SparseArray的作用和实现细节。

- MVP、MVVM

- 插件化原理

Android快稳省

- 卡顿问题

- ANR和Crash

- 耗电问题

- 内存优化

- 内存泄漏

- LRUCache

  - 传参cacheSize，重写sizeOf

  - 关键方法trimToSize：trimToSize()方法不断地删除LinkedHashMap中队首的元素，即近期最少访问的，直到缓存大小小于最大值。

  - LruCache中维护了一个集合LinkedHashMap，该LinkedHashMap是以访问顺序排序的。当调用put()方法时，就会在集合中添加元素，并调用trimToSize()判断缓存是否已满，如果满了就用LinkedHashMap的迭代器删除队首元素，即近期最少访问的元素。当调用get()方法访问缓存对象时，就会调用LinkedHashMap的get()方法获得对应集合元素，同时会更新该元素到队尾。
  
  - LRU的实现。让你自己实现一个，你会怎么做。
  
    ​	![](D:\Downloads\IMG_0015.jpg)

Java基础

- 泛型。

  ①泛型擦除的原因和效果，擦除的时机。

  ②为何会有协变和逆变

  ③通配符。

  ④PECS

- 反射

- 注解

  - Source和Class、Runtime注解的区别
  - 注解如何使用

- 方法内部的匿名内部类使用方法的局部变量时，为何要使用final修饰？

  局部变量是作为构造方法的参数传入匿名内部类的（以上代码经过了手动修改，真实的反编译结果中有一些不可读的命名）。

  如果Java允许匿名内部类访问非final的局部变量的话，那我们就可以在TryUsingAnonymousClass$1中修改paramInteger，但是这不会对number的值有影响，因为它们是不同的reference。

  这就会造成数据不同步的问题。

  所以，谜团解开了：Java为了避免数据不同步的问题，做出了匿名内部类只可以访问final的局部变量的限制。

- 容器

  - ArrayList  Vector 

  - CopyOnWriteList

  - LinkedList

  - HashSet

  - TreeSet

  - HashMap

    - HashMap添加元素的过程，hash方法细节；
    - 扩容的触发条件、扩容过程中是数据是整体复制么？
    - 链表转红黑树的阈值为何是8，红黑树转链表的阈值为何是6，为何不上同一个阈值？
    - 链表为何要转红黑树？红黑树有何特性？hashmap为何如此设计？
    -  如何使现有的HashMap线程安全？（Collections#synchronizedMap）

  - ConcurrentHashMap 

  - LinkedHashMap

  - LRUCache

  - 讲下equals和hashcode，他们为何必须一起重写？hashcode方法重写规则。

    1. 理想的哈希函数应当具有均匀性，即不相等的对象应当均匀分布到所有可能的哈希值上。这就要求了哈希函数要把所有域的值都考虑进来。可以将每个域都当成 R 进制的某一位，然后组成一个 R 进制的整数。

       R 一般取 31，因为它是一个奇素数，如果是偶数的话，当出现乘法溢出，信息就会丢失，因为与 2 相乘相当于向左移一位，最左边的位丢失。并且一个数与 31 相乘可以转换成移位和减法：31*x == (x<<5)-x，编译器会自动进行这个优化。

       ``@Override
       public int hashCode() {
       int result = 17;
       result = 31 * result + x;
       result = 31 * result + y;
       result = 31 * result + z;
       return result;
       }``

- 多线程

  - 线程状态

    - new、runnable、running、dead、waiting（主动等待）、timed waiting（主动睡眠）、blocked（被同步块或者IO阻塞）

    - notify（）该实例对象移除对象等待池，进入锁标志等待池

  - 线程池

    - 构造方法（核心线程数corePoolSize、总线程数maxPoolSize、存活时间keepAliveTime、时间单位timeUnit、阻塞队列workingqueue、ThreadFactory、RejectPolicy）
    - 线程池中的任务可以实现按照优先级执行么，如何实现？ -- PriorityBlockingQueue
    - 线程池的设计用到了那种设计思想？（生产者消费者模型）
    - 何为阻塞队列？

  - 线程安全问题
    
  - 本质时多线程的同步问题
    
    - T1、T2、T3三个线程，如何保证它们顺序执行？也就是异步转同步的方式。
    
      ①Thread#join
    
      ②等待多线程完成的CountDownLatch
    
      ③FutureTask
    
      ④Executors#newSingleThreadExecutor
    
      ⑤wait/notify
    
    - Java中 wait和sleep方法的不同？（wait释放锁，sleep不会释放锁）
    
  - 内存模型

    - ![img](https://api2.mubu.com/v3/document_image/07d17342-9394-4c6c-bd47-2d38a185b946-2012723.jpg)

    - Java内存模型描述了在多线程代码中，那些行为是正确合法的，以及多线程之间如何进行通信，代码中变量的读写行为如何反应到内存、CPU缓存的底层细节。

    - Java包含了几个关键字：volatile、synchronized、final

  - syncrhonized

    - 互斥同步：对于一个monitor对象，只能被一个线程持有，意味着一旦有线程进入了同步代码块，那么其他线程就不能进入，直到第一个进入的线程退出代码块。

    - 保证了线程在同步代码块期间的写入动作对其他线程是可见的。

      - 线程释放monitor对象，作用是把CPU缓存的数据刷新到主内存中。

      - 线程获取monitor对象，作用时候使cpu缓存失效，从而使变量从主内存中重新加载。然后就可以看到之前线程对该变量的修改。

      - 禁用指令重排序

  - final 
    
  - 如果一个类包含final字段，且在构造函数中初始化，那么正确的构造一个对象后，final字段被设置后对于其它线程是可见的。
    
  - volatile

    - 内存语义和synchronized获取和释放monitor的实现目的差不多（禁止指令重排序、保证修改的值会立即更新到主内存）
  - 保证可见性、有序性， 对单个读写具有原子性，复合操作除外如i++
    
  - JVM底层采用内存屏障来定义volatile语义
    
  - CAS

  - Java的锁类型
    
  - Java主流锁![img](https://api2.mubu.com/v3/document_image/fe575672-9982-4feb-a843-e7e17deded4c-2012723.jpg)
    
  - ThreadLocal
    
  - 原理![img](https://api2.mubu.com/v3/document_image/410f4830-a41f-49e6-89d3-c729802dc5a1-2012723.jpg)
    
  - AsyncTask

    - 使用

      - AsyncTask是一个抽象基类 AsyncTask<Params, Progress, Result>

      - 实现onPreExecute 、doInBackground、onPostProgress、onPostExecute

- JVM

  - 内存布局
    - 运行时数据区域分区，哪些线程私有，哪些线程共享。
    - 栈帧的数据结构。方法区存放哪些数据。
  - 类加载
    - 讲下Java的双亲委派。
    - 实例化对象的步骤
      - 1、分配内存空间 2、初始化对象、将内存空间的地址赋值给对应的引用
    - 简单描述一下 Person person = new Person() 对象实例化过程。最好有类加载过程。
      - 确认类无信息是否存在。当JVM接收到NEW指令时,首先在m e t a s p a c e内检查需要创建的类元信息是否存在。若不存在,那么在双亲委派模式下,使用当前类加载器以Cl a s s L o a d e r +包名+类名为K e y进行查找对应的. c l a s s 文件。如果没有找到文件,则抛出Cl a s s N o t F o u n d E x c e p t i o n 异常,如果找到,则进行类加载,并生成对应的Cl a s s 类对象。
      - 分配对象内存。首先计算对象占用空间大小,如果实例成员变量是引用变量,仅分配引用变量空间即可,即4 个字节大小,接着在堆中划分一块内存给新对象。在分配内存空间时,需要进行同步操作,比如果用C AS( Compare And Swap )失败重试、区域加锁等方式保证分配操作的原子性。
      - 设定默认值。成员变量值都需要设定为默认值,即各种不同形式的零值。设置对象头。设置新对象的哈希码、G C信息、锁信息、对象所属的类元信息等。这个过程的具体设置方式取决于JVM实现。
      - 执行i ni t 方法。初始化成员变量,执行实例化代码块,调用类的构造方法,并把堆内对象的首地址赋值给引用变量。
  - 内存回收
    - GCRoot的类型，举例说明
    - Activity中的匿名Handler导致的内存泄漏，最终的引用链
      - ThreadLocal——>looper——>messagequue——>message——>handler——>Acitivity

- 设计模式

- 算法

  - 排序

    - 冒泡排序

    - 选择排序

    - 插入排序

    - 快速排序

  - 查找
    - 二分查找

  - 二叉树

    - 前中后序遍历

    - 层序遍历

    - 重建二叉树

  - 链表

    - 链表反转

    - 查找链表倒数第n个数

  - 字符串 数组

  - 栈 队列

  - 单例

  - 写生产者消费者

业务流程

- SystemUI启动流程

- SystemUI架构

- Dagger2

- 生物解锁

- 锁屏壁纸

- 息屏时钟