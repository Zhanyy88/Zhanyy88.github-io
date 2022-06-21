---
title: oracle连接方式
tags:
  - oracle
categories: [Java,Oracle]
abbrlink: 94402b3c
---
##### 实例名，服务名



Oracle JDBC连接一共有三种方式，分别是：SERVICE_NAME、SID和TNSName。

1.SERVICE_NAME方式：jdbc:oracle:thin:@//<host>:<port>/<SERVICE_NAME>  

2.SID连接方式：jdbc:oracle:thin:@<host>:<port>:<SID> 
                    或：jdbc:oracle:thin:@<host>:<port>/<SID>

3.TNSName连接方式：jdbc:oracle:thin:@<TNSName>

打开oracle路径下的D:\oraclexe\app\oracle\product\11.2.0\server\network\ADMIN\tnsames.ora文件



红线框内的db25就是TNSName，是属于客户端的参数，其余内容都是服务端的参数。

SERVICE_NAME和SID的比较：
    SID是对内的，是实例级别的一个名字，用来内部之间称呼用。
    SERVICE_NAME是对外的，是数据库级别的一个名字，用来告诉外面的人，我数据库叫"SERVICE_NAME"。

访问数据库的过程：
要想访问数据库，必须把数据库文件加载进实例中。SID即INSTANCE_NAME是用来唯一标示实例的。SERVICE_NAME是oracle8i新引进的，8i之前，一个数据库只能由一个实例对应，但是随着高性能的需求，并行技术的使用，一个数据库可以由多个实例对应了，比较典型的应用如RAC。为了充分利用所有实例，并且令客户端连接配置简单，ORACLE提出了SERVICE_NAME的概念，该参数直接对应数据库，而不是某个实例。自此Oracle JDBC连接多使用SERVICE_NAME方式连接，逐渐替代SID方式连接。
————————————————
版权声明：本文为CSDN博主「qq_40391559」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/qq_40391559/article/details/87936681