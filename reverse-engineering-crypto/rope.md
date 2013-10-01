rope
====

Background
----------
For this challenge, we were only given a binary, however, there was also a hint: "Twine, cord, thread, strand."

Tools used
----------
* `strings`

Process
-------
The first thing that came into mind after reading problem name and hint was the `strings` command, which extracted strings from files. So we called `strings` on the binary, however, none of the output seemed to be the key we were looking for.

However, knowing that strings only outputs 4 character long strings, we tried `strings -n 1`, which would output all the characters, and near the end of the output, we could just read down the column to get:
````
DEBUG MODE: The key is: 1KsRezs4u8ksj6T26DMgvjjNsLjMmcH69s
````
