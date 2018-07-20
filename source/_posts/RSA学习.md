---
title: RSA学习
date: 2018-07-17 17:21:46
tags:
---

## openssl相关操作

提取pubkey.pem相关信息

```
 openssl rsa -pubin -text -modulus -in warmup -in pubkey.pem
```

解密:

```
openssl rsautl -decrypt -in 【flag.enc】 -inkey 【private.pem】
```

## 已知p,q

若已知质数p和q，则通过依次计算欧拉函数值phi、私钥d可解密。在选取加密指数e时要求phi，e互质，也就是gcd(phi,e)==1 ，如果不满足是无法直接解密的。

```
def rsa_decrypt(e, c, p, q):
    phi = (p - 1) * (q - 1)
    n = p * q
    try:
        d = gmpy2.invert(e, phi) #求e模phi的逆
        return pow(c, d, n)
    except Exception as e:
        print "e and phi are not coprime!"
        raise e
```

## 查询已知的n的可分解情况

在线查询：https://factordb.com/

api接口：

```
curl http://factordb.com/api?query=12345

response:

{"id":"12345","status":"FF","factors":[["3",1],["5",1],["823",1]]}

```

## 使用yafu分解N

适用情况：p,q相差较大或较小时可快速分解。

使用方法：yafu-x64.exe factor(233) ，yafu-x64.exe help

url: ```https://www.cnblogs.com/pcat/p/7508205.html```

## 模不互素 （gcd(N1,N2)!=1）

适用情况：存在两个或更多模数 ，且gcd(N1,N2)!=1 。

多个模数n共用质数，则可以很容易利用欧几里得算法求得他们的质因数之一gcd(N1,N2) ，然后这个最大公约数可用于分解模数分别得到对应的p和q，即可进行解密。实现参照本文欧几里得算法 部分和RSA解密 部分。

##  共模攻击

适用情况：明文m、模数n相同，公钥指数e、密文c不同，gcd(e1,e2)==1

```python
# -*- coding: utf-8 -*-
# by https://findneo.github.io/

import base64
import libnum
import gmpy2


def fix_py():
    # decode encryption.encrypted
    s1 = 'abdefghijklmpqrtuvwxyz'
    s2 = 'dmenwfoxgpyhirasbktclu'
    f1 = open('encryption.encrypted')
    with open('encryption.py', 'w') as f2:
        for i in f1.readlines():
            tmp = ''
            for j in i:
                tmp += s2[s1.index(j)] if j in s1 else j
            f2.write(tmp)
# fix_py()
def common_modulus(n, e1, e2, c1, c2):
    assert (libnum.gcd(e1, e2) == 1)
    _, s1, s2 = gmpy2.gcdext(e1, e2)
    m = pow(c1, s1, n) if s1 > 0 else pow(gmpy2.invert(c1, n), -s1, n)
    m *= pow(c2, s2, n) if s2 > 0 else pow(gmpy2.invert(c2, n), -s2, n)
    m %= n
    return m


[n2, n3] = map(lambda x: int(base64.b64decode(x).encode('hex'), 16),
               open('n2&n3').readlines())
[n1c1, n1c2] = map(lambda x: int(x, 16), open('n1.encrypted').readlines())
[msg1c1, msg2c2] = map(lambda x: int(x, 16), open('ciphertext').readlines())
# 通过共模攻击得到n1
e1 = 0x1001
e2 = 0x101
n1 = common_modulus(n3, e1, e2, n1c1, n1c2)
# n1,n2有一个共有质因数p1
# n1 += n3  # 存在n3比n1小的可能，并且确实如此;貌似主办方中途改题，把n1改成小于n3了。
p1 = gmpy2.gcd(n1, n2)
assert (p1 != 1)
p2 = n1 / p1
p3 = n2 / p1
e = 0x1001
d1 = gmpy2.invert(e, (p1 - 1) * (p2 - 1))
d2 = gmpy2.invert(e, (p1 - 1) * (p3 - 1))
msg1 = pow(msg1c1, d1, n1)
msg2 = pow(msg2c2, d2, n2)
msg1 = hex(msg1)[2:].decode('hex')
msg2 = hex(msg2)[2:].decode('hex')
print msg1, msg2
# XA{RP0I_0Itrsigi s.y
# MNCYT_55_neetnvmrap}
# XMAN{CRYPT0_I5_50_Interestingvim rsa.py}
```

