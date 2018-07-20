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

## LSB Oracle Attack

适用情况：可以选择密文并泄露最低位。

```python
import decimal
def oracle():
    return lsb == 'odd'


def partial(c, e, n):
    k = n.bit_length()
    decimal.getcontext().prec = k  # for 'precise enough' floats
    lo = decimal.Decimal(0)
    hi = decimal.Decimal(n)
    for i in range(k):
        if not oracle(c):
            hi = (lo + hi) / 2
        else:
            lo = (lo + hi) / 2
        c = (c * pow(2, e, n)) % n
        # print i, int(hi - lo)
    return int(hi)
```

## 选择密文攻击

选择密文攻击
适用情况：可以构造任意密文并获得对应明文。

这个好理解，在一个RSA加密过程中，明文为m，密文为c，模数为n，加密指数为e，选取x以满足`gcd(x,n)==1` 从而使x模n的逆存在，构造密文 `c'=c*(x^e) `使解密后明文为 `m'=(m*x)%n` ，则`m=m'*x^-1(mod n)` 。可参看模意义下的运算法则部分 。

## 广播攻击

适用情况：模数n、密文c不同，明文m、加密指数e相同。一般会是e=k，然后给k组数据

使用不同的模数n，相同的公钥指数e加密相同的信息。就会得到多个`(m^e) ==ci (mod ni)`，将(m^e)视为一个整体M，这就是典型的中国剩余定理适用情况。按照本文的中国剩余定理小节容易求得m^e的值，当e较小时直接开e方即可，可使用`gmpy2.iroot(M,e)` 方法。

案例:

```python
def CRT(mi, ai):
    # mi,ai分别表示模数和取模后的值,都为列表结构
    # Chinese Remainder Theorem
    # lcm=lambda x , y:x*y/gcd(x,y)
    # mul=lambda x , y:x*y
    # assert(reduce(mul,mi)==reduce(lcm,mi))
    # 以上可用于保证mi两两互质
    assert (isinstance(mi, list) and isinstance(ai, list))
    M = reduce(lambda x, y: x * y, mi)
    ai_ti_Mi = [a * (M / m) * gmpy2.invert(M / m, m) for (m, a) in zip(mi, ai)]
    return reduce(lambda x, y: x + y, ai_ti_Mi) % M

m = random.randint(0x100000000000, 0xffffffffffff)
e = 3
n1 = 0x43d819a4caf16806e1c540fd7c0e51a96a6dfdbe68735a5fd99a468825e5ee55c4087106f7d1f91e10d50df1f2082f0f32bb82f398134b0b8758353bdabc5ba2817f4e6e0786e176686b2e75a7c47d073f346d6adb2684a9d28b658dddc75b3c5d10a22a3e85c6c12549d0ce7577e79a068405d3904f3f6b9cc408c4cd8595bf67fe672474e0b94dc99072caaa4f866fc6c3feddc74f10d6a0fb31864f52adef71649684f1a72c910ec5ca7909cc10aef85d43a57ec91f096a2d4794299e967fcd5add6e9cfb5baf7751387e24b93dbc1f37315ce573dc063ecddd4ae6fb9127307cfc80a037e7ff5c40a5f7590c8b2f5bd06dd392fbc51e5d059cffbcb85555L
n2 = 0x60d175fdb0a96eca160fb0cbf8bad1a14dd680d353a7b3bc77e620437da70fd9153f7609efde652b825c4ae7f25decf14a3c8240ea8c5892003f1430cc88b0ded9dae12ebffc6b23632ac530ac4ae23fbffb7cfe431ff3d802f5a54ab76257a86aeec1cf47d482fec970fc27c5b376fbf2cf993270bba9b78174395de3346d4e221d1eafdb8eecc8edb953d1ccaa5fc250aed83b3a458f9e9d947c4b01a6e72ce4fee37e77faaf5597d780ad5f0a7623edb08ce76264f72c3ff17afc932f5812b10692bcc941a18b6f3904ca31d038baf3fc1968d1cc0588a656d0c53cd5c89cedba8a5230956af2170554d27f524c2027adce84fd4d0e018dc88ca4d5d26867L
n3 = 0x280f992dd63fcabdcb739f52c5ed1887e720cbfe73153adf5405819396b28cb54423d196600cce76c8554cd963281fc4b153e3b257e96d091e5d99567dd1fa9ace52511ace4da407f5269e71b1b13822316d751e788dc935d63916075530d7fb89cbec9b02c01aef19c39b4ecaa1f7fe2faf990aa938eb89730eda30558e669da5459ed96f1463a983443187359c07fba8e97024452087b410c9ac1e39ed1c74f380fd29ebdd28618d60c36e6973fc87c066cae05e9e270b5ac25ea5ca0bac5948de0263d8cc89d91c4b574202e71811d0ddf1ed23c1bc35f3a042aac6a0bdf32d37dede3536f70c257aafb4cfbe3370cd7b4187c023c35671de3888a1ed1303L
c1 = pow(m, e, n1)
c2 = pow(m, e, n2)
c3 = pow(m, e, n3)
print m == gmpy2.iroot(CRT([n1, n2, n3], [c1, c2, c3]), e)[0]
```

