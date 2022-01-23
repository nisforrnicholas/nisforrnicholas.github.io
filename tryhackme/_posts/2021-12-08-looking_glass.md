---
title: "Looking Glass"
tags:
  - Medium
  - Linux
  - nmap
---

|  | Difficulty |  |  IP Address   |  | Room Link |  |
|--| :--------: |--|:------------: |--| :--------:|--|
|  |   Medium   |  | 10.10.26.122  |  | [Looking Glass](https://tryhackme.com/room/lookingglass) |  |

### [ Get the user flag. ]

As always, let's run a full `nmap` scan on the target machine so as to enumerate the services that are running.

```
sudo nmap -p- -vv 10.10.26.122
```

 **Results:**

```
PORT      STATE SERVICE          REASON
22/tcp    open  ssh              syn-ack ttl 61
9000/tcp  open  cslistener       syn-ack ttl 61
9001/tcp  open  tor-orport       syn-ack ttl 61
9002/tcp  open  dynamid          syn-ack ttl 61
9003/tcp  open  unknown          syn-ack ttl 61
9004/tcp  open  unknown          syn-ack ttl 61
...
...
...
13995/tcp open  unknown          syn-ack ttl 61
13996/tcp open  unknown          syn-ack ttl 61
13997/tcp open  unknown          syn-ack ttl 61
13998/tcp open  unknown          syn-ack ttl 61
13999/tcp open  unknown          syn-ack ttl 61
```

Turns out we have a whole slew of ports open, ranging from port **9000** to **13999**. The usual port **22** for SSH is also open.

nmap didn't manage to enumerate the actual service running on these ports, so let's try to connect to one of them:

```
telnet 10.10.135.105 9000
```

![screenshot1](../assets/images/looking_glass/screenshot1.png)

Looks like these ports are running SSH as well! With that, let's try to connect using `ssh`:

```
ssh -p 9000 root@10.10.135.105
```

![screenshot2](../assets/images/looking_glass/screenshot2.png)

We get an interesting message: **Lower**

I first thought that we had to connect to a lower port, but this wouldn't make sense as I had connected to port 9000, which was the lowest port of the range. 

Let's try the highest port of the range:

![screenshot31](../assets/images/looking_glass/screenshot31.png)

Now we get another message: **Higher**. 

Maybe it's reversed and I had to go the opposite direction as to what was returned? Basically higher = lower and lower = higher.

Using this logic and after much trial and error, I finally found the port that showed a different message. *(Note that the port differs every time the machine is rebooted.)*

![screenshot3](../assets/images/looking_glass/screenshot3.png)

The message seems to be encrypted. Using an online [cipher identifier](https://www.dcode.fr/cipher-identifier), I found out that the encryption was done using the **Vigenere Cipher**.

Luckily for us, Boxentriq provides a nifty Vigenere Cipher brute-forcing [tool](https://www.boxentriq.com/code-breaking/vigenere-cipher):

![screenshot4](../assets/images/looking_glass/screenshot4.png)

After some time, the key used in the encryption was found:

> thealphabetcipher

We can then use the key to decode the entire ciphertext:

![screenshot5](../assets/images/looking_glass/screenshot5.png)

At the bottom of the decrypted message, we get the secret:

> bewareTheJabberwock

Let's now go back to the service and input the newfound secret:

![screenshot6](../assets/images/looking_glass/screenshot6.png)

And we get a set of SSH credentials! 

> jabberwock : WhileScramblingBalancesEnormous

With these creds, we can log into jabberwock's account on the SSH server (port 22). 

*(Note that the password also changes everytime the machine is rebooted!)*

![screenshot7](../assets/images/looking_glass/screenshot7.png)

The **user flag** can be found in the home directory of **jabberwock**:

![screenshot8](../assets/images/looking_glass/screenshot8.png)

However, it seems to be reversed. All we have to do is reverse it back and we get the flag!

![screenshot9](../assets/images/looking_glass/screenshot9.png)

---

### [ Get the root flag. ]

Now that we are in, let's do some manual enumeration.

![screenshot10](../assets/images/looking_glass/screenshot10.png)

jabberwock's home directory contains two other files: **poem.txt** and **twasBrillig.sh**:

![screenshot15](../assets/images/looking_glass/screenshot15.png)

**poem.txt** did not contain anything useful.

**twasBrillig.sh** simply writes the contents of poem.txt to all users on the machine. What's important to note is that we can write to this script:

Next, from the /home directory, we know that there are 5 other users:

![screenshot11](../assets/images/looking_glass/screenshot11.png)

Unfortunately, we do not have the permissions to navigate to their respective directories.

Let's check jabberwock's **sudo privileges**:

![screenshot12](../assets/images/looking_glass/screenshot12.png)

Interesting... jabberwock is able to run `/sbin/reboot` as root! Furthermore, we do not need to input a password when doing so. As the name suggests, this binary will reboot the entire machine.

After digging around further, I found something interesting in **/etc/crontab**:

![screenshot13](../assets/images/looking_glass/screenshot13.png)

it seems that every time the machine is rebooted, the user **tweedledum** will run the twasBrillig.sh script! With this, let's overwrite the contents of twasBrillig.sh with a [reverse shell script](found from https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md):

![screenshot14](../assets/images/looking_glass/screenshot14.png)

Now we just have to set up a netcat listener on our local machine, reboot the machine by running `/sbin/reboot` and we should be able to open a reverse shell as the user tweedledum:

![screenshot16](../assets/images/looking_glass/screenshot16.png)

And we're in tweededum's account!

tweedledum's home directory contains 2 other files: **humptydumpty.txt** and **poem.txt**:

![screenshot17](../assets/images/looking_glass/screenshot17.png)

Once again, **poem.txt** does not contain anything particularly useful.

**humptydumpty.txt**, on the other hand, contains what seems to be hashes:

![screenshot18](../assets/images/looking_glass/screenshot18.png)

Let's use an online [hash cracker](https://hashes.com/en/decrypt/hash) to crack these hashes:

![screenshot19](../assets/images/looking_glass/screenshot19.png)

And we've gained another password:

> zyxwvutsrqponmlk

Since there is a user called **humptydumpty** on our target machine, perhaps this is his password? Let's try to `su` into his account *(you have to upgrade your shell into a fully-interactive one first)*:

![screenshot20](../assets/images/looking_glass/screenshot20.png)

And we're now humpty dumpty!

humptydumpty's home directory only contains one file: **poetry.txt**

![screenshot21](../assets/images/looking_glass/screenshot21.png)

The file did not contain anything useful, so let's skip it.

It took me awhile at this point before I realized something odd in **/home**:

![screenshot22](../assets/images/looking_glass/screenshot22.png)

Notice how both **alice's** and **jabberwock's** home directories are **executable** by other users? This is a clear misconfiguration as this actually allows us to run commands within those directories, even though we cannot actually navigate into them. We'll ignore jabberwock as we have already compromised that account. Let's see if we can read out any files within alice's directory.

The first file that came to mind was alice's private ssh key. This is normally found in **~/.ssh/id_rsa**. However, we should not be able to read the file as it should be owned by alice. With that said, let's check if the file even exists, and if it does, what are its permissions:

![screenshot23](../assets/images/looking_glass/screenshot23.png)

Nice! alice's private key file does exist and it's actually owned by humptydumpty! :smile:

With that, we can read out the key:

![screenshot24](../assets/images/looking_glass/screenshot24.png)

I then copied the key to a file on my local machine, changed the file's permissions to the correct value (either 400 or 600), then used it to log into alice's ssh account:

![screenshot25](../assets/images/looking_glass/screenshot25.png)

Now we're alice!

alice's home directory contains a single file: **kitten.txt**. Once again, it was simply telling a story so nothing we can use there.

We cannot check alice's **sudo privileges** as we do not know her password. However, there is another method to check.

Let's visit **/etc/sudoers.d**:

![screenshot26](../assets/images/looking_glass/screenshot26.png)

This directory contains the sudo privileges for users on the machine. As we can see, the file for alice is actually world-readable! Let's take a look at alice's sudo privileges:

![screenshot27](../assets/images/looking_glass/screenshot27.png)

Turns out alice can actually run `/bin/bash` as root. No password is required as well. However, one thing to note is that we cannot simply run `sudo /bin/bash` as the **host** has been changed from **looking-glass** to **ssalg-gnikool**, as seen in the file above. Hence, we have to specify the host when running `sudo`, which can be done with the `-h` option.

With that, all we have to do is to run the following command:

```
sudo -h ssalg-gnikool /bin/bash
```

![screenshot28](../assets/images/looking_glass/screenshot28.png)

And we're now in root's account! 

The **root flag** can be found in /root:

![screenshot29](../assets/images/looking_glass/screenshot29.png)

It has been reversed, so let's reverse it back and obtain the flag:

![screenshot30](../assets/images/looking_glass/screenshot30.png)
