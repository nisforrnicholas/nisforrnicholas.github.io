---
title: "Simple CTF"
tags:
  - Easy
  - Linux
  - nmap
  - CMS Made Simple
  - SQLi
  - hashcat
  - vim
---

| Difficulty |  |  IP Address   |  |
| ---------- |--|:------------: |--|
|   Easy     |  |  10.10.225.30 |  |

---

### [ How many services are running under port 1000? ]

Running a basic `nmap` scan (top 1000 ports), we obtain the following results:

```
sudo nmap -sC -sV -vv 10.10.225.30
```

![screenshot1](../assets/images/simple_ctf/screenshot1.png)

From the results, we can see that there are 3 ports open: **21 (FTP)**, **80 (HTTP)** and **2222 (SSH)**

No of services running under port 1000: **2**

---

### [ What is running on the higher port? ]

**SSH** is running on the higher port.

---

### [ What's the CVE you're using against the application? ] 

At first, I tried to see if there was any exploit associated with the FTP server (**vsftpd 3.0.3**). However, I was unable to find a relevant exploit.

Fortunately, my `gobuster` scan (which I had running earlier) revealed that there was a directory on the HTTP webserver called **/simple**:

![screenshot2](../assets/images/simple_ctf/screenshot2.png)

Navigating to the directory, we are brought to the following webpage:

![screenshot3](../assets/images/simple_ctf/screenshot3.png)

Seems like a default 'CMS Made Simple' page. 

---

*CMS Made Simple is a free, open source content management system to provide developers, programmers and site owners a web-based development and administration area.*

---

Scrolling down the page, we find out that the server is running CMS Made Simple version **2.2.8**:

![screenshot4](../assets/images/simple_ctf/screenshot4.png)

With the version, I used `searchsploit` to search for exploits that we can use:

```
searchsploit cms made simple
```

![screenshot5](../assets/images/simple_ctf/screenshot5.png)

Hmmmm, seems like the only exploit we can use for this version of CMS Made Simple is:

```
CMS Made Simple < 2.2.10 - SQL Injection
```

To find out the CVE number of this exploit, we can use the `--examine` option in `searchsploit`. This will open the exploit in the terminal:

```
searchsploit php/webapps/46635.py --examine
```

![screenshot7](../assets/images/simple_ctf/screenshot7.png)

CVE we are using against the application: **CVE-2019-9053**

---

### [ To what kind of vulnerability is the application vulnerable? ]

From the exploit, we know that the application is vulnerable to **SQLi** (SQL Injection)

---

### [ What's the password? ]

Before running the exploit, I wanted to understand how it worked. 

From what I could gather, the exploit worked by achieving unauthenticated blind time-based SQL injection through the 'm1_idlist' parameter within the 'news' module.

---

**What is Blind SQL Injection?**

![screenshot8](../assets/images/simple_ctf/screenshot8.png)

**Below is a good example of how Blind SQL Injections work:**

![screenshot9](../assets/images/simple_ctf/screenshot9.png)

---

Hence, by injecting certain SQL commands into the 'm1_idlist' parameter, we can obtain important information like usernames, password hashes and even email addresses from the database!

Let's break down the [exploit script](https://www.exploit-db.com/exploits/46635) to understand how it works. We'll look at just the 'username' enumeration portion of the code.

First, we have the URL that we are targetting for this attack. It seems that we are targetting the '/moduleinterface.php?mact=News,m1_,default,0' URI: 

![screenshot10](../assets/images/simple_ctf/screenshot10.png)

Next, the exploit will cycle through the following dictionary of characters:

![screenshot12](../assets/images/simple_ctf/screenshot12.png)

Finally, we have the actual SQL Injection payload. This payload is appended to the final URL that will be used. As we can see, the payload is submitted as the value of the 'm1_idlist' parameter:

![screenshot11](../assets/images/simple_ctf/screenshot11.png)

The exploit repeats the SQL injection, checking for the response to determine whether the character from the dictionary used is correct, before moving on to the next position. This goes on until we form the entire username! 

![screenshot13](../assets/images/simple_ctf/screenshot13.png)

While the injection payload differs slightly for enumerating the passwords and email addresses, the logic of the attack remains the same. The exploit will simply repeat this process for dumping those values. 

After running the exploit, we managed to obtain the username, email address and password of the administrator:

![screenshot15](../assets/images/simple_ctf/screenshot15.png)

Looks like the administrator is called **mitch**. 

mitch's password has also been found, although it seems to be hashed using MD5. It has also been salted. Luckily for us, the exploit has managed to find the salt used. 

Let's use `hashcat` to crack the password:

```
hashcat -a 0 -m 20 0c01f4468bd75d7a84c7eb73846e8d96:1dac0d92e9fa6bb2 /usr/share/wordlists/rockyou.txt
```

* `-a` sets the attack mode. In this case, mode 0 = straight mode

* `-m` sets the hash format. In this case, format 20 = md5(salt.pass)

![screenshot16](../assets/images/simple_ctf/screenshot16.png)

Even though we know that the salt prepends the password, in `hashcat`, the way we enter the salt and password combo must be:

> hash:salt

![screenshot18](../assets/images/simple_ctf/screenshot18.png)

After a few moments, `hashcat` manages to crack mitch's password: 

> secret

---

### [ Where can you login with the details obtained? ]

Since there was a SSH server running on port 2222, I tried to log in with our newfound credentials. Fortunately for us, it worked! 

We can log into **SSH** with the details obtained.

---

### [ What's the user flag? ]

After logging into the SSH server (using `-p` to specify port 2222) as mitch, we can then easily retrieve **user.txt** located in his home directory:

![screenshot20](../assets/images/simple_ctf/screenshot20.png)

---

### [ Is there any other user in the home directory? What's its name? ]

If we take a look at the /home directory, we see that there is another user called **sunbath**:

![screenshot21](../assets/images/simple_ctf/screenshot21.png)

---

### [ What can you leverage to spawn a privileged shell? ]

Checking our **sudo privileges** with `sudo -l`, we see that we can actually run `vim` as root! Doing some research online, I found the following method to run shell commands from `vim`:

![screenshot22](../assets/images/simple_ctf/screenshot22.png)

Looks like we can run UNIX commands from within `vim`. This allows us to leverage it to spawn a privileged shell.

---

### [ What's the root flag? ]

Let's use `vim` to open a shell!

Firstly, we execute vim with sudo: 

```
sudo vim
```

Now, any command that we run within `vim` will be run as root. 

Next, we press `esc` to activate command mode, before typing `:sh` to open up a shell:

![screenshot23](../assets/images/simple_ctf/screenshot23.png)

Once we hit enter, we see that a privileged shell has been opened!

![screenshot24](../assets/images/simple_ctf/screenshot24.png)

With that, we can grab **root.txt** from /root:

![screenshot25](../assets/images/simple_ctf/screenshot25.png)

