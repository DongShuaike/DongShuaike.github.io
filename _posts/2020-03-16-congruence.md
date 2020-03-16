---
layout: post
title:  "From congruence to Euclidean, CRT and CRT-optimized RSA(part 1)"
date:   2020-03-09 13:56:45 +0800
categories: math crypto
---
## From congruence to Extended Euclidean, CRT and CRT-optimized RSA (part 1)

Recently I was taking the role of TA as a software security course. Part One of it focuses on standard cryptos. Instead of just showing the textbook crypto, we dive into how real crypto system is built and practical attacks around it. The course needs a lot of crypto background knowledge and I spend some time understanding them.

In this blog I dive into some number theory backgrounds and they are pretty important in understanding RSA algorithm.

The first concept is congruence. When $A$ and $B$ have the same result doing modular arithmetic, we say $A$ **is congruent to** $B$. The mathematical way of representing this is $A \equiv B\mod n$.
Note that $\equiv$ is to describe the equivalence relation between two elements. We usually pair an operation (in this case, the operation is "mod") with it. Therefore, we can say 
> $1 \equiv 3$ is true with the equivalence relation "has the same remainder upon division by 2"
> -- <cite> [Qiaochu Yuan][1]</cite>

[1]:https://math.stackexchange.com/questions/1058596/in-plain-language-whats-the-difference-between-two-things-that-are-equivalent 

From the outside looking we may infer congruence has a lot of in common with equality. Actually it is. Congruence has a lot of properties:

1. if we have $ a \equiv b \mod m $ and $c \equiv d \mod m$, we then hold following relations:

(1) $a+c \equiv b+d(\mod m)$

(2) $a-c \equiv b-d(\mod m)$

(3) $ac \equiv bd(\mod m)$

(4) $a^n \equiv b^n(\mod m)$

These can be easily proved from the definition of congruence: if $a \equiv b \mod m$, then $a = k*m+b$.

2. We cannot directly transfer division property of equality into congruence, which means:

If we have $ak \equiv bk \mod m$ where $k\neq 0$, we **cannot** get $a \equiv b \mod m$.

Actually when we are dealing with division in congruence arithmetic, we need to find the **modular inverse**. For $a \equiv b mod m$, we use $a^{-1}$ to be $a$'s modular inverse, then we have the following relation:

$aa^{-1}\equiv 1\mod m$.

The modular inverse seems not easy to understand, why do we use such concept?

As we know, for rational numbers, division is the inverse operation of multiplication. However, for congruence, the inverse operation of multiplication is ***modular inverse***, which means, if you want to calculate $\frac{a}{b} \mod m$, you should use $a*b^{-1} \mod m$ instead. Since $b^{-1}$ only exists when $b$ satisfies some conditions, we know getting $a \equiv b \mod m$ from $ak \equiv bk \mod m$ does not always hold.

When does it hold?
The answer is "when $gcd(k,m)=1$. 

The modular inverse plays a big role in RSA crypto. How to calculate it? We can use ***Extended Euclidean*** algorithm.
As we all know, the Euclidean algorithm works like below:
```
def gcd(a,b):
  if b == 0:
    return a
  return gcd(b, a%b)
```
So what is Extended Euclidean algorithm? It is used to solve a equation of two unknowns like below:
$ax+by = gcd(a,b)$ [1]
Using EEuc we can get $x$, $y$ and $gcd(a,b)$ at the same time. (There will be infinite solutions $x$ and $y$) How does it work?
From the definition of `gcd` function above, we know before `b` becoming zero, we have `gcd(a,b) == gcd(b,a%b)`, therefore, we use `b` and `a%b` to replace `a` and `b` in the above equation and get:
$bx+(a\%b)y = gcd(b,a\%b) = gcd(a,b)$ [2]
From the definition of modular arithmetic, we have $a\%b = a-\lfloor \frac{a}{b}\rfloor\times b$, taking it into the above equation we get $(x-\lfloor \frac{a}{b}\rfloor y)b+ya = gcd(a,b) = xa+yb$

Yep, since the extended euclidean algorithm is actually an iterative algorithm, therefore we use the word $x_1$ and $y_1$ to represent the $x$ and $y$ in [2] then we have
$y_1 = x_0$
$x_1 = y_0+\lfloor\frac{a}{b}\rfloor \times y_1$
Now we know how to write the `ext_euc` function:
Let's assume it takes 2 parameters `a` and `b` and will return three values `x`, `y` and `gcd(a,b)`.
The code would be like:
```
def ext_euc(a,b):
  if b == 0:
    x, y = 1, 0
    return x, y, a
  x, y, gcd = ext_euc(b, a%b)
  ny = x
  nx = y + a//b*ny
  return nx, ny, gcd
```
If you don't understand the above code, draw a recursive tree and you will get it.

<!--The common usage of Extended Euclidean algorithm is to calculate the modular inverse. Suppose we want to get the modular inverse of $b$, we need to find $b^{-1}$ such that $bb^{-1} \equiv 1 \mod n$. If we make $x=0$ then we have $by = gcd(a,b)$. Since the modular inverse of $b$ only exists when $gcd(b,n) = 1$, let $a=n$, the $by=gcd(a,b)$ will turn to $by=gcd(b,n)=1$, and $y$ is the modular inverse of $b$, which can be solved by Extended Euclidean algorithm, just invoke `ext_euc(n,b)`-->
The common usage of Extended Euclidean algorithm is to calculate the modular inverse. Suppose we want to calculate the modular inverse of $b$, $b^{-1}$ will only exist when $gcd(a,b)=1$. Take it into $ax+by=gcd(a,b)$ we have $ax+by=1$, make both sides of the equation mod $a$, we have $by=1 \mod a$, therefore we get $y$, which is the modular inverse of $b$.

In the second part, I will introduce what CRT is and how CRT makes RSA faster.





