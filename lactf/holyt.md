# LA CTF crypto/hOlyT
If you're coming here from my misc/gacha writeup, this one will probably be less degen. The challenge prompt isn't that relevant, but we are given a file called ``server.py``. We're also given an address to ``nc``.

Connecting with ``nc`` gives an interface where we're shown the following information:
```
ct = 7795478703951910298760800512251016604742954197898050479871934501928567145784802604542002268080404992780678217646124951840673797270491710309688112924721561151731605873112953711785868170966477869814463719384874577110299248963713396172759754647405773964179242806311505685143222093888093347835985735639754440152848146516109884968142336041568834166265316368983280221226951298952481486521619997411247561491239782604374151098554182647251791561517307089804991475650432858727962467022168414859936296189874338448996484184334288550871046087698350678567999518807001116688026839842507335016985102010047274042512865191879615627189
N = 17831104667040256134725887251427780309283441190346378241363820041577227275015801947889351608323543022862201231205425355290220006633567994093510271341296137236696435486220210052887153794743228135603531011013817226919624495642090379006823492988501642462491665647285840701843758122419384518395935175111969431981176852184961902749819593573017708973408100060762005631607410042285600792954074826599343773206465846004997125247332792765111462262606314067259767823386343138746280621268625953047895333695773948796251064112811522554255832429054369092856990234149099315668149570501794323907590910167206485076585099237050507816159
e = 65537
```
So these are very long, and we'll take a look at ``server.py`` for the specifics on these numbers.

Within ``server.py``, there are several important functions. We'll start from the top down.

First, there's a function called ``legendre()``:
```py
def legendre(a, p):
    return pow(a, (p - 1) // 2, p)
```
This one's pretty straightforward: it returns $a^{\frac{p-1}{2}} \mod p$.

The next function is much less kind to us. If you'd like to see the implementation, here it is:
```py
def tonelli(n, p):
    q = p - 1
    s = 0
    while q % 2 == 0:
        q //= 2
        s += 1
    if s == 1:
        return pow(n, (p + 1) // 4, p)
    for z in range(2, p):
        if p - 1 == legendre(z, p):
            break
    c = pow(z, q, p)
    r = pow(n, (q + 1) // 2, p)
    t = pow(n, q, p)
    m = s
    t2 = 0
    while (t - 1) % p != 0:
        t2 = (t * t) % p
        for i in range(1, m):
            if (t2 - 1) % p == 0:
                break
            t2 = (t2 * t2) % p
        b = pow(c, 1 << (m - i - 1), p)
        r = (r * b) % p
        c = (b * b) % p
        t = (t * c) % p
        m = i
    return r
```
The purpose of this function isn't immediately clear. However, with Professor ChatGPT on our side, there's nothing we can't conquer. It tells us that this function is something known as the Tonelli-Shanks Algorithm, which returns an $r$ such that $r^2 \equiv n \mod p$. I am inclined to believe it, mostly because I don't want to trawl through this mess of powers and moduli.

The next function is ``xgcd()``, which is fairly straightforward this time: it's just Extended Euclidean Algorithm.
```py
def xgcd(a, b):
    if a == 0 :
        return 0,1

    x1,y1 = xgcd(b%a, a)
    x = y1 - (b//a) * x1
    y = x1

    return x,y
```
This returns $x,y$ such that $ax + by = \gcd(a,b)$. However, we also see that this is just an auxiliary function to ``crt()``, given below:
```py
def crt(a, b, m, n):
    m1, n1 = xgcd(m, n)
    return ((b *m * m1 + a *n*n1) % (m * n))
```
This is just Chinese Remainder Theorem, which returns a value $y$ such that $y \equiv a \mod m$ and $y \equiv b \mod n$. 

Now let's skip to the ``main()`` function:
```py
def main():
    p = getPrime(1024)
    q = getPrime(1024)
    N = p * q
    e = 65537
    m = bytes_to_long(b"lactf{redacted?}")
    ct = pow(m, e, N)
    print(f"ct = {ct}")
    print(f"N = {N}")
    print(f"e = {e}")
    while 1:
        x = int(input("What do you want to ask? > "))
        ad = advice(x, p, q)
        print(ad)
```
So it's generating two primes $p$ and $q$ and encrypting a message $m$ with RSA. We're given the ciphertext ``ct`` as well as the public key and the modulus. Then, it loops forever, getting an integer from user input and passing it to the ``advice()`` function with $p$ and $q$. Let's finally take a look at that one:
```py
def advice(x, p, q):
    if legendre(x, p) != 1:
        exit()
    if legendre(x, q) != 1:
        exit()
    x1 = tonelli(x, p) * random.choice([1, -1])
    x2 = tonelli(x, q) * random.choice([1, -1])
    y = crt(x1, x2, p, q)
    return y
```
First, this function checks if $x^{\frac{p-1}{2}} \mod p$ or $x^{\frac{q-1}{2}} \mod q$ is 1; if not, then the function exits the program. In other words, we need to find numbers that satisfy this condition in order for ``advice()`` to be of use to us. After this, it'll find the ``tonelli()`` of $x$ and $p$, as well as $x$ and $q$ and randomly choose if it's positive or negative, before returning a value y calculated from the ``tonelli()`` values and the two primes.

