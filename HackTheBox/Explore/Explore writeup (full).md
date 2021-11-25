https://app.hackthebox.com/machines/356


### "Explore" machine - mapping

```bat
nmap -p- 10.10.10.247
```

![napm #1 result](https://is-going-to-rick-roll.me/1637878307.png)

Open ports on this machine:
* 2222 - TCP
* 5555 - TCP 
* 42135 - unknown TCP
* 59777 - unknown TCP

scan port 2222 to get more details about it:
```bat
nmap -oA explore 10.10.10.247 -sC -sV -p 2222
```