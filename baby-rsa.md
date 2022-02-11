## Baby RSA

GOAL: Given a system incorrectly generated RSA keys along with the generated ciphertext and public key, find the flag.

VERBAGE: “I messed up prime generation, and now my private key doesn't work!”

CODE (given):
```
from Crypto.Util.number import getPrime, bytes_to_long, long_to_bytes
def getAnnoyingPrime(nbits, e):
	while True:
		p = getPrime(nbits)
		if (p-1) % e**2 == 0:
			return p
nbits = 128
e = 17
p = getAnnoyingPrime(nbits, e)
q = getAnnoyingPrime(nbits, e)
flag = b"dice{???????????????????????}"
N = p * q
cipher = pow(bytes_to_long(flag), e, N)
print(f"N = {N}")
print(f"e = {e}")
print(f"cipher = {cipher}")
```

DATA (given):
```
C = 19441066986971115501070184268860318480501957407683654861466353590162062492971
e = 17
N = 57996511214023134147551927572747727074259762800050285360155793732008227782157
```


### Attempt 1 - use CCA

Since C = m^e (mod N), we can use r coprime with N and find r^(-1) to compute the following:

```
m’ = C * r^e * r^(-1) (mod N)
```

After trying this, it was unsuccessful since m’^e (mod N) does not equal the given C.


### Attempt 2 - find phi(N) using computing power to factor N -> compute private key

After using https://www.alpertron.com.ar/ECM.HTM to compute the factors of N, we successfully receive p and q as follows:

```
p = 172036442175296373253148927105725488217
q = 337117592532677714973555912658569668821
-> phi(N) = 57996511214023134147551927572747727073750608765342311271929088892243932625120
```

However, we realized phi(N) and e are not actually co-prime; hence, we cannot find the private key using traditional RSA because it was incorrectly generated.


### Attempt 3 - find the eth root

We thought maybe this could work since e is relatively small; however, the 17th root of C proved otherwise and led us nowhere since the number is too small to reveal the flag.


### Attempt 4 - using https://eprint.iacr.org/2020/1059.pdf 

Solve for phi_prime from dividing (p - 1)(q - 1) by e^4 since then gcd(phi_prime, e) = 1

```
phi_prime = 694394358472996421828664977343994050283768259064694044275440774083690720

p - 1 = e^2 * 595281806834935547588750612822579544 = e^2 * a
q - 1 = e^2 * 1166496859974663373610920113005431380 = e^2 * b
```

Trying to find the generator ```ge```:
```
g = 1
while True:
	g += 1
	ge = pow(g,phi_prime,N)
	if (ge != 1):
		break
print(ge)
```

However, no luck as this was running overnight and found nothing :(


### Attempt 5: Using this random tool from github

https://github.com/Ganapati/RsaCtfTool Note -  to get this to work you might need to uninstall and reinstall pycryptodome

Part of the message is known: ```flag = b"dice{???????????????????????}"```


Also maybe some a helpful resource for later: https://bitsdeep.com/posts/attacking-rsa-for-fun-and-ctf-points-part-1/
No luck with this guy either :(


### Attempt 6: Chinese Remainder Theorem

```
for x in range(1,p):
	if (pow(x,e,p) == C):
		print(x)

for y in range(1,q):
	if (pow(y,e,q) == C):
		print(y)
```

```C = x^e (mod p) and C = y^e (mod q)``` => m belongs to (x, y) where ```m = [q^(-1)(x - y) (mod p)]*q + y``` for some combo (x,y)

Code running… ```O(p) + O(q)``` -> in parallel ```max[O(p),O(q)]```

### Solution:

It turns out Sage has a built-in function ```nth_root``` that will retrieve such values without having to brute force them as described above with:

```
x = mod(cipher, p).nth_root(e, all=True) 
y = mod(cipher, q).nth_root(e, all=True)
```

x belongs to:

```
[94911252969483866512421216036175458459, 34738964150369315985690946219467529103, 144682460619756215222573932718288677574, 67563709743199931906521216375861736979, 120545244718334586693221558997648386455, 118213898942703354093543288836971676192, 128772705163182625631145329538392926939, 38708160654448268379983220297515882584, 36018877687403393827298757143339399960, 92970575832859594829754423641400360880, 124505686459227230067098278775453749813, 154762621876285543324659082322173822945, 10561998762116831527413059924606880197, 62110596170914033783530021092342334411, 82666927438416091048548053110384074632, 61262203689918592094867675087144220311, 3295652523751511096921356728636788302]
```

y belongs to:

```
[123051145405638103088362493325897286903, 120843497057329541955688325959441277291, 78637092053158916276190016971186129373, 191086485276595526784443510797142963252, 15712012687353508531014491012266887133, 255611288271724031432785750385992714047, 328249329163672772755604630559717300145, 237794584614487078176237333216459698226, 120209596880161384728992670472630166695, 42136900506030290620627812532051186078, 38232720319758305698078599656102519970, 305890500690536701236656282504903273821, 33039639366451899917550980181939511805, 8559227908703876401395512426625289245, 175190232418930946042340547709045502374, 54489953135730405848414192579838163591, 231088941972480715320508238318747811798]
```

Now that we have these, we iterate through each one until long_to_bytes contains ```b“dice{“``` to retrieve the flag after passing pairs into the built-in Chinese Remainder Theorem function with ```crt([Integer(x), Integer(y)], [p,q])```: ```dice{cado-and-sage-say-hello}```
