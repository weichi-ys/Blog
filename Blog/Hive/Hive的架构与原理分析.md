##  1. Hive的架构

　　![[Blog/Picture/efb98610192c36e3565458d412aadd26_MD5.png]]

 　　Hive的体系结构可以分为以下几部分：

1.  用户接口主要有三个：CLI，JDBC/ODBC和 Web UI。
    -   ①其中，最常用的是CLI，即Shell命令行；
    -   ②JDBC/ODBC Client是Hive的Java客户端，与使用传统数据库JDBC的方式类似，用户需要连接至Hive Server；
    -   ③Web UI是通过浏览器访问。
2.  Hive将元数据存储在数据库中，如mysql、derby。Hive中的元数据包括表的名字，表的列和分区及其属性，表的属性（是否为外部表等），表的数据所在目录等。
3.  解释器、编译器、优化器完成HQL查询语句从词法分析、语法分析、编译、优化以及查询计划的生成。生成的查询计划存储在HDFS中，并在随后有MapReduce调用 执行。
4.  Hive的数据存储在HDFS中，大部分的查询、计算由MapReduce完成（包含\*的查询，比如select \* from tbl不会生成MapReduce任务）

## 2. Hive的元数据存储

　　对于数据存储，Hive没有专门的数据存储格式，也没有为数据建立索引，用户可以非常自由的组织Hive中的表，只需要在创建表的时候告诉Hive数据中的列分隔符和行分隔符，Hive就可以解析数据。Hive中所有的数据都存储在HDFS中，存储结构主要包括数据库、文件、表和视图。Hive中包含以下数据模型：Table内部表，External Table外部表，Partition分区，Bucket桶。Hive默认可以直接加载文本文件，还支持sequence file、RCFile。

　　Hive将元数据存储在RDBMS中，有三种模式可以连接到数据库：

## 2.1 元数据内嵌模式（Embedded Metastore Database）

　　此模式连接到一个本地内嵌In-memory的数据库Derby，一般用于Unit Test，内嵌的derby数据库每次只能访问一个数据文件，也就意味着它不支持多会话连接。

　　　![[Blog/Picture/d97f9a0c8b45d1d03adbfa212da5316f_MD5.png]]

<table><tbody><tr><td>参数</td><td>描述</td><td>用例</td></tr><tr><td>javax.jdo.option.ConnectionURL&nbsp;</td><td>&nbsp;JDBC连接url</td><td>&nbsp;jdbc:derby:databaseName=metastore_db;create=true</td></tr><tr><td>&nbsp;javax.jdo.option.ConnectionDriverName</td><td>&nbsp;JDBC&nbsp;driver名称</td><td>&nbsp;org.apache.derby.jdbc.EmbeddedDriver</td></tr><tr><td>&nbsp;javax.jdo.option.ConnectionUserName</td><td>&nbsp;用户名</td><td>&nbsp;xxx</td></tr><tr><td>&nbsp;javax.jdo.option.ConnectionPassword</td><td>&nbsp;密码</td><td>&nbsp;xxxx</td></tr></tbody></table>

## 2.2 本地元数据存储模式（Local Metastore Server）

 　　通过网络连接到一个数据库中，是最经常使用到的模式。

![[Blog/Picture/08f7d58072247e24d633f42b8785a165_MD5.png]]

<table><tbody><tr><td>参数</td><td>描述</td><td>用例</td></tr><tr><td>javax.jdo.option.ConnectionURL</td><td>JDBC连接url</td><td>jdbc:mysql://&lt;host name&gt;/databaseName?createDatabaseIfNotExist=true</td></tr><tr><td>javax.jdo.option.ConnectionDriverName</td><td>JDBC&nbsp;driver名称</td><td>com.mysql.jdbc.Driver</td></tr><tr><td>javax.jdo.option.ConnectionUserName</td><td>用户名</td><td>xxx</td></tr><tr><td>javax.jdo.option.ConnectionPassword</td><td>密码</td><td>xxxx</td></tr></tbody></table>

## 2.3 远程访问元数据模式（Remote Metastore Server）

　　用于非Java客户端访问元数据库，在服务端启动MetaServer，客户端利用Thrift协议通过MetaStoreServer访问元数据库。

     ![[Blog/Picture/6afff8cef50cec1350c0ff5b16933e65_MD5.png]]

-    服务端启动HiveMetaStore

