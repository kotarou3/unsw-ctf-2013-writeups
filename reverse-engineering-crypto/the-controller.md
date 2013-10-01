The controller
==============

Background
----------
The goal was to find the "key" that was locking a piece of "malware"

Tools used
----------
* IDA Pro
* `node`

Process
-------
Opening up the executable in IDA, I noticed it called a function named `auth`, which I entered and got hexrays to decompile to:
````c
signed int __cdecl auth(const char *a1)
{
  signed int result; // eax@5
  signed int i; // [sp+10h] [bp-Ch]@1
  signed int j; // [sp+10h] [bp-Ch]@6

  for ( i = 0; i <= 19; ++i )
    flag[i] -= c[i];
  if ( strlen("thisistotallytheflag") == strlen(a1) - 1 )
  {
    for ( j = 0; j <= 19; ++j )
    {
      if ( flag[j] != a1[j] )
        return 0;
    }
    result = 1;
  }
  else
  {
    result = 0;
  }
  return result;
}
````

Knowing that hexrays decompilation is only meant to be used as a general guide, I ignored the parts that didn't look right and looked at the two important parts:
````c
  for ( i = 0; i <= 19; ++i )
    flag[i] -= c[i];
````
and
````c
    for ( j = 0; j <= 19; ++j )
    {
      if ( flag[j] != a1[j] )
        return 0;
    }
    result = 1;
````

This told me that the string stored in `flag` was subtracted by `c` to produce the real "key". `flag` contained `"thisistotallytheflag"` while `c` contained `32 16 28 2A 1B 20 53 2E 22 1C 1C 1A 34 20 14 0C 18 27 20 13`. I simply wrote a small javascript script to produce the real "key":
````javascript
$ node
> flag = new Buffer("thisistotallytheflag")
<Buffer 74 68 69 73 69 73 74 6f 74 61 6c 6c 79 74 68 65 66 6c 61 67>
> c = new Buffer("3216282A1B20532E221C1C1A3420140C18272013", "hex")
<Buffer 32 16 28 2a 1b 20 53 2e 22 1c 1c 1a 34 20 14 0c 18 27 20 13>
> for (var b = 0; b < flag.length; ++b) flag[b] -= c[b]
84
> flag.toString()
'BRAINS!AREPRETTYNEAT'
````
