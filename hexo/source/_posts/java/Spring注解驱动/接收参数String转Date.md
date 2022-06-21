---
abbrlink: '0'
---
#### Springboot 注解方式String转Date

```
@DateTimeFormat(pattern = "yyyy-MM-dd HH:mm:ss")
@JsonFormat(timezone = "GMT+8",pattern = "yyyy-MM-dd")
private Date addTime;
```

