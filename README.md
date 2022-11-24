# redpanda-htb-walkthrough
<h3> About the Lab:</h3>
This is a Linux machine that requires exploiting **SSTI** in a **Java SpringFramework** application via a search bar on the webpage for RCE and then initial access. For privilege escalation, we will need to emulate what group the user is in, discover a log file he/she has access to, use ***pspy*** to discover a **JAR** file root periodically run, study that JAR file to discover conditions required to pass, exploiting ***directory traversal***, and generate an **XXE XML** file to leak root’s SSH private key. The privilege escalation for this machine is **HARD** and shouldn’t be an easy category machine. But let’s get started!

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
A server-side template injection occurs when an attacker is able to use native template syntax to inject a malicious payload into a template, which is then executed server-side. See for further information.<br>

A list of payloads that can be used for checking if the site is vulnerable to SSTI:
