![[Picture/cdc00e74d9a3fd3f74731cdc1097bf32_MD5.jpg]]


![[Picture/53f34725f1ceb2248ffbad8350be46de_MD5.jpg]]

![[Picture/f0f70510f1919f50aadfebaa350c0576_MD5.jpg]]

![[Picture/e9adf513f7b5f08d575932db585ac7fc_MD5.jpg]]

两个月之前——

![[Picture/25e95cb03405c24f9af8a34e2d386774_MD5.jpg]]

![[Picture/817c318daa6cf97bb1991cfd40224f7a_MD5.jpg]]

![[Picture/8459943d7f63c770978a840eb5916c56_MD5.jpg]]

![[Picture/89fceb0751c0a3d15b74920f385b11cd_MD5.jpg]]

![[Picture/074d609c2c7427f23d3f81b6370395fb_MD5.jpg]]

为满足用户标签的统计需求，小灰利用Mysql设计了如下的表结构，每一个维度的标签都对应着Mysql表的一列：

![[Picture/40785fe3ab62aebbdeacc73d2b483051_MD5.jpg]]

要想统计所有90后的程序员该怎么做呢？

用一条求交集的SQL语句即可：  

**Select count（distinct Name） as 用户数 from table whare age = '90后' and Occupation = '程序员' ;**  

要想统计所有使用苹果手机或者00后的用户总合该怎么做？

用一条求并集的SQL语句即可：  

**Select count（distinct Name） as 用户数 from table whare Phone = '苹果' or age = '00后' ;**

![[Picture/a19279a964adcab0a5fc8c73b3b39977_MD5.jpg]]

两个月之后——

![[Picture/e7cc60191e5b685187c8120ece81f3cc_MD5.jpg]]

![[Picture/0cd6406fb90f52f16e5d33b78fa73be7_MD5.jpg]] 

![[Picture/7e23afd1565820368e3917ddb8e76681_MD5.jpg]]

![[Picture/7241efc5398f4331ddf615c709f5f857_MD5.jpg]]

———————————————

![[Picture/531572170410324c4a62c942d00db7f1_MD5.jpg]]

![[Picture/943ba4ce70ea68430efea24de4afff02_MD5.jpg]]

![[Picture/2ee3f2f4bbe900f8b744e3eebac2c9ab_MD5.jpg]]

![[Picture/7dce176d20c9a9f183335e04126db890_MD5.jpg]]

![[Picture/71e5ff1faa4b41fa586952fbe3687694_MD5.jpg]]

1\. 给定长度是10的bitmap，每一个bit位分别对应着从0到9的10个整型数。此时bitmap的所有位都是0。  

    ![[Picture/5610e2f3b4b71da2e2c4e286d57b59b5_MD5.png]]

2\. 把整型数4存入bitmap，对应存储的位置就是下标为4的位置，将此bit置为1。

![[Picture/2043597864aab9f433b94ce88e6c2137_MD5.png]]

3\. 把整型数2存入bitmap，对应存储的位置就是下标为2的位置，将此bit置为1。

![[Picture/a2e271e20f298d88554e23173538a845_MD5.png]]

4\. 把整型数1存入bitmap，对应存储的位置就是下标为1的位置，将此bit置为1。

![[Picture/a2367dab4e6ef67307ebcec570e7cbc4_MD5.png]]

5\. 把整型数3存入bitmap，对应存储的位置就是下标为3的位置，将此bit置为1。

![[Picture/732a0397f1753e89c03922b2b9646f75_MD5.png]]

要问此时bitmap里存储了哪些元素？显然是4,3,2,1，一目了然。

Bitmap不仅方便查询，还可以去除掉重复的整型数。

![[Picture/57a0eae344db62a547ca6761e6474d87_MD5.jpg]]

![[Picture/b7a363ab4045df87722c4a061569edb3_MD5.jpg]]

![[Picture/b20e00144e5b2c5b29ad2cf6772ecea6_MD5.jpg]]

![[Picture/b5665be1cdb5dfa458d14e6e8688c119_MD5.jpg]]

![[Picture/2672fed8bf972be9358e36a945fde03b_MD5.jpg]]

![[Picture/cc28f5a3c0475d9332be5a634497e643_MD5.jpg]]

1\. 建立用户名和用户ID的映射：

![[Picture/633327444b2156e0c532a189af1cca02_MD5.png]]

2\. 让每一个标签存储包含此标签的所有用户ID，每一个标签都是一个独立的Bitmap。

![[Picture/bb7d562ca9fdf1466a4a9f02f46ff794_MD5.png]]

3\. 这样，实现用户的去重和查询统计，就变得一目了然：

![[Picture/4584bbdf3e9f3d24c3acd729d7adb303_MD5.png]]

