espes
======

Background
----------
For this challenge, we were given a username, an address and a RSA public key, with the intent to find the corresponding private key to log in to the server and find the flag.

Tools used
----------
* `base64`
* `hexdump`
* `bc`
* `openssh`
* `ssh-keygen`

Process
-------
Firstly. I needed to understand the ssh-rsa public key format, so I read RFCs [4253](http://tools.ietf.org/html/rfc4253#section-6.6) and [4251](http://tools.ietf.org/html/rfc4251#section-5) and found the part after `ssh-rsa` is base64 encoded binary data in this format:
````
00 00 00 07 73 73 68 2d 72 73 61
<length> <public exponent>
<length> <public modulus>
````
where `<length>` is specified as a 4-byte BE integer.

I extracted these values from the public key:
````
Public Exponent: 0x10001
Public Modulus: 0xfffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffdffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff80000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000001
````

I needed to find the two prime factors for the modulus to work out the private key, and from past knowledge, I remembered that numbers in the form `0xfffff...fffff` multiplied together give a product in the form `0xfffff...fffff00000...00001`, and through a quick few checks, I was right.  
So now I knew that the prime factors of the modulus were probably primes in the form `0xfffff...fffff`, which I also remembered to be called Mersenne primes.  
Fortunately, the number of known Mersenne primes is [very low](http://en.wikipedia.org/wiki/Mersenne_prime#List_of_known_Mersenne_primes) so I did a brute force by hand using `bc` with the Mersenne primes which had near the number of digits the square root of the modulus had. I quickly found the primes to be `2^1279 - 1` and `2^2281 - 1`, shown by the command I used in `bc`:
````
$ echo 'obase=16;(2^1279-1)*(2^2281-1)' | bc
FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF\
FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF\
FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF\
FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF\
FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFDFFFFFFFFFFFFFFFFFFFF\
FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF\
FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF\
FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF\
FFFFFFFFFFFFFFFFFFFFFFFFFF800000000000000000000000000000000000000000\
00000000000000000000000000000000000000000000000000000000000000000000\
00000000000000000000000000000000000000000000000000000000000000000000\
00000000000000000000000000000000000000000000000000000000000000000000\
00000000000000000000000000000000000000000000000000000000000000000000\
000001
````

Now knowing the two prime factors, I needed a way to construct the private key in the appropriate format, which I resorted to hacking `openssl`'s `rsagen` source to use predefined keys:
````diff
--- a/crypto/rsa/rsa_gen.c	2013-09-30 23:06:41.283939601 +1000
+++ b/crypto/rsa/rsa_gen.c	2013-09-29 14:23:09.055065857 +1000
@@ -88,12 +88,12 @@
 		return 0;
 		}
 #endif
-	if(rsa->meth->rsa_keygen)
+	/*if(rsa->meth->rsa_keygen)
 		return rsa->meth->rsa_keygen(rsa, bits, e_value, cb);
 #ifdef OPENSSL_FIPS
 	if (FIPS_mode())
 		return FIPS_rsa_generate_key_ex(rsa, bits, e_value, cb);
-#endif
+#endif*/
 	return rsa_builtin_keygen(rsa, bits, e_value, cb);
 	}
 
@@ -130,50 +130,8 @@
 	BN_copy(rsa->e, e_value);
 
 	/* generate p and q */
-	for (;;)
-		{
-		if(!BN_generate_prime_ex(rsa->p, bitsp, 0, NULL, NULL, cb))
-			goto err;
-		if (!BN_sub(r2,rsa->p,BN_value_one())) goto err;
-		if (!BN_gcd(r1,r2,rsa->e,ctx)) goto err;
-		if (BN_is_one(r1)) break;
-		if(!BN_GENCB_call(cb, 2, n++))
-			goto err;
-		}
-	if(!BN_GENCB_call(cb, 3, 0))
-		goto err;
-	for (;;)
-		{
-		/* When generating ridiculously small keys, we can get stuck
-		 * continually regenerating the same prime values. Check for
-		 * this and bail if it happens 3 times. */
-		unsigned int degenerate = 0;
-		do
-			{
-			if(!BN_generate_prime_ex(rsa->q, bitsq, 0, NULL, NULL, cb))
-				goto err;
-			} while((BN_cmp(rsa->p, rsa->q) == 0) && (++degenerate < 3));
-		if(degenerate == 3)
-			{
-			ok = 0; /* we set our own err */
-			RSAerr(RSA_F_RSA_BUILTIN_KEYGEN,RSA_R_KEY_SIZE_TOO_SMALL);
-			goto err;
-			}
-		if (!BN_sub(r2,rsa->q,BN_value_one())) goto err;
-		if (!BN_gcd(r1,r2,rsa->e,ctx)) goto err;
-		if (BN_is_one(r1))
-			break;
-		if(!BN_GENCB_call(cb, 2, n++))
-			goto err;
-		}
-	if(!BN_GENCB_call(cb, 3, 1))
-		goto err;
-	if (BN_cmp(rsa->p,rsa->q) < 0)
-		{
-		tmp=rsa->p;
-		rsa->p=rsa->q;
-		rsa->q=tmp;
-		}
+	BN_hex2bn(&rsa->p, "7FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF");
+	BN_hex2bn(&rsa->q, "1FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF");
 
 	/* calculate n */
 	if (!BN_mul(rsa->n,rsa->p,rsa->q,ctx)) goto err;
