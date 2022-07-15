---
title: Buffer
tag: [NIO]
categories: [Java,基础,NIO]
abbrlink: 1dc09e78
date: 2022-07-14
---

### Buffer的方法
#### allocate()

```java
// 分配内存空间，
        //create buffer with capacity of 48 bytes
        ByteBuffer buffer = ByteBuffer.allocate(10);
        System.out.println("================= allocate 10 后 =================");
        System.out.println("capacity = " + buffer.capacity());
        System.out.println("position = " + buffer.position());
        System.out.println("limit = " + buffer.limit());
```
需要注意的是10并不是字节数，而是对象的数量
结果如下：
```java
================= allocate 10 后 =================
capacity = 10
position = 0
limit = 10
```

#### put()
调用allocate()分配内存后，实例对象处于写模式，put()方法想Buffer写入数据
```java
buffer.put(1);
buffer.put(2);
System.out.println("================= put 1,2 后 =================");
System.out.println("capacity = " + buffer.capacity());
System.out.println("position = " + buffer.position());
System.out.println("limit = " + buffer.limit());
```
结果如下：
```text
================= put 1,2 后 =================
capacity = 10
position = 2
limit = 10
```
#### flip() 切换为读模式
&emsp;&emsp;调用put()方法向Buffer中存储数据后，这时Buffer仍然处于写模式状态，在写模式状态下我们是不能直接从Buffer中读取数据的，需要调用flip()方法将
Buffer从写模式切换到读模式。

```java
buffer.flip();
System.out.println("================= flip 后 =================");
System.out.println("capacity = " + buffer.capacity());
System.out.println("position = " + buffer.position());
System.out.println("limit = " + buffer.limit());
```
结果如下
```text
================= flip 后 =================
capacity = 10
position = 0
limit = 2
```
&emsp;&emsp;Buffer的参数发生了变化，在读模式下limit代表Buffer的可读长度，等于写模式下的position，而position则是读的位置。


#### get()
&emsp;&emsp;get()读取数据很简单，每次从position的位置读取一个数据，并且将position向前移动1位。
```java
System.out.println("读取第 1 个位置的数据：" + buffer.get());
System.out.println("================= get 1 后 =================");
System.out.println("capacity = " + buffer.capacity());
System.out.println("position = " + buffer.position());
System.out.println("limit = " + buffer.limit());
```
结果如下
```text
读取第 1 个位置的数据：1.0
================= get 1 后 =================
capacity = 10
position = 1
limit = 2
```
&emsp;&emsp;上面说到limit为当前buffer最大可读位置，Buffer是一边读，position位置一边往前移动，那如果越界读取呢？
```java
System.out.println("读取第 2 个位置的数据：" + buffer.get());
System.out.println("读取第 3 个位置的数据：" + buffer.get());
System.out.println("读取第 4 个位置的数据：" + buffer.get());
```
&emsp;&emsp;如果越界读取，Buffer会抛出[BufferUnderflowException]{.label .danger}

```text
读取第 2 个位置的数据：2.0
java.nio.BufferUnderflowException
	at java.nio.Buffer.nextGetIndex(Buffer.java:500)
	at java.nio.HeapDoubleBuffer.get(HeapDoubleBuffer.java:135)
	at com.intellij.rt.junit.JUnitStarter.main(JUnitStarter.java:54)
```

#### rewind()
&emsp;&emsp;position是随着读取进度一直往前移动的，那如果我想在读取一遍数据呢？使用rewind()方法，可以进行重复读取。
```java
buffer.rewind();
System.out.println("================= rewind 后 =================");
System.out.println("capacity = " + buffer.capacity());
System.out.println("position = " + buffer.position());
System.out.println("limit = " + buffer.limit());
```
```text
================= rewind 后 =================
capacity = 10
position = 0
limit = 2
```

#### clear()和compact()
&emsp;&emsp;[flip()]{.label .danger}方法可以用于将Buffer从写模式切换到读模式，那怎么将Buffer从读模式切换至写模式呢？可以调用[clear()]{.label .danger}
和[compect()]{.label .danger}两个方法。
```java
buffer.clear();
System.out.println("================= clear 后 =================");
System.out.println("capacity = " + buffer.capacity());
System.out.println("position = " + buffer.position());
System.out.println("limit = " + buffer.limit());
```

```text
================= clear 后 =================
capacity = 10
position = 0
limit = 10
```
&emsp;&emsp;调用[clear()]{.label .danger}后，position的值变为0，limit值变成了10，也就是buffer清空了，回归到最迟的状态。但是里面的数据仍然是存在的，
只是没有标记那些数据是已读，那些是未读。

```java
buffer.compact();
System.out.println("================= compect 后 =================");
System.out.println("capacity = " + buffer.capacity());
System.out.println("position = " + buffer.position());
System.out.println("limit = " + buffer.limit());
```
```text
================= compect 后 =================
capacity = 10
position = 2
limit = 10
```
&emsp;&emsp;[compect()]{.label .danger}方法也可以将Buffer从读模式切换到写模式，它跟[clear()]{.label .danger}有一些区别。
可以看到position的值是2，从未读的数据后面开始写入数据，

#### mark()和reset()
&emsp;&emsp;调用[mark()]{.label .danger}方法可以标志一个指定的位置（即mark的位置）,之后调用[reset()]{.label .danger}时，position又
会回到之前标记的位。下面是源码：
```java
public final Buffer mark() {
    mark = position;
    return this;
}

public final Buffer reset() {
    int m = mark;
    if (m < 0)
        throw new InvalidMarkException();
    position = m;
    return this;
}
```

### Buffer的类型
[ByteBuffer]{.label}
[CharBuffer]{.label}
[DoubleBuffer]{.label}
[FloatBuffer]{.label}
[IntBuffer]{.label}
[LongBuffer]{.label}
[ShortBuffer]{.label}
[MappedByteBuffer]{.label}