　　第一种方式：

```
hive --service metastore -p <span>9083</span> &amp;
```

　　第二种方式：

　　如果在hive-site.xml里指定了hive.metastore.uris的port，就可以不指定端口启动了

```
  &lt;property&gt;
     &lt;name&gt;hive.metastore.local&lt;/name&gt;
     &lt;value&gt;<span>false</span>&lt;/value&gt;
   &lt;/property&gt;

   &lt;property&gt;
          &lt;name&gt;hive.metastore.uris&lt;/name&gt;
          &lt;value&gt;thrift:<span>//</span><span>node1:9083&lt;/value&gt;</span>
   &lt;/property&gt;
   &lt;property&gt;
      &lt;name&gt;javax.jdo.option.ConnectionURL&lt;/name&gt;
      &lt;value&gt;jdbc:mysql:<span>//</span><span>node1/hive_remote?createDatabaseIfNotExist=true&lt;/value&gt;</span>
   &lt;/property&gt;

   &lt;property&gt;
      &lt;name&gt;javax.jdo.option.ConnectionDriverName&lt;/name&gt;
      &lt;value&gt;com.mysql.jdbc.Driver&lt;/value&gt;
   &lt;/property&gt;

   &lt;property&gt;
      &lt;name&gt;javax.jdo.option.ConnectionUserName&lt;/name&gt;
      &lt;value&gt;root&lt;/value&gt;
   &lt;/property&gt;

   &lt;property&gt;
      &lt;name&gt;javax.jdo.option.ConnectionPassword&lt;/name&gt;
      &lt;value&gt;123456&lt;/value&gt;
   &lt;/property&gt;
```

-   客户端配置

<table><tbody><tr><td>参数</td><td>描述</td><td>用例</td></tr><tr><td>hive.metastore.uris</td><td>metastore server的url</td><td>thrift://&lt;host_name&gt;:9083</td></tr><tr><td>hive.metastore.local</td><td>metastore server的位置</td><td>false表示远程</td></tr></tbody></table>

## 2.4 三种模式汇总

　　 　![[Blog/Picture/414c3ec4acc99eacea4c8c4072723f43_MD5.png]]

## 3. Hive工作原理

## 3.1 Hive内部组件分布构成

　　　　　　　　　　　　　　　　　**No. 1** **Hive全局架构图**

　　　　 ![[Blog/Picture/cd754dc1e3f3e653449cdff59be69a03_MD5.png]]

　从图1 **Hive全局架构图**中可以看到Hive架构包括如下组件：CLI（command line interface）、JDBC/ODBC、Thrift Server、Hive WEB Interface（HWI）、metastore和Driver（Compiler、Optimizer）

　　**Metastore组件：**元数据服务组件，这个组件用于存储hive的元数据，包括表名、表所属的数据库、表的拥有者、列/分区字段、表的类型、表的数据所在目录等内容。hive的元数据存储在关系数据库里，支持derby、mysql两种关系型数据库。元数据对于hive十分重要，因此hive支持把metastore服务独立出来，安装到远程的服务器集群里，从而解耦hive服务和metastore服务，保证hive运行的健壮性。

　　**Driver组件：**该组件包括Parser、Compiler、Optimizer和Executor，它的作用是将我们写的HiveQL(类SQL)语句进行解析、编译、优化，生成执行计划，然后调用底层的mapreduce计算框架。

-   -   解释器（Parser）：将SQL字符串转化为抽象语法树AST；
    -   编译器（Compiler）：将AST编译成逻辑执行计划；
    -   优化器（Optimizer）：对逻辑执行计划进行优化；
    -   执行器（Executor）：将逻辑执行计划转成可执行的物理计划，如MR/Spark

　　 **CLI：**command line interface，命令行接口

　　**ThriftServers：**提供JDBC和ODBC接入的能力，它用来进行可扩展且跨语言的服务的开发，hive集成了该服务，能让不同的编程语言调用hive的接口。

## 3.2 Hive详细运行架构

　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　**No.2**  **Hive运行详细架构图**

![[Blog/Picture/d0d4ad2f0fc304302257b05a331b504b_MD5.jpg]]

 　工作流程步骤：