## 小明文攻击

适用情况：e较小，一般为3。

公钥e很小，明文m也不大的话，于是m^e=k*n+m 中的的k值很小甚至为0，爆破k或直接开三次方即可。

```python
def small_msg(e, n, c):
    print time.asctime(), "Let's waiting..."
    for k in xrange(200000000):
        if gmpy2.iroot(c + n * k, e)[1] == 1:
            print time.asctime(), "...done!"
            return gmpy2.iroot(c + n * k, 3)[0]
```

```python
import gmpy2,binascii,libnum,time
n=0xB0BEE5E3E9E5A7E8D00B493355C618FC8C7D7D03B82E409951C182F398DEE3104580E7BA70D383AE5311475656E8A964D380CB157F48C951ADFA65DB0B122CA40E42FA709189B719A4F0D746E2F6069BAF11CEBD650F14B93C977352FD13B1EEA6D6E1DA775502ABFF89D3A8B3615FD0DB49B88A976BC20568489284E181F6F11E270891C8EF80017BAD238E363039A458470F1749101BC29949D3A4F4038D463938851579C7525A69984F15B5667F34209B70EB261136947FA123E549DFFF00601883AFD936FE411E006E4E93D1A00B0FEA541BBFC8C5186CB6220503A94B2413110D640C77EA54BA3220FC8F4CC6CE77151E29B3E06578C478BD1BEBE04589EF9A197F6F806DB8B3ECD826CAD24F5324CCDEC6E8FEAD2C2150068602C8DCDC59402CCAC9424B790048CCDD9327068095EFA010B7F196C74BA8C37B128F9E1411751633F78B7B9E56F71F77A1B4DAAD3FC54B5E7EF935D9A72FB176759765522B4BBC02E314D5C06B64D5054B7B096C601236E6CCF45B5E611C805D335DBAB0C35D226CC208D8CE4736BA39A0354426FAE006C7FE52D5267DCFB9C3884F51FDDFDF4A9794BCFE0E1557113749E6C8EF421DBA263AFF68739CE00ED80FD0022EF92D3488F76DEB62BDEF7BEA6026F22A1D25AA2A92D124414A8021FE0C174B9803E6BB5FAD75E186A946A17280770F1243F4387446CCCEB2222A965CC30B3929
e=3
res=0
c=int(open('extremelyhardRSA.rar/flag.enc','rb').read().encode('hex'),16)
print time.asctime()
for i in xrange(200000000):
    if gmpy2.iroot(c+n*i,3)[1]==1:
        res=gmpy2.iroot(c+n*i,3)[0]
        print i,res
        print libnum.n2s(res)
        print time.asctime()
        break
```

## Rabin加密中的N可被分解

```
def rabin_decrypt(c, p, q, e=2):
    n = p * q
    mp = pow(c, (p + 1) / 4, p)
    mq = pow(c, (q + 1) / 4, q)
    yp = gmpy2.invert(p, q)
    yq = gmpy2.invert(q, p)
    r = (yp * p * mq + yq * q * mp) % n
    rr = n - r
    s = (yp * p * mq - yq * q * mp) % n
    ss = n - s
    return (r, rr, s, ss)
```

## Wiener's Attack(低解密指数攻击 )

```python
from Crypto.PublicKey import RSA
import ContinuedFractions, Arithmetic

def wiener_hack(e, n):
    # firstly git clone https://github.com/pablocelayes/rsa-wiener-attack.git !
    frac = ContinuedFractions.rational_to_contfrac(e, n)
    convergents = ContinuedFractions.convergents_from_contfrac(frac)
    for (k, d) in convergents:
        if k != 0 and (e * d - 1) % k == 0:
            phi = (e * d - 1) // k
            s = n - phi + 1
            discr = s * s - 4 * n
            if (discr >= 0):
                t = Arithmetic.is_perfect_square(discr)
                if t != -1 and (s + t) % 2 == 0:
                    print("Hacked!")
                    return d
    return False
```

## 私钥文件修复

