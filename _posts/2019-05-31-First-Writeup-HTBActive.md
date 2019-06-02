---
layout: post
title: My first writeup! Hack The Box, Active
tags: [Hack The Box, Active Directory, Bloodhound, Hashcat, GPP]
---
![logo](/boxes/htb/active/activeDirectory.PNG)

### i. Port Scan
![nmap portscan](/boxes/htb/active/nmap.PNG)
### ii. Enumeration
SMB ports were listed open on the nmap scan. Let's check if any shares are available with smbclient and smbmap.
```
smbclient -L 10.10.10.100
smbmap -H 10.10.10.100
```
smbclient:   
![smbclient](/boxes/htb/active/smbclient.PNG)

smbmap:  
![smbmap](/boxes/htb/active/smbmap.PNG)

Smbmap reveals a file share called Replication with read only permissions. Smbmap can be used to list the contents of the Replication directory:
```
smbmap -R Replication -H 10.10.10.100
```
Looking through the contents of the Replication directory there is a file called Groups.xml. This is a group policy file where local account information is typically stored for older Active Directory implementations.

![ReplicationFiles](/boxes/htb/active/groupsxml.PNG)

To download the file we use:
```
smbmap -R Replication -H 10.10.10.100 -A Groups.xml -q
```
Opening and reading the Groups.xml file, an account `SVC_TGS` and a encrypted password `edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH` was found. 

![groupsxmlhash](/boxes/htb/active/grouphash.PNG)


The password is encrypted with a known key:

https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-gppref/2c15cbf0-f086-4c74-8b70-1f2fa45dd4be  
![msftknownkey](/boxes/htb/active/aes key.jpg)

To decrpyt the password, we use gpp-decrpyt:  

![gppdecrypt](/boxes/htb/active/gppdecrypt.PNG)

The decrypted hash is `GPPstillStandingStrong2k18`. Using the credentials we found we enumerate more shares.
```
smbmap -d active.htb -u svc_tgs -p GPPstillStandingStrong2k18 -H 10.10.10.100
```
![moreshares](/boxes/htb/active/moreshares.PNG)

Logging in and enumerating these shares lead to the user flag under the `SVC_TGS\Desktop\` directory.

Now that we have credentials, let's use Bloodhound on the box. We switch to a Windows VM and run the following command:
```
runas /netonly /user:active.htb\svc_tgs cmd
```
Make sure to check your DNS settings on your network adapter and set it to 10.10.10.100. 

Run sharphound with the following command to gather information to use in Bloodhound:
```
sharphound.exe -c all -d active.htb --domaincontroller 10.10.10.100
```
A zip file should have been created with the infomartion to import into Bloodhound. Drag and drop the zip file created from into the user interface of Bloodhound to import the information. Checking for domain admin did not reveal anything. Checking for kerberoasting, the user `ADMINISTRATOR` is available for this attack.
![kerberoasting](/boxes/htb/active/kerberoasting.PNG)

Using GetUserSPN.py from impacket we will grab the hash of `ADMINISTRATOR`
```
GetUserSPN.py -request -dc-ip 10.10.10.100 active.htb/svc_tgs
```
![adminhash](/boxes/htb/active/getUserSPN.PNG)

We copy the hash into a file and use hashcat to crack the hash via the following command:
```
hashcat -m 13100 hash.txt rockyou.txt 
```
![crackedhash](/boxes/htb/active/adminpw.PNG)

The Administrator password is: `Ticketmaster1968`

Now we can use psexec and log into the box and get the root flag
```
psexec.py active.htb/Administrator@10.10.10.100
```
![rootflag](/boxes/htb/active/rootflag.PNG)
