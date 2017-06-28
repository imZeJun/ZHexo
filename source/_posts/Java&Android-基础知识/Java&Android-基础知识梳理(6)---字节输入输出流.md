---
title: Java&Android 基础知识梳理(6) - 字节输入输出流
date: 2017-03-21 15:35
categories : Java&Android 基础知识梳理
---
# 一、概述
`Java IO`库中的**流**代表有能力产出数据的数据源对象或者是有能力接收数据的接收端对象，我们一般把它分成输入和输出两部分：
- 继承自`InputStream`或`Reader`派生的类都含有名为`read`的方法，用于读取单个字节或字节数组。
- 继承自`OuputStream`或`Writer`派生的类都含有名为`write`的方法，用于写入单个字节或字节数组。

我们通常通过叠合多个对象来提供所期望的功能，这其实是一种装饰器设计模式。

## 二、字节输入流
# 2.1 `InputStream`的作用
它的作用是用来表示那些从不同数据源产生输入的类，而最终结果就是通过`read`方法获得数据源的内容，从数据源中读出的内容**用`int`或`byte[]`来表示**：
- 字节数组
- `String`对象
- 文件
- 管道
- 一个由其它种类的流组成的序列，以便我们可以将它们收集合并到一个流内
- 其它数据源，如网络等

