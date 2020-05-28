---
title: "TryHackMe WiresharkCTF walkthrough"
date: 2020-05-23
tags: [wireshark, CTF, tryhackme, flag, walkthrough]
header:
  image: "/images/tryhackme/thm.png"
---

## TryHackMe - [Wireshark CTFs](https://tryhackme.com/room/wirectf)

This is a medium difficulty room with two pcap files that need to be analyzed. This post only goes through the fist one (solving it was already exhausting :P)

### Task1: 

A CTF challenge set by csaw. During this task, you will be have to inspect a pcap file (using programs such as tshark and wireshark). You will analysis the file and release something has been... "transferred".

Download and look through the pcap file to analyse the traffic in order to find the flag.

pcap can be downloaded from the [room](https://tryhackme.com/room/wirectf)

#### Analysis:

The pcap looks huge and I don't know where to start. A good thing to check with such huge files is the protocol hierarchy (in statistics menu of wireshark). This shows which protocols are being used and how many packets are using each protocol.

![protocol-hierarchy]({{site.url}}{{site.baseurl}}/images/tryhackme/wiresharkctf1/protocol-hierarchy.png)

We can see that there is a lot of traffic over TLS (HTTPS) and some traffic over HTTP. Since we cannot decrypt HTTPS traffic, lets just start with the HTTP traffic.
Lets first filter the http traffic and start checking each stream (Follow -> TCP stream)

The first stream looks like a simple python-requests get request to reddit.com. The reponse code is 301. This must be because its requesting a http site and reddit redirects to a https site. The https site is mentioned in the location parameter of the response.

![stream1]({{site.url}}{{site.baseurl}}/images/tryhackme/wiresharkctf1/stream1.png)

The second stream is also very similar. This time the request is to twitter.com. Response is again 301 with location as https://twitter.com

The next stream is very interesting. The request looks like a python program.

![stream4]({{site.url}}{{site.baseurl}}/images/tryhackme/wiresharkctf1/stream4.png)

It also has an empty flag 'flag{xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx}'
The script is followed by a huge string, might be a hash. It doesn't look like a simple base64 encoded string.

Lets create a python script with this code, excluding the hash

```python
import string
import random
from base64 import b64encode, b64decode

FLAG = 'flag{xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx}'

enc_ciphers = ['rot13', 'b64e', 'caesar']
# dec_ciphers = ['rot13', 'b64d', 'caesard']

def rot13(s):
	_rot13 = string.maketrans( 
    	"ABCDEFGHIJKLMabcdefghijklmNOPQRSTUVWXYZnopqrstuvwxyz", 
    	"NOPQRSTUVWXYZnopqrstuvwxyzABCDEFGHIJKLMabcdefghijklm")
	return string.translate(s, _rot13)

def b64e(s):
	return b64encode(s)

def caesar(plaintext, shift=3):
    alphabet = string.ascii_lowercase
    shifted_alphabet = alphabet[shift:] + alphabet[:shift]
    table = string.maketrans(alphabet, shifted_alphabet)
    return plaintext.translate(table)

def encode(pt, cnt=50):
	tmp = '2{}'.format(b64encode(pt))
	for cnt in xrange(cnt):
		c = random.choice(enc_ciphers)
		i = enc_ciphers.index(c) + 1
		_tmp = globals()[c](tmp)
		tmp = '{}{}'.format(i, _tmp)

	return tmp

if __name__ == '__main__':
	print encode(FLAG, cnt=?)
```

Lets try to run the script with the default value of 'cnt=50' in the 'encode' function. I was trying to match the wordcount from this to the wordcount from the hash, but it didnt' match.
But it looks like there is a differnt word count everytime. So, the hash might still be the output of this script. The wordcount of hash is still way higher than the one with default flag. So, the appropriate 'cnt' value might also needs to be found.

![wc]({{site.url}}{{site.baseurl}}/images/tryhackme/wiresharkctf1/wc.png)

Lets try to understand the script. First the script takes the flag and base64 encodes it and appends to '2'. Then it is encoded with a randomly selected cipher from the three 'enc_ciphers'. The selected cipher's index is also added before the encoded part (this explains why 2 is added before base64 encoding the first time - 2 is the index (+1) of the base64 in enc_ciphers).

Our aim is to decode the ciphertext. The commented dec_ciphers might be hinting that we need to create decoding functions.

For rot13, the saame function can be used to decode. Same function can also be used for caesar cipher but with the negative value of the shift argument. This is also hinted with the function names in commented dec_ciphers list. Lets update the script with these functions

```python
dec_ciphers = ['rot13', 'b64d', 'caesard']

def b64d(s):
	return b64decode(s)

def caesard(plaintext, shift=-3):
    alphabet = string.ascii_lowercase
    shifted_alphabet = alphabet[shift:] + alphabet[:shift]
    table = string.maketrans(alphabet, shifted_alphabet)
    return plaintext.translate(table)
```

Now lets create the decode function. Remember, the fist character in each loop is the index +1 of the decoding function. And after all the deciphering, we need to do one last base64 decode.

```python
def decode(ct, cnt=50):
	tmp = ct
	for cnt in xrange(cnt):
		d = dec_ciphers[int(tmp[:1]) -1]
		tmp = globals()[d](tmp[1:])

	return b64d(tmp[1:])

import sys

if __name__ == '__main__':
	print decode(sys.argv[1], int(sys.argv[2]))
```

I've also added arguments so that we can test with differnt inputs for 'ct' and 'cnt'

The modified script works with the various outputs.

![decode]({{site.url}}{{site.baseurl}}/images/tryhackme/wiresharkctf1/decode.png)

Lets also eliminate the 'cnt' parameter, since we dont really need it. If the fist character is not an integer, we can return whatever is the value of 'tmp' at that point. Also, lets increase the default value of 'cnt' to 100, since we're expecting the 'cnt' of cipher text to be more than 50

```python
def decode(ct, cnt=100):
	tmp = ct
	for cnt in xrange(cnt):
		try:
			int(tmp[:1])
			d = dec_ciphers[int(tmp[:1]) -1]
			tmp = globals()[d](tmp[1:])
		except ValueError:
			return tmp

	return b64d(tmp[1:])

if __name__ == '__main__':
	print decode(sys.argv[1])
```

Lets verify this again.

![decode-nocnt]({{site.url}}{{site.baseurl}}/images/tryhackme/wiresharkctf1/decode-nocnt.png)

This seems to be working without the value of 'cnt'

Since the encoded string is too huge to paste as argument, lets use the input from `stdin` instead of `sys.argv`. We can run the script with output of `cat`

```python
if __name__ == '__main__':
	print [decode(ct) for ct in sys.stdin]
```

`cat ct | python tryhackme-wiresharkctf1.py`

This seems to be working fine. 

Now lets try with our original encoded string. Looks like it works and prints out the flag. Yay!!!

![yay]({{site.url}}{{site.baseurl}}/images/tryhackme/wiresharkctf1/yay.png)

Lets print the 'cnt' number at each loop to see whats value of 'cnt' was used while encoding.

```python
def decode(ct, cnt=100):
	tmp = ct
	for cnt in xrange(cnt):
		print cnt
		try:
			int(tmp[:1])
			d = dec_ciphers[int(tmp[:1]) -1]
			tmp = globals()[d](tmp[1:])
		except ValueError:
			return tmp

	return b64d(tmp[1:])
```

Looks like cnt of 61 was used.
The final script will all the modifications can be found [here]({{site.url}}{{site.baseurl}}/misc/tryhackme-wiresharkctf1.py)
