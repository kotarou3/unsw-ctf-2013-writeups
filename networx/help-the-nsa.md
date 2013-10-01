Help the NSA
============

Background
----------
For this challenge, the description was a back-story that revolved around the recent revealings of the NSA's misgivings of their citizen's data. However, this was not very relevant to the task at hand apart from needing to discover what was being sent.

Tools used
----------
* Wireshark
* `base64`

Process
-------
After opening up the pcap file in Wireshark, it was evidently an email transfer between `woot@babirusa` and `sugarbunny@gmail.com`. Scanning through the packets, `DATA` fragments caught our attention and so we right clicked->Follow TCP stream and something else struck our attention; a doc file was transferred. This section is relevant:
````
Content-Type: application/msword
Content-Disposition: attachment; filename="secret.doc"
Content-Transfer-Encoding: base64

0M8R4K<snip>AAAAAAAA
````

The section, `0M8R4K<snip>AAAAAAAA`,  was the base64'd data held in secret.doc. So we chucked that into `base64 -d`, just skimmed through the text (as only the flag was needed not the fonts, layout etc.) and this was the result: "Hey babe, thanks for last night. Sorry about crying afterward. I just love you so much, you know? Here's my Netflix password: unicornsAreCuteIwantToBelieve"

Thus, the flag was **unicornsAreCuteIwantToBelieve**.
