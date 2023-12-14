![](https://csdnimg.cn/release/blogv2/dist/pc/img/original.png)

[YiRan\_Zhao](https://blog.csdn.net/YiRan_Zhao "YiRan_Zhao") ![](https://csdnimg.cn/release/blogv2/dist/pc/img/newCurrentTime2.png) 于 2018-07-04 15:41:29 发布

### 一、[YARN](https://so.csdn.net/so/search?q=YARN&spm=1001.2101.3001.7020)是集群的资源管理系统

1、ResourceManager：负责整个集群的资源管理和分配。

2、ApplicationMaster：YARN中每个Application对应一个AM进程，负责与RM协商获取资源，获取资源后告诉NodeManager为其分配并启动[Container](https://so.csdn.net/so/search?q=Container&spm=1001.2101.3001.7020)。

3、NodeManager：每个节点的资源和任务管理器，负责启动/停止Container，并监视资源使用情况。

4、Container：YARN中的抽象资源。

### 二、[SPARK](https://so.csdn.net/so/search?q=SPARK&spm=1001.2101.3001.7020)的概念

1、Driver：和ClusterManager通信，进行资源申请、任务分配并监督其运行状况等。

2、ClusterManager：这里指YARN。

3、DAGScheduler：把spark作业转换成Stage的DAG图。

4、TaskScheduler：把Task分配给具体的Executor。

### 三、SPARK on YARN

#### 3.1、yarn-cluster模式下

**![](https://img-blog.csdn.net/20170409204821138?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMzU3MzgxMw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)**

(1)ResourceManager接到请求后在集群中选择一个NodeManager分配Container，并在Container中启动ApplicationMaster进程；

(2)在ApplicationMaster进程中初始化sparkContext；

(3)ApplicationMaster向ResourceManager申请到Container后，通知NodeManager在获得的Container中启动excutor进程；

(4)sparkContext分配Task给excutor，excutor发送运行状态给ApplicationMaster。

#### 3.2、yarn-clinet模式下

**![](https://img-blog.csdn.net/20170409204916592?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMzU3MzgxMw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)**

(1)ResourceManager接到请求后在集群中选择一个NodeManager分配Container，并在Container中启动ApplicationMaster进程；

(2)driver进程运行在client中，并初始化sparkContext；

(3)sparkContext初始化完后与ApplicationMaster通讯，通过ApplicationMaster向ResourceManager申请Container，ApplicationMaster通知NodeManager在获得的Container中启动excutor进程；

(4)sparkContext分配Task给excutor，excutor发送运行状态给driver。

#### 3.3、yarn-cluster与yarn-client的区别：

它们的区别就是ApplicationMaster的区别，yarn-cluster中ApplicationMaster不仅负责申请资源，并负责监控Task的运行状况，因此可以关掉client；

而yarn-client中ApplicationMaster仅负责申请资源，由client中的driver来监控调度Task的运行，因此不能关掉client。