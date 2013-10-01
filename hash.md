Hash
====

Background
----------
For this challenge, we were given a piece of code that would produce a hash of a string. We were also given the hash of a password, hashed using that code. We were also given details about the password - it was 8 characters and random.

Tools used
----------
* `gcc`
* `python`

Process
-------
For this task, the first thing we did was to go through the hashing code and see what the code was actually doing. The code first initialises the hash, which was a block of 16 bytes, to be 0. Then, for each 16 characters given, it would apply a function to transform it, and then xor it onto the hash. It would also pad the input with zeros if the input length was not a multiple of 16.

We also noted that we can simplify the problem, given the information about the password. As it is 8 characters, the function is only called once, and half the block of characters will be zeros.

For this function, the block of characters is casted into an array of four 32-bit unsigned integers, and then that array is extended to double its length. Each extra element was computed by getting the element four and two elements behind it, apply a function rot on each, which cyclically rotates each’s binary representation right by one, and finally xoring them together. So, if we represent the original elements as `A`, `B`, `C`, and `D`, the extra four are:
````c
rot(A) ^ rot(C)
rot(B) ^ rot(D)
rot(C) ^ rot(rot(A) ^ rot(C))
rot(D) ^ rot(rot(B) ^ rot(D))
````
However, since the password is only 8 characters, `C` and `D` are both 0, and xoring by 0 does nothing, so the extra four are actually:
````c
rot(A)
rot(B)
rot(rot(A))
rot(rot(B))
````

The next step it does is create another array of four 32-bit integers, and initialise it to the value of the hash (through casting). Then for each element in the extended array, it would store the value of that element xor the second element in the newly created array xor the fourth element in the same array, shift all elements in that array right one (so the last element is discarded), and set the first element of that array to be the stored value.
The array from this process would then be xored onto the hash (which is zero in our case, so it doesn’t matter).

If you follow that process through, due to elements cancelling each other out because of xor, the final hash value as an array of four ints are:
````c
B ^ rot(B) ^ rot(rot(B))
A ^ rot(A) ^ rot(rot(A))
rot(B)
rot(A)
````

This hash is printed out as a hexadecimal. So, in order to unhash it, all we needed to do was to disregard the first half of the hash, unrotate the last two elements, and extract the letters. We can easily achieve this in python, and using the following code:
````python
from struct import *

password = ''
orig = raw_input()
orig = orig[16:]

a = orig[8:]
a = unpack('I', a.decode('hex'))[0]
a = a << 1 & 2**32-1 | a >> 31
password += pack('I', a)

a = orig[:8]
a = unpack('I', a.decode('hex'))[0]
a = a << 1 & 2**32-1 | a >> 31
password += pack('I', a)
print password
````
getting the password to be **q6xQmS02**.