![[Picture/044389707c09b385a3c424526ce145c0_MD5.jpg]]

![[Picture/4a7e9956880657637d7b3de6c13025e1_MD5.jpg]]

![[Picture/93cb9e3c52a26c10079c870596f0685d_MD5.jpg]]

![[Picture/0f014cf61bf9ddfcdc33e27e82935ddf_MD5.jpg]]

1\. 如何查找使用苹果手机的程序员用户？

![[Picture/574ef8b983ce77023c4cc5cd072b7384_MD5.png]]

2\. 如何查找所有男性或者00后的用户？

![[Picture/90b842448cc2cb714ba0dbeab8272dbd_MD5.png]]

![[Picture/c67bbf090c23b38a63df02bf9fb17eae_MD5.jpg]]

![[Picture/52ec778dba8468d3cd1587f071d78ef3_MD5.jpg]]

![[Picture/9bc1812558cf0d552cde4b4e369ee3af_MD5.jpg]]

![[Picture/4898a549d6817869aea9267e864b78b2_MD5.jpg]]

![[Picture/cded56271f6f8cac66a7e2fd80ec0a7a_MD5.jpg]]

![[Picture/b476a49a9ca5312b2d6bcc2d7897e1e4_MD5.jpg]]

![[Picture/7fc482e731ed55cf9d2bf585a73e2c9d_MD5.jpg]]

![[Picture/92a235a86efc6a386e0b28bade6feceb_MD5.jpg]]

一周之后......

![[Picture/03ccfe7cf8bde761f4112193c12b3b56_MD5.jpg]]

![[Picture/bd3284a6eacedb6142e6369e41c6d601_MD5.jpg]]

![[Picture/bf447de88dd64d43817e73991c82b682_MD5.jpg]]

![[Picture/38135b2652ff29eb709fb2e0fbb24fb4_MD5.jpg]]

我们以上一期的用户数据为例，用户基本信息如下。按照年龄标签，可以划分成90后、00后两个Bitmap：

![[Picture/7239b9a422008dc3472edf9b286bc8b2_MD5.png]]

用更加形象的表示，90后用户的Bitmap如下：

![[Picture/b94b67afad37558f7a6b78d3570d9751_MD5.png]]

这时候可以直接求得**非**90后的用户吗？直接进行非运算？  

![[Picture/eaba360a9d0e217c51b03b32492cce22_MD5.png]]

显然，非90后用户实际上只有1个，而不是图中得到的8个结果，所以不能直接进行非运算。

![[Picture/16403f1753754c6dabdbd50694539883_MD5.jpg]]

![[Picture/cb8d1e83c361debff18ed621903ca3fa_MD5.jpg]]

同样是刚才的例子，我们给定90后用户的Bitmap，再给定一个全量用户的Bitmap。最终要求出的是存在于全量用户，但又不存在于90后用户的部分。

![[Picture/6266ccc3272bfa7b17a9d50f3f37a237_MD5.png]]

如何求出呢？我们可以使用**异或**操作，即相同位为0，不同位为1。

![[Picture/4b0a699121c320580a7a4314bd08315d_MD5.png]]

![[Picture/e2588cfd6113210236fe9d163096ee9f_MD5.jpg]]

![[Picture/94a4442c35e07f8d434278c9caeea377_MD5.jpg]]

![[Picture/3d713ab85b43640e328279cbae0aec0c_MD5.jpg]]

![[Picture/9fa672e157d933bded9ddb2b8f55ef06_MD5.jpg]]

![[Picture/86827a8ce8694d2957759b5e267a5f50_MD5.jpg]]

![[Picture/a80a12c316f4d94506d62b3d77b708cd_MD5.png]]

![[Picture/80a503da5ccd19fc5afd6cfc7f30c9ec_MD5.jpg]]

![[Picture/b8fe531eca10993fad832d945d751788_MD5.jpg]]

![[Picture/d6a6da8b57556afd72b49d56a54d2535_MD5.png]]

![[Picture/2943ae7cd59113e0e2045c38510769b0_MD5.jpg]]

![[Picture/9263463330754eb7a865fbc175707f98_MD5.jpg]]

![[Picture/dc07380469162ad5e15b13e9c928e4ea_MD5.png]]

![[Picture/9e42f690ab1ec3f9971a48b80c0a0e00_MD5.jpg]]

![[Picture/dcdc3ee0ec6826fde1137c55213bc8e1_MD5.jpg]]

![[Picture/6200cece5b5a5b74e44170970ea1c190_MD5.jpg]]

![[Picture/07130a1311cce9b7c7060543c4e4ade5_MD5.png]]

![[Picture/39e70c38ef46bc44dbf475f1e7796aac_MD5.jpg]]

![[Picture/3116dd6db151a47c11fa5bf7e5e96b73_MD5.png]]

