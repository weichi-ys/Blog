Spark 事件总线贯彻整个应用，TaskScheduler 、Executor、JobScheduler、SQLExecution等关键交互逻辑离不开event的传递，为了更好的发挥Spark性能，以及扩展Spark功能，掌握event传递机制显得尤其重要。总的来看，其流程相对清晰。基础过程如下图：

![[Picture/8f989d8c8c86f492223a48f13a11c2e6_MD5.png]]

在LiveListenerBus定义一个queue成员变量队列，该队列只会保存4个子队列，分别是SHARED 、EXECUTOR\_MANAGER、APP\_STATUS、EVENT\_LOG队列。每一个Async Event Queue包含EventQueue和Listeners。 4类子队列的作用分别如下：

SharedQueue：主要是用户代码中增加的监听器Listener，或通过配置添加的自定义监听器Listener。

ManagementQueue：可对关键服务进行监听，比如监听Executor的添加和移除事件。

StatusQueue：主要提供流任务和批任务的Job、Stage、Task的状态数据等。

EventLogQueue：将监听到的event通过监听器写入到某个指定存储里。

下面我们跟踪代码到addToQueue函数先分析监听器队列添加流程：  

![[Picture/907f7189330143f1a6fd0ddfea7b3a7c_MD5.png]]

![[Picture/45295064aa33c2bab784e893343c5fda_MD5.png]]

该函数会根据队列名字进行判断，如果是新队列，则创建一个Async Event Queue对象（下面简称AEQ），并把监听器添加进去；如果是已存在的，则直接添加监听器listener。其中addListener将新添加的listener加入到AEQ的listenerPlustimers内部list中。监听器添加之后，再分析下event事件投入过程，如下图：

![[Picture/da0e48b1c8cdb3621e49977f86541a8c_MD5.png]]

spark中所有事件的投递都是通过LiveListenersBus的post方法进行的，首先会做 一个判断，如果积压事件队列queuedEvents为空，则证明LiveListenersBus 线程已经启动，可以直接调用postToQueues方法发送给已注册的所有queue。如果不为空，则先判断LiveListenersBus线程是否启动成功，如果未启动成功，则先把消息放到积压队列queuedEvents中；如果启动成功则直接投递event。

LiveListenersBus 线程启动过程如下：

![[Picture/2651eedf6e4afe6f9ee282f23034a032_MD5.png]]

![[Picture/a458c29a65a303adbf0621261963baa7_MD5.png]]

LiveListenersBus 线程启动过程较简单，首先检查queue\[Async Event Queue\]是否为空并启动Async Event Queue线程（下面简称AEQ），然后把积压队列里的消息全部投递到每一个AEQ中，其投递流程是直接把消息放到自己的eventQueue中，等待event被AEQ线程处理。

AEQ线程调度过程如下图：

![[Picture/5ee7eabd5c72bee480c818ac5b1f05a3_MD5.png]]

![[Picture/3d5c2bfd3339512d352ad5da33d91157_MD5.png]]

线程dispatchThread是一个常驻守护线程，其会一直轮询AEQ的eventqueue,每获取一个event，则通过postToAll函数分发到该类AEQ的所有监听器listeners。每个Listener通过调用自己实现的都Post Event函数进行逻辑处理。Spark中ListernerBus接口的实现类如下，有StreamingListener、SQL的CataLogListener和批处理的SparkListener。其中SparkListenerd内的都Post Event函数定义如下，每个自定义的listener按需实现自己的event接口即可。  

![[Picture/d4058f4a028438c0cf7e71dc0e9b19b2_MD5.png]]

![[Picture/1193f036a27cd27f1b2183b9537f830c_MD5.png]]