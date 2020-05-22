---
title: "Mr Robot CTF walkthrough"
date: 2020-05-21
tags: [ctf, tryhackme, mrrobot, walkthrough]
header:
  image: "/images/tryhackme/MrRobotCTF/thm.png"
---

# Tryhackme - [Mr Robot CTF](https://tryhackme.com/room/mrrobot)

## Recon:

`nmap -A 10.10.227.36` shows that there are 997 filtered port and port 22, 80 and 443 are filtered. This means that there is some kind of firewall blocking the nmap scans

Lets open the website anyway. the http site give a browser based shell with only few commands. The https site has a self signed certificate, and it also does the same thing

![terminal]({{site.url}}{{site.baseurl}}/images/tryhackme/MrRobotCTF/1-terminal.png)

If we use `wakeup` command, then the port seems to be opened. 
Nothin interesting from the page source too

## File enumeration

lets run a gobuster on this.
`gobuster dir -u http://10.10.227.36/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt`

There is a /images page but we dont have permissions to access it. THere are lot of such pages with 301 return codes. 
/readme and /license have some text but not helpful

/robots has the hint for first key. 
![robots]({{site.url}}{{site.baseurl}}/images/tryhackme/MrRobotCTF/2-robots.png)

/key-1-of-3.txt page has the first key

## wpscan

We also see wp-content and wp-login pages. So, we can run wpscan to find any wordpress vulnerabilities. I ran a wpscan using `wpscan --url 10.10.227.36`
It reveals /xlmrpc.php page, which says that this page accepts only POST requests
this can be used to enumerate usernames. 
However, I didn't find any valid users with wpscan
`wpscan --url http://10.10.197.250/ --enumerate u`

fsociety.dic is also found on /robots page. Going to /fsociety.dic lets us download the file, which is a list of words
Wordpress sites has permalinks to posts and pages and also user's pages. So, now that we have a word list that resembles usernames, we can check if any of them are valid. We can use `gobuster` to do this
`gobuster --url http://10.10.197.250/author/ -w fsociety.dic`

![users]({{site.url}}{{site.baseurl}}/images/tryhackme/MrRobotCTF/4-users.png)

The fsociety.dic also has some strings resembling a password. We can use wpscan again to check if any password matches
Since it was taking a long time and the .dic is huge, i sorted the file
`cat fsociety.dic | sort -u | uniq > sorted.dic`
Now start the password brute forcing again
`wpscan --url http://10.10.197.250/ --usernames 'elliot','Elliot' --passwords sorted.dic`
Now the estimated time is around 30 mins. Earlier it was 4 hrs.

This gives us a valid password: ER28-0652 for elliot

![password]({{site.url}}{{site.baseurl}}/images/tryhackme/MrRobotCTF/5-password.png)

After logging in, the user's dashboard is shown. The user doesnt have any posts or comments, but is an administrator. We can also see another user in the users tab. There are alos some images in media secion

## Reverse shell

We can install a vulnerable plugin or we can install a reverse shell for wordpress. We can also use metasploit to get a reverse shell using the module `exploit/unix/webapp/wp_admin_shell_upload`

```
msfconsole
use exploit/unix/webapp/wp_admin_shell_upload
set RHOSTS 10.10.197.250
set USERNAME elliot
set PASSWORD ER28-0652
run
```
But somehow, it fails saying that the target doesnt appear to be a wordpress site

We can also inject malicious code into the appearence template. [reference](https://www.hackingarticles.in/wordpress-reverse-shell/)
In the appearence seciton of admin dashboard, we change the 404 error template to a php reverse shell. I am using the reverse shell from [pentestmonkey](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php)
Change the IP to your internal ip on tryhackme access page and the port to your netcat listener
`nc -lvnp 9001`

And now, going to the site 'http://10.10.197.250/wordpress/wp-content/themes/twentyfifteen/404.php' gives a reverse shell
The reverse shell is with user daemon. There is a second key key-2-of-3.txt and password.raw-md5 in /home/robot but can only be read by robot
So we need to escalate privileges

The password.raw-md5 file can be read by any user. And it gives a md5 hash
We can use hashcat to crack this md5
`hashcat -m 0 hash /usr/share/wordlists/rockyou.txt`
While it is running, we can also check at [hashkiller](https://hashes.com/decrypt/basic) if it can crack it.
Both have cracked it within few seconds 'c3fcd3d76192e4007dfb496cca67e13b:abcdefghijklmnopqrstuvwxyz'
So, the password is *abcdefghijklmnopqrstuvwxyz*

Lets login to robot user with this password. It works and now we can read the second key

## Privilege escalation

We need to escalate privileges to get a root login and find the third key.
Lets check if robot has any sudo permissions using `sudo -l`
Looks like there are none. Lets check if there are any binaries with SUID set using `find / -perm 4000 2>/dev/null`

We can find that nmap in /usr/local/bin/nmap has SUID bit.
We can [exploit](https://gtfobins.github.io/gtfobins/nmap/) this.
```
TF=$(mktemp)
echo 'os.execute("/bin/bash")' > $TF
/usr/local/bin/nmap --script=$TF
```
This was not working. I've checked the nmap version and it is 3.81, which is quite old.
It also supports interactive mode using --interactive, which can be used to escalate privileges

```
nmap --interactive
!sh
```

We can use !bash instead, but somehow I am still robot when i use bash. But when i use sh, i am root.

![shell]({{site.url}}{{site.baseurl}}/images/tryhackme/MrRobotCTF/6-shell.png)

the third key is in /root folder