![[Picture/9b7d1562769b1fca30f264b3f066543b_MD5.jpg]]

![[Picture/f592a39acaa828a66c7b702a10ee3da3_MD5.png]]

![[Picture/808a7d9db06a68f5da9ee9678e1ab602_MD5.jpg]]

![[Picture/24bd45082b7ced46c6bb688e9c09f738_MD5.png]]

![[Picture/d5fbaaf2cc60b9b1cb9d4345a7dfcdef_MD5.jpg]]

![[Picture/08d0e92b97bf62a3200d8e944116392b_MD5.jpg]]

![[Picture/883252a6e14afb1d74a1c1317a851b50_MD5.jpg]]

![[Picture/77a068917ec3d96ea1b6b6e8a2413fd6_MD5.jpg]]

![[Picture/782ae14ac135c8ced7aa74e5d79087c0_MD5.jpg]]

![[Picture/f7d6be190f1da95b093d0dc949e0b978_MD5.png]]

![[Picture/459909b38af4521ce1799ad23c1b2e08_MD5.jpg]]

![[Picture/f53e3979f02fe91507e8d4c5c88a30a0_MD5.jpg]]

![[Picture/0d4081882b922cdabe6f3a4a9cfaedc5_MD5.jpg]]

![[Picture/1759528ca109c089e257a6ede6f5fd83_MD5.jpg]]

![[Picture/870c2b21df887957c58ed32ed7f95f10_MD5.png]]

25769803776L  =  11000000000000000000000000000000000B

8589947086L = 1000000000000000000011000011001110B

![[Picture/ca82e6c84e5661e8b920549468206ad9_MD5.jpg]]

![[Picture/5edf67dd60d0d63965018ffc3c7dd840_MD5.jpg]]

![[Picture/eaee3509c1fae3ec0a4df51d4966a0f6_MD5.jpg]]

![[Picture/1dea874b5f6c2c16f259beb7a640d60a_MD5.jpg]]

![[Picture/5afb02536be5dae965c12a28a9b05241_MD5.jpg]]

![[Picture/4391ea4e10121de60cab853861a5699b_MD5.jpg]]

1.解析Word0，得知当前RLW横跨的空Word数量为0，后面有连续3个普通Word。  

2.计算出当前RLW后方连续普通Word的最大ID是  64 X  (0 + 3) -1 = 191。

3\. 由于 191 < 400003，所以新ID必然在下一个RLW（Word4）之后。

4.解析Word4，得知当前RLW横跨的空Word数量为6247，后面有连续1个普通Word。

5.计算出当前RLW（Word4）后方连续普通Word的最大ID是191 + （6247 + 1）X64  = 400063。

6.由于400003 < 400063，因此新ID 400003的正确位置就在当前RLW（Word4）的后方普通Word，也就是Word5当中。

最终插入结果如下：  

![[Picture/55d64b23597e71b29a86a8b436c2a34d_MD5.jpg]]

![[Picture/b1a8ccc350784a4f3fd9d10f84165884_MD5.jpg]]

![[Picture/e8c286e0c63c8ecc236ed1f2fb048dc8_MD5.jpg]]

![[Picture/703f368e514e4c4a29176afeea5bd906_MD5.png]]

![[Picture/55541c68e3ab415e21029c6e52ee2e2b_MD5.jpg]]

官方说明如下：

```
* Though you can set the bits in any order (e.g., set(100), set(10), set(1),* you will typically get better performance if you set the bits in increasing order (e.g., set(1), set(10), set(100)).* * Setting a bit that is larger than any of the current set bit* is a constant time operation. Setting a bit that is smaller than an * already set bit can require time proportional to the compressed* size of the bitmap, as the bitmap may need to be rewritten.
```

![[Picture/bb628497db5b4fda165f6650a94836f4_MD5.jpg]]

**几点说明：**

1\. 该项目最初的技术选型并非Mysql，而是内存数据库hana。本文为了便于理解，把最初的存储方案写成了Mysq数据库。

1.文中介绍的Bitmap优化方法在一定程度上做了简化，源码中的逻辑要复杂得多。比如对于插入数据400003的定位，和实际步骤是有出入的。

2.如果同学们有兴趣，可以亲自去阅读源码，甚至是尝试实现自己的Bitmap算法。虽然要花不少时间，但这确实是一种很好的学习方法。

```
EWAHCompressedBitmap对应的maven依赖如下：
```

```
<dependency>  <groupId>com.googlecode.javaewah</groupId>  <artifactId>JavaEWAH</artifactId>  <version>1.1.0</version></dependency>
```

—————END—————

喜欢本文的朋友们，欢迎长按下图关注订阅号**梦见**，收看更多精彩内容

![[Picture/de7c979a5a9805a67d2db14e2d35bf3a_MD5.jpg]]