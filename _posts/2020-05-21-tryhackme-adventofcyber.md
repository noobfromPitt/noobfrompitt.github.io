---
title: "Advent of Cyber walkthrough"
date: 2020-05-21
tags: [ctf, tryhackme, advent of cyber, walkthrough]
header:
  image: "/images/tryhackme/thm.png"
---

# Advent of Cyber


url: https://tryhackme.com/room/25daysofchristmas

## Day#1:

machine: 10.10.155.75
website: http://10.10.155.75:3000

#### 1: What is the name of the cookie used for authentication?

cookie will be created by the server when the login is succesful. We can create an account using register page. I used the username `McElferson` and after logging in , a `Set-Cookie` header is sent in the response with the cookie as `authid=TWNFbGZlcnNvbnY0ZXI5bGwxIXNz; Path=/`

![http response including cookie]({{site.url}}{{site.baseurl}}/images/tryhackme/AdventOfCyber/day1-1.png)

#### 2: If you decode the cookie, what is the value of the fixed part of the cookie?

The value of cookie can be base64 decoded. This can be done in command line using `echo TWNFbGZlcnNvbnY0ZXI5bGwxIXNz | base64 -d`. the decoded value of cookie is  `McElfersonv4er9ll1!ss`. It clearly has the username in it, which changes according the user logged in, so the rest of it must be the fixed part. I've created another account and verified that `v4er9ll1!ss` is the fixed part.

#### 3: After accessing his account, what did the user mcinventory request?

since we know the username `mcinventory`, we can craft a cookie with the username and fixed part and encode it. `echo mcinventoryv4er9ll1!ss | base64`. This will give the encoded value `bWNpbnZlbnRvcnl2NGVyOWxsMXNzaAo=`
This is actually not correct as `echo` will remove the `!` character or even worse, `!ss` will run the last command from histry that matches with 'ss' (In my case, I had a `ssh` in the histroy and `!ss` has appended the `ssh` command while base64 encoding)
We can use quotes `echo 'mcinventoryv4er9ll1!ss' | base64`. This will give `bWNpbnZlbnRvcnl2NGVyOWxsMSFzcwo=`
change the value of the `authid` cookie to this and reload the home page.
This didn't work. When i encoded from https://www.base64encode.org/ site, the encoded string is a little different. its `bWNpbnZlbnRvcnl2NGVyOWxsMSFzcw==` and this value of cookie works. This need to be further investigated **TODO**
The site lists the items requested by users and `mcinventory` requests `firewall` item

## Day2:

machine: 10.10.157.249
website: http://10.10.157.249:3000

#### 1: What is the path of the hidden page?

we can use `gobuster` to find all the pages. `gobuster dir -u http://10.10.157.249:3000 -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt`
Only few pages of the output have 200 reponse code. So the hidden page must be `/sysadmin`

#### 2: What is the password you found?

going through the `/sysadmin` page, there is a comment at the end of file saying ` Admin portal created by arctic digital design - check out our github repo`
this repo has a username and password in the readme.md file
https://github.com/ashu-savani/arctic-digital-design
these credentials work
so the password is `defaultpass`

#### 3:	What do you have to take to the 'partay'

after logging in to the `/sysadmin` page, there is a small image that says `Hey all - Please don't forget to BYOE(Bring Your Own Eggnog) for the partay!!`
so the answer should be `eggnog`

## Day3:

a pcap file is given

#### 1: Whats the destination IP on packet number 998?

from the pcap file, the packet number 998 has destination `63.32.89.195`

#### 2: What item is on the Christmas list?

Dont know what the christmas list means here. But if we follow the stream from the packet 998, we can see some commands being run.
```
echo 'ps4' > christmas_list.txt
cat /etc/shadow
```
from the first command, christmas_list.txt contains `ps4`

#### 3: Crack buddy's password!

from the output of the `cat /etc/shadow` command, we can see the hash of the user buddy
`buddy:$6$3GvJsNPG$ZrSFprHS13divBhlaKg1rYrYLJ7m1xsYRKxlLh0A1sUc/6SUd7UvekBOtSnSyBwk3vCDqBhrgxQpkdsNN6aYP1:18233:0:99999:7:::`

`$6` corresponds to the hashing algorithm `sha512` and the password hash is `3GvJsNPG$ZrSFprHS13divBhlaKg1rYrYLJ7m1xsYRKxlLh0A1sUc/6SUd7UvekBOtSnSyBwk3vCDqBhrgxQpkdsNN6aYP1`

We can use `hashcat` to try and get the password. from https://hashcat.net/wiki/doku.php?id=hashcat, the option we need to use for `$6` hash mode is `1800`. So the command is `hashcat -m 1800 hash /usr/share/wordlists/rockyou.txt --force` where hash is a file containing `$6$3GvJsNPG$ZrSFprHS13divBhlaKg1rYrYLJ7m1xsYRKxlLh0A1sUc/6SUd7UvekBOtSnSyBwk3vCDqBhrgxQpkdsNN6aYP1`

within a few seconds, hashcat cracks it and the password is `rainbow`
![hashcat output]({{site.url}}{{site.baseurl}}/images/tryhackme/AdventOfCyber/day3-3.png)

## Day4:

machine:10.10.233.224
the username and password are given, so we can `ssh` to the machine

#### 1: How many visible files are there in the home directory(excluding ./ and ../)?

there are 8 files (excluding the hidden files)

#### 2:	What is the content of file5?

recipes

#### 3:	Which file contains the string ‘password’?