## 2.2 `InputStream`源码
`InputStream`是一个抽象类，所有表示字节输入的流都是继承于它，它实现了以下接口：
![](http://upload-images.jianshu.io/upload_images/1949836-bd0b01206b763fc3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
比较关键的是前面四个方法：
- `public abstract int read() throws IOException`
返回输入流的**下一个字节（`next byte`）**，如果已经到达输入流的末尾，那么返回`-1`
- `public int read(byte b[]) throws IOException`
尝试从输入流中读取`b.length`长度的字节，存入到`b`中，如果已经到达末尾返回`-1`，否则返回成功写入到`b`中的字节数。
- `public int read(byte b[], int off, int len) throws IOException`
尝试从输入流中读取下`len`长度的字节，如果`len`为`0`，那么返回`0`，否则返回实际读入的字节数，读入的第一个字节存放在数据`b[off]`中，如果没有可读的，那么返回`-1`。
- `public long skip(long n) throws IOException`
跳过，并丢弃掉`n`个字节，其最大值为`2048`。

## 2.3 `InputStream`的具体实现类
- `ByteArrayInputStream`
它接收`byte[]`作为构造函数的参数，我们调用`read`方法时，就是从`byte[]`数组里，读取字节。
```
    public ByteArrayInputStream(byte buf[], int offset, int length) {
        this.buf = buf;
        this.pos = offset;
        this.count = Math.min(offset + length, buf.length);
        this.mark = offset;
    }

    public synchronized int read() {
        return (pos < count) ? (buf[pos++] & 0xff) : -1;
    }
```
- `StringBufferInputStream`
已经过时，推荐使用`StringReader`。
- `FileInputStream`
`FileInputStream`支持提供文件名、`File`和`FileDescription`作为构造函数的参数，它的`read`调用的是底层的`native`方法。
```
    public FileInputStream(File file) throws FileNotFoundException {
        String name = (file != null ? file.getPath() : null);
        SecurityManager security = System.getSecurityManager();
        if (security != null) {
            security.checkRead(name);
        }
        if (name == null) {
            throw new NullPointerException();
        }
        if (file.isInvalid()) {
            throw new FileNotFoundException("Invalid file path");
        }
        fd = new FileDescriptor();
        fd.attach(this);
        path = name;
        open(name);
    }

    public int read() throws IOException {
        return read0();
    }

    private native int read0() throws IOException;
```
- `PipedInputStream`
通过通信管道来交换数据，如果两个线程希望进行数据的传输，那么它们一个创建管道输出流，另一个创建管道输入流，它必须要和一个`PipedOutputStream`相连接。
```
    public PipedInputStream(int pipeSize) {
        if (pipeSize <= 0) {
            throw new IllegalArgumentException("pipe size " + pipeSize + " too small");
        }
        buffer = new byte[pipeSize];
    }

    public PipedInputStream(PipedOutputStream out, int pipeSize) throws IOException {
        this(pipeSize);
        connect(out);
    }

    @Override
    public synchronized int read() throws IOException {
        if (!isConnected) {
            throw new IOException("Not connected");
        }
        if (buffer == null) {
            throw new IOException("InputStream is closed");
        }
        lastReader = Thread.currentThread();
        try {
            int attempts = 3;
            while (in == -1) {
                // Are we at end of stream?
                if (isClosed) {
                    return -1;
                }
                if ((attempts-- <= 0) && lastWriter != null && !lastWriter.isAlive()) {
                    throw new IOException("Pipe broken");
                }
                notifyAll();
                wait(1000);
            }
        } catch (InterruptedException e) {
            IoUtils.throwInterruptedIoException();
        }
        int result = buffer[out++] & 0xff;
        if (out == buffer.length) {
            out = 0;
        }
        if (out == in) {
            in = -1;
            out = 0;
        }
        notifyAll();
        return result;
    }
```
- `SequenceInputStream`
将多个`InputStream`连接在一起，一个读完后就完毕，并读下一个，它接收两个`InputStream`对象或者一个容纳`InputStream`对象的容器`Enumeration`。
```
    public int read() throws IOException {
        while (in != null) {
            int c = in.read();
            if (c != -1) {
                return c;
            }
            nextStream();
        }
        return -1;
    }
```
- `ObjectInputStream`
它和一个`InputStream`相关联，源数据都是来自于这个`InputStream`，它继承于`InputStream`，并且它和传入的`InputStream`并不是直接关联的，中间通过了`BlockDataInputStream`进行中转，要关注的就是它的`readObject`方法，它会把一个之前序列化过的对象进行反序列化，然后得到一个`Object`对象，它的**目的在于将（把二进制流转换成为对象）和（从某个数据源中读出字节流）这两个操作独立开来，让它们可以随意地组合**。
```
    public ObjectInputStream(InputStream in) throws IOException {
        verifySubclass();
        bin = new BlockDataInputStream(in);
        handles = new HandleTable(10);
        vlist = new ValidationList();
        serialFilter = ObjectInputFilter.Config.getSerialFilter();
        enableOverride = false;
        readStreamHeader();
        bin.setBlockDataMode(true);
    }

    public final Object readObject() throws IOException, ClassNotFoundException {
        if (enableOverride) {
            return readObjectOverride();
        }

        // if nested read, passHandle contains handle of enclosing object
        int outerHandle = passHandle;
        try {
            Object obj = readObject0(false);
            handles.markDependency(outerHandle, passHandle);
            ClassNotFoundException ex = handles.lookupException(passHandle);
            if (ex != null) {
                throw ex;
            }
            if (depth == 0) {
                vlist.doCallbacks();
            }
            return obj;
        } finally {
            passHandle = outerHandle;
            if (closed && depth == 0) {
                clear();
            }
        }
    }
```
- `FilterInputStream`
它的构造函数参数就是一个`InputStream`：
```
    protected FilterInputStream(InputStream in) {
        this.in = in;
    }

    public int read() throws IOException {
        return in.read();
    }
```
这个类很特殊，前面的`InputStream`子类都是传入一个数据源（`Pipe/byte[]/File`）等等，然后通过重写`read`方法从数据源中读取数据，而`FilterInputStream`则是将`InputStream`组合在内部，它调用`in`去执行`InputStream`定义的抽象方法，也就是说它不会改变组合在内部的`InputStream`所对应的数据源。
另外，它还新增了一些方法，这些方法底层还是调用了`read`方法，但是它封装了一些别的操作，比如`DataInputStream`中的`readInt`，它调用`in`连续读取了四次，然后拼成一个`int`型返回给调用者，之所以采用组合，而不是继承，**目的是将（把二进制流转换成别的格式）和（从某个数据源中读出字节流）这两个操作独立开来，让它们可以随意地组合**。
```
    public final int readInt() throws IOException {
        int ch1 = in.read();
        int ch2 = in.read();
        int ch3 = in.read();
        int ch4 = in.read();
        if ((ch1 | ch2 | ch3 | ch4) < 0)
            throw new EOFException();
        return ((ch1 << 24) + (ch2 << 16) + (ch3 << 8) + (ch4 << 0));
    }
```

## 三、字节输出流
## 3.1 `OuputStream`的作用
`OuputStream`决定了输出要去往的目标：
- 字节数组
- 文件
- 管道

## 3.2 `OutputStream`源码
和`InputStream`类似，也是一个抽象类，它的子类代表了输出所要去往的目标，它的关键方法如下：
![](http://upload-images.jianshu.io/upload_images/1949836-4338a4e237b5505a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
我们主要关注的是`write`方法，前两个`write`方法最终都是调用了抽象的`write(int oneByte)`方法，最终怎么写入是由子类实现的。

## 3.3 `OutputStream`的具体实现类
- `ByteArrayOutputStream`
在`ByteArrayOutputStream`的内部，有一个可变长的`byte[]`数组，当我们调用`write`方法时，就是向这个数组中写入数据，它还提供了`toByteArray/toString`方法，来获得当前内部`byte[]`数组中的内容。
- `FileOutputStream`
它和上面类似，只不过写入的终点换成了所打开的文件。
- `PipedOutputStream`
和`PipedInputStream`相关联。
- `ObjectOutputStream`
和`ObjectInputStream`类似，只不过它内部组合的是一个`OutputStream`，当调用`writeObject(Object object)`方法时，其实是先将`Object`进行反序列化转换为`byte`，再输出到`OuputStream`所指向的目的地。
- `FilterOutputStream`
它的思想和`FilterInputStream`类似，都是在内部组合了一个`OuputStream`，`FilterOutputStream`提供了写入`int/long/short`的`write`重载函数，当我们调用这些函数之后，`FilterOutputStream`最终会通过内部的`OuputStream`向它所指向的目的地写入字节。
