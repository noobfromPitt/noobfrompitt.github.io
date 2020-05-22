---
title: "Overthewire Natas walkthrough"
date: 2020-05-21
tags: [overthewire, wargames, natas, walkthrough]
header:
  image: "/images/overthewire/otw.png"
---

##  [Overthewire](https://overthewire.org/wargames/) has some good challenges (wargames) from basic level. [Natas](https://overthewire.org/wargames/natas/) has challenges related to web security

#### Natas

Natas teaches the basics of serverside web-security.

Each level of natas consists of its own website located at http://natasX.natas.labs.overthewire.org, where X is the level number. There is no SSH login. To access a level, enter the username for that level (e.g. natas0 for level 0) and its password.

Each level has access to the password of the next level. Your job is to somehow obtain that next password and level up. All passwords are also stored in /etc/natas_webpass/. E.g. the password for natas5 is stored in the file /etc/natas_webpass/natas5 and only readable by natas4 and natas5.

Start here:

Username: natas0
Password: natas0
URL:      http://natas0.natas.labs.overthewire.org

#### Level 0

![level0]({{site.url}}{{site.baseurl}}/images/overthewire/natas/level0.png)

This is pretty simple, the password is in the comments of page source

#### Level 1

![level1]({{site.url}}{{site.baseurl}}/images/overthewire/natas/leve11.png)

Similar to level0, it says the password is on the page. Since right click is disabled, we can use shortcut `Ctrl+U` or just go to `view-source:http://natas1.natas.labs.overthewire.org/`

#### Level 2

![level2]({{site.url}}{{site.baseurl}}/images/overthewire/natas/level2.png)

It says there is nothing on the page. The header part of the source is common to all the levels, so there is nothing useful there.

![level2-source]({{site.url}}{{site.baseurl}}/images/overthewire/natas/level2-source.png)

Unlike other levels, this level includes an image `<img src="files/pixel.png">`. I've downloaded the image to see if there is anything interesting using `exiftool`. There was nothing interesting.
Since its using an image from "files" directory, I tried if we can access this path `http://natas2.natas.labs.overthewire.org/files/`

It has a users.txt with bunch of usernames and passwords, one of which is for natas3

#### Level 3

![level3]({{site.url}}{{site.baseurl}}/images/overthewire/natas/level3.png)

Similar to last one, it says there is nothing on the page. The source reveals a comment

![level3-source]({{site.url}}{{site.baseurl}}/images/overthewire/natas/level3-source.png)

/files gives a 404 error. We can check for /robots.txt, since the comment says google cannot find it and robots.txt has a list of paths for search engines to exclude while crawling, we can try if a /robots.txt exists.
http://natas3.natas.labs.overthewire.org/robots.txt does exist and it has a hint.

![level3-robots]({{site.url}}{{site.baseurl}}/images/overthewire/natas/level3-robots.png)

http://natas3.natas.labs.overthewire.org/s3cr3t/ leads to a users.txt which has the password for natas4

#### Level 4

![level4]({{site.url}}{{site.baseurl}}/images/overthewire/natas/level4.png)

It says authorized users should come only from "http://natas5.natas.labs.overthewire.org/".
The source also doesn't have anything interesting.

Lets check if we can redirect to http://natas4.natas.labs.overthewire.org/ from a different website and get the page to print it in "" (which is empty now).
I went to https://google.com and changed the window location to natas4 from js console `window.location.replace("http://natas4.natas.labs.overthewire.org/")`

![level4-redirect]({{site.url}}{{site.baseurl}}/images/overthewire/natas/level4-redirect.png)

As expected, it says we're visiting from google.com. The know how it knows we came from google.com, lets examine the actual request and response (from chrome inspector)

![level4-referer]({{site.url}}{{site.baseurl}}/images/overthewire/natas/level4-referer.png)

So, we need to somehow change the referer parameter in the header to get in.

We can do this using burp suite to do this. I've used firefox inspector as it allows to edit the request and resend.

![level4-natas5]({{site.url}}{{site.baseurl}}/images/overthewire/natas/level4-natas5.png)

This gives the password for the next level

![level4-done]({{site.url}}{{site.baseurl}}/images/overthewire/natas/level4-done.png)

#### Level 5

![level5]({{site.url}}{{site.baseurl}}/images/overthewire/natas/level5.png)

It says we're not logged in. I've checked if there are any /login, /signin, /signup pages. None of them exist. The only way the site knows if we're logged in or not is using cookies.

![level5]({{site.url}}{{site.baseurl}}/images/overthewire/natas/level5-cookie.png)

