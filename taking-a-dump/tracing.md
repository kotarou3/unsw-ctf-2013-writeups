Tracing
=======

Background
----------
A dump of a virtual machine's memory was provided for this challenge and we had to find out what was going on. It was evident that malware was downloaded and then executed on this machine, however this task was to figure out the domain at which this malware was downloaded from.

Tools used
----------
* `strings`
* `grep`

Process
-------
All that was needed in this task was to run `strings` on the vmem file as the domain would most likely appear as a string somewhere. After `grep`-ing for `http` and then manually looking through the results, one popped up as being suspicious: `totallylegit.bxx.me`.

Some files were downloaded from this domain, as evident in the HTTP requests that were revealed in further searching through the results of the initial `strings`, including `totallynotatrojan.exe`, `/excitingoffersforyou`, etc.
