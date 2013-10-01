The remote
==========

Background
----------
For this challenge, we were given a pcap file and the instruction "Someone is controlling this computer remotely. Figure out what's going on."

Tools used
----------
* Wireshark

Process
-------
After opening the pcap file and right clicking->Follow TCP stream, something caught our attention, a recurring "Relmtech Keyboard". A quick search revealed that this was likely to be a remote-controlling software made by Relmtech. This explains the transfer of data between an Android device (`android-46204fa038097bdb`) and a desktop computer (`WINXPPROSP3VB`). It appears that the Android device is pressing keys on a keyboard to send to the desktop and we deduced that it must be typing the key.

By filtering the packets to only those sent from `192.168.1.3` to `192.168.1.5`, we could see a sentence being typed, character by character:
````
Name.VK_T
Name.VK_H
Name.VK_E
Name.{SPACE}
Name.VK_K
Name.VK_E
Name.VK_Y
Name.{SPACE}
Name.VK_I
Name.VK_S
Name.:
Name.{SPACE}
Name.VK_W
Name.VK_L
Name.VK_A
Name.VK_N
Name.{SPACE}
Name.VK_R
Name.VK_E
Name.VK_M
Name.VK_O
Name.VK_T
Name.VK_E
Name.VK_S
Name.{SPACE}
Name.VK_A
Name.VK_R
Name.VK_E
Name.{SPACE}
Name.VK_S
Name.VK_W
Name.VK_E
Name.VK_E
Name.VK_T
Name.{SPACE}
Name.!
````
The flag was **wlan remotes are sweet!**.
