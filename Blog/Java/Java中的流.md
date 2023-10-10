# 字节输入流
## inputStream
用于从源头（通常是文件）读取数据（字节信息）到内存中，`java.io.InputStream`抽象类是所有字节输入流的父类

## FileInputStream（文件输入流）
是一个比较常用的字节输入流对象，可直接指定文件路径，可以直接读取单字节数据，也可以读取至字节数组中。

## BufferedInputStream（缓冲流）

## DataInputStream
用于读取指定类型数据，不能单独使用

# ObjectInputStream
用于从输入流中读取 Java 对象（反序列化)

# 字节输出流（）
## OutputStream（字节输出流）
用于将数据（字节信息）写入到目的地（通常是文件），`java.io.OutputStream`抽象类是所有字节输出流的父类。

# FileOutputStream
是最常用的字节输出流对象，可直接指定文件路径，可以直接输出单字节数据，也可以输出指定的字节数组。