As expected, there is a cookie named 'loggedin' with value 0. We can change it to 1 and reload. It now gives the password for natas6

#### Level 6

![level6]({{site.url}}{{site.baseurl}}/images/overthewire/natas/level6.png)

It just has a search box. The view source link opens http://natas6.natas.labs.overthewire.org/index-source.html, which has some interesting information regarding how the search is implemented.

![level6-source]({{site.url}}{{site.baseurl}}/images/overthewire/natas/level6-source.png)

If the seach term matches the $secret, it would give the password. Since this varaiable is not defined on this page, it must be in the "includes/secret.inc"

http://natas6.natas.labs.overthewire.org/includes/secret.inc defines a $secret. Lets use this in the serach bar to get the password.

#### Level 7

![level7]({{site.url}}{{site.baseurl}}/images/overthewire/natas/level7.png)

It has two pages, home and about. It loads the page based on the ?page parameter in the url. The source has a comment that says password is in '/etc/natas_webpass/natas8'.

![level7-source]({{site.url}}{{site.baseurl}}/images/overthewire/natas/level7-source.png)

This might be hinting at a local file inclusion vulnerability. Lets try if we can get some data using /etc/natas_webpass/natas8 as ?page parameter
http://natas7.natas.labs.overthewire.org/index.php?page=/etc/natas_webpass/natas8

![level7-done]({{site.url}}{{site.baseurl}}/images/overthewire/natas/level7-done.png)

This in fact reveals the password in file /etc/natas_webpass/natas8. All the passwords till natas8 can be found using this method.

#### Level 8

![level8]({{site.url}}{{site.baseurl}}/images/overthewire/natas/level8.png)

Similar to level6, it has a search box and a source file http://natas8.natas.labs.overthewire.org/index-source.html

![level8-source]({{site.url}}{{site.baseurl}}/images/overthewire/natas/level8-source.png)

There is a $encodedSecret variable, which has the right secret to search in the box to get the password.
Looks like its base64 encoded, reversed and then hex encoded.

$encodedSecret = 3d3d516343746d4d6d6c315669563362
hex decoded = ==QcCtmMml1ViV3b
reverse = b3ViV1lmMmtCcQ==
base64 decode = oubWYf2kBq	

Searching for this will reveal the password

#### Level 9

![level9]({{site.url}}{{site.baseurl}}/images/overthewire/natas/leve9.png)

Level9 again has search box and a source

**level9-source**
![level9-source]({{site.url}}{{site.baseurl}}/images/overthewire/natas/level9-source.png)

This time, it will take the ?needle parameter (which is the search item) and grep through a dictionary file http://natas9.natas.labs.overthewire.org/dictionary.txt

Here, grep is directly used with the $key, so we can use this for command injection. We can search for some string like 'test' followed by a ';' and next command. The command should be executed.

For example, http://natas9.natas.labs.overthewire.org/?needle=test;whoami; gives 'natas9' as output.

![level9-command]({{site.url}}{{site.baseurl}}/images/overthewire/natas/level9-command.png)

As we know from before, the password for natas10 is stored in /etc/natas_webpass/natas10

Lets see if we can `cat` this file. http://natas9.natas.labs.overthewire.org/?needle=test;cat%20/etc/natas_webpass/natas10;
This works and password is shown on the page.

#### Level 10

![level10]({{site.url}}{{site.baseurl}}/images/overthewire/natas/level10.png)

It is similar to the previous one, thre is a search box. But this time, it says they've filtered certain characters. Lets also check the source code

![level10-source]({{site.url}}{{site.baseurl}}/images/overthewire/natas/level10-source.png)

We know our password is in /etc/natas_webpass/natas11 and we cannot use the characters ';', '|' and '&'

After going through grep manual `man grep` we can see that there is a way to print file.

![level10-grep]({{site.url}}{{site.baseurl}}/images/overthewire/natas/level10-grep.png)

Lets try with this input http://natas10.natas.labs.overthewire.org/?needle=.*%20/etc/natas_webpass/natas11

This will reveal the password

#### Level 11

![level11]({{site.url}}{{site.baseurl}}/images/overthewire/natas/level11.png)

It has an input box which takes a hex value of color as input and changes background accordinly. It also says cookies are protected with XOR ecnryption. Again, there is a sourcecode page

![level11-source]({{site.url}}{{site.baseurl}}/images/overthewire/natas/level11-source.png)

Lets go over the code. First, the ?bgcolor key is taken from url and checked if it matches the pattern of hex color. It is then XOR encrypted using a $key and saved as cookie.

The cookie also has a ?showpassword key which is set to 'no'. The page shows the password if it is set toy 'yes'.



