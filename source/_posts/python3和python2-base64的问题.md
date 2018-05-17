---
title: python3和python2 base64的问题
date: 2018-03-26 17:12:56
tags:
- python
---
首先我们来看两段代码：

python2.7:
<pre>
text = 'system'
text1 = ''
for i in text:
    text1 += chr(ord(i) +10)
print base64.b64encode(text1)
</pre>
<pre>
结果:
fYN9fm93
</pre>

python3.5:
<pre>
text = 'system'
text1 = ''
for i in text:
    text1 += (chr(ord(i) +10))
print(base64.b64encode(str.encode(text1)))
</pre>
<pre>
结果:
b'fcKDfX5vdw=='
</pre>

我们可以发现，两次base64的结果并不相同。

原因:

我们可以将text1打印出来:

python2.7:
<pre>
text = 'system'
text1 = ''
for i in text:
    text1 += chr(ord(i) +10)
print text1
</pre>

<pre>
结果:
}儅~ow
</pre>

python3.5:
<pre>
text = 'system'
text1 = ''
for i in text:
    text1 += chr(ord(i) +10)
print(str.encode(text1))
</pre>

<pre>
结果:
b'}\xc2\x83}~ow'
</pre>

在经过
```
chr(ord(i) +10)
```
操作之后，text1的ascii码变为:

```
125,131,125,126,111,109
```
对应16进制

```
\x7d,\x83,\x7d,\x7e,\x6f,\x6d
```
我们可以发现，在python3.5的情况下，\x83变成了 \xc2\x83，这是因为str.encode默认采用utf-8编码来解码text1。

解决方法:
在python3.5下，我们可以使用bytesarray
<pre>
text = 'system'
text1 =  bytearray()
for i in text:
    text1.append(ord(i) + 10)
print(text1)
print(base64.b64encode(text1))
</pre>
<pre>
结果:
bytearray(b'}\x83}~ow')
b'fYN9fm93'
</pre>





















































