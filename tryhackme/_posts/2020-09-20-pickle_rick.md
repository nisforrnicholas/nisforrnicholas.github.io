---
title: "Pickle Rick"
tags:
  - Easy
  - Linux
  - nmap
  - OS Injection
---

| Difficulty |
| ---------- |
|   Easy     |

---

### Deploy the virtual machine on this task and explore the web application.

### [ What is the first ingredient Rick needs? ]

First, let's run a basic `nmap` scan (top 1000 ports) on the machine:

```
sudo nmap -sC -sV -vv 10.10.28.153
```

![screenshot1](../assets/images/pickle_rick/screenshot1.png)

From the results of the scan, we see that **2** ports are open: **22 (SSH)** and **80 (HTTP)**

Let's check out that HTTP webserver:

![screenshot2](../assets/images/pickle_rick/screenshot2.png)

We are brought to a Rick and Morty themed page. The first thing we can do is to run a `gobuster` directory scan to enumerate any hidden directories. We'll also be checking for PHP files, which we can specify by using the `-x` option:

```
gobuster dir -u http://10.10.28.153/ -x php -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

While Gobuster was working its magic, I went ahead and did some manual enumeration. I first looked at the source code of the site:

![screenshot3](../assets/images/pickle_rick/screenshot3.png)

Interestingly enough, we find a username: **R1ckRul3s**

Next, I took a look at the **/robots.txt** file on the webserver:

![screenshot4](../assets/images/pickle_rick/screenshot4.png)

It had a strange phrase: **Wubbalubbadubdub**. We'll take note of this phrase for now.

Checking back on our Gobuster scan, we see that it has managed to find two interesting directories: **/login.php** and **/portal.php**:

![screenshot5](../assets/images/pickle_rick/screenshot5.png)

Let's check them out!

**/login.php** brings us to a login page:

![screenshot6](../assets/images/pickle_rick/screenshot6.png)

**/portal.php** just redirects us back to **/login.php**. My first thought was to try and brute-force some login credentials using the username that we found earlier. However, before committing to that, I had a thought. What if the strange phrase that we found in the **robots.txt** file earlier was actually the password? 

I tried logging in with **R1ckRul3s:Wubbalubbadubdub** and it worked! We are brought to a command panel:

![screenshot12](../assets/images/pickle_rick/screenshot12.png)

Could we execute shell commands on the target machine through this panel? Before testing this out, I tried to visit the other pages. However they all required us to be in **rick's** account in order to access them:

![screenshot13](../assets/images/pickle_rick/screenshot13.png)

Let's try out that command panel. Running a simple `whoami` gives us:

![screenshot14](../assets/images/pickle_rick/screenshot14.png)

Nice! This proves that we can indeed inject commands here. Before doing so, I wanted to check the source code of this page in case I missed anything important:

![screenshot15](../assets/images/pickle_rick/screenshot15.png)

We get yet another strange phrase (in the green comment).

It seems that the phrase has been base64-encoded. After decoding it once, I realized that it has been base64-encoded multiple times in a row. Hence, after decoding and decoding, we find out that the phrase actually reads **"rabbit hole"**. What a rabbit hole indeed.

Alright, let's try executing some other commands. I first tried `ls`:

![screenshot16](../assets/images/pickle_rick/screenshot16.png)

There is a file that interests me: **Sup3rS3cretPickl3Ingred.txt**

I tried to `cat` it:

![screenshot17](../assets/images/pickle_rick/screenshot17.png)

Hmmm no luck there. With that, let's try to set up a reverse shell so that we can explore more freely within the target machine! We begin by checking whether Python is installed on the target (`python3 -V`):

![screenshot18](../assets/images/pickle_rick/screenshot18.png)

Great, looks like Python3 is installed.

Next, we'll be using **pentestmonkey's** [reverse shell cheatsheet](https://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet) to find a Python-based reverse shell command:

![screenshot19](../assets/images/pickle_rick/screenshot19.png)

I set up a netcat listener on my local machine, ran the command above and successfully opened the reverse shell:

![screenshot20](../assets/images/pickle_rick/screenshot20.png)

And we're in!

With that, we can obtain the first ingredient:

![screenshot21](../assets/images/pickle_rick/screenshot21.png)

---

### [ Whats the second ingredient Rick needs? ]

There is another interesting file called **clue.txt**:

![screenshot22](../assets/images/pickle_rick/screenshot22.png)

It tells us to look around the file system for the other ingredient. Doing just that, I found out that there are 2 users on the machine: **rick** and **ubuntu**:

![screenshot24](../assets/images/pickle_rick/screenshot24.png)

In rick's home directory, we find the second ingredient:

![screenshot25](../assets/images/pickle_rick/screenshot25.png)

---

### [ Whats the final ingredient Rick needs? ]

Before continuing on, let's check out our **sudo privileges**:

![screenshot23](../assets/images/pickle_rick/screenshot23.png)

It turns out that we actually have **full sudo privileges**! This means that we can basically run any program or command as root.

I'm guessing the last ingredient will be found in the **/root** folder. Since we can run **sudo** with any command, we can just open up another bash shell as root:

```
sudo bash
```

![screenshot26](../assets/images/pickle_rick/screenshot26.png)

From there, we can obtain the last ingredient that is located in **/root**:

![screenshot27](../assets/images/pickle_rick/screenshot27.png)

