# redpanda-htb-walkthrough
<h3> About the Lab:</h3>
This is a Linux machine that requires exploiting **SSTI** in a **Java SpringFramework** application via a search bar on the webpage for RCE and then initial access. For privilege escalation, we will need to emulate what group the user is in, discover a log file he/she has access to, use ***pspy*** to discover a **JAR** file root periodically run, study that JAR file to discover conditions required to pass, exploiting ***directory traversal***, and generate an **XXE XML** file to leak root’s SSH private key. The privilege escalation for this machine is **HARD** and shouldn’t be an easy category machine. But let’s get started!

## User Own:
