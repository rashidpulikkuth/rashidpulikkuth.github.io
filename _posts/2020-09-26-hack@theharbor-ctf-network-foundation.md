---
layout: post
title: Hack@TheHarbor CTF - Foundations - Networking 
tags: [CTF]
comments: true
---

Hack@TheHarbor was a good CTF. Point3 hosted the CTF on their Escalate platform. I will show you, how  i solved all **Foundations - Networking** challenges.

#### Lets start..


### 1 - numberlesslove

![Crepe](/assets/images/numb1.png)

Download the file to our system, it was a tcpdump capture file contains network data packets.

![Crepe](/assets/images/numb2.png)

Opened the file using **wireshark**

~~~
wireshark ee22a3b9fc2e8971c3bf812080d02714
~~~

There is only few packets in the file and i found a packet with GET request to /flag.html.

![Crepe](/assets/images/numb3.png)

Just select that packet and press **Ctrl+Alt+Shift+H** to follow HTTP Stream and you will get your flag.

Flag  : flag_{52877e8b1d224623957e36051d6a8e05}


### 2 - helpfulwine

Like previous one, it was also a pcap file with few packet data in it. Opened the file using whireshark and there is only some tcp protocols.

Select the first packet and press **Ctrl+Alt+Shift+T** to follow tcp Stream and you will get the flag.

Flag : flag_{7ab84f7168c2461e814e7e5eb1bb56a0}


### 3 - rhetoricalairplane

Lets analyse the downloaded file using wireshark and it was soo easy. There is only a single UDP packet and press **Ctrl+Alt+Shift+U** to follow UDP Stream.

You got the flag.

Flag : flag_{69791495c5334cf0a6dfe60eaf37769f}


### 4 - lemonswitch 

Unlike other challenges this was an interesting challenge.

Challenge description is given below.

![Crepe](/assets/images/lemon1.png)

We were given an IP address and a file, which is a SSH auth log file.

The web panel is for logging malicious behavior.

![Crepe](/assets/images/lemon2.png)

So we have to analyse the SSH auth log file and we have to find the malicious behavior.

We have to find malicious user's ip address, number of successful password login attempts made by adversary, number of successful public key authentication, and finally number of failed login attempts made by adversary.

Lets start our analysis..

After a lot of analysing i concluded by the below method.

I first selected all failed login attempts for all users and also collected IP addresses from which failed login originated.

~~~
cat 0228a4fb0b05408ad4e1a32ea2137ca8 | grep -i "failed" | cut -d" " -f9,11 | sort | uniq -c
~~~

Output : 

![Crepe](/assets/images/lemon3.png)

I concluded that a lot of failed login attempts are originated from **10.133.155.151** for **diffrent users**.

Total failed attempts made by 10.133.155.151:

![Crepe](/assets/images/lemon4.png)

Next i have to find how many successful attempts he made to access the server with both password auth and public key auth.

Luckily he is not successful with his password guessing.

![Crepe](/assets/images/lemon5.png)

So

* malicious user's ip : 10.133.155.151
* successful password attempt : 0
* successful PubKey attempt : 0
* number of failed attempts : 86

Submitted all these details on the web panel, we will get flag only if details are correct.

100% our analysis were correct and we got the flag.

![Crepe](/assets/images/lemon6.png)

![Crepe](/assets/images/lemon7.png)

Thanks.
