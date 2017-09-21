---
title: Miller-Rabin primality test
date: 2017-09-19
tags:
- algorithm
- math
categories:
- algorithm
---

利用随机算法判断一个数是合数还是素数。

<!-- more -->

## 大数求模的快速算法

计算 $$a^b \mod n$$，将 $$b$$ 拆分为二进制再进行计算：

当 $$b_{i} = 0$$ 时，$$a^{2c} \mod n = (a^c)^2 \mod n$$

当 $$b_{i} = 1$$ 时， $$a^{2c + 1} \mod n = a * (a^c)^2 \mod n$$

其中，$$c$$ 为 $$b_{i-1}b_{i-2}...b_{0}$$， $$b_{i}$$ 为 $$b$$ 二进制表示第 $$i$$ 位的值，当 $$c$$ 成倍增长时，$$d = a^c \mod n$$ 不变，直至 $$c = b$$

记 $$d \equiv a^b (\mod n)$$
``` python
def mod_exp(a, b, n):
    d = 1
    while b > 0:
        if b & 1 == 1:
            d = d * a % n
        b >>= 1
        t = (a * a) % n
    return d
```

## 二次探测定理

如果 $$n$$ 是素数，$$a$$ 为小于 $$n$$ 的正整数，则 $$a^2 \equiv 1 (\mod n)$$ 的解为 $$a=1$$ 或 $$a=n-1$$

设 $$n$$ 为奇素数且 $$n > 2$$，则有 $$n - 1 = 2^s * d$$，$$s$$ 和 $$d$$ 都为正整数且 $$d$$ 为奇数，对任意 $$(Z/nZ)^*$$ 范围内的 $$a$$ 和 $$0 \le r \le s - 1$$，必满足以下两种形式的一种：

$$a^d \equiv 1 (\mod n)$$

$$a^{2^rd} \equiv -1 (\mod n)$$

费马小定理，当 $$n$$ 为素数，$$a^{n-1} \equiv 1 (\mod n)$$

根据上述引理，不断对 $$a^{n-1}$$ 取平方根，总会得到 1 或 -1，这是 $$n$$ 为素数的必要非充分条件。我们通过多次选取不同的 $$a$$，得到多个“强伪证”去认为 $$n$$ 有很大概率为素数

``` python
def twice_detect(a, b, k):
    t = 0
    while b & 1 == 0:
        b >>= 1
        t += 1

    y = x = mod_exp(a, b, k)
    for _ in xrange(t):
        y = mod_exp(x, 2, k)
        if (y == 1 and x != 1 and x != k - 1):
            return 0
        x = y
    return y
```

## Miller Rabin Test

``` python
def miller_rabin(n):
    for a in prime_list:
        if n == a:
            return True
        if twice_detect(a, n-1, n) != 1:
            return False
    return True
```