-   1\. ExecuteQuery（执行查询操作）：命令行或Web UI之类的Hive接口将查询发送给Driver（任何数据驱动程序，如JDBC、ODBC等）执行；
-   2\. GetPlan（获取计划任务）：Driver借助编译器解析查询，检查语法和查询计划或查询需求；
-   3\. GetMetaData（获取元数据信息）：编译器将元数据请求发送到Metastore（任何数据库）；
-   4\. SendMetaData（发送元数据）：MetaStore将元数据作为对编译器的响应发送出去；
-   5\. SendPlan（发送计划任务）：编译器检查需求并将计划重新发送给Driver。到目前为止，查询的解析和编译已经完成；
-   6\. ExecutePlan（执行计划任务）：Driver将执行计划发送到执行引擎；
    -   6.1 ExecuteJob（执行Job任务）：在内部，执行任务的过程是MapReduce Job。执行引擎将Job发送到ResourceManager，ResourceManager位于Name节点中，并将job分配给datanode中的NodeManager。在这里，查询执行MapReduce任务；
    -   6.1 Metadata Ops（元数据操作）：在执行的同时，执行引擎可以使用Metastore执行元数据操作；
    -   6.2 jobDone（完成任务）：完成MapReduce Job；
    -   6.3 dfs operations（dfs操作记录）：向namenode获取操作数据；
-   7\. FetchResult（拉取结果集）：执行引擎将从datanode上获取结果集；
-   8\. SendResults（发送结果集至driver）：执行引擎将这些结果值发送给Driver；
-   9\. SendResults （driver将result发送至interface）：Driver将结果发送到Hive接口（即UI）；

## 3.3 Driver端的Hive编译流程

　　　　![[Blog/Picture/b1362f12d41f56638ba025013babd8ce_MD5.png]]

 　　Hive是如何将SQL转化成MapReduce任务的，整个编辑过程分为六个阶段：

-   1\. 词法分析/语法分析：使用Antlr定义SQL的语法规则，完成SQL词法，语法解析，将SQL语句解析成抽象语法树（AST Tree）；
-   2\. 语义分析：遍历AST Tree，抽象出查询的基本组成单元QueryBlock，并从Metastore获取模式信息，验证SQL语句中队表名、列名，以及数据类型（即QueryBlock）的检查和隐式转换，以及Hive提供的函数和用户自定义的函数（UDF/UAF）；
-   3. 逻辑计划生成：遍历QueryBlock，翻译生成执行操作树Operator Tree（即逻辑计划）；
-   4. 逻辑计划优化：逻辑层优化器对Operator Tree进行变换优化，合并不必要的ReduceSinkOperator，减少shuffle数据量；
-   5. 物理计划生成：将Operator Tree（逻辑计划）生成包含由MapReduce任务组成的DAG的物理计划——任务树；
-   6. 物理计划优化：物理层优化器对MapReduce任务树进行优化，并进行MapReduce任务的变换，生成最终的执行计划；

## 4. Hive源码分析

　　这里以hive-2.3.6为例。

## 4.1 源码目录构成分析

　　![[Blog/Picture/a86560586aac99324aa4bf6ffa3f73dc_MD5.png]]

　　**hive的三个重要组成部分**

-   serde：包含hive内置的序列化解析类，运行用户自定义序列化和发序列化解析器；
-   metastore：hive元数据服务器，用来存放数据仓库中所有表和分区的信息，hive元数据建表sql；
-   ql：解析sql生成的执行计划（了解hive执行流程的核心）

 　　**其他**

-   cli：hive命令行入口；
-   common：hive基础代码库，hive各组件信息的传递是通过hiveconf类管理的；
-   service：所有对外api接口的服务端，可以用于其他客户端与hive交互，例如：jdbc；
-   bin：hive执行的所有脚本；
-   beeline：hiveserver2提供的命令行工具；
-   findbugs：在java程序中查找bug的程序；
-   hwi：hive web页面的接口；
-   shim：用来兼容不容版本的hadoop和hive的版本；
-   hcatalog：Apache对于表和底层数据管理统一服务平台，hcatalog底层依赖于hive metastore；
-   ant：此组件包含一些ant任务需要的基础代码；

 　　**辅助组件**

-   conf：包含hive配置文件，hive-site.xml等；
-   data：hive所有的测试数据；
-   lib：hive运行所有的依赖包；

## 4.2 sql编译代码流程

　　对照节点3.3 Driver端的Hive编译流程，这里是具体的执行过程，如下：

 　![[Blog/Picture/fa68c5af86a96110abd92344d103fba3_MD5.jpg]]

