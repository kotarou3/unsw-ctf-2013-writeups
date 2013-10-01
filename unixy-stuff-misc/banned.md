banned
======

Background
----------
For this challenge, the only given information was a username, password and IP. However, we deduced that we had to SSH into this machine. Unfortunately, upon ssh-ing into said machine, we were "immediately" kicked off it, saying "This account has been banned".

Tools used
----------
* `ssh`
* `ls`
* `cat`

Process
-------
Initially, we had no idea on how to solve this challenge. We scanned for open ports, seeing if anything was open to exploitation however this revealed that only port 22 (SSH) was open. We assumed that this banning was via fail2ban or some similar application so we couldn't see any way in.

We were stumped until the hint was given out regarding the contents of `.profile`, which gets executed upon logging into the system:
````sh
echo
echo "     This account has been banned."
echo
exit
````

Thus the connection was closed from a point where the client can control, and all that was needed to prevent it from executing was pressing CTRL+C after logging in (right after you enter the password for SSH).

An `ls` showed that in the home directory, there was a `flag` file and upon `cat`-ing the file, the flag was revealed: **1DKQrWVcT4XVJSmjfvkDZ466r5dxP5jEaJ**.
