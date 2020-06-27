---
title: Java中的管道解析
date: 2018-09-15 10:46:38
tags: pipe
categories: java
---

# Java中的管道解析
@(java)

[TOC]

>线程间通信的一种，使用到了wait-notify

### 例子
#### 读取数据流
```java
public class ReadData {
    public void read(PipedReader in) {
        char[] bytes = new char[8];
        try {
            int l = 0;
            //读管道循环读(从与之建立连接的写管道读)
            while ((l = in.read(bytes)) != -1) {
                l = in.read(bytes);
                String s = new String(bytes, 0, l);
                System.out.println("消费：" + s);
            }
            System.out.println("--- OVER ---");
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

#### 写数据
```java
public class WriteData {
    public void write(PipedOutputStream out) {
        try {
            for (int i = 0; i < 100; i++) {
	            //写到管道（与读管道建立连接的管道）里边
                out.write(new String("" + i).getBytes());
                Thread.sleep(10);
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            try {
                out.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }

    }
}
```

#### 测试类
```java
public class Test {
    public static void main(String[] args) {
        WriteData writeData = new WriteData();
        ReadData readData = new ReadData();
        //定义写管道
        PipedWriter PipedWriter = new PipedWriter();
        //定义读管道
        PipedReader PipedReader = new PipedReader();
        try {
            //建立连接
            PipedReader.connect(PipedWriter);
            ThreadRead threadRead = new ThreadRead(readData, PipedReader);
            //读管道开始读--->等待写管道写
            threadRead.start();
            Thread.sleep(1000);
            ThreadWrite threadWrite = new ThreadWrite(writeData, PipedWriter);
            //写管道开始写
            threadWrite.start();
        } catch (IOException e) {
            e.printStackTrace();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

class ThreadWrite extends Thread {
    private WriteData writeData;
    private PipedWriter pipedWriter;
    public ThreadWrite(WriteData writeData, PipedWriter pipedWriter) {
        this.writeData = writeData;
        this.pipedWriter = pipedWriter;
    }
    @Override
    public void run() {
        super.run();
        writeData.write(pipedWriter);
    }
}

class ThreadRead extends Thread {
    private ReadData readData;
    private PipedReader pipedReader;

    public ThreadRead(ReadData readData, PipedReader pipedReader) {
        this.readData = readData;
        this.pipedReader = pipedReader;
    }

    @Override
    public void run() {
        super.run();
        readData.read(pipedReader);
    }
}
```

### 管道流
>使用管道流的过程中经常见的两种异常：
>* java.io.IOException: Write end dead
>* java.io.IOException: Read end dead
>Piped字节流和字符流完全一样，下面只分析字节流

**源码均位于PipedInputStream.java中**
```java
//Thread对象
Thread readSide;
Thread writeSide;

//发生在读操作
if (writeSide != null && !writeSide.isAlive() && !closedByWriter && (in < 0)) {
	//分析条件：如果写线程不为空 并且 写线程死掉 并且 没有写操作没有close 并且 读操作没有读到任何数据(-1)
	//则 抛出此异常
	throw new IOException("Write end dead");
}

//发生在写操作
if (readSide != null && !readSide.isAlive()) {
	//分析条件：若读线程不为空 并且 读线程已经死掉
    throw new IOException("Read end dead");
}

```

**总结**
* 若要在读操作不发生异常，则至少需要保证：
	1. 写线程存活
	2. 若写线程已经结束（死掉），那就要保证写操作被 close
* 若要在写操作不发生异常，则至少需要保证：
	1. 要保证有存活的读线程（即使执行一个死循环while(true); 因为只要保证读线程不结束）
	2. 看下图：只有读线程执行了read操作，才会给readSide赋值，所以，如果读线程没有读，那么即使它结束了也不会报错，尾音readSide为null

![Alt text](./1534070686097.png)


### 源码分析
>上述涉及到的主要操作就是：write个read

1. 建立连接（reader.connect(writer)  或   writer.connect(reader) ），最终都会走到writer的connect方法
2. 开始读 / 写
3. 开始写 / 读


```java
public synchronized int read()  throws IOException {
	...//省略一些代码：做一些基本判断，管道是否连接，是否关闭等
    readSide = Thread.currentThread();
    int trials = 2;
    //循环
    while (in < 0) {
	    //closedByWriter是否被写者关闭，Writer在close时会调用Reader中的receivedLast()方法来
	    //设置closedByWriter=true，并notifyAll读者
        if (closedByWriter) {
            /* closed by writer, return EOF */
            return -1;
        }
        //
        if ((writeSide != null) && (!writeSide.isAlive()) && (--trials < 0)) {
            throw new IOException("Pipe broken");
        }
        /* might be a writer waiting */
        notifyAll();
        try {
            wait(1000);
        } catch (InterruptedException ex) {
            throw new java.io.InterruptedIOException();
        }
    }
    int ret = buffer[out++];
    if (out >= buffer.length) {
        out = 0;
    }
    if (in == out) {
        /* now empty */
        in = -1;
    }
    return ret;
}
```






