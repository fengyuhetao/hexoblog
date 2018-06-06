---
title: phpinfo解读
date: 2018-05-31 08:52:57
tags:
password: 123456
---

# phpinfo解读

<!--more-->

我们以 [链接：](http://675ba661.2m1.pw/5a258c55-e0e9-495c-8b74-4ffd2885a2f8.php)[phpinfo()](http://675ba661.2m1.pw/5a258c55-e0e9-495c-8b74-4ffd2885a2f8.php) 为例来讲解吧。

 phpinfo里最显眼的信息，也就是其标题（图1），其中包含PHP的版本信息。 PHP 5.6 是官方最后一个支持的PHP 5版本，不过，这个版本也已经于2017年1月19号停止功能性更新，只提供安全更新。到2018年12月31号为止，PHP 5.6也将完全终止更新，PHP5彻底退出历史舞台。

 PHP7里提供了很多更好用的语法、函数，也极大改进了安全性。举几个简单的例子：

  1. 移除不支持SQL预编译的Mysql扩展：mysql 
  2. 移除preg_replace中容易导致代码执行漏洞的正则模式：e 
  3. assert从一个函数变成一个语法结构（类似eval，无法再动态调用。至此，大量PHP一句话木马将失效），7.2中废弃字符串形式的参数 
  4. hex字符串（如0xf4c3b00c）不再被作为数字，is_numeric也不再认可，可见 [链接：https://3v4l.org/ORuc7](https://3v4l.org/ORuc7) 
  5. 7.2中废弃可以动态执行字符串的 create_function 
  6. 7.2中废弃容易导致变量覆盖的无第二个参数的parse_str 
  7. 移除`<script language="php"\>`和`<%`，这两种另类的PHP标签 
  8. 移除dl函数

这里是关于PHP版本支持的一些说明 [链接：](http://php.net/supported-versions.php)[PHP: Supported Versions](http://php.net/supported-versions.php) PHP5从5.2到5.6，其实也做了很多改进，以前分享过一篇文章 [链接：](http://www.cnblogs.com/iamstudy/articles/study_from_php_update_log.html)[php各版本的姿势(2017-02-15更新) - l3m0n - 博客园](http://www.cnblogs.com/iamstudy/articles/study_from_php_update_log.html) 大家可以再温故一下，我就不再介绍了。

和版本有关的一些tricks：

 - [链接：](https://tricking.io/card/26/description)[梧桐百科 - 碎片化知识学习](https://tricking.io/card/26/description) - 
 - [链接：](https://tricking.io/card/24/description)[梧桐百科 - 碎片化知识学习](https://tricking.io/card/24/description) - 
 - [链接：](https://tricking.io/card/2/description)[梧桐百科 - 碎片化知识学习](https://tricking.io/card/2/description) 

这个两个在线工具可以用来在PHP多版本（包括小版本）中执行代码，进而比较他们之前的区别： 

- [链接：](https://3v4l.org/)[Online PHP editor | Run code in 200  PHP & HHVM ve...](https://3v4l.org/) 
- [链接：](http://sandbox.onlinephpfunctions.com/)[PHP Sandbox, test PHP online, PHP tester](http://sandbox.onlinephpfunctions.com/)

图1中的四项配置，与php.ini有关。

图一:

![](https://file.zsxq.com/148/e0/48e0d690fe8697f0a4129a049f1710a345dc94fd92d01dc4ba174f51ca695580.png)

 php.ini是php的配置文件，但其实并不一定每个php环境都会有这个文件。比如我图1中的这个环境，其“Loaded Configuration File”的值是“(none)”，其实就是没有。如果我们没有指定php.ini文件，那么php的所有配置都会使用默认配置。 

这里有一点值得注意，那就是“默认配置”的含义： 

1. 没有任何php.ini时，某个配置项默认的值 
2. 用apt等源管理工具安装php后，默认php.ini里配置的值 

上述两个“默认配置”在大部分情况下是相同的，但还是有些许不同。 举个简单的例子，配置项request_order用来指定PHP的\$_REQUEST变量以何种顺序取值。《Discuz! 6.x/7.x 全局变量防御绕过导致代码执行》（ [链接：](https://www.secpulse.com/archives/2338.html)[Discuz! 6.x/7.x 全局变量防御绕过导致命令执行 - SecPulse.COM | 安全...](https://www.secpulse.com/archives/2338.html) ） 就与这个配置有关。 (Discuz那个链接看着有点迷糊，我看的这个[链接：](http://wooyun.jozxing.cc/static/bugs/wooyun-2014-080723.html)[Discuz!某两个版本前台产品命令执行（无需登录） | WooYun-2014-80723 | W...](http://wooyun.jozxing.cc/static/bugs/wooyun-2014-080723.html))。

文中提到，因为PHP5.3以后request_order的默认值变成GP，不再包含\$\_COOKIE，所以导致通过\$_COOKIE来覆盖变量，造成代码执行。

 其实文中说的request_order默认值变成GP，指的是第2种情况。也就是说，如果一个完全没有任何php.ini文件的php环境，其默认的request_order，仍然是GPC，而非GP。 我们可以来实验一下，首先用docker启动一个php环境（docker的php环境是没有php.ini的）： 

> docker run --rm --name php1 -it -p 8080:80 -v `pwd`:/var/www/html php:7.2-apache 

再用docker启动一个默认的debian环境，并用apt源安装php5：

> docker run --rm --name php2 -it -p 8081:80 -v `pwd`:/var/www/html debian:8 bash apt-get update && apt-get install -y php5 
>
> php -S 0.0.0.0:80 -t /var/www/html 

然后，我们输出一下两个环境的request_order取值，以及\$\_REQUEST变量：图2。 可见，在request_order为空的时候，\$\_REQUEST可以取到$_COOKIE中的值；而request_order=GP的时候，就已经取不到了。我们查看第二个环境的php.ini文件，也可以看到request_order的取值：图3。

 所以，这两种默认值的含义，需要区分清楚，避免出现错误。

图二:

![](https://file.zsxq.com/1d6/59/d659ac1a237468325de98f2cfc0802548d77e8a7ebd9d1a80d0390d2d0b2b7e2.png)

图三:

![](https://file.zsxq.com/164/c8/64c85402cb752d1d7a5a2fbec6bf434338de9526b3836c08c28cb9382609a720.png)

win10: 本地win10+phpstudy+apache中5.3.29-nts与5.4.45-nts中的requests_order方式是"CGP"

除了默认php配置，还有两个配置项我没说。在编译PHP的时候，我们可以指定--with-config-file-path和--with-config-file-scan-dir。

 这两个配置项的意思是，PHP会在--with-config-file-path指定的目录下寻找php.ini文件，如果找到则加载之；除此之外，PHP还会在--with-config-file-scan-dir指定的目录下，寻找所有以.ini为后缀的文件，加载其为配置文件，这个配置是可以覆盖php.ini中配置的。 所以，通常用apt-get install php-pdo来安装php扩展，都会在--with-config-file-scan-dir下写入新的配置文件，而不是修改php.ini。 

这样，我们如果遇到phpinfo页面，即可用过这两个配置项，来定位php.ini以及额外配置文件的位置。甚至来说，如果这两个目录可写，我们就能写入自己的php配置，进而达成跨站、提权等目的。

图四:

![](https://file.zsxq.com/148/e0/48e0d690fe8697f0a4129a049f1710a345dc94fd92d01dc4ba174f51ca695580.png)



见图5，Server API 指的是当前PHP运行在哪一种模式下。SAPI是PHP内核的一个概念，相当于是PHP核心解释器和应用层（如Web中间件）的一个桥梁。我们需要在SAPI里实现一些函数，供PHP底层调用。 

这篇文章（ [链接：](https://foio.github.io/php-sapi/)[理解php内核中SAPI的作用 – 积木村の研究所](https://foio.github.io/php-sapi/) ）里举了个简单的例子：cli和cgi这两个SAPI，均需要实现sapi_cgi_read_cookies函数，但cli模式下没有cookie，所以该函数返回null；cgi模式下，cookie从HTTP头中获取，所以返回的是getenv("HTTP_COOKIE")。 

下面介绍一些常用的SAPI。 图5中的FPM/FastCGI指的就是一种SAPI，其提供了一种以fastcgi协议和Web中间件（如Nginx）通信的接口，具体流程可以阅读我的这篇文章（ [链接：](https://www.leavesongs.com/PENETRATION/fastcgi-and-php-fpm.html)[Fastcgi协议分析 && PHP-FPM未授权访问漏洞 && Exp编写 | 离别歌](https://www.leavesongs.com/PENETRATION/fastcgi-and-php-fpm.html)  ）。

图5:

![](https://file.zsxq.com/1a0/72/a072352975fb3a7f091f12f1f51c359d246c1ea1dded3a33be4300fc81d143c8.png)

 PHP为Apache提供了一个专用了SAPI——Apache 2.0 Handler。相当于是把PHP编译成一个动态链接库，作为Apache的一个模块。

 当然，Apache也不一定必须用这种SAPI，他同样也支持用Fastcgi和PHP-FPM通信。所以，即使你在实战中遇到了服务器是Apache的环境，也不能就此认定其一定存在Apache模块。你可以尝试下载一个PHPStudy，来看看是否有apache_get_version函数，答案是否定的。 我们平时在命令行里运行PHP，比如php -i （在命令行下执行phpinfo的方法），你可以看到此时的Server API是“Command Line Interface”：图6。

图6:

![](https://file.zsxq.com/1b6/61/b661242ec92dae5334dd695b75c7a3ad2f8a028c8cbc59151eec9230cc7d9a45.png)

这个SAPI就是给命令行用的。 

另外，PHP5.4及以后，我们可以用 php -S localhost:8080 来启动PHP内置的Web server。这个Web server其实也是一个SAPI，名为“Built-in HTTP server”：图7。 

图7:

![PHP](https://file.zsxq.com/1b2/bb/b2bbf2fcb9f5d5088f55dc290570ba84f8c0cc86af9786bbaa5b02b77372fdb8.png) 

Built-in Server相当于实现了一个Web文件服务器，然后将PHP有关的请求发给“Built-in HTTP server”这个SAPI，最终交给解释器解析。 

在很古老的时候，rfc3875（ [链接：http://www.ietf.org/rfc/rfc3875](http://www.ietf.org/rfc/rfc3875) ）定义了一种运行Web应用的方法——CGI，PHP也为其提供了名为“CGI/FastCGI”的SAPI，不过用的比较少，不介绍了。 

除此之外，还有一个比较有意思的SAPI，叫embed，这是一个嵌入式SAPI。这个功能就很有意思了，我们能很容易地将PHP解释器集成到自己的程序里。比如Laruence在08年曾写过一篇《使用PHP Embed SAPI实现Opcodes查看器》的文章：[链接：](http://www.laruence.com/2008/09/23/539.html)[使用PHP Embed SAPI实现Opcodes查看器 | 风雪之隅](http://www.laruence.com/2008/09/23/539.html) 。 

原理也比较简单，就是在外部调用PHP内核的zend_compile_file函数，将代码转换成OPCODE，然后输出。

 uwsgi的PHP插件，也是利用内嵌embed的PHP，我们可以在 [链接：](https://github.com/vulhub/vulhub/blob/master/base/uwsgi/php/2.0.16/Dockerfile)[vulhub/Dockerfile at master · vulhub/vulhub · GitH...](https://github.com/vulhub/vulhub/blob/master/base/uwsgi/php/2.0.16/Dockerfile) 看到整个编译的过程。 

phpdbg是php5.4及以后加入的一个php交互式调试器，他也是一个sapi。

