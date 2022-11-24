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

From above, we find out it is built in Spring framework Java. We found 3 accessible directories as well and that there is an SSTI vulnerability existence.<br>