At this point I was getting walled and I didn't know how to proceed, since this looked more than a little bit intimidating. So I took a four hour break.

When I returned, I noticed that since $p$ is prime, all perfect squares not divisible by $p$ pass the ``legendre()`` check by Euler's theorem. Therefore, an input of 1 may be the best way to get something out of ``advice()``, as the Tonelli algorithm would return 1, and thus we would be given $y$ such that $y \equiv \pm 1 \mod p,q$!

However, I then went down the entirely wrong path. I had the stairway to Heaven laid out to me, and I decided to instead hang a sharp left and walk straight into hell.

My team and I made the mistake of trying to figure out what other inputs would pass the ``legendre()`` test instead of analyzing the outputs that I was getting from inputting 1. We found that there was a certain list of primes that, when inputted to the ``legendre()``, would evaluate to -1, and another list that evaluated to 1. Therefore, an even number of primes from the first list multiplied by any number of primes from the second list would be inputs that returned something. Yet this reveals nothing about the actual nature of $p$ and $q$, so I decided to stare at the paper that I was working on for, another 2 hours. Then, I noticed something that helped me down the final stretch.

If you repeatedly input 1, it'll cycle between 4 values. One of them is 1, a second is $N - 1$. This is expected: if $y \equiv 1 \mod p,q$ or $y \equiv -1 \mod p,q$, then $y = 1$ or $y = pq - 1$ respectively. However, the other two numbers that are output are more interesting. They are as follows:
```
y1 = 12543327065801962610744075756846559617980246552922417100872603818493798966059706703055149059011972141365311478492748987671188435357925094775013208156046315891049104385142491732801619782976653736068904735144419446163489344131703025119761873944419880125761507869394862188182720955132188347678851544113421703302734727531242579531323262149507434400369268532075038572395448927911344696402028869425886723564234382595072991003810126823599461424382689088309859000473754968901328087419516027916743538080683918219102060429467560165753443642775973833304701656372325574307951794081942686364095841177702997728532810737341461151015
y2 = 5287777601238293523981811494581220691303194637423961140491216223083428308956095244834202549311570881496889752712676367619031571275642899318497063185249821345647331101077718320085534011766574399534626275869397780756135151510387353887061619044081762336730157777890978513661037167287196170717083630998547728678442124653719323218496331423510274573038831528686967059211961114374256096552045957173457049642231463409924134243522665941512000838223624978949908822912588169844952533849109925131151795615090030577149003683343962388502388786278395259552288577776773741360197776419851637543495068989503487348052288499709046665144
```
These are thus the other cases, where $y$ is congruent to different values mod $p,q$. Suppose $y1 \equiv 1 \mod p, -1 \mod q$ and $y2 \equiv -1 \mod p, 1 \mod q$. Therefore, it should follow that $\gcd(y1 - 1, y2 + 1) = p$, $\gcd(y2 - 1, y1 + 1) = q$. Plugging this into a calculator from <a href="https://www.dcode.fr"> dcode.fr </a> gives
```
p = 145451865620627255994423794806361945850102334492275287211990843130611262300325394155606209563629002979721763569246337055099205285966131382849487976636227400055027719431442836443745777646690258980901740967619221322690271390697990423858822096771947480132916085867119261426118850706911431405002249943301157526101
q = 122591103187001941531981116626088517272837042666009702021914587210570496297428345381271558003032110048175099726506196684469849960666768621613342548101515187188959282532721988792848139744093597910688850693268776677926655073601748929692187744665915785021760858938755364467804579756425006253062877958984156956259
```
Now that we have $p$ and $q$ this is just standard RSA decryption. I ran the following code:
```py
tot = math.lcm(p-1, q-1)
d = pow(e, -1, tot)
m = pow(ct, d, n)
print(long_to_bytes(m).decode())
```
and got the flag: ``lactf{1s_r4bin_g0d?}``