## 4.3 sql编译源码分析

　　ql文件目录下面可以找到以下几个文件：

-   HiveLexer.g 是做词法分析的，定义了所有用到的token；
-   HiveParser.g 是做语法解析的；
-   FromClauseParser.g from从句语法解析；
-   SelectClauseParser.g select从句语法解析；
-   IdentifiersParser.g 自定义函数的解析；

　　hive源码中语法文件之间的关系：

　　![[Blog/Picture/1d1f27d9ad54817cb7ee46aedf15db58_MD5.jpg]]

 　　查看hive源码Driver类，整个sql语句编译、执行的步骤，执行的入口方法是在Driver.run(String command)方法中，执行的参数也就是一个sql字符串，只要的方法是：

　　① Driver类中run(String command)方法调用Driver.runInternal(String command，boolean alreadyCompiled)方法，

　　![[Blog/Picture/8b8ba844fce91875b260b08393274b65_MD5.png]]

 　　② Driver类中，runInternal方法则是调用compileInternal(command，true)，

　　![[Blog/Picture/f09627c4849f0c7fb0c0ae6419d5dfb0_MD5.png]]

 　　③ Driver类中，compileInternal方法调用compile(command，true，deferClose)方法；这里的compile方法为核心的编译方法，主要是将sql字符串翻译成ast树，然后翻译成可执行的task树，然后再优化执行树，如下图：

　　![[Blog/Picture/021712e6ddfaebddd2a5240f55233db8_MD5.png]]

 　　④ Driver类中，complie方法里面调用ParseUtils.parse(command，ctx)方法

　　![[Blog/Picture/3843b7d13977dba29c6f3ad760e8b988_MD5.png]]

 　　⑤ ParserUtils.parse方法则是调用ParseDriver的parser解析方法

　　![[Blog/Picture/77ef33ce33b7ea6a3a0f733f517d6b17_MD5.png]]

 　　可以看出ParseDriver.parse方法获取AST Tree【抽象语法树（abstract syntax tree ）】信息

　　⑥ Hive中调用antlr类的代码org.apache.hadoop.hive.ql.parse.ParseDriver类 返回的HiveParser.statement\_return和上面一样，是棵ast的语法树，具体语法树的接口可以参见相应的HiveParse.g文件

 　　![[Blog/Picture/2a8ae2abc72b392e3f8d0194c5c0c3c5_MD5.png]]

 　　⑦ 得到语法树之后，返回到Driver类中，会根据语法树根节点的类型来选择相应的SemanticAnalyzer

　　![[Blog/Picture/12575a594054724289587254c8b42c22_MD5.png]]

 　　红框② BaseSemanticAnalyzer sem = SemanticAnalyzerFactory.get(queryState, tree); 主要是根据根节点的语法树类型来选择相应的analyzer，具体的选择analyzer代码如下：

　　 ![[Blog/Picture/c510e2f4553ae335fa91b4e8f8a5f3b6_MD5.png]]

 　　对于DDL操作，得到的就是DDLSemanticAnalyzer，对于一般的insert(hive中存select语句会被翻译成一个insert tmpDirectory的语句)得到的就是SemanticAnalyzer。

 　　⑧ 接着Driver类中，调用BaseSemanticAnalyzer.analyze(tree，ctx)来将语法树翻译成可执行的执行计划；

　　 ![[Blog/Picture/dc79723868fa430ca8d85fb35f9ef484_MD5.png]]

 　　BaseSemanticAnalyzer的analyzer方法如下：

　　![[Blog/Picture/eeb6d141a3f628658d040baa19c0ec62_MD5.png]]

 　　SemanticAnalyzer继承BaseSemanticAnalyzer并重写analyzerInternal(ast)方法，SemanticAnalyzer.java（语义分析器）对一个树的根节点AST就能对整棵树进行解析（深度优先探索）

　　 接下来是把抽象语法树变成一个QB（query block），如下：

　　 ![[Blog/Picture/7b3f3926750a9d9ff6edce8e7d16ef2c_MD5.png]]

 　　![[Blog/Picture/aa88213f11d2eee3729fdcb4d4aaeb90_MD5.png]]

 　　一个QB类为：

　　![[Blog/Picture/09db8965a884cb6db98da293384a49a5_MD5.png]]

