# crypto/Golden Ticket
The challenge is as follows:
```py
from Crypto.Util.number import *

#Some magic from Willy Wonka
def chocolate_generator(m:int) -> int:
    p = 396430433566694153228963024068183195900644000015629930982017434859080008533624204265038366113052353086248115602503012179807206251960510130759852727353283868788493357310003786807
    return (pow(13, m, p) + pow(37, m, p)) % p

#The golden ticket is hiding inside chocolate
flag = b"cyber{REDACTED}"
golden_ticket = bytes_to_long(flag)
flag_chocolate = chocolate_generator(golden_ticket)
chocolate_bag = []

#Willy Wonka is making chocolates
for i in range(golden_ticket):
    chocolate_bag.append(chocolate_generator(i))

#And he put the golden ticket at the end
chocolate_bag.append(flag_chocolate)

#Augustus ate lots of chocolates, but he can't eat all cuz he is full now :D
remain = chocolate_bag[-2:]

#Can you help Charles get the golden ticket?
print(remain)

#[301437878954814449819817577740623092118363353855361857460788950723312968591354868059451924156729631227605213221749853676372571545150239358204099334848100464521903105295653155922, 151389096682556308980806533285515529478197495767854850100329962613139509236892027198420459429204877145098325217414651385702891138933533143909117727398249975367690637972037457052]
```


## Solve Process
Let's call `bytes_to_long(flag)` `nflag` for simplicity. The first observation to make is that the two values in the array are the values `chocolate_generator(nflag-1)` and `chocolate_generator(nflag)`, respectively, which I'll call $c_0$ and $c_1$. Now, by definition of the `chocolate_generator` function, we have 
$$ 
\begin{align}
c_0 &= 13^{\text{nflag} - 1} + 37^{\text{nflag} - 1} \mod p \\
c_1 &= 13^{\text{nflag}} + 37^{\text{nflag}} \mod p
\end{align}
$$
At this point, I took a minute to think about what I was given. I noticed pretty quickly that there was a relationship between $c_0$ and $c_1$, namely, 
$$c_1 = 13c_0 + 24\cdot 37^{\text{nflag}-1} \mod p.$$
However, I was still pretty tentative about this, since $p$ was relatively large, and I still had a power that I wasn't sure how to reverse. Seeing no better way forward, though, I decided to continue down this train of thought. I computed the multiplicative inverse of $24\mod p$ and multiplied that by $c_1 - 13c_0 \mod p$. 

So what do I do about this giant guy now?
```
140351853102935967476854685661863035756294412330395394199432809611396294631438100799843168142777584447445693090821698867891215566136233493372215644447168898948992945862858941358
```

I sat here for like fifteen minutes since I'm sagelocked (why doesn't Fedora 40 have a sagemath package?). If I had Sagemath installed, I would probably have just tried to take the discrete log of that number base 37 and mod p, and let it run for a bit. However, since that wasn't an option, I had to think of something else. Online options are notorious for not being very good, and I couldn't get online Sagemath working.

But since I was done with nearly all of the other crypto challenges from recently, I decided I might as well give it a shot. Googling "discrete log calculator" gave me a result from Alpertron, and I gave it a whirl. Surprisingly, it ran in just 3 seconds, and I was able to shove the result into `long_to_bytes().decode()` to get the flag: `cyber{charles_and_the_chocolate_factory_wow!}`

## Solve Script
```py
# 13^k + 37^k
# 13*13^k + 37*37^k
# 13(13^k + 37^k) + 24(37^k)
from Crypto.Util.number import *
vals = [301437878954814449819817577740623092118363353855361857460788950723312968591354868059451924156729631227605213221749853676372571545150239358204099334848100464521903105295653155922, 151389096682556308980806533285515529478197495767854850100329962613139509236892027198420459429204877145098325217414651385702891138933533143909117727398249975367690637972037457052]

generic = vals[0]
gt = vals[1]
p = 396430433566694153228963024068183195900644000015629930982017434859080008533624204265038366113052353086248115602503012179807206251960510130759852727353283868788493357310003786807


c = gt - (13*generic)%p
print(c % p)
rev = pow(24, -1, p)
power = rev * (c + p) % p
# a = dlog base 37 power mod p found via alpertron
a = '912575 371662 083224 441696 912449 941507 252235 161212 845354 369953 450094 950577 743376 882408 762955 631347 932211 913084'
a=a.replace(' ', '')
num = int(a)
print(num)
print(long_to_bytes(num).decode())
```