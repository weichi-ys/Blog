我们知道Spark平台是用Scala进行开发的，但是使用Spark的时候最流行的语言却不是Java和Scala，而是Python。原因当然是因为Python写代码效率更高，但是Scala是跑在JVM之上的，JVM和Python之间又是如何进行交互的呢？

![[Picture/70805a01278fbfdcc9b54b29a1b09b0d_MD5.png]]

在实际运行过程中，JVM并不会直接和Python进行交互，JVM只负责启停Python脚本，而不会向Python发送任何特殊指令。启动脚本同执行外部任意进程的方法是一样的，就是调用Runtime.exec(command)生成python子进程。停止Python进行就是调用Process.destroy()和Process.destroyForcibly()杀死子进程，destroy方法使用SIGTERM信号通知Python进程主动退出，如果Python一段时间不响应，就会使用destroyForcibly方法发送SIGKIL信号强制杀死Python进程。

![[Picture/a89e26902f500e549ec216198dabe61b_MD5.png]]

Pyspark玄妙的地方在于Python在运行的过程中需要调用Spark的API，这些API的实现在JVM虚拟机里面，也就是说python脚本运行的进程同Spark的API实现不在一个进程里，当我们在Python里面调用SparkAPI的时候，实际的动作执行确是在JVM里面，这是如何做到的？

答案就是远程过程调用，也就是我们经常听到的词汇RPC。

在Pyspark中，Python作为RPC的客户端，JVM作为RPC的服务端。JVM会开启一个Socket端口提供RPC服务，Python需要调用Spark API时，它会作为客户端将调用指令序列化成字节流发送到Socket服务端口，JVM接受字节流后解包成对应的指令，然后找到目标对象和代码进行执行，然后将执行结果序列化成字节流通过Socket返回给客户端，客户端收到字节流后再解包成Python对象，于是Python客户端就成功拿到了远程调用的结果。

客户端的这些序列化过程不是很复杂，当然也不会太简单，不管怎样，作为pyspark的使用者来说并不需要关心内部实现的细节，这一切pyspark库已经帮我们封装好了。对于JVM提供的所有RPC API，pyspark都已经包装成了一个python方法，对于使用者来说，他只需要调用相应的Python方法，就好像不存在远程过程调用一样，假装所有的这些过程都发生在python进程内部而没有任何感知。

![[Picture/5d0ed465dff8ced90391b34e570e4cf2_MD5.png]]

仅仅在程序出现异常而在日志里面打印了复杂的堆栈信息的时候，我们才可以从中发现端倪。

pyspark的异常信息里面一般包含两部分堆栈信息，前一部分是Python堆栈，后一部分是JVM堆栈信息，原因是当JVM端执行代码出现异常的时候，会将错误信息包括堆栈信息通过RPC返回给客户端，Python客户端在输出错误日志时除了输出自己的堆栈信息之外还会将JVM返回回来的堆栈错误信息一同展现出来，方便开发者定位错误的发生原因。

Spark的开发者们并没有自己撸一个RPC库，他们使用了开源的Py4j库。Py4j是一个非常有趣的RPC库，我们接下来详细介绍这个库的使用和原理。

Py4j在JVM进程开辟一个ServerSocket监听客户端的链接，来一个链接开辟一个新线程处理这个链接上的消息，对于共享对象的状态，在JVM端实现API时需要考虑多线程并发问题。

Py4j在Python客户端会启动一个连接池连接到JVM，所有的远程调用都被封装成了消息指令，随机地从连接中挑选一个连接将消息指令序列化发送到JVM远程执行。

```
<dependency>
<groupId>net.sf.py4j</groupId>
<artifactId>py4j</artifactId>
<version>0.10.6</version>
</dependency>
```

我们看一个简单的示例，首先导入相关依赖

```
import py4j.GatewayServer;

public class Py4jTest {

public int fib(int n) {
if (n < 2) {
return 1;
}
return fib(n - 1) + fib(n - 2);
}

public static void main(String[] args) {
Py4jTest app = new Py4jTest();
GatewayServer server = new GatewayServer(app, 8000);
server.start();
}

}
```

