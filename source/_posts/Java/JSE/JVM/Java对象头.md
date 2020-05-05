---
title: Java对象头
categories:
  - Java
  - JSE
  - JVM
tags:
  - Java
  - JSE
  - JVM
  - Java对象头
abbrlink: 2eeb8779
date: 2020-05-06 00:27:00
---

> 下面是基于JDK8 64位

### 对象头的参看神器

```xml
<dependency>
  <groupId>org.openjdk.jol</groupId>
  <artifactId>jol-core</artifactId>
  <version>0.10</version>
</dependency>
```

代码如下：

```java
package com.github.mxsm;

import org.openjdk.jol.info.ClassLayout;

public class HeaderView {

    public static void main(String[] args) {

        HeaderView headerView = new HeaderView();
        System.out.println(ClassLayout.parseInstance(headerView).toPrintable());

    }

}

```

通过运行的结果如下：

```shell
com.github.mxsm.HeaderView object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           54 c3 00 f8 (01010100 11000011 00000000 11111000) (-134167724)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

通过发现在正常不设置任何参数的情况下，对象头的长度为12个字节。

增加一个JVM参数（取消对象指针压缩，默认情况下JDK是开启的）：

```
-XX:-UseCompressedOops
```

运行的结果：

```shell
com.github.mxsm.HeaderView object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           08 17 2e 1c (00001000 00010111 00101110 00011100) (472782600)
     12     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
Instance size: 16 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total
```

所以在不开启对象指针压缩的情况下对象头的长度为16个字节。

### 对象头的组成

- #### Mark Word

- #### class pointer

- #### array length

**普通对象**：

```java
//开启了指针压缩
|--------------------------------------------------------------|
|                     Object Header (96/128 bits)              |
|------------------------------------|-------------------------|
|        Mark Word (64 bits)         | Klass Word (32/64 bits) |
|------------------------------------|-------------------------|
```

**数组对象**：

```java
//开启指针压缩
|----------------------------------------------------------------------------------|
|                                 Object Header (128 bits)                         |
|--------------------------------|-----------------------|-------------------------|
|        Mark Word(64bits)       | Klass Word(32/64bits) |  array length(32bits)  |
|--------------------------------|-----------------------|-------------------------|
```

在开启指针压缩和非指针压缩

### Mark Word

