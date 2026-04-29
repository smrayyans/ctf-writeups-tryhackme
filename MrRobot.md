# MrRobot

## Room Info

This room is themed around the Mr. Robot TV show and walks you through a full attack chain on a WordPress site. The key concepts covered are web enumeration, credential brute-forcing with Hydra, WordPress theme editor abuse for remote code execution, and SUID-based privilege escalation through nmap's interactive mode. Three flags are hidden across the system, each requiring a deeper level of access than the last.

## Writeup

I started the machine, connected to the VPN, and checked connectivity with a quick `ping MACHINE_IP` before doing anything else.

Then I ran a basic service scan to see what was open.

`nmap -sV MACHINE_IP`

Screenshot:
![Nmap scan results](images/MrRobot/ss-01.png)

HTTP was open so I visited the IP in the browser. The page loaded some kind of interactive Mr. Robot themed terminal experience.

Screenshot:
![MrRobot themed landing page](images/MrRobot/ss-02.png)

I tried all the commands the terminal offered and poked around the page source too, but nothing useful turned up. Time to fuzz for hidden directories.

`dirb http://10.48.136.22/`

Screenshot:
![dirb fuzzing results](images/MrRobot/ss-03.png)

After looking through what dirb found, I checked `robots.txt` since it showed up in the results. It had two interesting entries.

Screenshot:
![robots.txt contents](images/MrRobot/ss-04.png)

The first was `key-1-of-3.txt`. I opened it and grabbed the first flag right there.

Screenshot:
![First flag](images/MrRobot/ss-05.png)

The second entry was `fsocity.dic`, a dictionary file. I pulled it down with wget.

`wget http://10.48.136.22/fsocity.dic`

Screenshot:
![Downloading fsocity.dic](images/MrRobot/ss-06.png)

I checked how large it was with `wc -l fsocity.dic` and it was massive.

Screenshot:
![Word count of fsocity.dic](images/MrRobot/ss-07.png)

Sorted it and stripped duplicates before using it for anything, which cut the size down significantly.

`sort -u fsocity.dic -o sorted.txt`

Screenshot:
![Sorted list is smaller](images/MrRobot/ss-08.png)

I went to the WordPress login page at `/wp-login.php` and tried a couple of obvious credentials. Nothing worked. I noticed something useful though: the error message when you enter a wrong username says "Invalid username", but when the username is correct it changes to a different message about the password. That difference is what lets Hydra split the problem into two steps.

Screenshot:
![WordPress login error message](images/MrRobot/ss-09.png)

First I brute-forced the username using the sorted wordlist, with a placeholder password.

`hydra -L sorted.txt -p default 10.48.136.22 http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In&testcookie=1:F=Invalid username"`

Screenshot:
![Hydra username brute-force results](images/MrRobot/ss-10.png)

I saved the found usernames to `usernames.txt` and then ran Hydra again, this time to get the password.

`hydra -L usernames.txt -P sorted.txt 10.48.136.22 http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In&testcookie=1:F=The password you entered for the username"`

Screenshot:
![Hydra password brute-force results](images/MrRobot/ss-11.png)

The same password worked for all the usernames. I logged into WordPress with `elliot:ER28-0652` and landed in the admin dashboard.

Screenshot:
![WordPress admin dashboard](images/MrRobot/ss-12.png)

From there I went to the theme editor at `/wp-admin/theme-editor.php` and injected a PHP reverse shell payload into the Twenty Fourteen theme's `archive.php`. I updated the IP and port to my own, set up a listener on that port in a separate terminal, then triggered the shell by navigating to the archive template URL.

Screenshot:
![Theme editor with reverse shell payload](images/MrRobot/ss-13.png)

`http://10.48.136.22/wp-content/themes/twentyfourteen/archive.php`

Screenshot:
![Reverse shell landed](images/MrRobot/ss-14.png)

The shell came back. I upgraded it to a proper TTY so it was easier to work with.

`python3 -c 'import pty; pty.spawn("/bin/bash")'`

Screenshot:
![TTY shell upgrade](images/MrRobot/ss-15.png)

I searched for flag files with `find / -name "flag2.txt" 2>/dev/null` but got nothing. I checked `/etc/passwd` next and noticed a user called `robot` in addition to root.

Screenshot:
![/etc/passwd showing robot user](images/MrRobot/ss-16.png)

I poked around manually and found `/home/robot`. There was a flag file in there but I couldn't read it as the current user. However, there was also a file called `password.raw-md5` that I could open.

Screenshot:
![robot's home directory contents](images/MrRobot/ss-17.png)

I threw the hash into CrackStation.

Screenshot:
![CrackStation cracking the hash](images/MrRobot/ss-18.png)

It came back with `abcdefghijklmnopqrstuvwxyz` as the plaintext. I upgraded my shell again, switched to robot with `su robot`, and read the second flag.

Screenshot:
![Second flag as robot user](images/MrRobot/ss-19.png)

Now for root. I checked `sudo -l` but robot had no sudo permissions. I looked for SUID binaries instead.

`find / -perm -4000 2>/dev/null`

nmap showed up with the SUID bit set, which is the interesting one here.

Screenshot:
![SUID binaries, nmap highlighted](images/MrRobot/ss-20.png)

Older versions of nmap ship with an interactive mode that drops you into a shell. I launched it and tried `!/bin/sh` first but that didn't work, so I tried `!sh` instead and got a root shell immediately.

`nmap --interactive`

`!sh`

Screenshot:
![Root shell via nmap interactive mode](images/MrRobot/ss-21.png)

Navigated to `/root` and grabbed the last flag.

Screenshot:
![Third flag in /root](images/MrRobot/ss-22.png)

The flag was there. Room done.
