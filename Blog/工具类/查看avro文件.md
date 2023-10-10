Sat, Jul 25, 2020, 22:22

使用avro提供的工具，可以通过命令行直接查看avro文件、查看avro schema、avro和json互转等。

1.  [avro-tools](https://puppylpg.github.io/2020/07/25/avro-tools/#avro-tools)
2.  [命令](https://puppylpg.github.io/2020/07/25/avro-tools/#%E5%91%BD%E4%BB%A4)
    1.  [base](https://puppylpg.github.io/2020/07/25/avro-tools/#base)
    2.  [查看avro：tojson](https://puppylpg.github.io/2020/07/25/avro-tools/#%E6%9F%A5%E7%9C%8Bavrotojson)
    3.  [查看schema：getschema](https://puppylpg.github.io/2020/07/25/avro-tools/#%E6%9F%A5%E7%9C%8Bschemagetschema)
    4.  [json转avro：fromjson](https://puppylpg.github.io/2020/07/25/avro-tools/#json%E8%BD%ACavrofromjson)
    5.  [生成Java code](https://puppylpg.github.io/2020/07/25/avro-tools/#%E7%94%9F%E6%88%90java-code)
3.  [Ref](https://puppylpg.github.io/2020/07/25/avro-tools/#ref)

-   https://mirrors.tuna.tsinghua.edu.cn/apache/avro/stable/java/

avro release里有很多东西，java文件夹下有各种工具，avro-tools是其中一个。

## 命令[¶](https://puppylpg.github.io/2020/07/25/avro-tools/#%E5%91%BD%E4%BB%A4 "Permanent link")

## base[¶](https://puppylpg.github.io/2020/07/25/avro-tools/#base "Permanent link")

-   java -jar avro-tools-1.10.0.jar：相当于help，显示各个子命令；

## 查看avro：tojson[¶](https://puppylpg.github.io/2020/07/25/avro-tools/#%E6%9F%A5%E7%9C%8Bavrotojson "Permanent link")

查看avro一般是转成json再看，要不然二进制也没法看：

-   java -jar avro-tools-1.10.0.jar tojson xxx.avro > xxx.json

> Avro To XXX，只需要指明xxx就行了，所以是tojson。

## 查看schema：getschema[¶](https://puppylpg.github.io/2020/07/25/avro-tools/#%E6%9F%A5%E7%9C%8Bschemagetschema "Permanent link")

-   java -jar avro-tools-1.10.0.jar getschema xxx.avro

## json转avro：fromjson[¶](https://puppylpg.github.io/2020/07/25/avro-tools/#json%E8%BD%ACavrofromjson "Permanent link")

json转avro需要指明avro的schema定义文件，即avsc文件，需要通过子命令fromjson的选项`--schema-file`：

-   java -jar avro-tools-1.10.0.jar fromjson –schema-file xxx.avsc xxx.json > xxx.avro

> Avro from xxx，只需要指明xxx就行了，所以是fromjson。

另外，avro也可以使用压缩，比如使用snappy压缩，使用`--codec`：

-   java -jar avro-tools-1.10.0.jar fromjson –codec snappy –schema-file xxx.avsc xxx.json > xxx.avro

fromjson的文档：

```
pichu@Archer ~/Utils/avro $ java -jar avro-tools-1.10.0.jar fromjson                                                                            
Expected 1 arg: input_file
Option                  Description                                        
------                  -----------                                        
--codec <String>        Compression codec (default: null)                  
--level <Integer>       Compression level (only applies to deflate, xz, and
                          zstandard) (default: -1)                         
--schema [String]       Schema                                             
--schema-file [String]  Schema File
```

## 生成Java code[¶](https://puppylpg.github.io/2020/07/25/avro-tools/#%E7%94%9F%E6%88%90java-code "Permanent link")

> 使用avro-maven-plugin可以直接在maven工程里生成Java 代码，没必要手撸。

手撸：

-   java -jar avro-tools-1.10.0.jar compile schema user.avsc .：将制定avsc编译为Java代码，到本目录，代码按照指定的package存放。

## Ref[¶](https://puppylpg.github.io/2020/07/25/avro-tools/#ref "Permanent link")

-   https://www.michael-noll.com/blog/2013/03/17/reading-and-writing-avro-files-from-the-command-line/

其他：

-   avro-tool的api：http://avro.apache.org/docs/current/api/java/org/apache/avro/tool/package-summary.html
-   avro doc：http://avro.apache.org/docs/current/

##### Feedback

Was this page helpful?

Glad to hear it! Please [tell us how we can improve](https://github.com/puppylpg/puppylpg.github.io/issues/new).

Sorry to hear that. Please [tell us how we can improve](https://github.com/puppylpg/puppylpg.github.io/issues/new).