---
title: String转Date
tag:
  - Spring
sticky: false
categories:
  - Java
  - Spring
abbrlink: 516f4dbe
---
#### Springboot 注解方式String转Date

```java
@DateTimeFormat(pattern = "yyyy-MM-dd HH:mm:ss")
@JsonFormat(timezone = "GMT+8",pattern = "yyyy-MM-dd")
private Date addTime;
```