上面是JVM Server端，GatewayServer需要提供一个entry\_point入口点，入口点是服务对外暴露的直接引用，客户端通过这个引用来访问RPC服务。GatewayServer默认端口25333，可以通过参数进行修改。

```
# pip install py4j
from py4j.java_gateway import GatewayClient, JavaGateway

client = GatewayClient(address="127.0.0.1", port=8000)
gw = JavaGateway(client)
app = gw.entry_point
print app.fib(10)
```

这是Python客户端代码。我们首先指定远端地址构造一个GatewayClient，再拿到入口点引用，然后就可以直接调用RPC服务了。如果有多个JVM Server，我们就可以指定不同的地址构造多个GatewayClient分别进行调用，GatewayClient已经封装了连接池的逻辑。

除了使用entry\_point属性暴露入口对象引用外，Gateway提供了默认的jvm对象引用，有了这个引用，你就可以远程导入任意的Java类，创建任意Java对象，自由地使用python语法操作Java的数据。

```
>>> from py4j.java_gateway import JavaGateway
>>> gateway = JavaGateway() # 使用默认地址                   
>>> random = gateway.jvm.java.util.Random()   
>>> print random.nextInt(10)              
```

Py4j对JVM中常用的集合对象List、Set、Map做了快捷处理，使得Python常用的集合操作方法可以直接应用于JVM远程对象。

```
>>> m = gateway.jvm.java.util.HashMap()
>>> m["a"] = 0
>>> m.put("b",1)
>>> m
{u'a': 0, u'b': 1}
>>> u"b" in m
True
>>> del(m["a"])
>>> m
{u'b': 1}
>>> m["c"] = 2
>>> for key in m:
...     print("%s:%i" % (key,m[key]))
...
b:1
c:2
```

客户端表面上是在对本地一个字典对象进行操作，但是每一个操作背后都涉及到网络IO。Python使用操作符重载实现了这个转换。

Gateway Server创建的任意对象都会携带由服务端生成的唯一的对象id，服务端会将生成的所有对象装在一个Map结构里。当Python客户端需要操纵远程对象时，会将对象id和操纵指令以及参数一起传递到服务端，服务端根据对象id找到对应的对象，然后使用反射方法执行指令。

![[Picture/52b2c028b2990f1d2533437c1bcfc5c1_MD5.png]]

Py4j除了可以让Python自由操纵Java外，还可以通过Java直接操纵Python代码，实现了Python和JVM之间的双向交互。只不过逆向操作没有正向的自由，因为Java语言不是一个动态的语言，任何方法的调用都必须预先定义。所以对于Python服务的入口类，需要映射到Java端定义的一个相对应的接口类，Java通过接口函数来调用Python代码。

![[Picture/f3dc26faf0bd5e053909b6c4cdc2da0f_MD5.png]]

Py4j考虑了垃圾回收问题。通过Py4j客户端在JVM内部生成的对象都会集中统一放到一个map中，通过这个map来保持住对象的引用。python客户端这边会使用weakref跟踪对象的引用状态，当weakref挂接的对象被回收了说明对象变成了垃圾，Py4j就会向JVM发送一个携带对象的id的回收对象的指令，这样JVM就可以从map中移除掉这个对象，使得对象的引用技术变成零，于是就可以被JVM GC回收掉。

同样对于逆向调用，JVM会通过finalize方法来跟踪对象是否变成了垃圾。当finalize被执行时，说明指向Python对象的引用已经消失了，就会向Python VM发送一个回收对象的指令。于是Python VM也可以避免了内存泄露问题。

当你开发一个工具软件时，将需要性能和高并发的逻辑放进JVM中，而那些配置型的不需要高性能的部分逻辑使用Python来实现，再将两者使用Py4j连接到一起就可以做到一个既可以满足性能又可以满足易用性的软件来。

这同使用Golang内嵌Lua脚本语言来开发工具一样，虽然机制上差距极大，却可以达成相似的目标，即同时满足软件的性能和易用性。

阅读相关文章，请关注知乎专栏【码洞】