we can write a simple command line script to cat throguh all the files and grep for `password`
`for file in file*;do echo $file; cat $file | grep password; done`
![password search]({{site.url}}{{site.baseurl}}/images/tryhackme/AdventOfCyber/day4-3.png)

looks like its present in `file6`
`passwordHpKRQfdxzZocwg5O0RsiyLSVQon72CjFmsV4ZLGjxI8tXYo1NhLsEply`

#### 4:	What is the IP address in a file in the home folder?

we can write a similar command line script like above to grep for the ip format (i feel like there must be a better way to do it using awk and regex)
`for file in file*;do echo $file; cat $file | grep '[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}';done`
This shows that there is an ip address `10.0.0.05` in file2
`10.0.0.05eXWx4auBc8Swra4aPvIoBre+PRsVgu9GVbGwD33X8bd7TWwlZxzSVYa`
![ip address from files]({{site.url}}{{site.baseurl}}/images/tryhackme/AdventOfCyber/day4-4.png)

#### 5:	How many users can log into the machine?

first, lets check all the users from `/etc/passwd`
looks like there are some users dont have a shell assigned and some have nologin (which means they cannot login)
we can `grep` for users with valid shell
`cat /etc/passwd | grep bash`
there are 3 users


#### 6:	What is the sha1 hash of file8?

sha1 can be computed using `sha1sum file8`
the hash is `fa67ee594358d83becdd2cb6c466b25320fd2835`


#### 7	What is mcsysadmin’s password hash?

this can be easily found from `/etc/shadow` file but the user doesn't have permissions to read this file. 
I was unable to find a solution for this, but from other walkthroughs, i discovered there is a backup file (which is also hinted in the supporting material)
we can find if a backup file exists using `find / -name shadow.bak`
looks like there is a file in `/var/`
from the file we can see the hash of mcsysadmin is `jbosYsU/$qOYToX/hnKGjT0EscuUIiIqF8GHgokHdy/Rg/DaB.RgkrbeBXPdzpHdMLI6cQJLdFlS4gkBMzilDBYcQvu2ro/`

## Day5:

image: https://blog.tryhackme.com/content/images/size/w2000/2019/12/thegrinch.jpg

#### 1:	What is Lola's date of birth? Format: Month Date, Year(e.g November 12, 2019)

`exiftool` is a command line tool which gives the metadata of the image
running it on this image reveals the creator name `JLolax1`
simple google search shows that this is twitter handle of Elf Lola https://twitter.com/jlolax1?lang=en
data of birth is given on the profile page - December 29, 1990
![exiftool output]({{site.url}}{{site.baseurl}}/images/tryhackme/AdventOfCyber/day5-1.png)



#### 2:	What is Lola's current occupation?

from the same page, her current ocuupation is Santa's helpers


#### 3:	What phone does Lola make?

from the same page, her only tweet says that she makes iphones


#### 4:	What date did Lola first start her photography? Format: dd/mm/yyyy

The twitter page links to her photography blog also https://lolajohnson1998.wordpress.com/
But not much useful information from that
in the supporting material page, they said we need to use three OSINT tools.
searching this blog on waybackmachine revelas that there are snapshots stored on Oct 23rd 2019, the earliest.
on this version of the blog, she says that its her 5 year celebration sice she started freelance photography. So the date shold be Oct 23rd 2014


#### 5:	What famous woman does Lola have on her web page?

reverse searching the first pic from the blog reveals that its of Ada lovelace (mother of computer)

## Day6:

we are given a pcap file


#### 1:	What data was exfiltrated via DNS?

looking at the dns traffic, it is clear that some data is encoded in the dns requests. And this is a good way to exfiltrate data as its not encrypted. the request is a dns query to a subdomain `43616e64792043616e652053657269616c204e756d6265722038343931.holidaythief.com`

the subdomain `43616e64792043616e652053657269616c204e756d6265722038343931` is hex encoded. we can decode using `xxd`
```
echo '43616e64792043616e652053657269616c204e756d6265722038343931' > hex.txt
xxd -r -p hex.txt
```
this will give the decoded value `Candy Cane Serial Number 8491`


#### 2:	What did Little Timmy want to be for Christmas?

while going through the tcp traffic, there is a http GET request for resource `/christmaslists.zip` and based on the response, it looks like it contains .txt files of christmaslists of few people. `christmaslisttimmy.txt` is one of them. 
![wireshark http traffic]({{site.url}}{{site.baseurl}}/images/tryhackme/AdventOfCyber/day6-2.png)

We can export the .zip from wireshark menu: File -> export objects -> HTTP -> and select the .zip file. Looks like we need a password to extract the zip file
From the supporting material, they mentioned about brute forcing zip file password cracking using `fcrackzip`
`fcrackzip -b --method 2 -D -p /usr/share/wordlists/rockyou.txt  -v day6/christmaslists.zip`
within a few seconds, it will crack the password and its `december`
opening the `christmaslisttimmy.txt` shows that Timmy wants to be a PenTester for christmas


#### 3:	What was hidden within the file?

Along with the .zip, there is also an image that has been transferred over http. I think this question is refering to that file. running `exiftool` on that didnt give any useful info.
From the supporting doc, its mentioned to use `steghide` for any steganography extractions
I tried using `steghide -sf TryHackMe.jpg` and it asked for passphrase. `december` didnt work and i was stuck for a few minutes. Then I discovered that it doesn't need a password.
`steghide` created a `christmasmonster.txt`. It had the contents of `RFC527` https://tools.ietf.org/html/rfc527














