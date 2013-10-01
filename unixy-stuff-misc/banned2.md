banned2
=======

Background
----------
This challenge is a "sequel" to banned, where we have to find the second flag on the server

Tools used
----------
* `ps`
* `grep`
* `cryptsetup`

Process
-------
After I logged in, I checked the process list with `ps -Af` and immediately saw `/bin/bash /root/backup_encrypted.sh` running. So I tried to find the contents of that script with `cat /root/backup_encrypted.sh`, finding this:
````sh
#!/bin/bash

# encryption challenge backup script

while true; do
    sudo sh -c "
        # unmount to flush the the filesystem
        umount /mnt
        cryptsetup luksClose secrets

        # back it up
        cat /encrypted > /var/log/bak

        # remount it
        echo -n $(</root/encryption_password) | cryptsetup luksOpen /encrypted secrets -d -
        mount /dev/mapper/secrets /mnt
    "

    # backup once every few hours
    sleep 960
done
````

I observed a few important facts about this script:
* Most of it was run through `sudo sh -c "..."`
* The encrypted volume was saved to `/var/log/bak`, which also happened to be world readable
* `bash` expands things like `$(</root/encryption_password)` when passing it as a command line argument
* It mounted to `/mnt`, which I checked inside with `ls /mnt`, showing that it indeed contained the flag, but it was only root-readable
* It ran every 16 minutes

This meant that whenever the `sudo sh -c "..."` part was run, it would have the full encryption password in it's command line arguments. Because it only ran every 16 minutes, and probably only for a few seconds, I ran `while true; do ps -Af | grep sudo; sleep 1; done` which eventually showed `when you call my name, its like a little prayer` in place of the `$(</root/encryption_password)` as it's output.

Now, knowing the key, I needed to copy `/var/log/bak` to my local computer, and because of the output from `.profile` saying I'm banned, I was unable to use the normal methods of `scp` or `sftp` to get the file. Through a bit of testing, I found an alternative way to get the file - doing a "reverse" `scp`. I created a dummy account on my computer and I `scp`'d from the server to my computer rather than from my computer to the server.

Having both the key and the encrypted volume, I installed `cryptsetup` and ran:
````
$ echo -n when you call my name, its like a little prayer | sudo cryptsetup luksOpen /tmp/bak secrets
$ sudo mount /dev/mapper/secrets /mnt
$ sudo cat /mnt/flag
1LF3kZ78yA6HWi6uMxdrhHayXMuYCZ24Fr
````