　　QB的两个重要变量是qbp和qbm他们都有QB的引用，这样组成了一棵树。

 　　在SemanticAnalyzer.analyzeInternal方法中Operator sinkOp = genOPTree(ast, plannerCtx);我们看一下Operator类的结构：

　　![[Blog/Picture/291450e4c0f294956e9d7caa32ab597f_MD5.png]]

 　　从代码中可以看到Operator有很多children和parent，由此这是一个有向无环图（DAG），QB经过genPlan()方法变成了一个DAG，接下来的Optimizer optm = new Optimizer()；是逻辑优化器。调用optm.initialize(conf)，Optimizer有以下优化器：

　　![[Blog/Picture/1ace88d282639f6a89e292d8355769ae_MD5.png]]

　　在SemanticAnalyzer.analyzeInternal方法中最终会调用compiler.compile(pCtx，rootTasks，inputs，outputs)；把可执行的计划存储在protected List<Task<? extends Serializable>> rootTasks;属性中，Task的executeTask()方法是可以直接执行的，最终实际的执行也是调用每个task的executeTask方法，依赖以及调度是在上层控制的，Task的继承关系如下：

　　![[Blog/Picture/23ed2609b5ded4bb1802a12919e99c2a_MD5.png]]

　　![[Blog/Picture/6800f1f133d56d4e1c3317b58ff14bc8_MD5.png]]

 　　Task是一个树形结构，每个task有一堆child task，这些child是在执行顺序上依赖于自己的task，rootTasks中存储的就是整个执行计划中需要最开始执行的task list，一棵“倒着的执行依赖树”。

　　⑨ Driver类中，执行task，Driver.execute()为入口

　　将可执行的task放入runnable中，初始为root task list，runnable表示正在运行running的task。

　　具体的执行流程如下：

-   不断去遍历runnable，选出一个执行launchTask(tsk，queryId，noName，running，jobname，jobs，driverCxt)，在这个方法中，启动task，其实就是调用task的executeTask()方法。

　　　　![[Blog/Picture/90cdb593880a936f73d2310931a35f9a_MD5.png]]

　　　这个里面hive是支持并发执行task的，若是需要并发的话每个task被封装成一个Thread的子类，然后自行启动。

-    找出执行完成的task，然后遍历该task的子task，选出可执行(pre task已经执行完)task放入runnable中，然后重复上一步骤。

　　　　![[Blog/Picture/19b847731b34ff259f6edef829998cdc_MD5.png]]

 　　　对于一些有多个pre task的child task，会在最后一个pre task执行完成后被启动，所以在这会被在child中过掉。

 　　以上逻辑就是整个hivesql的编译流程代码的大体脉络。

## 4.4 待补充

　　idea创建AstTreeTest测试类打印ast tree信息

```
<span>public</span> <span>class</span><span> AstTreeTest {
    </span><span>public</span> <span>static</span> <span>void</span><span> main(String[] args) {
        ParseDriver pd </span>=<span>new</span><span> ParseDriver();
        String sql</span>="select a,b,c from tab where age =222"<span>;
        ASTNode tree</span>=<span>null</span><span>;
        </span><span>try</span><span> {
             tree</span>=<span>pd.parse(sql);
        } </span><span>catch</span><span> (ParseException e) {
            e.printStackTrace();
        }
        System.out.println(</span>"AstNode:"+<span>tree.dump());
    }
}</span>
```

　　AstTree信息为：

　　![[Blog/Picture/fbd090475e0cfa620efc0c14932dc9d2_MD5.jpg]]

【参考资料】

 [https://www.cnblogs.com/bonelee/p/12441814.html](https://www.cnblogs.com/bonelee/p/12441814.html) Hive架构和工作原理

 [https://blog.csdn.net/oTengYue/article/details/91129850](https://blog.csdn.net/oTengYue/article/details/91129850) 一文弄懂Hive基本架构和原理

[https://www.cnblogs.com/lyr999736/p/9467854.html](https://www.cnblogs.com/lyr999736/p/9467854.html) Hive的架构和工作流程

[https://blog.csdn.net/wzq6578702/article/details/71250081](https://blog.csdn.net/wzq6578702/article/details/71250081) hive原理与源码分析-hive源码架构与理论（一）

[https://zhuanlan.zhihu.com/p/273263917](https://zhuanlan.zhihu.com/p/273263917) hive源码解读(3)-文件介绍