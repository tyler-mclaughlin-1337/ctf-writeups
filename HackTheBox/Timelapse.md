Timelapse is an easy machine from HackTheBox that starts with publicly accessible SMB shares and ends with abusing account permissions to pull the Administrator password from LAPS.

As with any CTF from HackTheBox, I always start with a Nmap scan of the target. Before I do that though, I typically like to set the target IP to a variable first:
target=10.129.227.113
From there, I run the following Nmap command:
```bash
sudo nmap -Pn -p- --min-rate 5000 -oN nmap-ports.txt $target && ports=$(grep ^[0-9] nmap-ports.txt | cut -d'/' -f1 | tr '\n' ',' | sed 's/,$//') && sudo nmap -Pn -n -p$ports -A -oN nmap-full.txt $target
```
This one-liner does a full scan of all 65,535 TCP ports, then whatever ports come back as open, those are saved to the ports variable. From there, those are fed into another Nmap scan with the -A flag. This performs OS detection, service version detection, and script scanning. I also send the output to a .txt file for future reference if needed.
Once this is complete, we see the following:
Could not load image
The image file may have been removed.
Could not load image
The image file may have been removed.
The main ports I am interested in here are 88, 389, 445, and 5986:
tcp/88 is going to be Kerberoas, indicating that we are dealing with a domain controller
tcp/389 is going to be LDAP, further strenghtining the domain controller theory
tcp/445 is SMB
tcp/5986 is the SSL implementation of WinRM
If SMB is open on a host, I'll typically start there, looking for quick wins. I'll try null authentication, and if that fails I'll try Guest authentication:
nxc smb $target -u '' -p '' --shares
nxc smb $target -u 'Guest' -p '' --shares
Could not load image
The image file may have been removed.
In this case, we have Guest authentication and read access over a non-standard share named "Shares". We also can see that the target's hostname is DC01 and the domain is timelapse.htb. 
I'll utilize smbclient to access this share and see what we can find:
Could not load image
The image file may have been removed.
We see two directories, "Dev" and "HelpDesk". Inside of "Dev", we see "winrm_backup.zip", so we grab that. Inside of HelpDesk, we see documentation for LAPS, hinting that LAPS is likely configured on this domain controller.
Upon attempting to unzip the .zip file we retrieved, we see it is password protected. Luckily, JohnTheRipper has a module available to create a hash for this, zip2john:
Putting that hash through John with the rockyou wordlist, we see the password to unzip the file is supremelegacy
Inside of the .zip, we see "legacyy_dev_auth.pfx". A .pfx file a file that contains both a users public key and private key. In this case, it appears they were used for WinRM authentication. This is also password protected though:
So, we must use pfx2john in order to get a password hash.
And once again, we can crack that with John and the rockyou wordlist.
The password for our .pfx file is thuglegacy .
Next, we can utilize openssl in order to separate the private key and SSL certificate for authentication
Awesome! We have initial access as the user legacyy.
Running a quick check (whoami /all) to see our current user's groups and privileges, I see nothing of interest:
Next, I transferred winPEAS to the target via PowerShell's Invoke-WebRequest and saved that as wp.exe, then executed it:
While this was running, I had a lightbulb moment and remembered LAPS was likely enabled on this host. However, I knew that in order to abuse it, I was going to need an account with credentials.
winPEAS confirmed my LAPS suspicions, and it also noted that I had a PowerShell console history text file with some data inside.
Upon listing out the contents of the PowerShell history file, I saw credentials for the user "svc_deploy":
Using those credentials, I could now attempt to retrieve the Administrator password using the Netexec laps module:
And that is a success! We can confirm the credentials are valid by using a tool I like to use called nxcspray - 
https://github.com/NTHSec/nxcspray
Our credentials are valid, so we can utilize impacket-psexec to gain access
And with that, we have a SYSTEM shell, and that is the box!
Previous
About me
Next - OffSec Proving Grounds Writeups
AuthBy
Tyler
Last modified 15m ago
