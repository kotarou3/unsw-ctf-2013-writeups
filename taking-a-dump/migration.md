Migration
=========

Background
----------
The goal of this challenge was to find which process the malware injected itself into

Tools used
----------
* volatility

Process
-------
Using the `pstree` module of volatility, it showed this output:
````
Name                                                  Pid   PPid   Thds   Hnds Time
-------------------------------------------------- ------ ------ ------ ------ --------------------
<snip>
 0x81890228:explorer.exe                             1336   1296     14    323 2013-09-14 08:55:03
. 0x8191e6e0:IEXPLORE.EXE                            1448   1336      5    104 2013-09-14 08:56:05
.. 0x81680888:cmd.exe                                1636   1448      1     37 2013-09-14 08:56:24
. 0x814cd5c8:IEXPLORE.EXE                            1704   1336     12    294 2013-09-14 08:58:17
````
Internet explorer launching `cmd.exe` is pretty suspicious, so at this point I already suspected PID 1448 was the culprit.

Running the `malfind` module, it added further evidence that 1448 was the culprit, with 27/41, or ~66% of the entries in PID 1448.

Now having a decent amount of evidence that PID 1448 was the culprit, I inputted it as the flag, and it was correct!
