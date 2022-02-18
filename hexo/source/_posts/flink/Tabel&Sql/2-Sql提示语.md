---
title: Sql提示语
tags: [Flink,sql]
abbrlink: 686ec8f1
categories: 
  - Flink
  - Tabel&Sql
date: 2022-01-17 11:15:57
---


**SQL Hints**

------

#### 

```sql
-- override table options in query source
select id, name from kafka_table1 /*+ OPTIONS('scan.startup.mode'='earliest-offset') */;

-- override table options in join
select * from
    kafka_table1 /*+ OPTIONS('scan.startup.mode'='earliest-offset') */ t1
    join
    kafka_table2 /*+ OPTIONS('scan.startup.mode'='earliest-offset') */ t2
    on t1.id = t2.id;
    
    
-- override table options for INSERT target table
insert into kafka_table1 /*+ OPTIONS('sink.partitioner'='round-robin') */ select * from kafka_table2;
```