````

Compiling this and then running `./openssl genrsa -out id_rsa 3560` generated this key:
````
-----BEGIN RSA PRIVATE KEY-----
MIIHuAIBAAKCAb4A////////////////////////////////////////////////
////////////////////////////////////////////////////////////////
////////////////////////////////////////////////////////////////
/////////////////////////////////////f//////////////////////////
////////////////////////////////////////////////////////////////
////////////////////////////////////////////////////////////////
////////////gAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAQIDAQABAoIBvSp/1YAqf9WAKn/VgCp/
1YAqf9WAKn/VgCp/1YAqf9WAKn/VgCp/1YAqf9WAKn/VgCp/1YAqf9WAKn/VgCp/
1YAqf9WAKn/VgCp/1YAqf9WAKn/VgCp/1YAqf9WAKn/VgCp/1YAqf9WAKn/VgCp/
1YAqf9WAKn/VgCp/1YAqf9WAKn/VgCp/1YAqf9WAKn/VgCp/1YAqf9WAKn/VgCp/
1X+AgH9/gIB/f4CAf3+AgH9/gIB/f4CAf3+AgH9/gIB/f4CAf3+AgH9/gIB/f4CA
f3+AgH9/gIB/f4CAf3+AgH9/gIB/f4CAf3+AgH9/gIB/f4CAf3+AgH9/gIB/f4CA
f3+AgH9/gIB/f4CAf3+AgH9/gIB/f4CAf3+AgH9/gFX/qgBV/6oAVf+qAFX/qgBV
/6oAVf+qAFX/qgBV/6oAVf+qAFX/qgBV/6oAVf+qAFX/qgBV/6oAVf+qAFX/qgBV
/6oAVf+qAFX/qgBV/6oAVf+qAFX/qgBV/6oAVf+qAFX/qgBV/6oAVf+qAFX/qgBV
/6oAVf+qAFX/qgBV/6oAVf+qAFX/qgBV/6oAVf+qAFX/qgBV/6oAVf+qAFX/qgEC
gaB/////////////////////////////////////////////////////////////
////////////////////////////////////////////////////////////////
////////////////////////////////////////////////////////////////
////////////////////////AoIBHgH/////////////////////////////////
////////////////////////////////////////////////////////////////
////////////////////////////////////////////////////////////////
////////////////////////////////////////////////////////////////
////////////////////////////////////////////////////////////////
////////////////////////////////////////////////////////////////
//////////////////////////8CgaBVVaqqVVWqqlVVqqpVVaqqVVWqqlVVqqpV
VaqqVVWqqlVVqqpVVaqqVVWqqlVVqqpVVaqqVVWqqlVVqqpVVaqqVVWqqlVVqqpV
VaqqVVWqqlVVqqpVVaqqVVWqqlVVqqpVVaqqVVWqqlVVqqpVVaqqVVWqqlVVqqpV
VaqqVVWqqlVVqqpVVaqqVVWqqlVVqqpVVaqqVVWqqlVVqqpVVaqpAoIBHgGAgH9/
gIB/f4CAf3+AgH9/gIB/f4CAf3+AgH9/gIB/f4CAf3+AgH9/gIB/f4CAf3+AgH9/
gIB/f4CAf3+AgH9/gIB/f4CAf3+AgH9/gIB/f4CAf3+AgH9/gIB/f4CAf3+AgH9/
gIB/f4CAf3+AgH9/gIB/f4CAf3+AgH9/gIB/f4CAf3+AgH9/gIB/f4CAf3+AgH9/
gIB/f4CAf3+AgH9/gIB/f4CAf3+AgH9/gIB/f4CAf3+AgH9/gIB/f4CAf3+AgH9/
gIB/f4CAf3+AgH9/gIB/f4CAf3+AgH9/gIB/f4CAf3+AgH9/gIB/f4CAf3+AgH9/
gIB/f4CAf3+AgH9/gIB/f4CAf3+AgH9/gIB/f4CAf3+AgH9/gIB/f38CgaASJEkS
JEkSJIkSJIkSJIkSRIkSRIkiRIkiRIkiRJEiRJEiRJEiSJEiSJEkSJEkSJEkSJIk
SJIkSRIkSRIkSRIkiRIkiRIkiRJEiRJEiSJEiSJEiSJEkSJEkSJIkSJIkSJIkSRI
kSRIkSRIkiRIkiRJEiRJEiRJEiSJEiSJEkSJEkSJEkSJIkSJIkSJIkSRIkSRIkiR
IkiRIkiRJEiRJEiR
-----END RSA PRIVATE KEY-----
````
which I then verified matched the provided public key with `chmod go-rwx && ssh-keygen -y -f id_rsa`.

Following that, I copied the public key over and simply logged into the server with `ssh -i id_rsa ctf@54.252.234.85` and ran the following commands:
````
ctf@ip-172-31-23-157:~$ ls
flag
ctf@ip-172-31-23-157:~$ cat flag
1GoMKytCA3RKYuka6HVpBMzsG34DJVX1Nx
````
