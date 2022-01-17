
---
title: FlinkSql读取Hive配置
tags: [Flink,配置]
abbrlink: 32987c8c
categories:
  - flink
  - Tabel&Sql
date: 2021-12-30 11:15:57
---

#### 1 配置hive Metastore

##### 	1.1 配置 hive-site.xml

```xml
<configuration>
   <property>
      <name>javax.jdo.option.ConnectionURL</name>
      <value>jdbc:mysql://localhost/metastore?createDatabaseIfNotExist=true</value>
      <description>metadata is stored in a MySQL server</description>
   </property>

   <property>
      <name>javax.jdo.option.ConnectionDriverName</name>
      <value>com.mysql.jdbc.Driver</value>
      <description>MySQL JDBC driver class</description>
   </property>

   <property>
      <name>javax.jdo.option.ConnectionUserName</name>
      <value>...</value>
      <description>user name for connecting to mysql server</description>
   </property>

   <property>
      <name>javax.jdo.option.ConnectionPassword</name>
      <value>...</value>
      <description>password for connecting to mysql server</description>
   </property>

   <property>
       <name>hive.metastore.uris</name>
       <value>thrift://localhost:9083</value>
       <description>IP address (or fully-qualified domain name) and port of the metastore host</description>
   </property>

   <property>
       <name>hive.metastore.schema.verification</name>
       <value>true</value>
   </property>

</configuration>
```

#### 2 Flink Sql Client

##### 	2.1 配置sql-cli-defaults.xml

```yaml
execution:
    planner: blink
    type: streaming
    ...
    current-catalog: myhive  # set the HiveCatalog as the current catalog of the session
    current-database: mydatabase
    
catalogs:
   - name: myhive
     type: hive
     hive-conf-dir: /opt/hive-conf  # contains hive-site.xml
```

#### 3 设置Hive方言

##### 	3.1 yaml文件配置

```yaml
execution:
  planner: blink
  type: batch
  result-mode: table

configuration:
  table.sql-dialect: hive
```

##### 	3.2 手动设置

```sql
Flink SQL> set table.sql-dialect=hive; -- to use hive dialect
[INFO] Session property has been set.

Flink SQL> set table.sql-dialect=default; -- to use default dialect
[INFO] Session property has been set.
```

##### 	3.3 Table API

```java
EnvironmentSettings settings = EnvironmentSettings.newInstance().useBlinkPlanner()...build();
TableEnvironment tableEnv = TableEnvironment.create(settings);
// to use hive dialect
tableEnv.getConfig().setSqlDialect(SqlDialect.HIVE);
// to use default dialect
tableEnv.getConfig().setSqlDialect(SqlDialect.DEFAULT);
```

