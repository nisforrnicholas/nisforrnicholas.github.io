---
title: "Agent Sudo"
tags:
  - Easy
  - Linux
  - nmap
  - user-agent
  - burpsuite
  - ftp
  - hydra
  - binwalk
  - john
  - steghide
  - sudo
---

* Difficulty: Easy

---

### [ Deploy the machine ]

Done!

---

### Enumerate the machine and get all the important information
### [ How many open ports? ]

To find out what ports are open on our target machine, let's run a basic **nmap** scan (top 1000 ports) on the target machine:

```
sudo nmap -sC -sV -vv 10.10.63.185
```

![screenshot1](../assets/images/agent_sudo/screenshot1.png)

Seems like **ftp (21)**, **SSH (22)** and **HTTP (80)** is up and running.

**No of open ports: 3**

---

### [ How do you redirect yourself to a secret page? ]

Let's check out the HTTP webserver first:

![screenshot2](../assets/images/agent_sudo/screenshot2.png)

Interesting. Before doing any manual enumeration, we shall run a **Gobuster** directory scan to see if we can enumerate any hidden directories. We can also add extensions to our search options by using the `-x` option. In this case, we can check if there's any PHP or HTML files hidden within the web server. 

```
gobuster dir -u http://10.10.63.185/ -x php,html -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

While Gobuster is running, we can also check the site's source code to see if there is any hidden information we can use. Unfortunately, there was no such hidden information found.

Gobuster wasn't giving any promising results, so let's look more closely at the information given on the main page. They mention logging into the webpage with the agent's name as the **user-agent**. Doing some research online, I was able to find some useful information on [user-agents](https://betanews.com/2017/03/22/user-agent-based-attacks-are-a-low-key-risk-that-shouldnt-be-overlooked/).

![screenshot3](../assets/images/agent_sudo/screenshot3.png)

Looks like we should be able to redirect ourselves to the secret page using the **user-agent** header!

---

### [ What is the agent name? ]

Let’s use **Burpsuite** to intercept the request and change the **user-agent** header. 

**Intercepted Request:**

![screenshot4](../assets/images/agent_sudo/screenshot4.png)

We can see the **User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:68.0) Gecko/20100101 Firefox/68.0**

Let's send this request to Burpsuite Repeater so that we can automate the process of changing the user-agent field. From the webpage, we know that there is an agent called **'Agent R'**. However, trying that, it seems like nothing happens:

![screenshot5](../assets/images/agent_sudo/screenshot5.png)

After trying a couple of variations, I realized that we were supposed to change the field to just the letter **'R'**:

![screenshot6](../assets/images/agent_sudo/screenshot6.png)

Thus, to access the agent's pages, we just have to change the user-agent to the agent's codename/letter accordingly. It is also mentioned that there are 25 employees. Minus 'R', that would account for the rest of the English alphabet. Does that mean that every letter has an existing agent page?

To find the other agent pages, we can brute-force through every alphabet using the **Intruder** module in Burpsuite.

Firstly, we send the intercepted request to Intruder. Then, we set a simple payload of all the alphabets in uppercase (A-Z). We then highlight the location in which we want Intruder to set the payload. In our case, that would be the User-Agent field. Once that is done, we can run the attack and look at the responses.

**Attack Running:**

![screenshot7](../assets/images/agent_sudo/screenshot7.png)

**Results**:

![screenshot8](../assets/images/agent_sudo/screenshot8.png)

Nice! Turns out that when **'C'** is used as the user-agent, we get a redirect code from the webserver, bringing us to **'/agent_C_attention.php'**. Let's check that file out:

![screenshot9](../assets/images/agent_sudo/screenshot9.png)

We now know that agent C's real name is actually **Chris**.

Since it is mentioned that his password is weak, we can probably crack his SSH/FTP password easily. 

**Agent's name: chris**

---

### Done enumerate the machine? Time to brute your way out

### [ FTP password ]

Let's explore the FTP server. Firstly, we check if anonymous login is enabled on it. This is a common misconfiguration on FTP servers as it grants access without needing to authenticate.

![screenshot10](../assets/images/agent_sudo/screenshot10.png)

Looks like anonymous access is not enabled. Since we know a potential username, **chris**, let's use `Hydra` to crack his password for the FTP server:

```
hydra -l chris -P /usr/share/wordlists/rockyou.txt -o ftp_pass ftp://10.10.63.185
```

**Results:**

![screenshot11](../assets/images/agent_sudo/screenshot11.png)

After a short while, we successfully crack Chris's password: **crystal**

---

### [ Obtain the Zip file password ]

With the password cracked, we can now login as **chris** into the FTP server:

![screenshot12](../assets/images/agent_sudo/screenshot12.png)

**Contents of the FTP server:**

![screenshot13](../assets/images/agent_sudo/screenshot13.png)

Listing all of the files on the FTP server, we see that there isn't any further directories that we can explore. There are only a couple of images and a text file. Let's download all of the files to our local machine using the `get` command.

**To_agent.txt:**

![screenshot14](../assets/images/agent_sudo/screenshot14.png)

Clearly, there is some secret data embed within the photos.

I tried some common steganography techniques to extract the secret data. Firstly, I used the `strings` command to check for any human-readable strings within the image files. However, there was nothing found.

Next, I used `exiftool` to extract the metadata from the images. However, there was also nothing of interesting there.

I then got to learn and try out various steganography tools. The tools I tried were: `steghide`, `stegcracker`, `zsteg`, `binwalk`.

*(Note: I learned that `steghide` only works for **JPEG, BMP, WAV and AU** files. If we want to potentially extract data from **PNG** files, we must use `zsteg` instead.)*

I tried using `steghide`:

![screenshot15](../assets/images/agent_sudo/screenshot15.png)

Looks like we need a passphrase. I tried inputting an empty passphrase but it didn't work. Let's use [StegCracker](https://github.com/Paradoxis/StegCracker) to try and crack the passphrase. Note that StegCracker is a python module, so it has to be run with ```python3 -m stegcracker …```

```
python3 -m stegcracker cute-alien.jpg
```

![screenshot16](../assets/images/agent_sudo/screenshot16.png)

StegCracker ran for some time, but I stopped it as no passphrase was found.

Next, I tried using `zsteg` on the **cutie.png** file. The `-a` tag denotes to use all extraction methods:

```
zsteg -a cutie.png
```

![screenshot17](../assets/images/agent_sudo/screenshot17.png)

Unfortunately, it was also unable to find any hidden data within the image. With that said, one interesting thing is that in the **extradata** section, it did mention '**file: Zip archive data…**' Is this the zip file that we have been looking for?

Next, I tried using `binwalk`, which is another tool for searching binary files like images and audio files for embedded files and data. I will use the `-e` tag to extract data.

```
binwalk -e cute-alien.jpg
```

![screenshot18](../assets/images/agent_sudo/screenshot18.png)

Nice! We managed to extract the zip file (**8702.zip**) within the **cutie.png** file. Let's open it:

![screenshot19](../assets/images/agent_sudo/screenshot19.png)

Looks like we need a password to unzip the file. We can use **John The Ripper** to try and brute-force this password. Firstly, we will need to use the `zip2john` tool to convert the zip file into a format that can be cracked by John:

```
sudo zip2john 8702.zip > for_john.txt
```

![screenshot20](../assets/images/agent_sudo/screenshot20.png)

After this, we just have to run `john` on the **for_john.txt** file that was outputted:

```
sudo john --wordlist=/usr/share/wordlists/rockyou.txt for_john.txt
```

![screenshot21](../assets/images/agent_sudo/screenshot21.png)

Using the **rockyou.txt** wordlist, we managed to obtain the zip file password: **alien**

---

### [ Obtain the steg password ]

After extracting the zip file, we managed to obtain the text file within: **To_agentR.txt**.

![screenshot22](../assets/images/agent_sudo/screenshot22.png)

Could the strange phrase in the text file be the passphrase for the **cute-alien.jpg** file?

![screenshot23](../assets/images/agent_sudo/screenshot23.png)

Oops… looks like it isn't.

I figured that the phrase was probably encoded, but I was unsure of what encoding scheme was used. After messing around for a bit, I realized it was just simple **base64 encoding**! We can decode it in the CLI:

![screenshot24](../assets/images/agent_sudo/screenshot24.png)

An alternate method to decode the string:

![screenshot25](../assets/images/agent_sudo/screenshot25.png)

Looks like we got the passphrase: **Area51**

---

### [ Who is the other agent (in full name)? ]

Let's use `Steghide` to extract the hidden data within the **cute-alien.jpg** file:

![screenshot26](../assets/images/agent_sudo/screenshot26.png)

We found agent J's name: **James**

---

### [ SSH password ]

His password is: **hackerrules!**

---

### You know the drill.

### [ What is the user flag? ]

With James's credentials, we can log into the SSH server with his account:

![screenshot27](../assets/images/agent_sudo/screenshot27.png)

We're in!

The user flag can be found in james' home directory:

![screenshot28](../assets/images/agent_sudo/screenshot28.png)

---

### [ What is the incident of the photo called? ]

I noticed another JPEG file called '**Alien_autospy.jpg**'. Seeing as python3 is installed on the remote machine, we can set up a simple python HTTP server and transfer that image file to our local machine. 

![screenshot29](../assets/images/agent_sudo/screenshot29.png)

However, I wanted to practice using the `scp` command. To download the image file from the SSH server with scp, we run the following command on our local machine:

```
scp james@10.10.54.221:/home/james/Alien_autospy.jpg /home/fatalfuric/Desktop/agent_sudo
```

![screenshot30](../assets/images/agent_sudo/screenshot30.png)

**Alien_autospy.jpg:**

![screenshot31](../assets/images/agent_sudo/screenshot31.png)

First thing I thought to do was to use google reverse-image search. This was the results:

![screenshot32](../assets/images/agent_sudo/screenshot32.png)

I tried inputting the answer as "**Roswell UFO incident**", but that was not it.

After looking at numerous different articles, I decided to use the hint, which mentioned **Fox News**. Searching for fox news articles on the subject matter, I came across this article:

![screenshot33](../assets/images/agent_sudo/screenshot33.png)

**Incident of the photo: Roswell Alien Autopsy**

---

### Enough with the extraordinary stuff? Time to get real.

### [ CVE number for the escalation (Format: CVE-xxxx-xxxx) ]

First,let's check the **sudo** privileges that James has on the machine:

![screenshot34](../assets/images/agent_sudo/screenshot34.png)

The `(ALL, !root)` tells us that James can run the specified program as any user **OTHER THAN ROOT**. In this case, the program is `/bin/bash` Let's try running `sudo /bin/bash` regardless:

![screenshot35](../assets/images/agent_sudo/screenshot35.png)


Expected. Doing some search online, I found the following [exploit](https://www.exploit-db.com/exploits/47502) on exploit-db:

![screenshot36](../assets/images/agent_sudo/screenshot36.png)

This is the exploit we will be using to allow us to escalate our privileges using `sudo`.

CVE number: **2019-14287** 

---

### [ What is the root flag? ]

One thing to note is that this exploit only works for **Sudo < 1.2.28**. 

![screenshot37](../assets/images/agent_sudo/screenshot37.png)

Let's check our sudo version on the remote machine. This can be done with `sudo -V`:

![screenshot38](../assets/images/agent_sudo/screenshot38.png)

Looks like the exploit will work!

---

**How the exploit works:**

It works by using the `-u` option when running sudo, which specifies a user/user ID to run sudo as.

![screenshot39](../assets/images/agent_sudo/screenshot39.png)

In this case, for outdated versions of sudo, when we use `userid = -1`, sudo will incorrectly treat the userid as `0` instead. Since userid 0 belongs to root, that means that we will actually run the command as root!

---

Let's carry out the exploit:

*(Note that when using ID as input, we need to use the **#** prefix)*

![screenshot40](../assets/images/agent_sudo/screenshot40.png)

And we have root access!

![screenshot40](../assets/images/agent_sudo/screenshot41.png)

With that, we will be able to obtain the root flag located in /root and complete the room.

---

### [ (Bonus) Who is Agent R? ]

**DesKel**

