#Linux #  # 

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

![image](https://user-images.githubusercontent.com/99097743/170733365-d1597c75-3cac-4cb0-a7b8-0e3402861fd6.png)

It seems that there is no vulnerability documented for **SquirrelMail version 1.4.23**

![image](https://user-images.githubusercontent.com/99097743/170743722-89617c7d-6878-4ec2-b530-1676178865ba.png)

OK, Let's switch to the Port 445/tcp SMB server. We can try the **enum4linux** for enumerating information.


```sh
  enum4linux -a 10.10.14.219
  ```
* To do all simple enumerations(-a)

There are interesting outcomes such as the known usernames:

![image](https://user-images.githubusercontent.com/99097743/170734669-57ec1e84-403f-4464-a5a9-6c62c66ebe0b.png)


Here are the sharenames:

![image](https://user-images.githubusercontent.com/99097743/170734874-50cb0cc9-7a02-484e-a69e-1e9aa92a5772.png)


We can use the **smbclient** to enter the anonymous share. Here are the commonly used commands for smbclient


```sh
  smbclient '\\10.10.48.162\anonymous'
  ```
  
 We captures a number of logs files, one of which includes a password list potentiall for **milesdyson**. Let's use **hydra** for brute-forcing the email account 
 
 ```sh
 hydra -l milesdyson -P log1.txt 10.10.48.162 http-post-form "/squirrelmail/src/redirect.php:login_username=^USER^&secretkey=^PASS^:F=incorrect" -V -F -u
```
