## 一.配置从私服下载

从私服下载主要是将 central 库的下载地址从`https://repo1.maven.org/maven2/`修改为私服地址，比如`http://localhost:8081/repository/maven-public/`。然后配置好访问私服的用户名和密码即可。

## 了解settings.xml文件结构

```xml
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 https://maven.apache.org/xsd/settings-1.0.0.xsd"> <localRepository/> <interactiveMode/> <offline/> <pluginGroups/> <servers/> <mirrors/> <proxies/> <profiles/> <activeProfiles/> </settings>
```

-   localRepository: 配置本地存储库的位置，默认为`${user.home}/.m2/repository`
-   interactiveMode: 是否与用户开启交互模式，默认为 true
-   offline: 离线模式，默认为 false
-   pluginGroups: 比如`<pluginGroup>org.eclipse.jetty</pluginGroup>`, 默认有`org.apache.maven.plugins and org.codehaus.mojo`。
-   servers: 配置私服的用户名和密码
-   mirrors: mirror相当于一个拦截器，它会拦截maven对remote repository的相关请求，把请求里的remote repository地址，重定向到mirror里配置的地址。
-   proxies: 代理配置
-   profiles: 配置环境
-   activeProfiles: 配置默认激活的环境

## 1-配置用户名和密码

```xml
<!-- 访问私服需要的用户名和密码 --> <servers> <server> <id>repo-releases</id> <username>admin</username> <password>admin</password> </server> <server> <id>repo-snapshots</id> <username>admin</username> <password>admin</password> </server> </servers>
```

## 2-配置profile

下面的私服地址是假的。

```xml
<!-- 配置 zero-rdc-repo --> <profile> <id>me-repo</id> <repositories> <!-- 配置的顺序决定了下载 jar 包的顺序 --> <!-- 阿里云的 release 版本 --> <repository> <id>central</id> <url>https://maven.aliyun.com/repository/central</url> <snapshots> <enabled>false</enabled> </snapshots> </repository> <!-- 私服的 release 版本 --> <repository> <id>repo-releases</id> <url>https://repo.rdc.aliyun.com/repository/release/</url> <snapshots> <enabled>false</enabled> </snapshots> </repository> <!-- 私服的 snapshot 版本 --> <repository> <id>repo-snapshots</id> <url>https://repo.rdc.aliyun.com/repository/snapshot/</url> <releases> <enabled>false</enabled> </releases> </repository> </repositories> <pluginRepositories> <!-- 阿里云插件的 release 版本 --> <pluginRepository> <id>central</id> <url>https://maven.aliyun.com/repository/central</url> <snapshots> <enabled>false</enabled> </snapshots> </pluginRepository> </pluginRepositories> </profile>
```

这里的 repositories 如果不配置的话，默认会有一个 Maven 中央仓库的配置，同样 pluginRepositories 中如果没有配置的话，默认也是有一个 Maven 中央仓库的配置。  
还有！如果 repositories 中没有配置 repository.id 是 central 的 repository，会自动增加一个 Maven 中央仓库的配置，并且是以追加的方式，也就是配置在 repositories 的最后一个。所以如果只配置了私服的 repository 情况下，就会先去私服中下载，私服中下载不到时再去追加上来的 Maven 中央仓库中下载。

## 3-配置 mirror

```xml
<!-- 配置拦截 repository 内的 url 进行重定向 --> <mirrors> <!-- 将 central 的请求重定向到阿里云的公共 Maven 仓库 --> <!-- 其它的不重定向到阿里云 --> <mirror> <id>Nexus-aliyun</id> <mirrorOf>central</mirrorOf> <name>Nexus aliyun</name> <url>https://maven.aliyun.com/repository/central</url> </mirror> </mirrors>
```

这里的 mirror 类似于重定向操作，改变 repository 的 url 属性。  
**如果在 repositories 配置了 central 的地址，则这里不配置也可以！！！**  
注意，这里写的是`central`而不是`*`，是因为我们只想把 Maven 中央仓库的请求重定向到阿里云上，而不是把所有的请求都重定向到阿里云上。Maven 中央仓库仅仅是一个仓库，打开 [https://mvnrepository.com/repos](https://mvnrepository.com/repos) 发现我们经常使用的`https://repo1.maven.org/maven2/`仅仅是众多仓库中的一个，只不过这个是比较大而全的仓库而已。如果我们把所有的请求都重定向到这个仓库，那么就会有依赖找不到。

## 二. 配置部署到私服

部署到私服就简单了，在项目中的 pom.xml 文件中加入如下内容后，并将访问私服的用户名和密码配置好即可。

在 pom.xml 中加入配置:

```xml
<distributionManagement> <repository> <id>local-release</id> <url>http://localhost:8081/repository/maven-releases/</url> </repository> <snapshotRepository> <id>local-snapshot</id> <url>http://localhost:8081/repository/maven-snapshots/</url> </snapshotRepository> </distributionManagement>
```

然后在 settings.xml 文件中加入访问私服的用户名和密码:

```xml
<servers> <server> <id>local-release</id> <username>snail</username> <password>admin</password> </server> <server> <id>local-snapshot</id> <username>snail</username> <password>admin</password> </server> </servers>
```

**注意：repository.id 和 server.id 必须是一致的！！！**  
配置好之后，在项目中执行`mvn clean deploy -Dmaven.test.skip`即可部署到私服了。

Q：还有一个问题就是我是部署到 release 库了还是 snapshot 库了？？？  
A：根据`<version>1.0.0</version>`中的内容是否是以`-SNAPSHOT`为结尾的进行区分，如果想发布到 snapshot 库则必须以`-SNAPSHOT`为结尾，否则就发布到了 release 库。

\------------------------------我是博客签名------------------------------  
**座右铭**：不要因为知识简单就忽略，不积跬步无以至千里。  
**版权声明**：自由转载-非商用-非衍生-保持署名。  
本作品采用[知识共享署名 4.0 国际许可协议](http://creativecommons.org/licenses/by/4.0/)进行许可。  
\----------------------------------------------------------------------